# 多租户 Pod — 企业级沙箱设计方案

> **适用场景**：企业级多用户并发，Skill 沙箱隔离执行，兼顾资源效率与安全性
> **核心关键词**：1 Pod : N 用户 / Linux 用户级隔离 / 预热 Pod 池 / OSS 产物持久化 / Session 级粘滞调度

---

## 一、方案概述

### 1.1 核心思路

采用 **「1 Pod = N 用户」** 的多租户共享模型。预热固定数量的 Pod（如 20 个），每个 Pod 内通过 **Linux 用户账号** 实现用户间隔离。用户连接到
agent_backend 时立刻分配到负载最低的 Pod，Pod 内为其创建专属 Linux 账号和工作目录，文件权限限制在用户自己的目录内。产物通过
OSS 持久化，Pod 销毁不丢数据。

**一句话总结**：多个用户共享 Pod，操作系统级权限隔离，OSS 持久化产物。

### 1.2 设计决策总览

| #  | 设计要素      | 决策                                                             |
|----|-----------|----------------------------------------------------------------|
| 1  | Pod 与用户关系 | 1 Pod : N 用户（多租户共享）                                            |
| 2  | 隔离方式      | Linux 用户账号 + 文件权限                                              |
| 3  | 目录结构      | `user_data/{user_id}/{session_id}/skills+outputs`              |
| 4  | 粘滞粒度      | session_id → pod_id（best-effort，7 天过期）                         |
| 5  | 身份切换      | su 切换方式（root 进程切换到目标用户执行命令）                                    |
| 6  | Tool 封装   | 4 个语义化工具（execute_script / read_file / write_file / list_files） |
| 7  | 管理层形态     | agent_backend 内部模块（sandbox_manager）                            |
| 8  | 与 Pod 通信  | Pod 内 HTTP 服务（命令执行）+ kubernetes SDK（Pod 生命周期管理）                |
| 9  | 弹性策略      | 初期固定数量预热，后续考虑自动扩缩容                                             |
| 10 | Skills 更新 | Pod 定时轮询 OSS，发现新版本自动替换                                         |
| 11 | 用户初始化时机   | WebSocket 连接时异步启动沙箱准备（不阻塞对话），Skill 执行时检查就绪                     |
| 12 | 产物同步      | MD5 增量对比，每 5 分钟同步变更文件到 OSS                                     |
| 13 | 沙箱启用条件    | Scene 配置项 `sandbox_enabled` 控制，仅需要沙箱的 Scene 才加载                |
| 14 | Pod 崩溃策略  | 接受最多 5 分钟数据丢失，从 OSS 恢复                                         |

---

## 二、架构设计

### 2.1 三层架构总览

```
┌───────────────────────────────────────────────────────────────────────┐
│                    第一层：Agent Backend (FastAPI)                      │
│                                                                       │
│   WebSocket ──→ chat_service ──→ assembler ──→ Agent 执行              │
│        │                                           │                  │
│        │ 连接时立刻初始化沙箱                        │ Skill 调用       │
│        ▼                                           ▼                  │
│   ┌────────────────────────────────────────────────────────────────┐  │
│   │  sandbox_manager (内部模块)                                     │  │
│   │                                                                │  │
│   │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │  │
│   │  │ 调度器        │  │ 粘滞映射表    │  │ Pod 生命周期管理     │ │  │
│   │  │ (最少连接)    │  │ (session→pod) │  │ (kubernetes SDK)     │ │  │
│   │  └──────────────┘  └──────────────┘  └──────────────────────┘ │  │
│   └───────────────────────────┬────────────────────────────────────┘  │
│                               │                                       │
└───────────────────────────────┼───────────────────────────────────────┘
                                │ HTTP（通过 Pod ClusterIP 直连）
                                ▼
┌───────────────────────────────────────────────────────────────────────┐
│                    第二层：Pod 集群（预热 N 个）                        │
│                                                                       │
│   Namespace: kt-agent-sandbox                                         │
│                                                                       │
│   ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐       │
│   │  sandbox-001    │ │  sandbox-002    │ │  sandbox-003    │ ...   │
│   │  users: 47      │ │  users: 45      │ │  users: 50      │       │
│   │  ┌───────────┐  │ │  ┌───────────┐  │ │  ┌───────────┐  │       │
│   │  │ HTTP 服务  │  │ │  │ HTTP 服务  │  │ │  │ HTTP 服务  │  │       │
│   │  │ :8080     │  │ │  │ :8080     │  │ │  │ :8080     │  │       │
│   │  └───────────┘  │ │  └───────────┘  │ │  └───────────┘  │       │
│   │  /skills/       │ │  /skills/       │ │  /skills/       │       │
│   │  /user_data/    │ │  /user_data/    │ │  /user_data/    │       │
│   └─────────────────┘ └─────────────────┘ └─────────────────┘       │
│                                                                       │
└───────────────────────────────┬───────────────────────────────────────┘
                                │ 网络请求（OSS SDK）
                                ▼
┌───────────────────────────────────────────────────────────────────────┐
│                    第三层：OSS 对象存储                                 │
│                                                                       │
│   Bucket: kt-agent-sandbox                                            │
│   ├── skills/                     ← 全量 Skill 脚本                   │
│   │   ├── {scene_id_1}/                                               │
│   │   │   ├── skill_a/                                                │
│   │   │   └── skill_b/                                                │
│   │   └── {scene_id_2}/                                               │
│   └── sessions/                   ← 用户产物                          │
│       └── {scene_id}/{user_id}/{session_id}/                          │
│           └── outputs/                                                │
└───────────────────────────────────────────────────────────────────────┘
```

### 2.2 完整请求时序图

