# 技术方案 B：Agent Sandbox CRD + OSS 对象存储持久化

> **适用场景**：自建 K8s 集群，Skill 沙箱隔离执行，文件通过对象存储持久化
> **核心关键词**：Agent Sandbox CRD / SandboxWarmPool 预热池 / OSS 对象存储 / 云原生

---

## 一、方案概述

### 1.1 核心思路

采用 Kubernetes SIG Apps 官方子项目 **[Agent Sandbox](https://github.com/kubernetes-sigs/agent-sandbox)** 提供的 CRD 体系管理沙箱生命周期。核心模型是 **「共享 Pod 池」**——通过 **SandboxWarmPool** 预热一批通用空白 Pod，用户请求到来时从池中分配一个，根据 **user_id / session_id 决定从 OSS 的哪个路径恢复数据**，执行完毕后清理 emptyDir 并归还池（或销毁后池自动补充）。

**一句话总结**：Pod 是通用的、可复用的，数据身份由 OSS 路径决定，而非 Pod 本身。

### 1.1.1 生命周期模型

```
                    SandboxWarmPool (维持 N 个空白 Pod)
                    ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
                    │Pod-1 │ │Pod-2 │ │Pod-3 │ │Pod-4 │
                    │(空白) │ │(空白) │ │(空白) │ │(空白) │
                    └──┬───┘ └──────┘ └──────┘ └──────┘
                       │
用户 A 请求执行 Skill  │
    │                  │
    ├── 从池中分配 Pod-1
    ├── Init 容器: 从 OSS 下载 users/A/sessions/xxx/ 到 Pod-1
    ├── 执行 Skill → Sidecar 同步产物到 OSS
    ├── 执行完毕
    ├── 清理 Pod-1 的 emptyDir → Pod-1 归还池
    │   （或销毁 Pod-1，池自动补充 Pod-5）
    │
    │                  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
    │                  │Pod-1 │ │Pod-2 │ │Pod-3 │ │Pod-4 │
    │                  │(空白) │ │(空白) │ │(空白) │ │(空白) │
    │                  └──────┘ └──┬───┘ └──────┘ └──────┘
    │                              │
用户 B 请求执行 Skill              │
    │                              │
    ├── 从池中分配 Pod-2
    ├── Init 容器: 从 OSS 下载 users/B/sessions/yyy/ 到 Pod-2
    ├── 执行 Skill → Sidecar 同步产物到 OSS
    └── ...

关键特征：
  - Pod 和用户没有固定绑定关系
  - 任何空白 Pod 都可以服务任何用户
  - 用户身份由 OSS 路径（user_id/session_id）决定
  - Pod 池由 Controller 自动维护，始终保持 N 个就绪
  - 同一用户的不同请求可能分配到不同 Pod
  - 资源消耗 = 池大小（固定），不随在线用户数线性增长
```

### 1.2 Agent Sandbox 项目概况

| 维度 | 说明 |
|------|------|
| **项目归属** | Kubernetes SIG Apps 官方子项目 |
| **GitHub** | https://github.com/kubernetes-sigs/agent-sandbox |
| **当前版本** | v0.1.1（2025 年 2 月发布） |
| **成熟度** | Alpha 阶段，API 可能变动 |
| **许可证** | Apache-2.0 |
| **Star** | ~1.1k |
| **兼容性** | 任何标准 K8s 集群（Kind / minikube / 自建 / 云托管） |

### 1.3 四个 CRD 资源

| CRD | API Group | 功能 |
|-----|-----------|------|
| **Sandbox** | `agents.x-k8s.io/v1alpha1` | 核心资源，管理单个有状态 Pod |
| **SandboxTemplate** | `extensions.agents.x-k8s.io/v1alpha1` | 可复用的沙箱模板 |
| **SandboxWarmPool** | `extensions.agents.x-k8s.io/v1alpha1` | 预热池，维护就绪 Pod 缓存 |
| **SandboxClaim** | `extensions.agents.x-k8s.io/v1alpha1` | 按需申请沙箱（类似 PVC 模式） |

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
│                                   │  SandboxClient           │                │
│                                   │  (Python SDK)            │                │
│                                   │                          │                │
│                                   │  k8s-agent-sandbox       │                │
│                                   │  pip install             │                │
│                                   │  k8s-agent-sandbox       │                │
│                                   └────────────┬─────────────┘                │
│                                                │                              │
└────────────────────────────────────────────────┼──────────────────────────────┘
                                                 │ K8s API
                                                 ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                          Kubernetes Cluster (自建)                            │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │  Agent Sandbox Controller (Deployment)                                  │  │
│  │  - 监听 Sandbox / SandboxClaim / SandboxWarmPool 资源变化               │  │
│  │  - 自动创建/回收/分配 Pod                                               │  │
│  │  - 维护预热池状态                                                       │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
│  ┌─ SandboxWarmPool ──────────────────────────────────────────────────────┐  │
│  │  replicas: 3 (维持 3 个预热 Pod)                                       │  │
│  │                                                                         │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                              │  │
│  │  │ Pod (空闲)│  │ Pod (空闲)│  │ Pod (空闲)│  ← 预热就绪，等待分配       │  │
│  │  │ owner:   │  │ owner:   │  │ owner:   │                              │  │
│  │  │ WarmPool │  │ WarmPool │  │ WarmPool │                              │  │
│  │  └──────────┘  └──────────┘  └──────────┘                              │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                           │ 分配                                              │
│                           ▼                                                   │
│  ┌─ 活跃 Sandbox 实例 ───────────────────────────────────────────────────┐   │
│  │                                                                         │  │
│  │  ┌─ Sandbox: session-aaa ─────────────────────────────────────────┐    │  │
│  │  │  owner: SandboxClaim (已分配给用户)                              │    │  │
│  │  │                                                                 │    │  │
│  │  │  ┌─ Pod ──────────────────────────────────────────────────┐    │    │  │
│  │  │  │                                                         │    │    │  │
│  │  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │    │    │  │
│  │  │  │  │ init:        │  │ main:        │  │ sidecar:     │  │    │    │  │
│  │  │  │  │ oss-restore  │→→│ skill-runtime│  │ oss-sync     │  │    │    │  │
│  │  │  │  │              │  │              │  │              │  │    │    │  │
│  │  │  │  │ 从 OSS 恢复  │  │ 执行 Skill   │  │ 定期同步到   │  │    │    │  │
│  │  │  │  │ 工作区文件    │  │ 脚本         │  │ OSS          │  │    │    │  │
│  │  │  │  └──────────────┘  └──────────────┘  └──────────────┘  │    │    │  │
│  │  │  │                         │                               │    │    │  │
│  │  │  │                    emptyDir Volume                       │    │    │  │
│  │  │  │                    /workspace (三个容器共享)              │    │    │  │
│  │  │  └─────────────────────────────────────────────────────────┘    │    │  │
│  │  └─────────────────────────────────────────────────────────────────┘    │  │
│  │                                                                         │  │
│  │  ┌─ Sandbox: session-bbb ─────────────────────────────────────────┐    │  │
│  │  │  (结构同上)                                                      │    │  │
│  │  └─────────────────────────────────────────────────────────────────┘    │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
│  ┌─ sandbox-router Service ───────────────────────────────────────────────┐  │
│  │  统一路由入口，将请求分发到对应的 Sandbox Pod                           │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
                                     │
                              网络请求（非 Volume）
                                     ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                    OSS 对象存储（阿里云 OSS / 自建 MinIO）                     │
│                                                                               │
│   Bucket: kt-agent-sandbox                                                    │
│   │                                                                           │
│   ├── skills/                          ← Skill 脚本存储                       │
│   │   ├── data_analysis/                                                      │
│   │   │   ├── main.py                                                         │
│   │   │   └── requirements.txt                                                │
│   │   └── web_scraper/                                                        │
│   │       └── main.py                                                         │
│   │                                                                           │
│   └── sessions/                        ← 用户会话数据                          │
│       ├── {user_id_1}/                                                        │
│       │   ├── {session_id_1}/                                                 │
│       │   │   ├── uploads/                                                    │
│       │   │   ├── workspace/                                                  │
│       │   │   └── outputs/                                                    │
│       │   └── {session_id_2}/                                                 │
│       └── {user_id_2}/                                                        │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 预热池分配流程

```
                         SandboxWarmPool Controller
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │ Pod-1    │ │ Pod-2    │ │ Pod-3    │
              │ (就绪)   │ │ (就绪)   │ │ (就绪)   │
              │ owner:   │ │ owner:   │ │ owner:   │
              │ WarmPool │ │ WarmPool │ │ WarmPool │
              └──────────┘ └──────────┘ └──────────┘


用户请求到达 → 创建 SandboxClaim
                    │
                    ▼
              Controller 从预热池分配 Pod-1
                    │
                    ├─→ Pod-1 的 ownerRef 从 WarmPool → Sandbox
                    ├─→ Pod-1 状态变为「已分配」
                    │
                    ▼
              Controller 自动补充新 Pod-4 到预热池
                    │
                    ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │ Pod-2    │ │ Pod-3    │ │ Pod-4    │
              │ (就绪)   │ │ (就绪)   │ │ (启动中) │
              └──────────┘ └──────────┘ └──────────┘

              预热池始终维持 replicas 数量
```

### 2.3 数据流时序图

```
┌──────┐     ┌──────────────┐     ┌────────────┐     ┌──────┐     ┌──────┐
│ 用户  │     │ Agent Backend│     │ K8s / Agent│     │ Pod  │     │ OSS  │
│      │     │              │     │ Sandbox    │     │      │     │      │
└──┬───┘     └──────┬───────┘     └─────┬──────┘     └──┬───┘     └──┬───┘
   │                │                    │               │            │
   │  1. 发送消息   │                    │               │            │
   │───────────────>│                    │               │            │
   │                │                    │               │            │
   │                │ 2. Agent 判断      │               │            │
   │                │    需要执行 Skill  │               │            │
   │                │                    │               │            │
   │                │ 3. SandboxClient   │               │            │
   │                │    创建 Claim      │               │            │
   │                │───────────────────>│               │            │
   │                │                    │               │            │
   │                │                    │ 4. 从预热池   │            │
   │                │                    │    分配 Pod   │            │
   │                │                    │    (~100ms)   │            │
   │                │                    │──────────────>│            │
   │                │                    │               │            │
   │                │                    │               │ 5. Init    │
   │                │                    │               │   容器启动 │
   │                │                    │               │            │
   │                │                    │               │ 6. 从 OSS  │
   │                │                    │               │   恢复文件 │
   │                │                    │               │───────────>│
   │                │                    │               │<───────────│
   │                │                    │               │            │
   │                │                    │               │ 7. 主容器  │
   │                │                    │               │   启动完成 │
   │                │                    │               │            │
   │                │ 8. sandbox.run()   │               │            │
   │                │    执行 Skill 脚本 │               │            │
   │                │──────────────────────────────────>│            │
   │                │                    │               │            │
   │                │                    │               │ 9. 执行    │
   │                │                    │               │   脚本     │
   │                │                    │               │            │
   │                │                    │               │ 10. Sidecar│
   │                │                    │               │   定期同步 │
   │                │                    │               │───────────>│
   │                │                    │               │            │
   │                │ 11. 返回结果       │               │            │
   │                │<──────────────────────────────────│            │
   │                │                    │               │            │
   │ 12. 推送结果   │                    │               │            │
   │<───────────────│                    │               │            │
   │                │                    │               │            │
   │   [会话结束]   │                    │               │            │
   │                │ 13. 删除 Claim     │               │            │
   │                │───────────────────>│               │            │
   │                │                    │               │            │
   │                │                    │ 14. 最终同步  │            │
   │                │                    │──────────────>│            │
   │                │                    │               │───────────>│
   │                │                    │               │            │
   │                │                    │ 15. 销毁 Pod  │            │
   │                │                    │──────────────>│            │
   │                │                    │               │            │
   │                │                    │ 16. 补充      │            │
   │                │                    │   预热 Pod    │            │
   │                │                    │               │            │
```

### 2.4 Pod 内部结构（三容器模式）

```
┌─ Sandbox Pod ────────────────────────────────────────────────────────────────┐
│                                                                               │
│  ownerReference: Sandbox (已分配) 或 SandboxWarmPool (预热中)                 │
│                                                                               │
│  ┌─ initContainer: oss-restore ──────────────────────────────────────────┐   │
│  │                                                                         │   │
│  │  Image: kt-oss-sync:latest                                              │   │
│  │  作用: Pod 启动时，从 OSS 下载工作区文件到 /workspace                    │   │
│  │                                                                         │   │
│  │  逻辑:                                                                  │   │
│  │    1. 读取环境变量 USER_ID, SESSION_ID                                  │   │
│  │    2. 构造 OSS 前缀: sessions/{user_id}/{session_id}/                   │   │
│  │    3. 下载所有文件到 /workspace/                                         │   │
│  │    4. 如果是首次（OSS 无数据），创建默认目录结构                          │   │
│  │                                                                         │   │
│  │  Mount: /workspace (emptyDir, ReadWrite)                                │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│      │                                                                         │
│      │ 执行完毕后                                                              │
│      ▼                                                                         │
│  ┌─ container: skill-runtime ────────────────────────────────────────────┐   │
│  │                                                                         │   │
│  │  Image: kt-skill-runtime:latest                                         │   │
│  │  Port: 8080                                                              │   │
│  │                                                                         │   │
│  │  作用: Skill 执行主容器                                                  │   │
│  │    - 提供 HTTP API: /execute, /healthz                                  │   │
│  │    - 接收执行请求 → 运行 Skill 脚本 → 返回结果                           │   │
│  │                                                                         │   │
│  │  Mount:                                                                  │   │
│  │    /workspace    ← emptyDir (ReadWrite) 工作目录                         │   │
│  │    /mnt/skills   ← ConfigMap/emptyDir (ReadOnly) Skill 脚本              │   │
│  │                                                                         │   │
│  │  Resources:                                                              │   │
│  │    requests: cpu=250m, mem=512Mi                                         │   │
│  │    limits:   cpu=1000m, mem=1Gi                                          │   │
│  │                                                                         │   │
│  │  SecurityContext:                                                        │   │
│  │    runAsNonRoot: true                                                    │   │
│  │    allowPrivilegeEscalation: false                                      │   │
│  │    capabilities: { drop: [ALL] }                                        │   │
│  │    (可选) runtimeClassName: gvisor                                      │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  ┌─ container: oss-sync (Sidecar) ──────────────────────────────────────┐   │
│  │                                                                         │   │
│  │  Image: kt-oss-sync:latest                                              │   │
│  │  作用: 运行期间定期将 /workspace 增量同步到 OSS                          │   │
│  │                                                                         │   │
│  │  逻辑:                                                                  │   │
│  │    while True:                                                          │   │
│  │      sleep(30)    # 每 30 秒                                            │   │
│  │      sync(/workspace → OSS sessions/{user_id}/{session_id}/)            │   │
│  │                                                                         │   │
│  │  Mount: /workspace (emptyDir, ReadOnly)                                 │   │
│  │                                                                         │   │
│  │  Resources:                                                              │   │
│  │    requests: cpu=50m, mem=64Mi                                           │   │
│  │    limits:   cpu=200m, mem=128Mi                                         │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  Volumes:                                                                     │
│    - workspace: emptyDir {}    ← 三个容器共享                                 │
│    - skills: configMap / emptyDir                                             │
│                                                                               │
│  (可选) runtimeClassName: gvisor / kata-qemu                                  │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## 三、Agent Sandbox 安装与配置

### 3.1 安装 Controller

- 官方提供 manifest.yaml + extensions.yaml 两个安装文件
- 一行 `kubectl apply` 即可完成安装
- 安装后集群中会出现 4 个 CRD 资源

### 3.2 SandboxTemplate 模板要点

- 定义三容器 Pod 模板：Init 容器（OSS 恢复）+ 主容器（Skill 运行时）+ Sidecar（OSS 同步）
- 三个容器通过 emptyDir Volume 共享 `/workspace` 目录
- OSS 凭证通过 K8s Secret 注入环境变量
- USER_ID / SESSION_ID 在创建 SandboxClaim 时动态注入
- 可选配置 `runtimeClassName: gvisor` 启用强隔离

### 3.3 配置预热池

- 一个 SandboxWarmPool YAML，指定 `replicas` 数量和引用的 SandboxTemplate
- replicas 根据并发量调整（如 3-50 个）

### 3.4 OSS 凭证配置

- 使用 K8s Secret 存储 OSS 的 endpoint / access_key_id / access_key_secret
- 敏感信息不硬编码在 YAML 模板中

---

## 四、核心组件说明

### 4.1 Python SDK 集成

- 使用官方 `k8s-agent-sandbox` SDK 创建 SandboxClaim，从预热池获取 Pod
- 通过 `sandbox.run()` 执行命令，通过环境变量注入 USER_ID / SESSION_ID
- 销毁时删除 SandboxClaim，Controller 自动回收 Pod

### 4.2 OSS 同步镜像

需要自行开发一个 Docker 镜像（`kt-oss-sync`），包含两个脚本：

- **restore.py**（Init 容器）：根据 USER_ID/SESSION_ID 从 OSS 下载文件到 /workspace，首次访问时创建空目录结构
- **sync_loop.py**（Sidecar 容器）：每 30 秒增量同步 /workspace 到 OSS，基于 MD5 比较只上传变更文件

### 4.3 存储后端抽象

- 支持阿里云 OSS 和 MinIO 两种实现，通过环境变量 `STORAGE_BACKEND` 切换
- 统一接口：upload_file / download_file / list_objects / delete_object

> 代码实现详见开发阶段的 Issue 任务，此处仅做架构说明。

---

## 五、数据持久化详解

### 5.1 存储模型

```
OSS Bucket: kt-agent-sandbox
│
├── skills/                              ← Skill 脚本（全局共享）
│   ├── data_analysis/
│   │   ├── main.py
│   │   └── requirements.txt
│   └── web_scraper/
│       └── main.py
│
└── sessions/                            ← 用户会话数据
    └── {user_id}/
        └── {session_id}/
            ├── uploads/                 ← 用户上传文件
            ├── workspace/               ← 工作临时文件
            └── outputs/                 ← Skill 产物


Pod 内部（emptyDir）:
/workspace/                              ← emptyDir，三容器共享
├── uploads/
├── workspace/
├── outputs/
└── skills/                              ← 从 OSS 或 ConfigMap 加载
```

### 5.2 Pod 池的生命周期（与方案 A 的核心区别）

```
                ┌─────────────────────────────────────────────────────────────┐
                │                  SandboxWarmPool                             │
                │       维持 N 个通用空白 Pod，与用户无关                      │
                │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐                    │
                │  │Pod-1 │  │Pod-2 │  │Pod-3 │  │Pod-4 │                    │
                │  │(空白) │  │(空白) │  │(空白) │  │(空白) │                    │
                │  └──────┘  └──────┘  └──────┘  └──────┘                    │
                └─────────────────────────────────────────────────────────────┘

用户 A 请求执行 Skill (user_id=A, session_id=xxx)
    │
    ├── 1. Controller 从池中取出 Pod-1（~100ms）
    │
    ├── 2. 注入环境变量：USER_ID=A, SESSION_ID=xxx
    │      Init 容器根据这些变量，从 OSS 下载：
    │      sessions/A/xxx/ → /workspace (emptyDir)
    │
    ├── 3. 在 Pod-1 内执行 Skill 脚本
    │      Sidecar 将产物同步到 OSS：sessions/A/xxx/
    │
    ├── 4. 执行完毕 → 最终同步确认
    │
    ├── 5. 清理 emptyDir → Pod-1 归还池（或销毁，池自动补充 Pod-5）
    │
    ▼
用户 B 请求执行 Skill (user_id=B, session_id=yyy)
    │
    ├── 1. Controller 从池中取出 Pod-1（刚归还的，或 Pod-2）
    │
    ├── 2. Init 容器从 OSS 下载：sessions/B/yyy/ → /workspace
    │      （Pod-1 现在服务用户 B，之前服务的是用户 A）
    │
    └── ...

关键对比：
  ┌──────────────────────────────────────────────────────────┐
  │  方案 A：1 thread = 1 Pod                                │
  │  Pod-A 专属 thread-A，Pod-B 专属 thread-B               │
  │  空闲超时 10 分钟后自动销毁（非对话结束即销毁）           │
  │  10 个活跃 thread = 10 个 Pod                            │
  │                                                          │
  │  方案 B：N 个 Pod 服务所有用户                             │
  │  Pod 是通用的，通过 OSS 路径切换「身份」                    │
  │  10 个并发用户，可能只需要 5 个 Pod（如果不同时执行）       │
  │  预热池 Pod 是固定开销，不随用户数线性增长                  │
  └──────────────────────────────────────────────────────────┘
```

### 5.3 与方案 A 的关键差异

| 维度 | 方案 A (hostPath) | 方案 B (OSS) |
|------|------------------|-------------|
| **Pod 绑定** | 1 thread = 1 Pod（强绑定） | 共享 Pod 池（无绑定） |
| **Pod 生命周期** | 空闲超时销毁（默认 10 分钟） | 池级（预热创建，执行完归还） |
| **创建时机** | 首次 Skill 调用时按需创建（lazy_init） | 预热池提前创建 |
| **用户身份** | thread_id → sandbox_id（确定性哈希） | OSS 路径代表用户 |
| **代码执行方式** | Pod 内 HTTP 服务，backend 通过 HTTP API 调用 | Pod 内 HTTP 服务，通过 HTTP API 调用 |
| **跨进程恢复** | 文件锁 + 状态文件 + 后端 discover | Controller 管理 |
| **资源消耗** | 与活跃 thread 数线性相关 | 与池大小相关（相对固定） |
| **数据位置** | 宿主机本地磁盘 | OSS 对象存储 |
| **恢复速度** | 0 秒（直接挂载） | 取决于文件大小和网络 |
| **写入性能** | 本地磁盘 IOPS | 本地 emptyDir + 异步上传 |
| **Pod 调度** | 必须固定到同一节点 | 可以调度到任何节点 |
| **数据安全** | 节点故障可能丢失 | OSS 多副本，高可用 |
| **容量限制** | 宿主机磁盘容量 | OSS 理论无限 |
| **额外成本** | 无 | OSS 存储 + 流量费 |

---

## 六、优势分析

### 6.1 真正的云原生

- **Pod 完全无状态**：可以调度到集群中任何节点
- **弹性伸缩**：加节点即扩容，无需迁移数据
- **标准 K8s API**：`kubectl get sandbox`、`kubectl get sandboxclaim`
- **生态兼容**：可以用 ArgoCD、Helm 等标准工具管理

### 6.2 预热池性能

| 场景 | 无预热池 | 有预热池 |
|------|---------|---------|
| Pod 调度 | 1-5 秒 | 0 秒（已调度） |
| 镜像拉取 | 0-60 秒 | 0 秒（已拉取） |
| 容器启动 | 1-3 秒 | 0 秒（已启动） |
| 就绪探针通过 | 1-5 秒 | 0 秒（已就绪） |
| **总计** | **3-73 秒** | **~100ms（仅 ownerRef 切换）** |

### 6.3 数据安全

- **OSS 多副本**：数据跨多个可用区存储
- **版本控制**（可选）：OSS 支持版本管理，可回滚
- **备份**：OSS 原生支持跨区域复制
- **不受节点故障影响**：节点崩溃不影响数据

### 6.4 水平扩展

```
扩展方式：
1. 增加 K8s 节点 → 自动分散 Pod
2. 增加 WarmPool replicas → 更多预热 Pod
3. OSS 无需扩容 → 理论无限存储

不存在方案 A 的「单节点绑定」问题
```

### 6.5 强隔离支持

```
隔离层级：
1. Pod 级隔离（标准）
2. SecurityContext 加固
3. gVisor 用户态内核隔离（可选）
4. Kata Containers VM 级隔离（可选）
5. NetworkPolicy 网络隔离

方案 A 只能做到 1-2 级
方案 B 可以做到 1-5 级
```

---

## 七、劣势分析

### 7.1 架构复杂度高

```
问题：引入了多个新组件
  - Agent Sandbox Controller（需要安装和维护）
  - OSS / MinIO（需要部署和维护）
  - OSS Sync 镜像（需要自己开发和维护）
  - 三容器 Pod（调试更复杂）

对比方案 A：
  - 方案 A 只需要 K8s API + 一个 Provisioner 服务
  - 方案 B 需要 Agent Sandbox Controller + OSS + Sync 镜像 + 三容器
```

### 7.2 OSS 恢复延迟

```
问题：每次 Pod 创建（即使从预热池分配后），Init 容器需要从 OSS 下载文件

恢复时间估算（内网环境）：
  - 10MB 工作区：~1 秒
  - 100MB 工作区：~5 秒
  - 1GB 工作区：~30 秒

对比方案 A：
  - 方案 A 恢复时间 = 0 秒（hostPath 直接挂载）

缓解措施：
  1. 压缩后传输（tar.gz）
  2. 并发下载
  3. 控制工作区大小
  4. PVC 缓存层（高级方案）
```

### 7.3 Alpha 阶段风险

```
问题：Agent Sandbox 项目处于 v0.1.1 Alpha 阶段
  - API 可能变动（CRD 字段、SDK 接口）
  - 可能有未发现的 Bug
  - 社区支持有限（虽然是 K8s SIG 官方项目）

缓解措施：
  1. 锁定版本，不盲目升级
  2. 做好单元测试和集成测试
  3. 准备回退方案（可以退回到方案 A）
```

### 7.4 OSS 网络依赖

```
问题：OSS 不可用 → 沙箱无法创建（Init 容器下载失败）
     OSS 延迟高 → 用户体验差

对比方案 A：
  - 方案 A 没有外部网络依赖

缓解措施：
  1. 使用内网 OSS 端点（延迟更低）
  2. Init 容器增加重试逻辑
  3. 自建 MinIO（在同一集群内，延迟极低）
  4. 降级策略：OSS 不可用时，创建空工作区
```

### 7.5 成本更高

```
额外成本项：
  - OSS 存储费
  - OSS 流量费（内网免费，外网收费）
  - OSS API 调用费
  - Sync 容器的 CPU/Memory 资源
  - 预热池 Pod 的资源占用（即使空闲也消耗资源）
```

---

## 八、扩展性分析

### 8.1 水平扩展能力

```
方案 B 的扩展路径：

1. 增加 K8s 节点
   → Pod 自动分散到新节点
   → OSS 无需变动
   → 无单节点绑定问题

2. 增加预热池
   → 修改 WarmPool replicas
   → 更多并发请求可以立即得到响应

3. OSS 自动扩展
   → 阿里云 OSS：无需操作，自动扩容
   → MinIO：加节点即可
```

### 8.2 容量评估

| 规模 | 并发 Pod | 预热池 | K8s 节点 | OSS 存储 | 建议 |
|------|---------|--------|---------|---------|------|
| 小 | < 50 | 3-5 | 2-3 | < 100GB | 基础配置 |
| 中 | 50-200 | 10-20 | 4-8 | 100GB-1TB | 加强配置 |
| 大 | 200-1000 | 30-50 | 8-20 | 1-10TB | 生产级配置 |
| 超大 | > 1000 | 50+ | 20+ | > 10TB | 需要额外优化 |

---

## 九、成本分析

### 9.1 基础设施成本（以阿里云为例）

**场景假设**：200 并发用户，平均每人每天 10 次 Skill 执行

```
计算资源（含预热池）：
  - 活跃 Pod: 50 × (250m CPU + 512Mi) = 12.5 CPU, 25Gi
  - Sidecar:  50 × (50m CPU + 64Mi)   = 2.5 CPU, 3.2Gi
  - 预热 Pod: 10 × (250m CPU + 512Mi) = 2.5 CPU, 5Gi
  - Controller: 2 × (100m CPU + 256Mi) = 0.2 CPU, 0.5Gi
  - 总计: ~18 CPU, ~34Gi
  - 服务器: 8C32G × 3 台 + 16C64G × 1 台
  - 费用: ≈ ¥8000/月

OSS 存储：
  - 200 用户 × 500MB/用户 = 100GB
  - 标准存储: ¥0.12/GB/月 = ¥12/月

OSS 内网流量：
  - 免费（同区域内网访问不收费）

OSS 外网流量（如果 K8s 和 OSS 不在同一区域）：
  - 2000 次/天 × 10MB/次 × 30 天 = 600GB/月
  - ¥0.5/GB = ¥300/月
  - 建议：使用内网端点，流量费 = ¥0

OSS API 调用：
  - PUT/GET: ¥0.01/万次
  - 每天约 5 万次 × 30 = 150 万次/月
  - ¥0.01 × 150 = ¥1.5/月

MinIO（自建替代）：
  - 3 节点 × 1TB SSD = ≈ ¥1500/月
  - 无流量费，无 API 费

总计:
  - 阿里云 OSS 方案: ≈ ¥8314/月（使用内网端点）
  - MinIO 自建方案:  ≈ ¥9500/月
```

### 9.2 与方案 A 的成本对比

| 项目 | 方案 A (hostPath) | 方案 B (OSS 内网) | 方案 B (MinIO) |
|------|------------------|------------------|---------------|
| 计算资源 | ¥8000/月 | ¥8000/月 | ¥8000/月 |
| 本地存储 | ¥400/月 | ¥200/月 | ¥200/月 |
| 对象存储 | ¥0 | ¥14/月 | ¥1500/月 |
| 网络流量 | ¥0 | ¥0（内网） | ¥0（内网） |
| **总计** | **¥8400/月** | **¥8214/月** | **¥9700/月** |
| **差异** | 基准 | **-¥186/月** | **+¥1300/月** |

结论：如果使用阿里云 OSS 内网端点，成本其实与方案 A 接近。MinIO 自建方案由于需要额外服务器，成本略高。

---

## 十、实施路线

### Phase 1：环境搭建

- 搭建 K8s 集群（或使用已有集群）
- 安装 Agent Sandbox Controller + Extensions
- 部署 OSS（阿里云 OSS 或 MinIO）
- 构建 `kt-oss-sync` 镜像
- 构建 `kt-skill-runtime` 镜像

### Phase 2：CRD 配置

- 创建 SandboxTemplate
- 创建 SandboxWarmPool
- 创建 OSS Secret
- 验证 SandboxClaim → Pod 分配流程

### Phase 3：SDK 集成

- 安装 `k8s-agent-sandbox` Python SDK
- 实现 SandboxService
- 打通 Agent Backend → SandboxClient → Pod 链路
- 端到端测试

### Phase 4：安全加固

- gVisor 安装和配置（可选）
- NetworkPolicy 配置
- PodSecurity Admission 配置
- OSS 访问权限最小化（RAM 子账号 / STS 临时凭证）

### Phase 5：生产优化

- 预热池参数调优
- OSS 同步策略优化（压缩、并发、增量）
- 监控接入（Prometheus metrics）
- 告警规则配置

---

## 十一、风险评估

| 风险 | 概率 | 影响 | 应对措施 |
|------|------|------|---------|
| Agent Sandbox API 变动 | 中 | 中 | 锁定版本 + 单元测试 |
| OSS 不可用 | 低 | 高 | 降级策略 + 重试 + 告警 |
| 预热池耗尽 | 中 | 中 | 监控 + 动态调整 replicas |
| OSS 同步丢数据 | 低 | 中 | 最终同步 + 重试 + 日志 |
| gVisor 兼容性问题 | 中 | 低 | 部分 syscall 不支持，需测试 |
| Controller 故障 | 低 | 高 | 多副本 + 自动重启 |

---

## 十二、gVisor 隔离说明

### 12.1 什么是 gVisor

gVisor 是 Google 开源的应用内核（application kernel），在用户态拦截容器的系统调用，提供额外的安全隔离层。

```
普通容器:
  Container → syscall → Host Kernel  (共享内核，风险高)

gVisor 容器:
  Container → syscall → gVisor Sentry → Host Kernel  (用户态拦截，风险低)
```

### 12.2 是否必须使用

**不是必须的**。gVisor 是可选的安全增强：

| 场景 | 建议 |
|------|------|
| 执行团队内部编写的 Skill 脚本 | 不需要 gVisor |
| 执行 LLM 生成的代码 | 建议使用 gVisor |
| 执行用户上传的代码 | 强烈建议使用 gVisor |
| 开发和测试环境 | 不需要 gVisor |

### 12.3 启用方式

```yaml
# 在 SandboxTemplate 中添加
spec:
  podTemplate:
    spec:
      runtimeClassName: gvisor    # 添加这一行
```

前提：K8s 节点需要安装 gVisor runsc。

---

## 十三、参考资料

### Agent Sandbox 官方

1. **Agent Sandbox 项目主页**
   - https://agent-sandbox.sigs.k8s.io/
   - 官方文档站点

2. **Agent Sandbox GitHub**
   - https://github.com/kubernetes-sigs/agent-sandbox
   - 源代码、Issue、Release

3. **Getting Started**
   - https://agent-sandbox.sigs.k8s.io/docs/getting_started/
   - 安装和快速上手指南

4. **Python SDK**
   - https://agent-sandbox.sigs.k8s.io/docs/python_sdk/
   - Python SDK 用法和 API 参考
   - PyPI: `pip install k8s-agent-sandbox`

5. **LangChain + Agent Sandbox 指南**
   - https://agent-sandbox.sigs.k8s.io/docs/guides/langchain/
   - 在 Kind 上运行 LangChain Agent 的完整示例

6. **Releases**
   - https://github.com/kubernetes-sigs/agent-sandbox/releases
   - 版本历史和安装文件

### 背景文章

7. **Google 开源博客 — 为什么 K8s 需要 Agent 执行新标准**
   - https://opensource.googleblog.com/2025/11/unleashing-autonomous-ai-agents-why-kubernetes-needs-a-new-standard-for-agent-execution.html
   - 项目背景和动机

8. **Google Cloud 博客 — Agentic AI on K8s and GKE**
   - https://cloud.google.com/blog/products/containers-kubernetes/agentic-ai-on-kubernetes-and-gke
   - 大规模沙箱调度的挑战和解决方案

9. **预热池深度分析**
   - https://pacoxu.wordpress.com/2025/12/02/agent-sandbox-pre-warming-pool-makes-secure-containers-cold-start-lightning-fast/
   - SandboxWarmPool 机制的技术细节

10. **InfoQ 报道**
    - https://www.infoq.com/news/2025/12/agent-sandbox-kubernetes/
    - 业界视角和分析

### OSS / MinIO

11. **阿里云 OSS Python SDK**
    - https://help.aliyun.com/document_detail/32026.html
    - GitHub: https://github.com/aliyun/aliyun-oss-python-sdk

12. **MinIO Python SDK**
    - https://min.io/docs/minio/linux/developers/python/minio-py.html
    - GitHub: https://github.com/minio/minio-py

13. **MinIO 部署指南**
    - https://min.io/docs/minio/kubernetes/upstream/
    - Kubernetes 上部署 MinIO

### 安全 / gVisor

14. **gVisor 官方文档**
    - https://gvisor.dev/docs/
    - 安装、配置、兼容性说明

15. **Kata Containers**
    - https://katacontainers.io/
    - VM 级别隔离的替代方案

16. **K8s Pod Security Standards**
    - https://kubernetes.io/docs/concepts/security/pod-security-standards/

---

## 十四、总结

### 方案定位

Agent Sandbox + OSS 方案是一个 **「面向未来」** 的云原生选择。它用标准化的 K8s CRD 体系管理沙箱，用对象存储解耦计算和存储，代价是增加了架构复杂度和额外依赖。

### 核心取舍

```
✅ 获得：水平扩展、高可用、标准化、强隔离、预热池性能
❌ 放弃：简单性、零依赖、本地 I/O 性能、Alpha 阶段稳定性
```

### 适合的团队

- 团队有 K8s 运维经验
- 用户规模可能快速增长（> 500 并发）
- 对数据安全要求高（不能因节点故障丢数据）
- 需要多节点集群部署
- 愿意承担额外的架构复杂度换取长期收益
