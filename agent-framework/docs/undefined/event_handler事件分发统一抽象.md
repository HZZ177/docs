# event_handler 事件分发统一抽象

> 对 `agent_backend/engine/event_handler.py` 进行轻量重构，收拢散乱的事件解析、副作用处理和通道投递逻辑。
> 本文档是「多通道 WS 重构」和「调用链路追踪重构」的前置依赖，三者一起实施。

---

## 一、问题概述

当前 `process_events()` 约 550 行，所有事件处理逻辑以 if-elif 平铺展开，三种关注点完全混杂：

| 关注点 | 问题 |
|--------|------|
| **事件解析** | SubAgent 上下文判断重复 4 处，call_stack 入栈/出栈分散 6 处 |
| **副作用** | `is_debug` 检查散落 12 处，调试数据收集模式（append + for 查找更新）重复 8 次 |
| **事件投递** | 每个分支末尾直接调用 `send_func`，无统一路由，多通道需求下会进一步膨胀 |

多通道 WS 和链路追踪两个需求会在每个分支中继续叠加（通道路由判断、TraceCollector DB 写入、trace_run WS 推送），不重构则不可维护。

---

## 二、重构方案

不引入新架构，仅在 `event_handler.py` 内部通过 **提取函数 + 收拢状态 + 路由表** 完成重构。

### 2.1 `_EventContext` — 状态收拢

将现有 `process_events()` 内的十余个局部变量收拢为一个 dataclass：

```python
@dataclass
class _EventContext:
    trace_id: str
    session_id: str
    # 运行时状态
    active_subagents: dict = field(default_factory=dict)   # checkpoint_ns -> subagent_info
    call_stack: list = field(default_factory=list)          # run_id 栈
    tool_start_times: dict = field(default_factory=dict)    # run_id -> start_time
    final_content: str = ""
    # 统计
    token_count: int = 0
    event_count: int = 0
    tool_call_count: int = 0
    stream_start_time: float = 0.0

    def resolve_subagent(self, event: dict) -> dict:
        """统一判断当前事件是否在 SubAgent 上下文内，返回 subagent 信息字典"""
        ...

    def register_subagent(self, event: dict) -> dict:
        """注册新 SubAgent，返回 subagent 信息"""
        ...

    def unregister_subagent(self, event: dict) -> dict:
        """注销 SubAgent，返回 subagent 信息"""
        ...

    def push_stack(self, run_id: str): ...
    def pop_stack(self, run_id: str): ...
```

### 2.2 `_parse_event()` — 统一解析

将 LangGraph 原始事件解析为 `(action, data)` 元组，SubAgent 上下文判断和 call_stack 管理统一在此处完成：

```python
def _parse_event(event: dict, ctx: _EventContext) -> tuple[str, dict] | None:
    """
    LangGraph 原始事件 → (action, data)
    返回 None 表示跳过该事件
    """
    event_type = event.get("event", "")
    event_name = event.get("name", "")
    run_id = event.get("run_id", "")
    data = event.get("data", {})
    subagent_info = ctx.resolve_subagent(event)

    # on_chat_model_stream → stream
    if event_type == "on_chat_model_stream":
        chunk = data.get("chunk")
        if not (chunk and hasattr(chunk, "content") and chunk.content):
            return None
        ctx.final_content += chunk.content
        ctx.token_count += 1
        return ("stream", {"content": chunk.content, **subagent_info, ...})

    # on_tool_start → tool_start 或 subagent_start
    if event_type == "on_tool_start":
        if event_name == "task":
            info = ctx.register_subagent(event)
            return ("subagent_start", {**info, ...})
        ctx.push_stack(run_id)
        ctx.tool_call_count += 1
        return ("tool_start", {**subagent_info, ...})

    # on_tool_end → tool_end 或 subagent_end
    if event_type == "on_tool_end":
        if event_name == "task":
            info = ctx.unregister_subagent(event)
            return ("subagent_end", {**info, ...})
        ctx.pop_stack(run_id)
        return ("tool_end", {**subagent_info, ...})

    # on_chat_model_start → llm_start
    if event_type == "on_chat_model_start":
        return ("llm_start", {...})

    # on_chat_model_end → llm_end
    if event_type == "on_chat_model_end":
        return ("llm_end", {...})

    # on_chain_start/end → middleware_start/end（仅 Middleware 名称匹配时）
    if event_type == "on_chain_start" and _is_middleware(event_name):
        ctx.push_stack(run_id)
        return ("middleware_start", {...})

    if event_type == "on_chain_end" and _is_middleware(event_name):
        ctx.pop_stack(run_id)
        return ("middleware_end", {...})

    # on_chain_error / on_tool_error → 对应 error 事件
    ...

    return None  # 未识别的事件类型，静默跳过
```

### 2.3 路由表 — 通道分配

```python
_CHAT_ACTIONS = frozenset({
    "stream", "completed", "cancelled", "error",
    "tool_start", "tool_end",
    "subagent_start", "subagent_end", "subagent_error",
})

_DEBUG_ACTIONS = frozenset({
    "debug_context",
    "llm_start", "llm_end",
    "middleware_start", "middleware_end", "middleware_error",
    "memory_recalled",
    "trace_run_start", "trace_run_end",   # 链路追踪新增
})
```