```
┌──────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐    ┌─────┐
│ 用户  │    │Agent Backend │    │sandbox_manager│   │ Pod-001  │    │ OSS │
└──┬───┘    └──────┬───────┘    └──────┬───────┘    └────┬─────┘    └──┬──┘
   │               │                   │                  │             │
   │ 1. WebSocket  │                   │                  │             │
   │   连接        │                   │                  │             │
   │──────────────>│                   │                  │             │
   │               │                   │                  │             │
   │               │ 2. 查询 Scene     │                  │             │
   │               │  sandbox_enabled? │                  │             │
   │               │                   │                  │             │
   │               │ [sandbox_enabled = true]             │             │
   │               │                   │                  │             │
   │               │ 3. 异步启动       │                  │             │
   │               │   沙箱准备        │                  │             │
   │               │──────────────────>│ (后台 asyncio.Task，不阻塞)    │
   │               │                   │                  │             │
   │ 4. 对话立刻   │                   │ 5. 查粘滞映射    │             │
   │   开始        │                   │  session→pod?    │             │
   │──────────────>│                   │                  │             │
   │               │                   │ [未命中，按最少   │             │
   │ (用户正常     │                   │  连接数分配]     │             │
   │  对话中，     │                   │                  │             │
   │  不受沙箱     │                   │ 6. POST /init_user             │
   │  准备影响)    │                   │─────────────────>│             │
   │               │                   │                  │             │
   │               │                   │                  │ 7. 创建     │
   │               │                   │                  │ Linux 用户  │
   │               │                   │                  │ + 目录      │
   │               │                   │                  │             │
   │               │                   │                  │ 8. 从 OSS   │
   │               │                   │                  │ 下载产物    │
   │               │                   │                  │────────────>│
   │               │                   │                  │<────────────│
   │               │                   │                  │             │
   │               │                   │                  │ 9. 复制     │
   │               │                   │                  │ Scene Skills│
   │               │                   │                  │ 到用户目录  │
   │               │                   │                  │             │
   │               │                   │ 10. 准备完成     │             │
   │               │                   │   标记 ready     │             │
   │               │                   │<─────────────────│             │
   │               │                   │                  │             │
   │               │ ... 对话继续 ...  │                  │             │
   │               │                   │                  │             │
   │               │ 11. Agent 判断    │                  │             │
   │               │   需要执行 Skill  │                  │             │
   │               │                   │                  │             │
   │               │ 12. 调用 sandbox tool                │             │
   │               │   → 检查沙箱是否 ready               │             │
   │               │   → [已 ready] 直接执行               │             │
   │               │   → [未 ready] await 等待准备完成     │             │
   │               │──────────────────────────────────── >│             │
   │               │                   │                  │             │
   │               │                   │                  │ 13. su 切换 │
   │               │                   │                  │ 用户执行    │
   │               │                   │                  │             │
   │               │ 14. 返回执行结果                      │             │
   │               │< ────────────────────────────────────│             │
   │               │                   │                  │             │
   │ 15. 推送结果  │                   │                  │             │
   │<──────────────│                   │                  │             │
   │               │                   │                  │             │
   │               │                   │      [每 5 分钟]  │             │
   │               │                   │                  │ 16. MD5     │
   │               │                   │                  │ 对比同步    │
   │               │                   │                  │────────────>│
   │               │                   │                  │             │
```

---

## 三、Pod 内部结构

### 3.1 容器设计

每个 Pod 只有一个容器，内部运行：

- **HTTP 服务**（FastAPI，root 身份，监听 8080）：接收 agent_backend 的指令
- **Skill 轮询进程**：定时检查 OSS 上 Skills 是否更新
- **产物同步进程**：定时 MD5 对比并上传变更文件到 OSS

