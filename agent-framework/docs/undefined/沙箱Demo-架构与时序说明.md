# 沙箱 Demo — 架构与时序说明

> **文档定位**：描述当前 `sandbox_demo` + `agent_backend/sandbox` + `sandbox_runtime` 三层 Demo 实现的实际架构与交互时序。
>
> **对应设计文档**：`docs/沙箱方案-多租户Pod-企业级沙箱设计.md`

---

## 一、架构总览

### 1.1 三层架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         浏览器（内嵌 HTML 前端）                              │
│                                                                             │
│   [连接 WS]  [Init]  [Exec: whoami]  [Exec: ls]  [Status]  [Cleanup]       │
│   [自定义命令输入框]                                                         │
│                                                                             │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ WebSocket (ws://localhost:9000/ws)
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│               第一层：sandbox_demo (FastAPI, :9000)                          │
│                                                                             │
│   sandbox_demo/app.py                                                       │
│   ┌───────────────────────────────────────────────────────────────────────┐ │
│   │  GET /           → 返回内嵌 HTML 前端页面                              │ │
│   │  WS  /ws         → WebSocket 端点，解析 action 分发到沙箱层            │ │
│   │                                                                       │ │
│   │  action=init     → sandbox_manager.init_sandbox()                     │ │
│   │  action=exec     → pod_client.exec_cmd()                              │ │
│   │  action=status   → pod_client.status()                                │ │
│   │  action=cleanup  → pod_client.cleanup_user()                          │ │
│   └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│   启动时：初始化内存 SQLite（aiosqlite），建 sandbox_session 表              │
│   Mock 数据：固定 user_id / session_id / scene_id                           │
│                                                                             │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ 调用 agent_backend/sandbox 模块
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│            第二层：agent_backend/sandbox（沙箱管理模块）                      │
│                                                                             │
│   ┌──────────────────┐  ┌──────────────────┐  ┌────────────────────────┐   │
│   │ sandbox_manager   │  │ scheduler         │  │ pod_client             │   │
│   │                  │  │                  │  │                        │   │
│   │ • 获取节点列表   │  │ • 粘滞优先复用   │  │ • GET  /healthz        │   │
│   │   (fixed_list    │  │ • 最少连接数选择 │  │ • GET  /status         │   │
│   │    / kubernetes) │  │ • 满载异常抛出   │  │ • POST /init_user      │   │
│   │ • 拉取 /status   │  │                  │  │ • POST /exec           │   │
│   │ • 查/写粘滞表    │  └──────────────────┘  │ • POST /cleanup_user   │   │
│   │ • 调用 init_user │                        │ • POST /sync_now       │   │
│   │ • on_disconnect  │  ┌──────────────────┐  │                        │   │
│   └──────────────────┘  │ cleanup_task      │  └────────────────────────┘   │
│                         │ • 定时清理过期    │            │                   │
│                         │   粘滞记录       │            │                   │
│                         └──────────────────┘            │                   │
│                                                         │                   │
└─────────────────────────────────────────────────────────┼───────────────────┘
                               │ DB (SQLite 内存)         │ HTTP (Pod :8080)
                               ▼                          ▼
┌────────────────────┐  ┌─────────────────────────────────────────────────────┐
│    数据库           │  │         第三层：sandbox_runtime (Docker, :8080)      │
│                    │  │                                                     │
│ sandbox_session 表 │  │   sandbox_runtime/app.py (FastAPI)                  │
│ ┌────────────────┐ │  │   ┌─────────────────────────────────────────────┐   │
│ │ session_id     │ │  │   │  GET  /healthz     → 探活 + uptime + 用户数 │   │
│ │ pod_id         │ │  │   │  GET  /status      → 负载指标（CPU/MEM/磁盘）│  │
│ │ scene_id       │ │  │   │  POST /init_user   → 创建用户/目录/复制Skill│   │
│ │ user_id        │ │  │   │  POST /exec        → su 切换用户执行命令    │   │
│ │ last_active_at │ │  │   │  POST /cleanup_user→ 同步产物 + 删除目录    │   │
│ └────────────────┘ │  │   │  POST /sync_now    → 立即同步产物到 OSS     │   │
│                    │  │   └─────────────────────────────────────────────┘   │
└────────────────────┘  │                                                     │
                        │   内部模块：                                         │
                        │   ┌───────────────┐ ┌──────────────┐ ┌───────────┐ │
                        │   │ user_manager   │ │skill_updater │ │artifact_  │ │
                        │   │               │ │              │ │syncer     │ │
                        │   │ • useradd     │ │ • 轮询 OSS   │ │ • MD5对比 │ │
                        │   │ • 目录创建    │ │   .version   │ │ • 增量上传│ │
                        │   │ • 复制 skills │ │ • 增量下载   │ │           │ │
                        │   │ • chown/chmod │ │  (待实现)    │ │           │ │
                        │   │ • 会话清理    │ │              │ │           │ │
                        │   └───────────────┘ └──────────────┘ └───────────┘ │
                        │                                                     │
                        │   文件系统：                                         │
                        │   /skills/{scene_id}/         ← 全量 Skill 源       │
                        │   /user_data/{user_id}/{session_id}/                │
                        │       ├── skills/             ← 复制的 Skill 副本   │
                        │       └── outputs/            ← 执行产物            │
                        │                                                     │
                        └──────────────────────┬──────────────────────────────┘
                                               │ OSS SDK / Mock 本地目录
                                               ▼
                        ┌─────────────────────────────────────────────────────┐
                        │                 OSS / Mock 本地目录                   │
                        │                                                     │
                        │   skills/                  ← Skill 脚本源           │
                        │   sessions/{scene}/{user}/{session}/outputs/        │
                        │                            ← 用户产物持久化         │
                        │                                                     │
                        │   Demo 模式下通过 local_path 做本地目录 Mock         │
                        └─────────────────────────────────────────────────────┘
```

### 1.2 组件职责一览

| 组件                  | 位置                                         | 职责                                                                       |
|---------------------|--------------------------------------------|--------------------------------------------------------------------------|
| **sandbox_demo**    | `sandbox_demo/app.py`                      | 独立 WebSocket 演示服务，内嵌前端，Mock 业务数据，驱动沙箱全流程                                 |
| **sandbox_manager** | `agent_backend/sandbox/sandbox_manager.py` | 节点发现、粘滞查询、调度分配、init_user 编排、粘滞写入                                         |
| **scheduler**       | `agent_backend/sandbox/scheduler.py`       | 调度算法：粘滞优先 → 最少连接数 → 满载异常                                                 |
| **pod_client**      | `agent_backend/sandbox/pod_client.py`      | Pod HTTP 客户端，封装 6 个接口调用                                                  |
| **tools**           | `agent_backend/sandbox/tools.py`           | 4 个 LangChain Tool（execute_script / read_file / write_file / list_files） |
| **cleanup_task**    | `agent_backend/sandbox/cleanup_task.py`    | 定时清理超过粘滞期（7 天）的过期会话                                                      |
| **sandbox_runtime** | `sandbox_runtime/app.py`                   | Pod 内 FastAPI 服务，处理用户管理和命令执行                                             |
| **user_manager**    | `sandbox_runtime/user_manager.py`          | Linux 用户创建/删除、目录权限管理、Skill 复制                                            |
| **artifact_syncer** | `sandbox_runtime/artifact_syncer.py`       | 产物 MD5 增量对比同步到 OSS                                                       |
| **skill_updater**   | `sandbox_runtime/skill_updater.py`         | 定时轮询 OSS 检查 Skills 版本更新                                                  |
| **SandboxSession**  | `common/models/sandbox_session.py`         | 粘滞映射表 ORM 模型                                                             |

---

## 二、核心时序图

### 2.1 WebSocket 连接 + Init 沙箱初始化

这是 Demo 的核心流程，从浏览器点击「Init」按钮到容器内用户和目录就绪的完整链路。

```
┌──────┐    ┌───────────┐    ┌─────────────────┐    ┌──────────┐    ┌──────────┐    ┌─────┐
│浏览器 │    │sandbox_demo│   │sandbox_manager  │    │ scheduler │    │Pod:8080  │    │ DB  │
└──┬───┘    └─────┬─────┘    └───────┬─────────┘    └────┬─────┘    └────┬─────┘    └──┬──┘
   │              │                  │                   │               │             │
   │ 1. GET /     │                  │                   │               │             │
   │─────────────>│                  │                   │               │             │
   │  HTML 页面   │                  │                   │               │             │
   │<─────────────│                  │                   │               │             │
   │              │                  │                   │               │             │
   │ 2. WS 连接   │                  │                   │               │             │
   │  /ws         │                  │                   │               │             │
   │─────────────>│                  │                   │               │             │
   │              │ accept()         │                   │               │             │
   │  ready 消息  │                  │                   │               │             │
   │<─────────────│                  │                   │               │             │
   │              │                  │                   │               │             │
   │ 3. {"action":│"init"}           │                   │               │             │
   │─────────────>│                  │                   │               │             │
   │              │                  │                   │               │             │
   │              │ 4. init_sandbox( │                   │               │             │
   │              │    user_id,      │                   │               │             │
   │              │    session_id,   │                   │               │             │
   │              │    scene_id)     │                   │               │             │
   │              │─────────────────>│                   │               │             │
   │              │                  │                   │               │             │
   │              │                  │ 5. 获取节点列表    │               │             │
   │              │                  │  _get_pod_nodes   │               │             │
   │              │                  │  _with_status()   │               │             │
   │              │                  │                   │               │             │
   │              │                  │ 5a. 解析           │               │             │
   │              │                  │ SANDBOX_POD_URLS  │               │             │
   │              │                  │ (fixed_list 模式) │               │             │
   │              │                  │                   │               │             │
   │              │                  │ 5b. GET /status ──────────────────>│             │
   │              │                  │    (拉取负载)     │               │             │
   │              │                  │<──── {pod_id,  ───────────────────│             │
   │              │                  │       user_count} │               │             │
   │              │                  │                   │               │             │
   │              │                  │ 6. 查粘滞表       │               │             │
   │              │                  │  _get_sticky      │               │             │
   │              │                  │  _pod_id()        │               │             │
   │              │                  │──────────────────────────────────────────────── >│
   │              │                  │<──────────────────────────────────────────────── │
   │              │                  │  (None 或 pod_id) │               │             │
   │              │                  │                   │               │             │
   │              │                  │ 7. 调度选择 Pod    │               │             │
   │              │                  │──────────────────>│               │             │
   │              │                  │                   │ 粘滞优先      │             │
   │              │                  │                   │ → 最少连接数  │             │
   │              │                  │  选中 node        │               │             │
   │              │                  │<──────────────────│               │             │
   │              │                  │                   │               │             │
   │              │                  │ 8. POST /init_user ──────────────>│             │
   │              │                  │    {user_id,      │               │             │
   │              │                  │     session_id,   │               │ 8a. useradd │
   │              │                  │     scene_id,     │               │  (Linux 用户)│
   │              │                  │     oss_config}   │               │             │
   │              │                  │                   │               │ 8b. mkdir   │
   │              │                  │                   │               │  skills/    │
   │              │                  │                   │               │  outputs/   │
   │              │                  │                   │               │             │
   │              │                  │                   │               │ 8c. 复制    │
   │              │                  │                   │               │ /skills/    │
   │              │                  │                   │               │ {scene_id}/ │
   │              │                  │                   │               │ → 用户skills│
   │              │                  │                   │               │             │
   │              │                  │                   │               │ 8d. 下载 OSS│
   │              │                  │                   │               │ 产物(或mock)│
   │              │                  │                   │               │             │
   │              │                  │                   │               │ 8e. chown + │
   │              │                  │                   │               │ chmod 700   │
   │              │                  │                   │               │             │
   │              │                  │  {status:ok,      │               │             │
   │              │                  │   workdir,        │               │             │
   │              │                  │   skills_copied}  │               │             │
   │              │                  │<─────────────────────────────────-│             │
   │              │                  │                   │               │             │
   │              │                  │ 9. 写入粘滞表     │               │             │
   │              │                  │  _upsert_sticky() │               │             │
   │              │                  │──────────────────────────────────────────────── >│
   │              │                  │<──────────────────────────────────────────────── │
   │              │                  │                   │               │             │
   │              │  sandbox_info    │                   │               │             │
   │              │  {pod_id,        │                   │               │             │
   │              │   base_url,      │                   │               │             │
   │              │   workdir, ...}  │                   │               │             │
   │              │<─────────────────│                   │               │             │
   │              │                  │                   │               │             │
   │ init_result  │                  │                   │               │             │
   │ + sandbox_info                  │                   │               │             │
   │<─────────────│                  │                   │               │             │
   │              │                  │                   │               │             │
```

### 2.2 Exec 命令执行

用户点击执行按钮或输入自定义命令，经 WebSocket → pod_client → Pod `/exec` → `su` 切换用户执行。

```
┌──────┐    ┌───────────┐    ┌──────────┐    ┌──────────────────────────────┐
│浏览器 │    │sandbox_demo│   │pod_client │    │         Pod :8080             │
└──┬───┘    └─────┬─────┘    └────┬─────┘    │  app.py    user_manager      │
   │              │               │          └──────┬───────────────────────┘
   │              │               │                 │
   │ 1. {"action":│"exec",        │                 │
   │     "command":│"whoami"}      │                 │
   │─────────────>│               │                 │
   │              │               │                 │
   │              │ 2. 从          │                 │
   │              │ sandbox_info   │                 │
   │              │ 取 base_url,   │                 │
   │              │ workdir        │                 │
   │              │               │                 │
   │              │ 3. exec_cmd(  │                 │
   │              │    base_url,  │                 │
   │              │    user_id,   │                 │
   │              │    session_id,│                 │
   │              │    command,   │                 │
   │              │    workdir)   │                 │
   │              │──────────────>│                 │
   │              │               │                 │
   │              │               │ 4. POST /exec   │
   │              │               │ {user_id,       │
   │              │               │  session_id,    │
   │              │               │  command,       │
   │              │               │  timeout,       │
   │              │               │  workdir}       │
   │              │               │────────────────>│
   │              │               │                 │
   │              │               │                 │ 5. _run_as_user():
   │              │               │                 │    su -s /bin/bash
   │              │               │                 │       -  user_{user_id}
   │              │               │                 │       -c "cd {workdir}
   │              │               │                 │           && {command}"
   │              │               │                 │
   │              │               │                 │    [非 Linux 降级]
   │              │               │                 │    sh -c "{command}"
   │              │               │                 │
   │              │               │  {stdout,       │
   │              │               │   stderr,       │
   │              │               │   exit_code,    │
   │              │               │   execution_    │
   │              │               │   time_ms}      │
   │              │               │<────────────────│
   │              │               │                 │
   │              │  exec result  │                 │
   │              │<──────────────│                 │
   │              │               │                 │
   │ exec_result  │               │                 │
   │ {stdout,     │               │                 │
   │  stderr,     │               │                 │
   │  exit_code}  │               │                 │
   │<─────────────│               │                 │
   │              │               │                 │
```

### 2.3 Status 查询

```
┌──────┐    ┌───────────┐    ┌──────────┐    ┌──────────┐
│浏览器 │    │sandbox_demo│   │pod_client │    │Pod :8080 │
└──┬───┘    └─────┬─────┘    └────┬─────┘    └────┬─────┘
   │              │               │               │
   │ 1. {"action":│"status"}      │               │
   │─────────────>│               │               │
   │              │               │               │
   │              │ 2. status(    │               │
   │              │    base_url)  │               │
   │              │──────────────>│               │
   │              │               │ 3. GET /status│
   │              │               │──────────────>│
   │              │               │               │ psutil 采集
   │              │               │               │ CPU/MEM/磁盘
   │              │               │               │ 统计用户数
   │              │               │  {pod_id,     │
   │              │               │   user_count, │
   │              │               │   active_     │
   │              │               │   sessions,   │
   │              │               │   cpu_usage,  │
   │              │               │   memory_mb,  │
   │              │               │   disk_mb}    │
   │              │               │<──────────────│
   │              │  status       │               │
   │              │<──────────────│               │
   │ status_result│               │               │
   │<─────────────│               │               │
   │              │               │               │
```

### 2.4 Cleanup 清理

```
┌──────┐    ┌───────────┐    ┌──────────┐    ┌──────────────────────────────┐
│浏览器 │    │sandbox_demo│   │pod_client │    │         Pod :8080             │
└──┬───┘    └─────┬─────┘    └────┬─────┘    │  app.py    user_manager      │
   │              │               │          └──────┬───────────────────────┘
   │              │               │                 │
   │ 1. {"action":│"cleanup"}     │                 │
   │─────────────>│               │                 │
   │              │               │                 │
   │              │ 2. cleanup_   │                 │
   │              │    user(      │                 │
   │              │    base_url,  │                 │
   │              │    user_id,   │                 │
   │              │    session_id,│                 │
   │              │    sync=false)│                 │
   │              │──────────────>│                 │
   │              │               │                 │
   │              │               │ 3. POST         │
   │              │               │ /cleanup_user   │
   │              │               │────────────────>│
   │              │               │                 │
   │              │               │                 │ 4. 清除内存状态
   │              │               │                 │    _user_sessions
   │              │               │                 │    _session_oss_config
   │              │               │                 │
   │              │               │                 │ 5. cleanup_session():
   │              │               │                 │    rm -rf
   │              │               │                 │    user_data/{user_id}
   │              │               │                 │              /{session_id}
   │              │               │                 │
   │              │               │                 │ 6. [无其他 session]
   │              │               │                 │    userdel user_{user_id}
   │              │               │                 │
   │              │               │  {status: ok,   │
   │              │               │   synced_files, │
   │              │               │   cleaned_size} │
   │              │               │<────────────────│
   │              │               │                 │
   │              │  cleanup ok   │                 │
   │              │<──────────────│                 │
   │              │               │                 │
   │cleanup_result│               │                 │
   │ sandbox_info │               │                 │
   │ = None       │               │                 │
   │<─────────────│               │                 │
   │              │               │                 │
```

---

## 三、调度算法流程

sandbox_manager 调用 scheduler.select_pod 的内部决策逻辑：

```
                    开始：为 session 分配 Pod
                              │
                              ▼
                   ┌─────────────────────┐
                   │  pod_nodes 列表     │
                   │  是否为空？          │
                   └──────────┬──────────┘
                         是 / │ \ 否
                           /  │  \
                          ▼   │   ▼
              SandboxNo-  │   │  ┌──────────────────┐
              CapacityError│   │  │ 有 sticky_pod_id │
                          │   │  │ 且在 pod_nodes   │
                          │   │  │ 中存在？          │
                          │   │  └────────┬─────────┘
                          │   │      是 / │ \ 否
                          │   │        /  │  \
                          │   │       ▼   │   ▼
                          │   │  ┌────────────────┐  ┌───────────────────────┐
                          │   │  │粘滞 Pod 的     │  │ 从 pod_nodes 中筛选   │
                          │   │  │user_count      │  │ user_count <          │
                          │   │  │< max_users?    │  │ max_users 的节点      │
                          │   │  └───────┬────────┘  └───────────┬───────────┘
                          │   │     是 / │ \ 否                  │
                          │   │       /  │  \                    ▼
                          │   │      ▼   │   ▼         ┌──────────────────┐
                          │   │  返回    │  走最少       │ 有可用节点？     │
                          │   │  粘滞Pod │  连接数       └────────┬─────────┘
                          │   │          │  ────────>        是 / │ \ 否
                          │   │          │                    /   │  \
                          │   │          │                   ▼    │   ▼
                          │   │          │             选 user-   │  SandboxNo-
                          │   │          │             count      │  CapacityError
                          │   │          │             最小的 Pod │
                          │   │          │             返回       │
```

---

## 四、Pod 内部 /init_user 详细流程

```
POST /init_user 请求到达
         │
         ▼
┌─────────────────────────────────┐
│ 1. 注册到内存状态                │
│    _user_sessions.add(          │
│      (user_id, session_id))     │
│    _session_oss_config[...] =   │
│      oss_config                 │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│ 2. ensure_user_and_dirs()       │
│    (user_manager)               │
│                                 │
│    ┌───────────────────────┐    │
│    │ 2a. useradd           │    │
│    │   -M (不创建 home)    │    │
│    │   -d /user_data/{uid} │    │
│    │   -s /usr/sbin/nologin│    │
│    │   user_{user_id}      │    │
│    │   (已存在则跳过)      │    │
│    └───────────┬───────────┘    │
│                │                │
│    ┌───────────▼───────────┐    │
│    │ 2b. 创建目录          │    │
│    │   /user_data/         │    │
│    │     {uid}/{sid}/      │    │
│    │       skills/         │    │
│    │       outputs/        │    │
│    └───────────┬───────────┘    │
│                │                │
│    ┌───────────▼───────────┐    │
│    │ 2c. 复制 Skills       │    │
│    │   /skills/{scene_id}/ │    │
│    │   → skills/           │    │
│    └───────────┬───────────┘    │
│                │                │
│    ┌───────────▼───────────┐    │
│    │ 2d. chown + chmod     │    │
│    │   chown -R user:user  │    │
│    │   chmod 700           │    │
│    └───────────────────────┘    │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│ 3. 下载 OSS 产物到 outputs/     │
│    _download_oss_to_outputs()   │
│                                 │
│    [oss_config 含 local_path]   │
│    → 本地目录复制（Mock 模式）   │
│                                 │
│    [oss_config 含 endpoint+     │
│     bucket+prefix+key+secret]   │
│    → oss2 SDK 下载              │
│                                 │
│    [都没有]                      │
│    → 跳过，返回 0               │
└────────────────┬────────────────┘
                 │
                 ▼
            返回响应
         {status, workdir,
          linux_user,
          skills_copied,
          artifacts_downloaded}
```

---

## 五、数据模型

### sandbox_session 表

```
┌─────────────────────────────────────────────────────────────┐
│                    sandbox_session                           │
├──────────────────┬──────────┬───────────────────────────────┤
│ 字段              │ 类型     │ 说明                          │
├──────────────────┼──────────┼───────────────────────────────┤
│ id (PK)          │ String32 │ UUID 主键                     │
│ session_id (UDX) │ String64 │ 对话会话 ID                   │
│ pod_id (IDX)     │ String64 │ 沙箱节点标识                  │
│ scene_id         │ String32 │ 关联 Scene                    │
│ user_id          │ String64 │ 用户 ID                       │
│ last_active_at   │ DateTime │ 最近活跃时间（7 天过期判断）    │
│ created_at       │ DateTime │ 创建时间（继承自 BaseModel）   │
│ updated_at       │ DateTime │ 更新时间（继承自 BaseModel）   │
│ is_deleted       │ Boolean  │ 软删除标记（继承自 BaseModel） │
└──────────────────┴──────────┴───────────────────────────────┘

索引：
  udx_sandbox_session_session_id     — session_id 唯一索引
  idx_sandbox_session_pod_id         — pod_id 普通索引
  idx_sandbox_session_last_active_at — last_active_at 普通索引（清理查询）
```

---

## 六、Demo 运行拓扑

```
┌─ 本机 ──────────────────────────────────────────────────────────────────┐
│                                                                         │
│   ┌─ 终端 1 ─────────────────────────┐                                  │
│   │  Docker 容器: kt-sandbox-runtime  │                                  │
│   │  端口: 8080                       │                                  │
│   │                                   │                                  │
│   │  挂载:                            │                                  │
│   │   local_skills → /skills/         │                                  │
│   │     /demo_scene_001               │                                  │
│   │   local_artifacts → (可选)        │                                  │
│   └───────────────────────────────────┘                                  │
│                    ▲                                                     │
│                    │ HTTP :8080                                          │
│                    │                                                     │
│   ┌─ 终端 2 ─────────────────────────┐                                  │
│   │  sandbox_demo (Python)            │                                  │
│   │  端口: 9000                       │                                  │
│   │                                   │                                  │
│   │  环境变量:                         │                                  │
│   │   SANDBOX_POD_URLS=               │                                  │
│   │     http://localhost:8080         │                                  │
│   │   SANDBOX_NODE_SOURCE=            │                                  │
│   │     fixed_list                    │                                  │
│   └───────────────────────────────────┘                                  │
│                    ▲                                                     │
│                    │ HTTP :9000 + WS                                     │
│                    │                                                     │
│   ┌─ 浏览器 ─────────────────────────┐                                  │
│   │  http://localhost:9000            │                                  │
│   │  WebSocket → ws://localhost:9000  │                                  │
│   └───────────────────────────────────┘                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

启动步骤：

```bash
# 终端 1：启动 sandbox_runtime Docker 容器
docker build -t kt-sandbox-runtime:latest ./sandbox_runtime
docker run --rm -p 8080:8080 \
  -v ./sandbox_runtime/local_skills:/skills/demo_scene_001 \
  -e POD_ID=sandbox-dev-1 \
  kt-sandbox-runtime:latest

# 终端 2：启动 sandbox_demo
.venv\Scripts\activate
pip install aiosqlite
set SANDBOX_POD_URLS=http://localhost:8080
python -m sandbox_demo
```
