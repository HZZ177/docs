# WebSocket 三通道架构重构 — 需求摘要

> 面向 Coding 平台的需求文档，仅保留需求与约束，不含实现细节。

---

## 一、背景与问题

当前仅有一个 WebSocket 端点，所有信息通过同一通道推送，导致：

| 问题 | 说明 |
|------|------|
| **场景覆盖不全** | 定时任务结果通知、会话状态变更等需要 user 级推送，无法通过仅 session 级的 Chat 通道满足 |
| **Chat 与 Debug 混杂** | 对话流式事件与调试专用事件（debug_context、debug_llm_*、middleware_*）混在同一通道，改 debug 逻辑易波及 chat |
| **前端缺乏实时更新** | Session 列表状态等需手动刷新，无独立事件通道 |

---

## 二、重构范围与边界

**在范围内：**

- agent_backend：新增 Event / Debug 两个 WS 端点，拆分连接管理与事件推送
- admin_backend：新增对应 WS 代理端点
- admin_frontend：WS 连接管理拆为多通道，Store 与页面状态按通道调整

**不在范围内：**

- Chat 通道端点路径与 action 协议保持不变，现有三方与定时任务客户端无需改
- LangSmith 风格 trace/span 仅预留，本期不实现
- Scene 级状态监控仅预留事件类型，本期不实现

---

## 三、三通道职责要求

### 3.1 通道划分

| 维度 | Event 通道 | Chat 通道 | Debug 通道 |
|------|------------|-----------|------------|
| **连接级别** | user 级（一用户一连接） | session 级（一会话一连接） | session 级（仅调试会话） |
| **用途** | 页面级独立事件实时推送 | 对话流式事件推送 | 调试专用事件推送 |
| **连接时机** | 页面加载且 userId 非空 | 创建/恢复会话时 | 调试会话创建后 |
| **断开时机** | 页面关闭 | 关闭会话或页面关闭 | 关闭调试会话或页面关闭 |
| **与 Session 生命周期** | 无关（断连不影响 session） | 绑定（断连触发宽限期） | 无关（断连不影响 session） |

### 3.2 事件归属

**Event 通道事件：**

- `task_result_notification`：定时任务执行完成
- `session_status_changed`：session 被关闭（主动/超时/宽限期到期）
- `system_notice`：系统通知（预留）

**Debug 通道事件（从 Chat 迁出）：**

- `debug_context`、`debug_llm_start`、`debug_llm_end`
- `middleware_start`、`middleware_end`、`middleware_error`
- `memory_recalled`、`debug_snapshot_updated`

**Chat 通道事件（保留）：**

- `stream`、`completed`、`tool_start`、`tool_end`、`subagent_*`、`cancelled`、`error`、`session_created`、`session_closed`、`pong`

---

## 四、Event 通道需求

### 4.1 连接与协议

- **端点**：agent_backend 提供独立 Event 端点；admin_backend 提供代理端点并透传。
- **认证**：客户端发送 `auth`（带 `user_id`），服务端回应 `auth_ok`；支持 `ping`/`pong` 心跳。
- **连接管理**：user 级，同一 user_id 仅保留一个连接，重连时替换旧连接。
- **推送能力**：支持向指定用户推送、以及可选广播；用户不在线时跳过推送（如定时任务通知）。

### 4.2 事件与数据

- `task_result_notification`：需含 task_id、session_id、结果摘要、成功与否等。
- `session_status_changed`：需含 session_id、old_status、new_status。
- `system_notice`：预留 type、message、level。

### 4.3 调用边界

- 定时任务执行完成后，通过统一入口向任务创建者推送（用户不在线则跳过）。
- session 关闭时（主动/超时/宽限期），通过统一入口向该 session 的 user 推送状态变更。

---

## 五、Debug 通道需求

### 5.1 连接与协议

- **端点**：agent_backend 提供独立 Debug 端点；admin_backend 代理端点需与现有 Chat 代理区分（避免路径冲突）。
- **绑定**：客户端发送 `bind_session`（带 `session_id`），服务端回应 `bind_ok`；支持 `ping`/`pong`。
- **连接管理**：session 级，一个调试 session 最多一个 debug 连接。

### 5.2 事件分流

- 所有 debug 相关事件（debug_context、debug_llm_*、middleware_*、memory_recalled、debug_snapshot_updated）**只从 Debug 通道推送**，不再经 Chat 通道。
- 非调试模式（无 Debug 连接）时，debug 事件静默丢弃；调用链与快照等数据仍可正常收集与落库，不依赖推送。

### 5.3 后端行为

- 事件处理逻辑需支持「按事件类型分流」：chat 事件走 Chat 推送（并保持原有持久化），debug 事件走 Debug 推送；Chat 通道不再承载 debug 事件。

---

## 六、Chat 通道（变更说明）

- 端点路径与 action 协议**不变**。
- **移除**经 Chat 推送的 debug 类事件：debug_context、debug_llm_*、middleware_*、memory_recalled。
- 流式对话、工具、子 agent、会话生命周期、错误与取消等事件**保持现状**。
- 对外兼容：非调试场景行为不变；调试场景下前端需改为从 Debug 通道接收上述事件。

---

## 七、定时任务结果推送

- 定时任务执行完成后，除现有结果回写外，需**向任务创建者**推送通知（Event 通道）。
- 用户不在线时可跳过推送；不改变现有定时任务执行与 WS 客户端协议。

---

## 八、Session 生命周期

- **保活与宽限期**：仅与 Chat 通道绑定，逻辑不变；Event/Debug 断连**不**触发宽限期、不影响任何 session。
- **关闭时**：所有关闭场景（主动关闭、保活超时、宽限期到期、服务重启批量关闭）均需在变更 session 状态后，通过 Event 通道向对应用户推送 `session_status_changed`（需在关闭前能解析到 user_id）。

---

## 九、前端（admin_frontend）需求

### 9.1 连接管理

- 拆分为三条独立连接：Event（user 级）、Chat（session 级）、Debug（session 级）。
- 连接时机：页面加载且 userId 非空 → Event；创建/恢复会话 → Chat；调试会话创建后 → Debug 并 bind_session；关闭会话或离开页面时断开对应通道。

### 9.2 状态与数据

- 调试相关数据（调用链、LLM 详情、记忆、快照、debug_context）与 Chat 对话数据分离存储/展示；Event 事件（如 session 状态、定时任务通知）独立处理。
- Session 列表：收到 `session_status_changed` 时更新对应 session 的 status，实现实时状态更新，无需依赖手动刷新。

### 9.3 UI 与体验

- 在配置/调试区域展示各通道连接状态（如 Event / Chat / Debug 已连接/未连接），颜色区分。
- Event 通道断线时，可给出轻量提示（如“事件通道已断开，部分实时更新不可用”）。
- 收到 `task_result_notification` 时，以通知形式（如 message.info）展示任务结果摘要。

---

## 十、与其他模块的边界

- 定时任务结果推送：执行结果由现有定时任务流程产生，通过统一事件入口推送给对应用户（回调或等价机制，避免 common 与 agent_backend 强耦合）。
- 事件分流：由 chat_service 与 event_handler 协作，按事件类型分别投递到 Chat 与 Debug 通道，不改变事件语义与持久化策略（仅改变推送路径）。
- 前端历史与状态：从现有接口与事件流获取；Session 状态以 Event 通道推送为准。

---

*文档版本：需求摘要，基于《WebSocket 三通道架构重构 — 需求文档》提炼，去掉实现细节，便于放入 Coding 平台作为需求依据。*