```
┌─ Pod: sandbox-001 ──────────────────────────────────────────────────┐
│                                                                      │
│  Labels:                                                             │
│    app: kt-agent-sandbox                                             │
│    pod-id: sandbox-001                                               │
│                                                                      │
│  ┌─ Container: sandbox-runtime ────────────────────────────────────┐ │
│  │                                                                  │ │
│  │  Image: kt-sandbox-runtime:latest                                │ │
│  │  Port: 8080                                                      │ │
│  │  运行身份: root（需要 su 切换用户）                                │ │
│  │                                                                  │ │
│  │  进程:                                                           │ │
│  │    PID 1: HTTP 服务 (FastAPI, :8080)                             │ │
│  │    PID 2: Skill 轮询 (定时检查 OSS 更新)                         │ │
│  │    PID 3: 产物同步 (定时 MD5 对比上传)                            │ │
│  │                                                                  │ │
│  │  Volumes:                                                        │ │
│  │    /skills/        ← emptyDir（Pod 启动时从 OSS 拉取全量 Skill） │ │
│  │    /user_data/     ← emptyDir（多用户工作目录）                   │ │
│  │                                                                  │ │
│  │  Resources:                                                      │ │
│  │    requests: cpu=500m, mem=1Gi                                   │ │
│  │    limits:   cpu=4000m, mem=8Gi                                  │ │
│  │                                                                  │ │
│  │  SecurityContext:                                                │ │
│  │    capabilities: { drop: [ALL], add: [SETUID, SETGID] }         │ │
│  │    # 需要 SETUID/SETGID 来实现 su 切换用户                      │ │
│  │    # 其余 capabilities 全部移除                                  │ │
│  │                                                                  │ │
│  │  Probes:                                                         │ │
│  │    readiness: GET /healthz :8080                                 │ │
│  │    liveness:  GET /healthz :8080                                 │ │
│  │                                                                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  RestartPolicy: Always                                               │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 3.2 目录结构

```
Pod 文件系统:
/
├── skills/                                ← 全量 Skill（只读引用源）
│   ├── {scene_id_1}/
│   │   ├── skill_a/
│   │   │   ├── main.py
│   │   │   └── requirements.txt
│   │   └── skill_b/
│   │       └── main.py
│   ├── {scene_id_2}/
│   │   └── skill_c/
│   │       └── main.py
│   └── .skills_manifest.json             ← Skills 版本清单（用于轮询更新对比）
│
├── user_data/                             ← 多用户工作区
│   ├── {user_id_1}/                       ← Linux 用户 user_{user_id_1} 的权限边界
│   │   ├── {session_id_1}/                ← 会话 1 的独立工作空间
│   │   │   ├── skills/                    ← 从 /skills/{scene_id}/ 复制的该 Scene 的 Skill
│   │   │   │   ├── skill_a/
│   │   │   │   └── skill_b/
│   │   │   ├── outputs/                   ← Skill 执行产物（同步到 OSS）
│   │   │   └── .sync_manifest.json        ← 产物同步 MD5 清单
│   │   └── {session_id_2}/                ← 会话 2（可能是不同 Scene）
│   │       ├── skills/
│   │       ├── outputs/
│   │       └── .sync_manifest.json
│   │
│   └── {user_id_2}/                       ← 另一个用户（权限完全隔离）
│       └── {session_id_3}/
│           ├── skills/
│           ├── outputs/
│           └── .sync_manifest.json
│
└── tmp/                                   ← 系统临时目录（按用户隔离，见安全章节）
```

### 3.3 Pod 内 HTTP 服务 API

Pod 内的 HTTP 服务运行在 8080 端口，提供以下接口：

| 接口              | 方法   | 功能                       | 调用方             |
|-----------------|------|--------------------------|-----------------|
| `/healthz`      | GET  | 健康检查                     | K8s 探针          |
| `/status`       | GET  | Pod 状态（用户数、资源使用）         | sandbox_manager |
| `/init_user`    | POST | 创建用户账号 + 目录 + 从 OSS 下载产物 | sandbox_manager |
| `/exec`         | POST | 以指定用户身份执行命令              | sandbox tool    |
| `/cleanup_user` | POST | 清理用户目录 + 删除账号            | sandbox_manager |
| `/sync_now`     | POST | 立刻触发指定用户的产物同步            | sandbox_manager |

#### 接口详情

**GET /healthz**

```json
// Response 200
{
  "status": "ok",
  "uptime": 86400,
  "user_count": 47,
  "skills_version": "2025-03-07T10:00:00Z"
}
```

**GET /status**

```json
// Response 200
{
  "pod_id": "sandbox-001",
  "user_count": 47,
  "active_sessions": 52,
  "cpu_usage_percent": 35.2,
  "memory_usage_mb": 3200,
  "disk_usage_mb": 15000,
  "users": [
    {
      "user_id": "u_123",
      "session_count": 2,
      "last_activity": "2025-03-07T14:30:00Z"
    }
  ]
}
```

**POST /init_user**

```json
// Request
{
  "user_id": "u_123",
  "session_id": "sess_abc",
  "scene_id": "scene_001",
  "oss_config": {
    "endpoint": "https://oss-cn-hangzhou-internal.aliyuncs.com",
    "bucket": "kt-agent-sandbox",
    "prefix": "sessions/scene_001/u_123/sess_abc/"
  }
}

// Response 200
{
  "status": "ok",
  "linux_user": "user_u_123",
  "workdir": "/user_data/u_123/sess_abc/",
  "skills_copied": [
    "skill_a",
    "skill_b"
  ],
  "artifacts_downloaded": 3
}
```

初始化流程：

1. 检查 Linux 用户 `user_{user_id}` 是否存在，不存在则 `useradd`
2. 创建目录 `user_data/{user_id}/{session_id}/skills/` 和 `outputs/`
3. 将 `/skills/{scene_id}/` 下的 Skill 复制到 `user_data/{user_id}/{session_id}/skills/`
4. 从 OSS 下载该 session 的历史产物到 `outputs/`
5. 设置目录权限：`chown -R user_{user_id} /user_data/{user_id}/`

**POST /exec**

```json
// Request
{
  "user_id": "u_123",
  "session_id": "sess_abc",
  "command": "python skills/skill_a/main.py --input data.csv",
  "timeout": 300,
  "workdir": "/user_data/u_123/sess_abc/"
}

// Response 200
{
  "stdout": "Processing complete. Output saved to outputs/result.csv\n",
  "stderr": "",
  "exit_code": 0,
  "execution_time_ms": 12340
}
```

执行逻辑：

```bash
su - user_{user_id} -c "cd {workdir} && {command}"
```

**POST /cleanup_user**

```json
// Request
{
  "user_id": "u_123",
  "session_id": "sess_abc",
  "sync_before_cleanup": true
}

// Response 200
{
  "status": "ok",
  "synced_files": 5,
  "cleaned_size_mb": 120
}
```

清理流程：

1. 如果 `sync_before_cleanup=true`，先触发产物同步到 OSS
2. 删除 `user_data/{user_id}/{session_id}/` 目录
3. 如果该 user_id 下已无任何 session 目录，删除 Linux 用户账号

---

## 四、调度策略

### 4.1 最少连接数调度

新 session 分配 Pod 时，优先选择当前活跃用户数最少的 Pod：

```
sandbox_manager 维护的 Pod 状态表:

┌──────────────┬────────────┬────────────┐
│ Pod ID       │ 活跃用户数  │ 状态        │
├──────────────┼────────────┼────────────┤
│ sandbox-001  │ 47         │ Ready      │
│ sandbox-002  │ 45  ← 最少  │ Ready      │
│ sandbox-003  │ 50         │ Ready      │
│ sandbox-004  │ 48         │ Ready      │
│ ...          │            │            │
└──────────────┴────────────┴────────────┘