路由规则：
- `action in _CHAT_ACTIONS` → 走 `chat_send`
- `action in _DEBUG_ACTIONS` 且 `debug_send is not None` → 走 `debug_send`
- TraceCollector DB 写入不看通道，由 `emit()` 内部统一调用

### 2.4 `emit()` — 统一发射（闭包）

在 `process_events()` 内部定义，替代现有所有散落的 `await send_func({...})` 调用：

```python
async def emit(action: str, data: dict):
    # ① 通道路由投递
    if action in _CHAT_ACTIONS:
        await chat_send({"action": action, "data": data})
    if action in _DEBUG_ACTIONS and debug_send:
        await debug_send({"action": action, "data": data})

    # ② Trace DB 写入（全量采集，不看 is_debug）
    if trace_collector:
        await trace_collector.on_event(action, data)
```

### 2.5 `process_events()` 主函数瘦身

```python
async def process_events(
    event_stream: AsyncIterator,
    chat_send: Callable,
    debug_send: Callable | None = None,
    trace_collector=None,
    cancel_event: asyncio.Event | None = None,
    debug_context: dict | None = None,
) -> str | None:
    ctx = _EventContext(
        trace_id=get_trace_id() or "",
        session_id=get_session_id() or "",
        stream_start_time=time.time(),
    )

    async def emit(action, data):
        ...  # 如 2.4

    try:
        # 首次 LLM 调用前推送 debug_context
        debug_context_sent = False

        async for event in event_stream:
            if cancel_event and cancel_event.is_set():
                await event_stream.aclose()
                await emit("cancelled", {"session_id": ctx.session_id, "trace_id": ctx.trace_id})
                return None

            ctx.event_count += 1

            # debug_context 推送（首次 on_chat_model_start 时）
            if not debug_context_sent and event.get("event") == "on_chat_model_start" and debug_context:
                debug_context_sent = True
                await emit("debug_context", {**debug_context, "trace_id": ctx.trace_id})

            result = _parse_event(event, ctx)
            if result is None:
                continue

            action, data = result
            await emit(action, data)

    except asyncio.CancelledError:
        await event_stream.aclose()
        await emit("cancelled", {"session_id": ctx.session_id, "trace_id": ctx.trace_id})
        return None
    except Exception as e:
        logger.error(f"[event_handler] 事件处理错误: {e}")
        await emit("error", {"message": str(e), "trace_id": ctx.trace_id})
        return None

    await emit("completed", {"session_id": ctx.session_id, "trace_id": ctx.trace_id})

    logger.info(
        f"[event_handler] 事件流完成 | "
        f"事件数: {ctx.event_count} | token: {ctx.token_count} | "
        f"工具调用: {ctx.tool_call_count} | 耗时: {time.time() - ctx.stream_start_time:.2f}s"
    )
    return ctx.final_content
```

---

## 三、签名变更

### 3.1 process_events

| 参数 | 变更 |
|------|------|
| `send_func` | **移除**，拆分为 `chat_send` + `debug_send` |
| `chat_send` | **新增**，Chat 通道推送函数 |
| `debug_send` | **新增**，Debug 通道推送函数，非调试模式传 None |
| `trace_collector` | **新增**，链路追踪采集服务实例，详见「调用链路追踪重构需求文档」 |
| `is_debug` | **移除**，由 `debug_send is not None` 隐式表达 |
| `debug_context` | 保留不变 |
| 返回值 | 统一返回 `str | None`，不再返回 dict（调试数据改由 TraceCollector 写 DB） |

### 3.2 chat_service 适配

`PersistentSendWrapper` 仅包装 `chat_send`（只有 Chat 通道事件需要持久化）。其 `SKIP_ACTIONS` 列表中 debug/middleware 相关项可移除，因为这些事件不再经过 `chat_send`。

---

## 四、文件组织

不新建子包，在 `agent_backend/engine/event_handler.py` 内部完成：

```
event_handler.py
├── _EventContext           (dataclass)
├── _CHAT_ACTIONS           (frozenset 常量)
├── _DEBUG_ACTIONS          (frozenset 常量)
├── _is_middleware()        (辅助函数)
├── _parse_event()          (统一解析)
├── _serialize_messages()   (保留)
├── _make_json_serializable() (保留)
├── process_events()        (主函数，含 emit 闭包)
├── send_completed()        (保留)
├── send_error()            (保留)
└── send_task_result()      (保留)
```

---

## 五、与其他需求的关系

| 需求文档 | 依赖关系 |
|---------|---------|
| 多通道 WS 重构 | 本文档提供 `chat_send` / `debug_send` 双通道签名，多通道文档定义通道端点和 ConnectionManager |
| 调用链路追踪重构 | 本文档提供 `trace_collector` 参数和 `emit()` 内的统一写入点，追踪文档定义 TraceCollector 服务和数据模型 |
| chat_service | 适配新签名，构建 `chat_send` 和 `debug_send` 两个函数传入 |
