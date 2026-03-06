# 技术方案 A：DeerFlow 风格 — K8s Pod + hostPath 本地持久化

> **适用场景**：自建 K8s 集群，Skill 沙箱隔离执行，文件通过宿主机磁盘持久化
> **核心关键词**：K8s Pod / NodePort Service / hostPath Volume / 无外部存储依赖

---

## 一、方案概述

### 1.1 核心思路

参考字节跳动 [DeerFlow](https://github.com/bytedance/deer-flow) 开源项目的 Provisioner + AioSandboxProvider 设计，采用 **「1 thread（对话线程） = 1 Pod」** 的强绑定模型。首次 Skill 调用时按需创建专属 Pod + NodePort Service，同一 thread 内的后续 Skill 执行通过 **HTTP API** 复用该 Pod，**空闲超时（默认 10 分钟）后自动销毁**。文件通过 **hostPath** 直接挂载宿主机目录，Pod 销毁后数据仍保留在宿主机磁盘上。

**一句话总结**：每个 thread 独占一个 Pod，空闲超时后销毁；数据留在宿主机上。

### 1.1.1 生命周期模型

```
用户 A 开启对话（thread_id = abc123）
    │
    │  （Pod 并非此刻创建！lazy_init=True，延迟到首次 Skill 调用）
    │
    ├── 首次 Skill 调用 → 创建 Pod-A + Service-A
    │   Pod Name: sandbox-{sha256(thread_id)[:8]}
    │   sandbox_id 通过 thread_id 的 SHA256 确定性生成（跨进程一致）
    │
    ├── Skill 执行 1 ──→ HTTP API 调用 Pod-A 内服务执行
    ├── Skill 执行 2 ──→ HTTP API 复用同一 Pod-A（命中进程内缓存）
    ├── Skill 执行 3 ──→ HTTP API 复用同一 Pod-A（命中进程内缓存）
    │
    ├── 用户停止操作... 后台空闲检查线程每 60 秒扫描
    │
    ├── 空闲超过 10 分钟（DEFAULT_IDLE_TIMEOUT = 600s）
    │
    └── 自动销毁 Pod-A + Service-A（hostPath 数据保留）
        或：应用进程关闭时统一 shutdown 销毁所有 Pod

用户 B 同时在线（thread_id = def456）
    │
    ├── 创建 Pod-B + Service-B（与 Pod-A 完全独立）
    └── ...

关键特征（基于 DeerFlow 源码验证）：
  - Pod 和 thread 是 1:1 绑定关系（不是 user 级别，是 thread 级别）
  - sandbox_id = sha256(thread_id)[:8]（确定性生成，跨进程一致）
  - 三层一致性：进程内缓存 → 跨进程文件锁(sandbox.lock) → 后端 discover
  - Pod 内运行 HTTP 服务，backend 通过 HTTP API 调用（非 kubectl exec）
  - 销毁时机：空闲超时（默认 10 分钟）或进程关闭时 shutdown
  - 不存在 Pod 池、不存在 Pod 复用到其他 thread
  - N 个活跃 thread = N 个 Pod（资源消耗与活跃线程数线性相关）
```

### 1.2 DeerFlow 原始实现概要（基于源码分析）

DeerFlow 的沙盒系统分为三层：

**第一层：AioSandboxProvider**（`backend/src/community/aio_sandbox/aio_sandbox_provider.py`）
核心编排逻辑，管理 thread → sandbox 的映射关系：

| 能力 | 实现方式 |
|------|---------|
| 确定性 ID | `sha256(thread_id)[:8]`，跨进程一致 |
| 进程内缓存 | `dict[thread_id, sandbox_id]`，最快路径 |
| 跨进程同步 | 文件锁 `sandbox.lock` + 状态文件 `sandbox.json` |
| 后端发现 | `discover()` 方法检查容器是否仍存活 |
| 空闲超时 | 后台守护线程每 60 秒扫描，默认 600 秒超时 |
| 优雅关闭 | atexit + SIGTERM/SIGINT 信号处理 |

**第二层：SandboxBackend**（可插拔后端）
- `LocalContainerBackend`：本地 Docker / Apple Container
- `RemoteSandboxBackend`：通过 Provisioner 服务管理 K8s Pod

**第三层：Provisioner 服务**（`docker/provisioner/app.py`）
FastAPI 服务，封装 K8s API 调用：

| 接口 | 方法 | 功能 |
|------|------|------|
| `/api/sandboxes` | POST | 创建 Pod + NodePort Service |
| `/api/sandboxes/{id}` | GET | 查询沙箱状态 |
| `/api/sandboxes/{id}` | DELETE | 删除 Pod + Service |
| `/api/sandboxes` | GET | 列出所有沙箱 |

**代码执行方式**：Pod 内运行 `all-in-one-sandbox` HTTP 服务（监听 8080），backend 通过 NodePort 暴露的 HTTP API 调用 `exec_command`，**不是 kubectl exec**。

DeerFlow 的 Volume 配置（**全部 hostPath，无 PVC / 无 OSS**）：

| Volume | hostPath | 容器挂载点 | 权限 | 说明 |
|--------|----------|-----------|------|------|
| skills | `/skills` | `/mnt/skills` | 只读 | Skill 脚本文件 |
| user-data | `/.deer-flow/threads/{thread_id}/user-data` | `/mnt/user-data` | 读写 | 会话工作区 |

### 1.3 我们需要做的适配

在 DeerFlow 基础上，需要针对我们的场景做以下调整：

1. **命名空间**：`deer-flow` → `kt-agent-sandbox`
2. **镜像**：火山引擎 `all-in-one-sandbox` 镜像 → 自建 Skill 运行时镜像
3. **hostPath 路径**：适配我们的目录结构
4. **绑定粒度**：DeerFlow 按 thread_id 绑定，我们按 session_id 绑定（语义一致）
5. **安全加固**：DeerFlow 开启了 `allow_privilege_escalation: true`，我们需要关闭，并添加 `runAsNonRoot`

---

## 二、架构设计

### 2.1 整体架构图

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                           Agent Backend (FastAPI)                             │
│                                                                               │
│   WebSocket ──→ chat_service ──→ assembler ──→ Agent 执行                     │
│                                                    │                          │
│                                           需要执行 Skill                      │
│                                                    │                          │
│                                                    ▼                          │
│                                   ┌──────────────────────────┐                │
│                                   │  SandboxProvisioner      │                │
│                                   │  (FastAPI 内部服务)       │                │
│                                   │                          │                │
│                                   │  POST /api/sandboxes     │                │
│                                   │  DELETE /api/sandboxes/x │                │
│                                   │  GET /api/sandboxes      │                │
│                                   └────────────┬─────────────┘                │
│                                                │                              │
└────────────────────────────────────────────────┼──────────────────────────────┘
                                                 │ K8s API
                                                 ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                          Kubernetes Cluster (自建)                            │
│                                                                               │
│   Namespace: kt-agent-sandbox                                                 │
│                                                                               │
│   ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐  │
│   │  Pod: sandbox-aaa   │  │  Pod: sandbox-bbb   │  │  Pod: sandbox-ccc   │  │
│   │  ┌───────────────┐  │  │  ┌───────────────┐  │  │  ┌───────────────┐  │  │
│   │  │  Container     │  │  │  │  Container     │  │  │  │  Container     │  │  │
│   │  │  skill-runtime │  │  │  │  skill-runtime │  │  │  │  skill-runtime │  │  │
│   │  │               │  │  │  │               │  │  │  │               │  │  │
│   │  │ /mnt/skills   │──┤  │  │ /mnt/skills   │──┤  │  │ /mnt/skills   │──┤  │
│   │  │  (只读)       │  │  │  │  (只读)       │  │  │  │  (只读)       │  │  │
│   │  │               │  │  │  │               │  │  │  │               │  │  │
│   │  │ /mnt/user-data│──┤  │  │ /mnt/user-data│──┤  │  │ /mnt/user-data│──┤  │
│   │  │  (读写)       │  │  │  │  (读写)       │  │  │  │  (读写)       │  │  │
│   │  └───────────────┘  │  │  └───────────────┘  │  │  └───────────────┘  │  │
│   └──────────┬──────────┘  └──────────┬──────────┘  └──────────┬──────────┘  │
│              │                        │                        │              │
│   ┌──────────▼──────────┐  ┌──────────▼──────────┐  ┌──────────▼──────────┐  │
│   │  Service (NodePort) │  │  Service (NodePort) │  │  Service (NodePort) │  │
│   │  :31001             │  │  :31002             │  │  :31003             │  │
│   └─────────────────────┘  └─────────────────────┘  └─────────────────────┘  │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
                                     │
                            hostPath Volume 映射
                                     ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                          宿主机文件系统                                        │
│                                                                               │
│   /data/kt-agent/                                                             │
│   ├── skills/                          ← 所有 Skill 脚本（只读挂载）           │
│   │   ├── data_analysis/               ← Skill: 数据分析                      │
│   │   │   ├── main.py                                                         │
│   │   │   └── requirements.txt                                                │
│   │   ├── web_scraper/                 ← Skill: 网页抓取                      │
│   │   └── code_runner/                 ← Skill: 代码执行                      │
│   │                                                                           │
│   └── sessions/                        ← 用户会话数据（读写挂载）              │
│       ├── {user_id_1}/                                                        │
│       │   ├── {session_id_1}/                                                 │
│       │   │   └── user-data/                                                  │
│       │   │       ├── uploads/         ← 用户上传的文件                        │
│       │   │       ├── workspace/       ← 工作区                               │
│       │   │       └── outputs/         ← Skill 产物                           │
│       │   └── {session_id_2}/                                                 │
│       │       └── user-data/                                                  │
│       └── {user_id_2}/                                                        │
│           └── ...                                                             │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 请求时序图

```
┌──────┐     ┌──────────────┐     ┌────────────┐     ┌──────┐     ┌──────────┐
│ 用户  │     │ Agent Backend│     │ Provisioner│     │K8s API│    │ 宿主机磁盘│
└──┬───┘     └──────┬───────┘     └─────┬──────┘     └──┬───┘    └────┬─────┘
   │                │                    │               │             │
   │  1. 发送消息   │                    │               │             │
   │───────────────>│                    │               │             │
   │                │                    │               │             │
   │                │ 2. Agent 判断      │               │             │
   │                │    需要执行 Skill  │               │             │
   │                │                    │               │             │
   │                │ 3. POST /api/sandboxes             │             │
   │                │───────────────────>│               │             │
   │                │                    │               │             │
   │                │                    │ 4. 检查沙箱   │             │
   │                │                    │    是否已存在  │             │
   │                │                    │──────────────>│             │
   │                │                    │<──────────────│             │
   │                │                    │               │             │
   │                │                    │   [不存在]     │             │
   │                │                    │               │             │
   │                │                    │ 5. 创建 Pod   │             │
   │                │                    │──────────────>│             │
   │                │                    │               │             │
   │                │                    │               │ 6. hostPath │
   │                │                    │               │    挂载目录 │
   │                │                    │               │────────────>│
   │                │                    │               │             │
   │                │                    │               │ 7. 自动创建 │
   │                │                    │               │  user-data/ │
   │                │                    │               │  (若不存在) │
   │                │                    │               │<────────────│
   │                │                    │               │             │
   │                │                    │ 8. 创建       │             │
   │                │                    │  NodePort Svc │             │
   │                │                    │──────────────>│             │
   │                │                    │               │             │
   │                │                    │ 9. 轮询等待   │             │
   │                │                    │  NodePort分配 │             │
   │                │                    │──────────────>│             │
   │                │                    │<──────────────│             │
   │                │                    │               │             │
   │                │ 10. 返回 sandbox_url               │             │
   │                │<───────────────────│               │             │
   │                │                    │               │             │
   │                │ 11. 通过 sandbox_url               │             │
   │                │     HTTP API 调用 Pod 内服务        │             │
   │                │     执行 Skill                      │             │
   │                │──────────────────────────────────> Pod           │
   │                │                                    │             │
   │                │                                    │ 12. 读写    │
   │                │                                    │  user-data  │
   │                │                                    │────────────>│
   │                │                                    │<────────────│
   │                │                                    │             │
   │                │ 13. 返回执行结果                    │             │
   │                │<────────────────────────────────── Pod           │
   │                │                    │               │             │
   │ 14. 推送结果   │                    │               │             │
   │<───────────────│                    │               │             │
   │                │                    │               │             │
   │ [空闲超时 10min] │                    │               │             │
   │                │ 15. DELETE         │               │             │
   │                │  /api/sandboxes/x  │               │             │
   │                │───────────────────>│               │             │
   │                │                    │ 16. 删除 Svc  │             │
   │                │                    │──────────────>│             │
   │                │                    │ 17. 删除 Pod  │             │
   │                │                    │──────────────>│             │
   │                │                    │               │             │
   │                │                    │               │ 18. 数据    │
   │                │                    │               │  仍在磁盘上 │
   │                │                    │               │  (不会删除) │
   │                │                    │               │             │
```

### 2.3 Pod 内部结构

```
┌─ Pod: sandbox-{session_id} ──────────────────────────────────────┐
│                                                                   │
│  Labels:                                                          │
│    app: kt-agent-sandbox                                          │
│    session-id: {session_id}                                       │
│    user-id: {user_id}                                             │
│                                                                   │
│  ┌─ Container: skill-runtime ─────────────────────────────────┐   │
│  │                                                             │   │
│  │  Image: kt-skill-runtime:latest                             │   │
│  │  Port: 8080                                                 │   │
│  │                                                             │   │
│  │  Mounts:                                                    │   │
│  │    /mnt/skills     ← hostPath (ReadOnly)                    │   │
│  │    /mnt/user-data  ← hostPath (ReadWrite)                   │   │
│  │                                                             │   │
│  │  Resources:                                                 │   │
│  │    requests: cpu=100m, mem=256Mi                             │   │
│  │    limits:   cpu=1000m, mem=1Gi                              │   │
│  │                                                             │   │
│  │  SecurityContext:                                            │   │
│  │    privileged: false                                        │   │
│  │    allowPrivilegeEscalation: false    ← 加固（DeerFlow未做）│   │
│  │    runAsNonRoot: true                 ← 加固（DeerFlow未做）│   │
│  │    capabilities: { drop: [ALL] }      ← 加固（DeerFlow未做）│   │
│  │                                                             │   │
│  │  Probes:                                                    │   │
│  │    readiness: GET /healthz :8080                             │   │
│  │    liveness:  GET /healthz :8080                             │   │
│  │                                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                   │
│  RestartPolicy: Always                                            │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

---

## 三、核心组件说明

### 3.1 Provisioner 服务

- **定位**：FastAPI 内部服务，封装 K8s API 调用
- **核心接口**：POST/GET/DELETE `/api/sandboxes`
- **职责**：创建/查询/销毁 Pod + NodePort Service，幂等处理，空闲回收
- **参考**：DeerFlow `docker/provisioner/app.py`（约 500 行）

### 3.2 Skill 执行器

- **定位**：Agent Backend 内的服务层，串联 Provisioner → Pod HTTP API
- **流程**：获取或创建沙箱 → 等待就绪 → HTTP POST 调用 Pod 内 API 执行 Skill → 返回结果

### 3.3 Skill 运行时镜像

- **定位**：运行在沙箱 Pod 内的 FastAPI HTTP 服务（监听 8080）
- **接口**：`GET /healthz`（健康检查）、`POST /execute`（执行 Skill 脚本）
- **执行方式**：接收 skill_id + params → 调用 `python /mnt/skills/{skill_id}/main.py` → 返回 stdout/stderr/outputs

> 代码实现详见开发阶段的 Issue 任务，此处仅做架构说明。

---

## 四、数据持久化详解

### 4.1 存储模型

```
宿主机磁盘（hostPath）
│
├── /data/kt-agent/skills/               ← 全局共享，只读
│   ├── data_analysis/
│   │   ├── main.py                      ← Skill 入口
│   │   ├── requirements.txt
│   │   └── utils/
│   └── web_scraper/
│       └── main.py
│
└── /data/kt-agent/sessions/             ← 按用户/会话隔离，读写
    └── {user_id}/
        └── {session_id}/
            └── user-data/
                ├── uploads/             ← 用户上传文件
                ├── workspace/           ← 工作临时文件
                ├── outputs/             ← Skill 产物
                └── params.json          ← 执行参数
```

### 4.2 Pod 与 thread 的生命周期绑定

```
首次 Skill 调用（lazy_init=True，延迟到需要时才创建）
    │
    ├── AioSandboxProvider.acquire(thread_id)
    │   ├── 三层一致性检查：
    │   │   Layer 1: 进程内缓存（dict[thread_id → sandbox_id]）
    │   │   Layer 2: 跨进程文件锁（sandbox.lock + sandbox.json）
    │   │   Layer 3: 后端 discover（检查容器是否存活）
    │   │
    │   ├── 若已存在 → 直接复用（更新 last_activity 时间戳）
    │   └── 若不存在 → 创建新沙盒
    │       ├── sandbox_id = sha256(thread_id)[:8]（确定性，跨进程一致）
    │       ├── Provisioner → K8s API → 创建 Pod + NodePort Service
    │       ├── wait_for_sandbox_ready(timeout=60s)
    │       └── 持久化状态到 sandbox.json（供其他进程发现）
    │
    ▼
对话进行中（Pod 常驻运行 HTTP 服务，监听 8080 端口）
    │
    ├── Skill 执行 1 → HTTP POST sandbox_url/v1/shell/exec → 产物写入 hostPath
    ├── Skill 执行 2 → HTTP POST 复用同一 Pod                → 产物写入 hostPath
    ├── Skill 执行 3 → HTTP POST 复用同一 Pod                → 产物写入 hostPath
    │
    │   ⚡ 第 2 次之后命中进程内缓存，无需文件锁和后端检查，延迟极低
    │   每次执行都更新 last_activity 时间戳
    │
    ▼
空闲超时触发销毁（默认 10 分钟无活动）
    │
    ├── 后台守护线程每 60 秒扫描 last_activity 字典
    ├── idle_duration > 600s → 触发 release(sandbox_id)
    │   ├── 清理进程内缓存
    │   ├── 清理 sandbox.json 状态文件
    │   └── Provisioner → K8s API → 删除 Pod + Service
    ├── 宿主机上的 hostPath 目录 **不会被删除**
    │   （Pod 销毁 ≠ 数据销毁）
    │
    ▼
同一 thread 再次调用 Skill（会话恢复）
    │
    ├── acquire(thread_id) → 检查发现不存在 → 创建新 Pod
    ├── 新 Pod 重新挂载同一个 hostPath
    ├── 上次的文件仍然在，无需从任何地方恢复
    │
    ▼
应用进程关闭（或节点重启）
    │
    ├── atexit + SIGTERM/SIGINT → shutdown() 统一销毁所有 Pod
    │
    ▼
会话数据过期（CronJob 定期清理 / 管理员手动）
    │
    └── rm -rf /data/kt-agent/sessions/{user_id}/{session_id}/
```

**核心要点**：
- Pod 的创建时机 = 首次 Skill 调用（非对话开启时）
- Pod 的销毁时机 = 空闲超时 10 分钟后（非对话关闭时）
- 同一 thread 内多次 Skill 调用共享同一 Pod，通过 HTTP API 执行
- 跨进程恢复：即使 backend 进程重启，也能通过文件状态 + 后端 discover 恢复已有 Pod

### 4.3 关键特点

| 特性 | 说明 |
|------|------|
| **零拷贝** | 文件直接在宿主机磁盘上读写，无网络传输 |
| **Pod 无状态** | Pod 销毁后重建即可，数据不丢失 |
| **无外部依赖** | 不需要 OSS / S3 / NFS 等外部存储 |
| **天然持久** | 只要宿主机磁盘不坏，数据就在 |

---

## 五、优势分析

### 5.1 架构简单

- **无额外组件**：不需要 OSS、MinIO、NFS 等外部存储
- **无数据同步**：不存在「上传/下载」的开销和复杂度
- **调试方便**：直接 SSH 到宿主机查看文件
- **代码量少**：Provisioner 核心逻辑约 300 行

### 5.2 性能优秀

| 操作 | hostPath 方案 | OSS 方案（对比） |
|------|-------------|----------------|
| 读文件 | 本地磁盘 IOPS | 网络往返 + OSS IOPS |
| 写文件 | 本地磁盘 IOPS | 本地写 + 异步上传 |
| 恢复工作区 | **0 秒**（直接挂载） | 下载时间（取决于文件大小） |
| 同步产物 | **0 秒**（已在磁盘上） | 上传时间（取决于文件大小） |

### 5.3 成本低

- 无 OSS 流量费
- 无 OSS 存储费
- 无 OSS API 调用费
- 只需宿主机本地磁盘

### 5.4 故障模式简单

- 容器崩溃 → Pod 自动重启 → 数据仍在
- 节点重启 → 数据仍在磁盘上 → 重建 Pod 即可
- 不存在「OSS 不可用导致执行失败」的问题

---

## 六、劣势分析

### 6.1 单节点绑定（最大硬伤）

```
问题：hostPath 是节点本地路径
     → 同一个会话的 Pod 必须调度到同一个节点
     → 否则新 Pod 看不到老数据

解决方案：
  1. nodeSelector / nodeAffinity 固定调度
  2. 单节点部署（小规模场景）
  3. NFS hostPath（变成共享存储，但引入了 NFS）
```

### 6.2 磁盘容量有限

```
问题：宿主机磁盘容量有限，且所有用户共享
     → 大量用户数据可能撑满磁盘

解决方案：
  1. 设置 ephemeral-storage limits
  2. 定期清理过期会话数据
  3. 使用外挂磁盘（如云盘）
```

### 6.3 无高可用

```
问题：节点故障 → 数据丢失（如果磁盘损坏）
     → 没有异地备份

解决方案：
  1. RAID 磁盘阵列
  2. 定期 rsync 到备份节点
  3. 接受风险（沙箱数据本身可以重新生成）
```

### 6.4 无标准化

```
问题：自定义 Provisioner 不是 K8s 标准方案
     → 无法利用 K8s 生态工具（如 kubectl get sandbox）
     → 维护成本在团队手上

解决方案：
  1. 接受（够用就行）
  2. 后期迁移到 Agent Sandbox CRD
```

### 6.5 安全性局限

```
问题：hostPath 本身有安全隐患
     → 容器内如果路径配置错误，可能访问宿主机敏感目录
     → 没有 gVisor / Kata 级别的强隔离

解决方案：
  1. 严格限制 hostPath 路径
  2. securityContext 加固
  3. 使用 PodSecurityPolicy / PodSecurity Admission
```

---

## 七、扩展性分析

### 7.1 水平扩展策略

```
方式 1：单节点纵向扩展
  → 加 CPU / 内存 / 磁盘
  → 简单但有上限

方式 2：多节点 + 节点亲和性
  → 用户 Hash → 固定节点
  → 同一用户的 Pod 始终调度到同一节点
  → 扩展性提升，但运维复杂度增加

方式 3：多节点 + NFS
  → hostPath 指向 NFS 挂载点
  → 所有节点共享同一份数据
  → 本质上变成了网络存储方案
  → 这时不如直接用方案 B
```

### 7.2 容量评估

| 规模 | 并发 Pod | 磁盘需求 | 节点数 | 建议 |
|------|---------|---------|--------|------|
| 小 | < 50 | < 500GB | 1-2 | 单节点即可 |
| 中 | 50-200 | 500GB-2TB | 2-4 | 多节点 + 亲和性 |
| 大 | > 200 | > 2TB | > 4 | 建议切换到方案 B |

---

## 八、成本分析

### 8.1 基础设施成本（以阿里云为例）

**场景假设**：200 并发用户，平均每人每天 10 次 Skill 执行

```
计算资源：
  - 单 Pod: 100m~1000m CPU, 256Mi~1Gi 内存
  - 峰值 50 并发 Pod: 50 CPU, 50Gi 内存
  - 服务器: 16C64G × 4 台
  - 费用: ≈ ¥8000/月

存储资源：
  - 本地 SSD: 1TB × 4 节点
  - 费用: ≈ ¥0/月（服务器自带，或 ≈ ¥400/月 如果外挂云盘）

总计: ≈ ¥8400/月
```

### 8.2 与方案 B 的成本对比

| 项目 | 方案 A (hostPath) | 方案 B (OSS) |
|------|------------------|-------------|
| 计算资源 | ¥8000/月 | ¥8000/月（相同） |
| 本地存储 | ¥400/月 | ¥200/月（更少） |
| OSS 存储 | ¥0 | ¥60/月 |
| OSS 流量 | ¥0 | ¥500-1500/月 |
| OSS API 调用 | ¥0 | ¥50/月 |
| **总计** | **≈ ¥8400/月** | **≈ ¥9810/月** |

---

## 九、实施路线

### Phase 1：基础搭建

- 搭建 K8s 集群（Kind 本地 / 云上自建）
- 构建 `kt-skill-runtime` 镜像
- 实现 SandboxProvisioner 核心逻辑
- 打通 Agent Backend → Provisioner → Pod 链路

### Phase 2：安全加固

- SecurityContext 配置
- NetworkPolicy 限制 Pod 间通信
- 资源限额（LimitRange / ResourceQuota）
- 目录权限控制

### Phase 3：运维完善

- 空闲回收机制
- 过期数据清理任务
- 监控指标（Pod 数量、资源使用、磁盘容量）
- 告警规则（磁盘使用率 > 80%）

### Phase 4：优化（可选）

- 镜像预热（DaemonSet 预拉取）
- 节点亲和性调度
- 多节点扩展

---

## 十、风险评估

| 风险 | 概率 | 影响 | 应对措施 |
|------|------|------|---------|
| 磁盘写满 | 中 | 高 | 监控 + 清理策略 + 告警 |
| 节点故障 | 低 | 高 | 数据可接受丢失 / 定期备份 |
| 容器逃逸 | 低 | 高 | SecurityContext + PodSecurity |
| 并发超限 | 中 | 中 | 资源配额 + 排队机制 |
| hostPath 误配置 | 低 | 高 | 代码审查 + 路径白名单 |

---

## 十一、参考资料

### 开源项目

1. **DeerFlow — 字节跳动开源多 Agent 框架**
   - GitHub: https://github.com/bytedance/deer-flow
   - Provisioner 核心文件: `docker/provisioner/app.py`
   - 我们方案 A 的直接参考来源

2. **Kubernetes Python Client**
   - GitHub: https://github.com/kubernetes-client/python
   - 文档: https://kubernetes.readthedocs.io/
   - 用于操作 K8s API 创建 Pod / Service

### K8s 官方文档

3. **Volumes — hostPath**
   - https://kubernetes.io/docs/concepts/storage/volumes/#hostpath
   - hostPath Volume 的类型说明（Directory / DirectoryOrCreate）

4. **Pod Security Standards**
   - https://kubernetes.io/docs/concepts/security/pod-security-standards/
   - Restricted / Baseline / Privileged 三级安全标准

5. **Jobs**
   - https://kubernetes.io/docs/concepts/workloads/controllers/job/
   - 如果选择 Job 模式而非常驻 Pod 模式

6. **Resource Management**
   - https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
   - CPU / Memory requests 和 limits 配置

### 安全参考

7. **Docker Security Best Practices — OWASP**
   - https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html

8. **Kubernetes Security Checklist**
   - https://kubernetes.io/docs/concepts/security/security-checklist/

### Python SDK

9. **Docker SDK for Python**
   - https://docker-py.readthedocs.io/
   - 如果选择 Docker 单机方案而非 K8s

10. **kubernetes (Python)**
    - https://pypi.org/project/kubernetes/
    - `pip install kubernetes`

---

## 十二、总结

### 方案定位

DeerFlow hostPath 方案是一个 **「够用就好」** 的务实选择。它用最简单的方式解决了沙箱隔离和文件持久化两个核心问题，代价是牺牲了水平扩展性和高可用性。

### 核心取舍

```
✅ 获得：简单、快速、低成本、高性能（本地 I/O）
❌ 放弃：水平扩展、高可用、标准化、强隔离
```

### 适合的团队

- 团队规模小（3-5 人），需要快速交付
- K8s 运维经验一般
- 用户规模可控（< 500 并发）
- 对数据丢失有一定容忍度（沙箱数据可重新生成）