新 session 请求 → 分配到 sandbox-002
```

Pod 状态信息来源：

- **主动轮询**：sandbox_manager 定期（如每 30 秒）调用各 Pod 的 `GET /status` 接口
- **被动更新**：每次 init_user / cleanup_user 后更新本地缓存

### 4.2 Session 粘滞（7 天）

```
粘滞映射表（持久化到数据库）:

┌──────────────┬──────────────┬─────────────────────┐
│ session_id   │ pod_id       │ last_active          │
├──────────────┼──────────────┼─────────────────────┤
│ sess_abc     │ sandbox-002  │ 2025-03-07 14:30:00 │
│ sess_def     │ sandbox-001  │ 2025-03-06 09:15:00 │
│ sess_ghi     │ sandbox-003  │ 2025-03-01 08:00:00 │ ← 超过 7 天，已失效
└──────────────┴──────────────┴─────────────────────┘
```

分配逻辑：

1. 查询粘滞映射：`session_id → pod_id`
2. **命中且 Pod 存活** → 直接复用该 Pod（省去 OSS 下载，产物已在 Pod 本地）
3. **命中但 Pod 已销毁/重启** → 清除映射，走最少连接数分配
4. **未命中或已过期（> 7 天）** → 走最少连接数分配
5. 分配后写入映射表

### 4.3 Pod 负载监控

sandbox_manager 需要监控每个 Pod 的健康状态和负载：

| 监控指标     | 数据来源               | 触发动作                 |
|----------|--------------------|----------------------|
| Pod 存活状态 | K8s API（Pod Phase） | 不存活则从映射表清除相关 session |
| 用户数      | GET /status        | 用于最少连接数调度            |
| CPU 使用率  | GET /status        | 超过阈值告警               |
| 内存使用率    | GET /status        | 超过阈值告警               |
| 磁盘使用量    | GET /status        | 超过阈值触发清理             |

### 4.4 Pod 容量上限

每个 Pod 设置最大用户数上限（如 100），超过后不再接受新用户分配。当所有 Pod 都满载时，sandbox_manager 返回「资源不足」错误，由
agent_backend 降级处理（如提示用户排队等待）。

---

## 五、生命周期流程

### 5.1 Pod 创建（系统启动时预热）

```
系统启动 / 运维操作
    │
    ├── sandbox_manager 读取配置: pod_count=20
    │
    ├── 通过 kubernetes SDK 创建 20 个 Pod
    │   名称: sandbox-001 ~ sandbox-020
    │   命名空间: kt-agent-sandbox
    │
    ├── 等待所有 Pod Ready（readiness probe 通过）
    │
    ├── 每个 Pod 启动时自动执行:
    │   ├── 启动 HTTP 服务 (:8080)
    │   ├── 从 OSS 拉取全量 Skills 到 /skills/
    │   ├── 生成 .skills_manifest.json（记录版本信息）
    │   ├── 启动 Skill 轮询进程（定时检查更新）
    │   └── 启动产物同步进程（定时 MD5 对比上传）
    │
    └── sandbox_manager 初始化 Pod 状态表
        记录每个 Pod 的 ID、IP、状态
```

### 5.2 用户连接（异步准备沙箱）

```
用户 WebSocket 连接到 agent_backend
    │
    ├── chat_service 接收连接
    │   获取 user_id, session_id, scene_id
    │
    ├── 查询 Scene 配置: sandbox_enabled?
    │
    │   [sandbox_enabled = false]
    │   └── 跳过沙箱初始化，正常对话
    │
    │   [sandbox_enabled = true]
    │   │
    │   ├── 异步启动沙箱准备（asyncio.Task，不阻塞对话）
    │   │   sandbox_task = asyncio.create_task(
    │   │       sandbox_manager.init_sandbox(user_id, session_id, scene_id)
    │   │   )
    │   │
    │   ├── 将 sandbox_task 存入上下文（ContextVar 或 session 级缓存）
    │   │
    │   └── 对话立刻开始，用户无需等待沙箱就绪
    │
    │
    │   沙箱准备（后台并行执行）:
    │   │
    │   ├── sandbox_manager:
    │   │   ├── 查粘滞映射 session_id → pod_id
    │   │   │
    │   │   │   [命中且 Pod 存活]
    │   │   │   ├── 检查 Pod 内是否已有该 session 的目录
    │   │   │   │   [已有] → 直接复用，无需下载，标记 ready
    │   │   │   │   [没有] → 调 POST /init_user 初始化
    │   │   │   └── 更新 last_active 时间
    │   │   │
    │   │   │   [未命中或 Pod 不可用]
    │   │   │   ├── 按最少连接数选择 Pod
    │   │   │   ├── 调 POST /init_user 初始化
    │   │   │   └── 写入粘滞映射
    │   │   │
    │   │   └── 返回 sandbox_info {pod_id, pod_ip, workdir}，标记 ready
    │   │
    │   └── 准备完成，sandbox_task 可被 await
    │
    │
    │   Skill 执行时（Tool 内部）:
    │   │
    │   ├── sandbox_info = await sandbox_task   ← 如果已 ready，立刻返回
    │   │                                       ← 如果未 ready，等待准备完成
    │   └── 使用 sandbox_info 执行命令
```

**核心设计**：沙箱准备和对话是并行的。绝大多数情况下，用户发第一条消息、Agent 推理、判断需要 Skill
这一系列步骤需要数秒，而沙箱准备（粘滞命中时几乎为零，未命中时 1-5 秒）在此期间已完成。只有极端情况（用户第一句话就触发 Skill
且沙箱未就绪）才会出现短暂等待。

### 5.3 Skill 执行

```
Agent 决定调用 Skill（通过 Tool）
    │
    ├── Tool: execute_script / read_file / write_file / list_files
    │
    ├── Tool 内部:
    │   ├── 从上下文获取 sandbox_info (pod_ip, user_id, session_id, workdir)
    │   ├── 将工具调用转换为 shell 命令
    │   │   例: execute_script(language="python", script_path="skills/analysis/main.py")
    │   │   → "python skills/analysis/main.py"
    │   │
    │   ├── POST http://{pod_ip}:8080/exec
    │   │   {
    │   │     "user_id": "u_123",
    │   │     "session_id": "sess_abc",
    │   │     "command": "python skills/analysis/main.py",
    │   │     "timeout": 300
    │   │   }
    │   │
    │   └── 返回 stdout / stderr / exit_code 给 Agent
    │
    └── Agent 继续推理或返回结果给用户
```

### 5.4 用户断开

```
用户 WebSocket 断开
    │
    ├── chat_service 收到断开事件
    │
    ├── 调用 sandbox_manager.on_user_disconnect(user_id, session_id)
    │
    ├── sandbox_manager:
    │   ├── 调 POST /sync_now 立即同步该 session 的产物到 OSS
    │   ├── 更新粘滞映射的 last_active 时间
    │   └── 注意：不清理用户目录（7 天粘滞期内保留）
    │
    └── 粘滞期内用户再次连接可直接复用
```

### 5.5 用户目录清理（7 天过期）

```
定时任务（CronJob 或 sandbox_manager 内部定时器）
    │
    ├── 每天执行一次
    │
    ├── 遍历粘滞映射表:
    │   筛选 last_active < 当前时间 - 7 天 的记录
    │
    ├── 对每条过期记录:
    │   ├── 调 POST /cleanup_user 清理 Pod 内目录
    │   │   （清理前会先同步到 OSS）
    │   └── 删除粘滞映射记录
    │
    └── 产物仍保留在 OSS 中（OSS 数据不受 7 天限制）
        用户再次连接时从 OSS 重新下载
```

### 5.6 Pod 销毁

```
运维操作 / Pod 崩溃 / 节点故障
    │
    ├── [计划销毁（运维操作）]
    │   ├── sandbox_manager 调用各用户的 /sync_now 同步产物
    │   ├── 等待同步完成
    │   ├── 通过 kubernetes SDK 删除 Pod
    │   └── 清除该 Pod 相关的粘滞映射
    │
    ├── [异常崩溃]
    │   ├── K8s 自动重启 Pod（RestartPolicy: Always）
    │   ├── Pod 重启后 emptyDir 数据丢失
    │   ├── Pod 重新从 OSS 拉取 Skills
    │   ├── 用户下次连接时重新 init_user（从 OSS 恢复产物）
    │   └── 最多丢失 5 分钟未同步的产物变更
    │
    └── sandbox_manager 检测到 Pod 状态变化
        更新 Pod 状态表，清除失效的粘滞映射
```

---

## 六、数据持久化

### 6.1 OSS 目录结构

```
Bucket: kt-agent-sandbox
│
├── skills/                                ← Skill 脚本（全局共享）
│   ├── {scene_id_1}/
│   │   ├── skill_a/
│   │   │   ├── main.py
│   │   │   ├── requirements.txt
│   │   │   └── utils/
│   │   └── skill_b/
│   │       └── main.py
│   ├── {scene_id_2}/
│   │   └── skill_c/
│   │       └── main.py
│   └── .version                           ← 全局版本号（用于轮询对比）
│
└── sessions/                              ← 用户产物（按 scene/user/session 隔离）
    ├── {scene_id_1}/
    │   ├── {user_id_1}/
    │   │   ├── {session_id_1}/
    │   │   │   └── outputs/
    │   │   │       ├── result.csv
    │   │   │       └── chart.png
    │   │   └── {session_id_2}/
    │   │       └── outputs/
    │   └── {user_id_2}/
    │       └── ...
    └── {scene_id_2}/
        └── ...
```

### 6.2 Skills 分发与更新

**初始拉取**（Pod 启动时）：

```
Pod 启动
  → 从 OSS 下载 skills/ 前缀下的所有文件到 /skills/
  → 生成 .skills_manifest.json:
    {
      "version": "2025-03-07T10:00:00Z",
      "files": {
        "scene_001/skill_a/main.py": "md5_hash_1",
        "scene_001/skill_b/main.py": "md5_hash_2"
      }
    }
```

**定时轮询更新**：

```
每 5 分钟:
  → 读取 OSS 上的 skills/.version 文件
  → 与本地 .skills_manifest.json 中的 version 对比
  → [版本相同] → 跳过
  → [版本不同] → 增量下载变更文件，更新本地 /skills/ 和 manifest
```

**管理后台发布 Skill 时**：

```
管理后台修改 Skill 脚本
  → 上传到 OSS skills/{scene_id}/{skill_name}/
  → 更新 skills/.version 为当前时间戳
  → 各 Pod 在下一次轮询时自动发现并更新
  → 延迟：最多 5 分钟
```

注意：已有用户目录中复制的 Skills 不会自动更新。新 Skill 版本只对新初始化的 session 生效。如果需要对已有 session
生效，需要考虑额外机制（如用户主动刷新或符号链接替代复制）。

### 6.3 产物同步（MD5 增量）

**同步流程**（每 5 分钟执行一次）：

```python
# 伪代码
async def sync_artifacts():
    for user_dir in scan("/user_data/"):
        for session_dir in scan(user_dir):
            outputs_dir = f"{session_dir}/outputs/"
            manifest_path = f"{session_dir}/.sync_manifest.json"

            # 读取上次同步的 MD5 清单
            old_manifest = load_json(manifest_path) or {}

            # 扫描当前文件，计算 MD5
            new_manifest = {}
            for file in walk(outputs_dir):
                new_manifest[file] = md5(file)

            # 对比找出变更
            to_upload = []
            to_delete = []

            for path, md5_hash in new_manifest.items():
                if path not in old_manifest or old_manifest[path] != md5_hash:
                    to_upload.append(path)  # 新增或修改

            for path in old_manifest:
                if path not in new_manifest:
                    to_delete.append(path)  # 已删除

            # 执行同步
            for path in to_upload:
                oss_upload(local=path, remote=oss_prefix + path)
            for path in to_delete:
                oss_delete(remote=oss_prefix + path)

            # 更新清单
            save_json(manifest_path, new_manifest)
```

**Pod 销毁前最终同步**：

收到 SIGTERM 信号后，立即执行一次全量同步，确保最新产物不丢失。

### 6.4 Pod 崩溃恢复

| 事件       | 影响            | 恢复方式                          |
|----------|---------------|-------------------------------|
| Pod 正常重启 | emptyDir 数据丢失 | 重新拉取 Skills；用户下次连接时从 OSS 恢复产物 |
| Pod 异常崩溃 | 最多丢失 5 分钟产物   | K8s 自动重启 Pod；同上               |
| 节点故障     | 该节点所有 Pod 受影响 | Pod 调度到其他节点；从 OSS 完整恢复        |

---

## 七、Tool 封装设计

### 7.1 工具列表

Agent 通过 4 个语义化工具与沙箱交互，底层统一走 Pod HTTP 服务的 `/exec` 接口：

#### execute_script — 执行脚本

在沙箱中执行脚本或命令。支持 Python、Shell、Node.js 等语言。

```
参数:
  language: str      — 脚本语言（python / shell / node）
  script_path: str   — 脚本文件路径（相对于工作目录），与 code 二选一
  code: str          — 内联代码（直接执行，不需要文件），与 script_path 二选一
  args: str          — 命令行参数（可选）
  timeout: int       — 超时秒数（可选，默认 300）

返回:
  stdout: str        — 标准输出
  stderr: str        — 标准错误
  exit_code: int     — 退出码
```

转换逻辑：

```
execute_script(language="python", script_path="skills/analysis/main.py", args="--input data.csv")
→ "python skills/analysis/main.py --input data.csv"

execute_script(language="shell", code="grep 'ERROR' logs/app.log | wc -l")
→ "bash -c \"grep 'ERROR' logs/app.log | wc -l\""

execute_script(language="python", code="print(1+1)")
→ "python -c \"print(1+1)\""
```

#### read_file — 读取文件

读取沙箱中的文件内容。

```
参数:
  path: str          — 文件路径（相对于工作目录）
  start_line: int    — 起始行（可选，用于大文件部分读取）
  end_line: int      — 结束行（可选）

返回:
  content: str       — 文件内容
  total_lines: int   — 文件总行数
```

转换逻辑：

```
read_file(path="outputs/result.csv")
→ "cat outputs/result.csv"

read_file(path="outputs/big.log", start_line=100, end_line=200)
→ "sed -n '100,200p' outputs/big.log"
```

#### write_file — 写入文件

在沙箱中创建或覆盖文件。

```
参数:
  path: str          — 文件路径（相对于工作目录）
  content: str       — 文件内容

返回:
  success: bool      — 是否成功
  size_bytes: int    — 文件大小
```

转换逻辑：

```
write_file(path="scripts/preprocess.py", content="import pandas as pd\n...")
→ "cat > scripts/preprocess.py << 'SANDBOX_EOF'\nimport pandas as pd\n...\nSANDBOX_EOF"
```

#### list_files — 列出目录

列出沙箱中的目录内容。

```
参数:
  path: str          — 目录路径（可选，默认为工作目录根）
  recursive: bool    — 是否递归列出（可选，默认 false）

返回:
  files: list        — 文件列表（含名称、大小、修改时间）
```

转换逻辑：

```
list_files(path="outputs/")
→ "ls -la outputs/"

list_files(path=".", recursive=true)
→ "find . -type f -printf '%p %s %T@\n'"
```

### 7.2 底层执行层

所有工具共享同一个底层执行函数：

```python
# 伪代码
async def sandbox_exec(
        pod_ip: str,
        user_id: str,
        session_id: str,
        command: str,
        timeout: int = 300,
) -> dict:
    """统一的沙箱命令执行入口。"""
    workdir = f"/user_data/{user_id}/{session_id}"

    response = await http_client.post(
        f"http://{pod_ip}:8080/exec",
        json={
            "user_id": user_id,
            "session_id": session_id,
            "command": command,
            "timeout": timeout,
            "workdir": workdir,
        },
    )
    return response.json()
```

### 7.3 与 Agent 集成

这 4 个工具作为 LangChain Tool 注册到 Agent 的工具列表中。在 `assembler.py` 组装 Agent 时，如果 Scene 启用了沙箱（
`sandbox_enabled=true`），则将沙箱工具加入 tools 列表：

```python
# assembler.py 中的集成逻辑（伪代码）
if scene_config.get("sandbox_enabled", False):
    sandbox_info = await sandbox_manager.init_sandbox(user_id, session_id, scene_id)
    sandbox_tools = create_sandbox_tools(sandbox_info)
    tools.extend(sandbox_tools)
```

---

## 八、安全隔离

### 8.1 Linux 用户级隔离

#### 用户创建

```bash
# 创建用户，指定 home 目录，使用 nologin shell（防止交互式登录）
useradd -M -d /user_data/{user_id} -s /usr/sbin/nologin user_{user_id}

# 设置目录权限（700：仅用户自己可读写执行）
chown -R user_{user_id}:user_{user_id} /user_data/{user_id}/
chmod 700 /user_data/{user_id}/
```

#### 权限效果

```
user_u_123 可以:
  ✅ 读写 /user_data/u_123/ 下的所有文件
  ✅ 执行 /user_data/u_123/ 下的脚本

user_u_123 不能:
  ❌ 读写 /user_data/u_456/（权限 700，其他用户无权访问）
  ❌ 读写 /skills/（只读挂载或权限控制）
  ❌ 修改系统文件
  ❌ 查看其他用户的进程（hidepid=2）
```

### 8.2 安全加固措施

| 加固项          | 实现方式                                       | 防护目标          |
|--------------|--------------------------------------------|---------------|
| 进程隔离         | `/proc` 挂载 `hidepid=2`                     | 防止用户间窥探进程信息   |
| 资源限制         | cgroups v2 对每个用户设置 CPU/内存/进程数上限            | 防止单用户资源耗尽影响他人 |
| 磁盘配额         | `quota` 或检查脚本限制用户目录大小                      | 防止磁盘写满        |
| /tmp 隔离      | 每个用户使用 `/user_data/{user_id}/tmp/` 作为临时目录  | 防止 /tmp 信息泄露  |
| 网络限制         | `iptables` 限制非 root 用户的出站连接                | 防止横向渗透        |
| capabilities | `drop ALL, add SETUID SETGID`（仅 root 进程需要） | 最小权限原则        |
| seccomp      | 限制高危系统调用（`mount`、`ptrace`、`unshare` 等）     | 防止提权          |
| 只读 Skills    | `/skills/` 目录对普通用户只读                       | 防止篡改 Skill 脚本 |
| 命令超时         | 所有执行命令强制超时（默认 300 秒）                       | 防止死循环/挂起      |
| 文件路径校验       | 工具层校验路径不包含 `..` 和绝对路径                      | 防止目录穿越        |

### 8.3 风险与缓解

| 风险               | 概率    | 影响 | 缓解措施                                 |
|------------------|-------|----|--------------------------------------|
| 用户间进程信息泄露        | 高(默认) | 中  | `hidepid=2` 挂载 /proc                 |
| LLM 生成恶意命令导致资源耗尽 | 中     | 高  | cgroups 资源限制 + 命令超时                  |
| 内核提权漏洞           | 极低    | 极高 | seccomp + capabilities drop + 及时更新内核 |
| 目录穿越攻击           | 低     | 高  | 工具层路径校验 + 权限 700                     |
| 网络嗅探/端口访问        | 低     | 中  | iptables 规则                          |

**风险评估结论**：在企业级场景下（用户为内部员工，Skill 由团队编写），Linux 用户级隔离配合上述加固措施，安全性足够。核心防护重点是
**cgroups 资源限制**（防止 LLM 生成的异常命令影响其他用户）。

---

## 九、与现有系统集成

### 9.1 Scene 配置扩展

在 Scene 表中新增沙箱相关配置字段：

```python
# Scene 模型扩展
class Scene(BaseModel):
    # ... 已有字段 ...

    sandbox_enabled: bool = False  # 是否启用沙箱（comment="是否启用 Skill 沙箱执行"）
    sandbox_timeout: int = 300  # 沙箱命令默认超时秒数（comment="沙箱命令超时时间（秒）"）
```

### 9.2 chat_service 集成点

在 `chat_service.handle_chat()` 的 Phase 1（上下文设置）之后，异步启动沙箱准备，不阻塞后续流程：

```python
# chat_service.py 变更（伪代码）

async def handle_chat(...):
    # Phase 1: 上下文设置
    set_request_context(user_id=user_id, scene_id=scene_id, ...)

    # Phase 2: Skill 手动触发检测（已有）
    ...

    # Phase 2.5: 异步启动沙箱准备（新增，不阻塞）
    sandbox_task = None
    if scene_config.get("sandbox_enabled", False):
        sandbox_task = asyncio.create_task(
            sandbox_manager.init_sandbox(
                user_id=user_id,
                session_id=session_id,
                scene_id=scene_id,
            )
        )
        # 将 task 存入上下文，供 Tool 执行时 await
        set_sandbox_task(sandbox_task)

    # Phase 3: assembler.assemble（传入 sandbox_task）
    # 对话立刻开始，不等待沙箱就绪
    main_agent = await assembler.assemble(
        scene_id=scene_id,
        session_id=session_id,
        sandbox_task=sandbox_task,  # 新增：传递 task 而非 info
        ...
    )
```

### 9.3 assembler 集成

`assembler.assemble()` 接收 `sandbox_task`，创建工具时将其传入，Tool 执行时才 await：

```python
# assembler.py 变更（伪代码）

async def assemble(scene_id, session_id, sandbox_task=None, ...):
    # ... 已有逻辑 ...

    # 如果启用了沙箱，创建沙箱工具（此时不等待沙箱就绪）
    if sandbox_task is not None:
        from agent_backend.sandbox.tools import create_sandbox_tools
        sandbox_tools = create_sandbox_tools(sandbox_task)  # 传入 task，不是 info
        tools.extend(sandbox_tools)

    # ... 创建 Agent ...
```

Tool 内部执行时才 await sandbox_task：

```python
# tools.py 中的执行逻辑（伪代码）

async def execute_script(language, script_path, ...):
    # 首次调用时 await sandbox_task，获取 sandbox_info
    # 如果沙箱已就绪，立刻返回；否则等待准备完成
    sandbox_info = await sandbox_task
    # 使用 sandbox_info 执行命令
    return await sandbox_exec(sandbox_info, command)
```

---

## 十、模块结构

```
agent_backend/
├── sandbox/                           ← 沙箱模块（新增）
│   ├── __init__.py
│   ├── sandbox_manager.py             ← 沙箱管理器（调度、粘滞、Pod 生命周期）
│   ├── pod_client.py                  ← Pod HTTP 客户端（调用 Pod 内服务）
│   ├── scheduler.py                   ← 调度策略（最少连接数 + 粘滞）
│   └── tools.py                       ← 4 个语义化 Tool 定义
│
├── middleware/                        ← 中间件模块（已有）
├── engine/                            ← 引擎模块（已有）
├── services/                          ← 服务模块（已有）
└── ...

common/
├── models/
│   └── sandbox_session.py             ← 粘滞映射表 ORM 模型（新增）
└── ...

sandbox_runtime/                       ← Pod 内运行时（独立项目/目录）
├── Dockerfile                         ← 沙箱运行时镜像
├── app.py                             ← FastAPI HTTP 服务（/exec, /init_user, ...）
├── user_manager.py                    ← Linux 用户创建/删除
├── skill_updater.py                   ← Skill 轮询更新
├── artifact_syncer.py                 ← 产物 MD5 同步
└── requirements.txt
```

---

## 十一、容量评估

### 11.1 资源规格

**单 Pod 规格（承载约 50-100 个并发用户）**：

| 资源  | Requests | Limits          | 说明                 |
|-----|----------|-----------------|--------------------|
| CPU | 500m     | 4000m           | 多用户共享，突发负载需要余量     |
| 内存  | 1Gi      | 8Gi             | 每用户约 50-100Mi 工作内存 |
| 磁盘  | -        | 50Gi (emptyDir) | Skills + 用户数据      |

**集群总资源（以 5000 并发用户为例）**：

```
Pod 数量:    5000 用户 / 50 用户每Pod = 100 个 Pod
CPU 总量:    100 × 4 CPU = 400 CPU (limits)
内存总量:    100 × 8Gi = 800Gi (limits)
服务器:      16C64G × 8 台 ≈ 128 CPU, 512Gi（实际 requests 远小于 limits）
```

### 11.2 成本估算

| 项目               | 估算              |
|------------------|-----------------|
| Pod 数量 (5000 用户) | ~100            |
| 服务器数量            | 8-10 台          |
| 计算成本             | ¥8,000-10,000/月 |
| OSS 成本           | ¥14/月           |
| **总计**           | **~¥10,000/月**  |

---

## 十二、实施路线

### Phase 1：Pod 运行时

- 构建 `sandbox-runtime` 镜像（FastAPI HTTP 服务）
- 实现 `/healthz`、`/exec`、`/init_user`、`/cleanup_user` 接口
- 实现 Linux 用户创建/删除逻辑
- 实现 su 身份切换执行
- 本地 Docker 环境验证

### Phase 2：沙箱管理模块

- 实现 `sandbox_manager`（Pod 生命周期管理）
- 实现调度器（最少连接数 + 粘滞映射）
- 集成 kubernetes Python SDK（创建/监控 Pod）
- 创建粘滞映射表（数据库 ORM）

### Phase 3：OSS 集成

- 实现 Skill 初始拉取和轮询更新
- 实现产物 MD5 增量同步
- 实现用户初始化时的 OSS 产物下载
- 对接阿里云 OSS SDK

### Phase 4：Tool 封装与集成

- 实现 4 个语义化 Tool（LangChain Tool）
- 集成到 assembler（sandbox_enabled 条件加载）
- 集成到 chat_service（连接时初始化沙箱）
- 端到端联调测试

### Phase 5：安全加固

- hidepid=2 进程隔离
- cgroups v2 资源限制
- seccomp profile 配置
- 路径校验和命令过滤
- 安全测试

### Phase 6：生产优化

- 监控接入（Prometheus metrics）
- Pod 容量告警
- 用户目录 7 天过期清理
- 弹性扩缩容（可选）

---

## 十三、风险评估

| 风险            | 概率 | 影响 | 应对措施                      |
|---------------|----|----|---------------------------|
| 单 Pod 故障影响多用户 | 低  | 高  | 多 Pod 分散 + 自动重启 + OSS 恢复  |
| LLM 生成恶意命令    | 中  | 中  | cgroups 限制 + 命令超时 + 路径校验  |
| 用户间数据泄露       | 低  | 高  | 文件权限 700 + hidepid + 路径校验 |
| OSS 不可用       | 低  | 中  | 降级策略（已有本地数据继续使用）          |
| 产物丢失（Pod 崩溃）  | 低  | 中  | 5 分钟同步间隔，最多丢 5 分钟数据       |
| Pod 满载        | 中  | 中  | 容量上限 + 监控告警 + 后续扩缩容       |
| 磁盘写满          | 中  | 高  | 用户磁盘配额 + 监控告警             |

---

## 十四、总结

### 方案定位

本方案是面向 **企业级大规模用户** 的务实选择。通过 1 Pod : N 用户的多租户模型，大幅降低资源消耗（100 个 Pod 支撑 5000
用户），同时通过 Linux 用户级隔离保障安全性。

### 核心取舍

```
✅ 获得：
  - 资源效率极高（100 个 Pod 支撑 5000 用户）
  - 架构相对简单（无 CRD、无三容器、无预热池调度）
  - OSS 持久化（数据安全，多节点支持）
  - Session 粘滞（7 天内复用，减少 OSS 下载）
  - 灵活扩展（加 Pod 即扩容）

❌ 放弃：
  - Pod 级隔离（换成 Linux 用户级隔离，安全性略低）
  - 单用户故障隔离（一个 Pod 崩溃影响该 Pod 上所有用户）
  - gVisor 等强隔离能力（多租户场景下不适用）

⚠️ 核心风险点：
  - LLM 生成异常命令导致资源耗尽 → 必须做好 cgroups 限制
  - Pod 崩溃影响面大 → 必须做好多 Pod 分散和自动恢复
```

### 适合的场景

- 企业级应用，用户为内部员工（非外部攻击者）
- 并发用户数 > 500，资源效率优先
- Skill 由团队编写（非用户上传的任意代码）
- 可接受 Linux 用户级隔离（非 Pod 级隔离）
