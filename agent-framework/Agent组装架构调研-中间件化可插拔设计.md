# Agent 组装架构调研 — 中间件化可插拔设计

> 基于 DeerFlow 源码深度分析，结合本项目现状，探讨 Agent 组装架构的改造方向。

---

## 第一部分：DeerFlow 的中间件化组装架构

### 1.1 架构总览

DeerFlow 的 Agent 组装采用 **工厂函数 + 中间件链 + 配置驱动 + 反射加载** 模式。核心入口是 `make_lead_agent(config: RunnableConfig)`，一个纯工厂函数，组装过程分 7 步：

```
make_lead_agent(RunnableConfig)
  │
  ├─ 1. 解析运行时参数 (thinking_enabled, model_name, is_plan_mode, subagent_enabled)
  ├─ 2. 加载 Agent 配置 (agents/{name}/config.yaml → AgentConfig)
  ├─ 3. 创建 Model (create_chat_model → 反射加载 config.yaml 中的 use 路径)
  ├─ 4. 组装 Tools (get_available_tools → 配置工具 + MCP工具 + 内置工具)
  ├─ 5. 构建 Middleware 链 (_build_middlewares → 11个中间件)
  ├─ 6. 生成 System Prompt (apply_prompt_template → Soul + Memory + Skills + Subagent)
  └─ 7. create_agent(model, tools, middleware, system_prompt, state_schema)
```

最终调用 LangChain 的 `create_agent()`，将 model、tools、middleware、prompt、state_schema 五要素一次性传入，得到一个 LangGraph CompiledStateGraph。

### 1.2 中间件系统的底层机制

#### 1.2.1 中间件是什么

DeerFlow 的中间件不是 HTTP 中间件（洋葱模型），而是 **LangChain `AgentMiddleware` 基类提供的 Agent 执行循环生命周期钩子**。

`AgentMiddleware` 基类定义了 6 个钩子方法：

| 钩子 | 调用时机 | 调用频率 | 返回值 |
|------|---------|---------|--------|
| `before_agent` | Agent 执行开始前 | 每次 invoke 调一次 | `dict` 更新 state，`None` 不动 |
| `before_model` | 每次调 LLM 前 | 循环中可能多次 | 同上 |
| `after_model` | 每次 LLM 返回后 | 循环中可能多次 | 同上 |
| `after_agent` | Agent 执行完成后 | 每次 invoke 调一次 | 同上 |
| `wrap_model_call` | 包装 LLM 调用 | 循环中可能多次 | 可重试、短路、修改请求/响应 |
| `wrap_tool_call` | 包装工具调用 | 每个工具调用一次 | 可拦截、重试、修改 |

#### 1.2.2 执行流程

`create_agent()` 内部将中间件编织为 LangGraph 图节点，形成如下执行流：

```
用户消息进入
    │
    ▼
┌─ before_agent ──────────────────────────────┐  ← 整个Agent启动前，只跑一次
│  所有中间件的 before_agent() 按顺序执行       │
└──────────────────────────────────────────────┘
    │
    ▼
┌─ Agent Loop (可能循环多次) ──────────────────┐
│                                              │
│  ┌─ before_model ─────────────────────────┐  │  ← 每次调LLM前
│  │  所有中间件的 before_model()            │  │
│  └────────────────────────────────────────┘  │
│      │                                       │
│      ▼                                       │
│  ┌─ LLM 调用 ─────────────────────────────┐  │
│  │  被 wrap_model_call 链式包装             │  │
│  └────────────────────────────────────────┘  │
│      │                                       │
│      ▼                                       │
│  ┌─ after_model ──────────────────────────┐  │  ← 每次LLM返回后
│  │  所有中间件的 after_model()              │  │
│  └────────────────────────────────────────┘  │
│      │                                       │
│      ▼                                       │
│   LLM返回了tool_calls？                      │
│      │是                    │否               │
│      ▼                      └──→ 退出循环     │
│  ┌─ 工具执行 ─────────────────────────────┐  │
│  │  每个工具被 wrap_tool_call 链式包装      │  │
│  └────────────────────────────────────────┘  │
│      │                                       │
│      └──→ 回到 before_model，再次调LLM ─────┘
│
└──────────────────────────────────────────────┘
    │
    ▼
┌─ after_agent ───────────────────────────────┐  ← 整个Agent完成后，只跑一次
│  所有中间件的 after_agent() 按顺序执行        │
└──────────────────────────────────────────────┘
    │
    ▼
  返回结果
```

**核心机制**：钩子返回 `dict` 会自动合并进 state，返回 `None` 则不修改 state。这是中间件修改 Agent 状态的唯一方式。

### 1.3 DeerFlow 的 11 个中间件

| # | 中间件 | 必选 | 使用的钩子 | 职责 |
|---|--------|------|-----------|------|
| 1 | ThreadDataMiddleware | 是 | `before_agent` | 创建线程隔离目录（workspace/uploads/outputs） |
| 2 | UploadsMiddleware | 是 | `before_agent` | 扫描上传文件，注入 `<uploaded_files>` 到用户消息 |
| 3 | SandboxMiddleware | 是 | `before_agent` | 获取沙箱环境，sandbox_id 写入 state |
| 4 | DanglingToolCallMiddleware | 是 | `before_agent` | 为中断的工具调用补充占位 ToolMessage，防止 LLM 报错 |
| 5 | SummarizationMiddleware | 条件 | `before_agent` | 上下文接近 token 上限时自动摘要压缩（配置开关） |
| 6 | TodoListMiddleware | 条件 | — | Plan 模式下的任务管理（运行时参数 `is_plan_mode` 控制） |
| 7 | TitleMiddleware | 是 | `after_agent` | 第一轮对话完成后，调 LLM 自动生成对话标题 |
| 8 | MemoryMiddleware | 是 | `after_agent` | 过滤消息→入队→后台 LLM 提取记忆（防抖 30s） |
| 9 | ViewImageMiddleware | 条件 | `before_model` | view_image 工具完成后，在下次调 LLM 前注入 base64 图片 |
| 10 | SubagentLimitMiddleware | 条件 | `after_model` | LLM 返回超限 task 调用时硬截断（比 prompt 约束可靠） |
| 11 | ClarificationMiddleware | 是 | `wrap_tool_call` | 拦截 ask_clarification 工具→中断 Agent→问题推给用户 |

**顺序是硬约束**，代码中有详细注释说明依赖关系：
- ThreadData 必须在 Sandbox 之前（确保 thread_id 可用）
- Uploads 必须在 ThreadData 之后（需要 thread_id）
- Clarification 必须在最后（拦截后中断整个流程）

### 1.4 典型中间件的实现解析

#### MemoryMiddleware — `after_agent` 钩子

**时机**：Agent 整个执行完成后，只跑一次。

```python
class MemoryMiddleware(AgentMiddleware):
    def after_agent(self, state, runtime) -> dict | None:
        messages = state.get("messages", [])
        # 过滤：只保留 user 消息 + 最终 AI 回复（去掉 tool 调用和中间步骤）
        filtered = _filter_messages_for_memory(messages)
        # 入队（30s防抖，后台 LLM 提取记忆）
        queue = get_memory_queue()
        queue.add(thread_id=thread_id, messages=filtered)
        return None  # 不修改 state，异步更新
```

**设计思路**：记忆更新是"旁路"操作，不影响当前对话流。放在 `after_agent` 保证拿到完整对话。

#### SubagentLimitMiddleware — `after_model` 钩子

**时机**：每次 LLM 返回后，循环中可能多次执行。

```python
class SubagentLimitMiddleware(AgentMiddleware):
    def after_model(self, state, runtime) -> dict | None:
        last_msg = state["messages"][-1]
        tool_calls = last_msg.tool_calls
        task_indices = [i for i, tc in enumerate(tool_calls) if tc["name"] == "task"]
        if len(task_indices) <= self.max_concurrent:
            return None  # 没超限，不动
        # 超限：截断多余的 task 调用
        truncated = [tc for i, tc in enumerate(tool_calls)
                     if i not in set(task_indices[self.max_concurrent:])]
        updated_msg = last_msg.model_copy(update={"tool_calls": truncated})
        return {"messages": [updated_msg]}  # 用截断后的消息替换原消息
```

**设计思路**：LLM 不可靠，prompt 说"最多 3 个"它可能输出 5 个。中间件在模型输出后硬截断，比 prompt 约束可靠得多。

#### ClarificationMiddleware — `wrap_tool_call` 拦截器

**时机**：每个工具执行时拦截特定工具。

```python
class ClarificationMiddleware(AgentMiddleware):
    def wrap_tool_call(self, request, handler):
        if request.tool_call["name"] != "ask_clarification":
            return handler(request)  # 不是目标工具，放行
        # 是 ask_clarification → 拦截，不调 handler（工具不真正执行）
        return Command(
            update={"messages": [ToolMessage(content=格式化的问题)]},
            goto=END,  # 直接跳到图的终点，中断 Agent 执行
        )
```

**设计思路**：Agent 想澄清问题时，中间件拦截后中断执行流，把问题推给用户。用户回答后从断点继续。

#### ViewImageMiddleware — `before_model` 钩子

**时机**：每次调 LLM 前，循环中可能多次执行。

```python
class ViewImageMiddleware(AgentMiddleware):
    def before_model(self, state, runtime) -> dict | None:
        if not self._should_inject_image_message(state):
            return None
        viewed_images = state.get("viewed_images", {})
        human_msg = HumanMessage(content=[
            {"type": "text", "text": "Here are the images:"},
            {"type": "image_url", "image_url": {"url": f"data:{mime};base64,{data}"}},
        ])
        return {"messages": [human_msg]}  # 追加到 state 的 messages
```

**设计思路**：Agent 上一轮调了 view_image 工具拿到了图片数据，但 LLM 还没"看到"。在下一次调 LLM 前，中间件把 base64 图片注入消息流。

### 1.5 配置驱动 + 反射加载

DeerFlow 的所有可插拔组件通过 `config.yaml` 的 `use` 字段声明模块路径，运行时反射加载：

```yaml
# Model — 通过 resolve_class() 动态实例化
models:
  - name: gpt-4
    use: langchain_openai:ChatOpenAI

# Tool — 通过 resolve_variable() 动态获取
tools:
  - name: web_search
    use: src.community.tavily.tools:web_search_tool

# Sandbox — 通过 resolve_class() 动态实例化
sandbox:
  use: src.sandbox.local:LocalSandboxProvider
```

反射系统（`src/reflection/resolvers.py`）做的事：
1. `resolve_variable("module.path:variable_name")` → `importlib.import_module()` + `getattr()`
2. `resolve_class("module.path:ClassName", base_class)` → 同上 + `issubclass` 校验
3. 内置缺失依赖提示（如 `uv add langchain-openai`）

### 1.6 工具聚合模式

`get_available_tools()` 从 4 个来源聚合工具：

```
配置工具 (config.yaml tools[])       → resolve_variable() 反射加载
    +
MCP 工具 (extensions_config.json)    → get_cached_mcp_tools() 懒加载 + mtime 缓存失效
    +
内置工具 (present_file, ask_clarification) → 始终包含
    +
条件工具 (view_image, task)          → 运行时参数控制（supports_vision, subagent_enabled）
```

工具还支持 `group` 分组，Agent 可通过 `tool_groups` 选择性加载特定分组。

### 1.7 子代理递归组装

子代理使用 **相同的 `create_agent()` 接口**，但做了精简：

| 维度 | 主 Agent | 子代理 |
|------|---------|--------|
| Model | 可选模型 + thinking | 继承父级或自定义，关闭 thinking |
| Tools | 全量工具 | 白名单/黑名单过滤，禁止 `task` 防递归 |
| Middleware | 11 个完整链 | 只保留 ThreadData + Sandbox（最小集） |
| State | 独立 ThreadState | 复用父级的 sandbox_state 和 thread_data |

---

## 第二部分：本项目改造方向分析

### 2.1 现状诊断

#### 当前组装流程

```
WebSocket → chat_service → assembler.assemble() → Agent执行 → event_handler → 推送
```

`assembler.assemble()` 是一个过程式流水线，做了以下事情：

1. 从数据库加载 Scene → Agent 配置
2. 创建 SubAgent 实例（CompiledSubAgent）
3. 构建 System Prompt（L1 安全 + L2 Agent + L3 画像 + L4 Skill 索引）
4. 解析工具绑定（TODO：返回空列表）
5. 创建 LLM（PatchedChatOpenAI + 缓存）
6. 调 `create_deep_agent()` 创建最终 Agent

#### 关键发现：deepagents 已原生支持 middleware

`deepagents` v0.3.8 的 `create_deep_agent()` 签名：

```python
def create_deep_agent(
    model, tools, *,
    system_prompt,
    middleware: Sequence[AgentMiddleware] = (),  # ← 已支持
    subagents, skills, memory,
    checkpointer, ...
) -> CompiledStateGraph:
```

内部已自带一套中间件栈：

```
deepagents 内置中间件（当前项目已在使用，但未感知）：
  1. TodoListMiddleware          ← 任务管理
  2. MemoryMiddleware            ← 记忆（需传 memory 参数开启）
  3. SkillsMiddleware            ← 技能加载（需传 skills 参数开启）
  4. FilesystemMiddleware        ← 文件操作
  5. SubAgentMiddleware          ← 子代理调度
  6. SummarizationMiddleware     ← 上下文摘要
  7. AnthropicPromptCachingMiddleware
  8. PatchToolCallsMiddleware
  └─ 用户传入的 middleware 追加在末尾 ─→  9, 10, 11...
```

**结论**：不需要换框架，只需在 `create_deep_agent()` 调用时传入自定义 middleware 列表。

### 2.2 核心问题：组装时 vs 运行时

当前 assembler 把两类本质不同的事情混在一起：

| 类型 | 含义 | 示例 | 当前做法 |
|------|------|------|---------|
| **组装时逻辑** | 决定 Agent "长什么样" | 选 LLM、选 Tools、选 SubAgent | assembler 做 ✓ |
| **运行时逻辑** | 决定 Agent "跑起来时的行为" | 画像注入、历史裁剪、Skill 触发、标题生成、记忆更新 | 也写在 assembler/chat_service ✗ |

**问题**：运行时逻辑写在组装阶段，意味着只在 Agent 创建时执行一次。无法响应运行时的状态变化（如画像更新、消息增长、工具调用拦截）。

### 2.3 具体改造点

#### 改造 1：用户画像注入 → `UserProfileMiddleware`

**现状**（assembler.py:48-52）：

组装时从数据库加载画像，拼接到 system_prompt。整个对话期间画像不变。

**改为中间件**：

```python
class UserProfileMiddleware(AgentMiddleware):
    """每次 Agent 执行前，从数据库加载最新用户画像注入到消息流。"""

    def before_agent(self, state, runtime) -> dict | None:
        session_id = runtime.context.get("session_id")
        profile = load_user_profile(session_id)
        if not profile:
            return None
        return {"messages": [SystemMessage(content=f"<user_profile>\n{profile}\n</user_profile>")]}
```

**收益**：每轮对话拿最新画像；画像逻辑从 assembler 解耦。

#### 改造 2：历史消息裁剪 → `HistoryWindowMiddleware`

**现状**（assembler.py:66-71）：

TODO 注释，未实现。

**改为中间件**：

```python
class HistoryWindowMiddleware(AgentMiddleware):
    """每次调 LLM 前，裁剪历史消息到指定窗口大小。"""

    def __init__(self, max_rounds: int = 20):
        self.max_rounds = max_rounds

    def before_model(self, state, runtime) -> dict | None:
        messages = state.get("messages", [])
        if count_rounds(messages) <= self.max_rounds:
            return None
        trimmed = trim_to_rounds(messages, self.max_rounds)
        return {"messages": trimmed}
```

**关键**：放在 `before_model` 而非 `before_agent`，因为 Agent 可能循环多次调 LLM，每次都需检查。

#### 改造 3：Skill 触发检测 → `SkillTriggerMiddleware`

**现状**（chat_service.py:63-66）：

TODO，`_check_skill_trigger()` 返回 None。

**改为中间件**：

```python
class SkillTriggerMiddleware(AgentMiddleware):
    """检测用户消息中的 /command 前缀，匹配并加载 Skill 内容。"""

    def before_agent(self, state, runtime) -> dict | None:
        messages = state.get("messages", [])
        last_msg = messages[-1]
        if not isinstance(last_msg, HumanMessage) or not last_msg.content.startswith("/"):
            return None
        skill_content = match_and_load_skill(last_msg.content, runtime.context)
        if not skill_content:
            return None
        return {"messages": [SystemMessage(content=skill_content)]}
```

#### 改造 4：对话标题生成 → `TitleMiddleware`

**现状**：无自动标题生成。

**改为中间件**：

```python
class TitleMiddleware(AgentMiddleware):
    """第一轮对话完成后，自动生成对话标题并写回数据库。"""

    def after_agent(self, state, runtime) -> dict | None:
        if state.get("title"):
            return None
        messages = state.get("messages", [])
        if not has_complete_exchange(messages):
            return None
        title = generate_title_via_llm(messages)
        update_session_title(runtime.context["session_id"], title)
        return {"title": title}
```

#### 改造 5：记忆更新 → `MemoryUpdateMiddleware`

**现状**（memory/ 目录）：全是 TODO 占位函数。

**改为中间件**：

```python
class MemoryUpdateMiddleware(AgentMiddleware):
    """Agent 执行完成后，异步提取记忆并更新。"""

    def after_agent(self, state, runtime) -> dict | None:
        messages = state.get("messages", [])
        filtered = filter_for_memory(messages)
        memory_queue.add(
            session_id=runtime.context["session_id"],
            messages=filtered,
        )
        return None  # 异步更新，不修改 state
```

### 2.4 改造后的 assembler 结构

```python
async def assemble(scene_id, session_id, checkpointer):
    scene_config = await _load_scene_config(scene_id)

    # ── 组装时逻辑（不变）──
    subagents = [await _create_sub_agent(cfg) for cfg in scene_config["subagents"]]
    system_prompt = context_builder.build(agent_prompt, render_params)
    skill_index = skill_assembler.build_index(skill_bindings)
    if skill_index:
        system_prompt += "\n\n" + skill_index
    tools = await _resolve_tools(...)
    llm = agent_factory.get_or_create_llm(...)

    # ── 运行时逻辑（新增：构建中间件链）──
    middleware = build_middleware_chain(scene_config, session_id)

    # ── 创建 Agent（传入 middleware）──
    return agent_factory.create_deep_agent(
        model=llm,
        tools=tools,
        system_prompt=system_prompt,
        subagents=subagents,
        middleware=middleware,       # ← 新增
        checkpointer=checkpointer,
    )
```

```python
def build_middleware_chain(scene_config, session_id) -> list[AgentMiddleware]:
    """根据 Scene 配置动态构建中间件链。"""
    middlewares = []

    memory_cfg = scene_config.get("memory_config", {})

    # 用户画像（按配置启用）
    if memory_cfg.get("user_profile_enabled"):
        middlewares.append(UserProfileMiddleware())

    # Skill 触发检测
    if scene_config.get("skill_bindings"):
        middlewares.append(SkillTriggerMiddleware())

    # 历史窗口裁剪
    window = scene_config.get("scene", {}).get("history_window_length", 0)
    if window > 0:
        middlewares.append(HistoryWindowMiddleware(max_rounds=window))

    # 标题生成（始终启用）
    middlewares.append(TitleMiddleware())

    # 记忆更新（按配置启用）
    if memory_cfg.get("enabled"):
        middlewares.append(MemoryUpdateMiddleware())

    return middlewares
```

### 2.5 改造路线图

```
Phase 0  验证链路
         agent_factory.create_deep_agent() 加上 middleware=[] 参数
         写一个最简单的中间件（如日志中间件）验证整个钩子机制能跑通

Phase 1  剥离现有逻辑
         把 assembler 中的运行时逻辑逐个迁移为中间件：
         - 用户画像注入 → UserProfileMiddleware
         - 历史消息裁剪 → HistoryWindowMiddleware（当前是 TODO）

Phase 2  实现新功能
         用中间件模式直接实现原先的 TODO 项：
         - Skill 触发检测 → SkillTriggerMiddleware
         - 对话标题生成 → TitleMiddleware
         - 记忆更新 → MemoryUpdateMiddleware
         - Token 监控 → TokenMonitorMiddleware

Phase 3  后续功能全部用中间件
         新需求不再改 assembler，而是新建中间件类：
         - 调试追踪 → TraceMiddleware
         - 敏感词过滤 → ContentFilterMiddleware
         - 限流/降级 → RateLimitMiddleware
         - ...
```

---

## 第三部分：对比总结

### 3.1 架构模式对比

| 维度 | DeerFlow | 本项目现状 |
|------|----------|-----------|
| **组装模式** | 工厂函数 + 中间件链 | `assembler.assemble()` 过程式流水线 |
| **配置来源** | YAML 文件 + 反射加载 | 数据库（Scene → SceneAgent → Agent 三层查询） |
| **中间件** | 11 个，条件启用，统一接口 | 无中间件概念，逻辑写死在 assembler |
| **运行时钩子** | 6 个钩子覆盖 Agent 执行全生命周期 | 无（只有组装阶段逻辑） |
| **工具加载** | 配置声明 + MCP + 内置，反射解析 | TODO（返回空列表） |
| **Prompt 构建** | 模板 + 6 个动态注入块（Soul/Memory/Skills/Subagent） | L1-L4 四层字符串拼接 |
| **记忆系统** | 完整实现：队列 + 防抖 + LLM 提取 + 注入 | TODO 占位函数 |
| **子代理** | 递归组装，精简中间件集 | `create_react_agent` 直接创建 |
| **Model 管理** | 反射加载任意 LangChain Chat 类 | `PatchedChatOpenAI` 硬编码 |
| **状态 Schema** | `ThreadState` 扩展（sandbox/thread_data/artifacts/todos/viewed_images） | 使用 LangGraph 默认 state |
| **扩展性** | 新增功能 = 新 config + 新中间件，零改核心 | 新增功能 = 改 assembler 代码 |

### 3.2 核心设计差异

| 差异点 | DeerFlow 的做法 | 为什么更好 |
|--------|----------------|-----------|
| **组装时 vs 运行时分离** | assembler 只管"用什么造 Agent"，中间件管"跑起来时的行为" | 职责清晰，互不干扰 |
| **横切关注点解耦** | 每个横切关注点（记忆/摘要/标题/图像/限流/拦截）是独立中间件 | 新增功能不碰已有代码 |
| **配置即组装** | `config.yaml` 定义"用什么"，`RunnableConfig` 定义"怎么用" | 同一份代码可组装出完全不同的 Agent |
| **LLM 输出后置校验** | `after_model` 钩子可以修改 LLM 的返回（如截断 tool_calls） | 对 LLM 不可靠输出的工程化兜底 |
| **工具调用拦截** | `wrap_tool_call` 可以短路/重试/修改任意工具调用 | 实现如"澄清中断"等复杂控制流 |

### 3.3 改造可行性评估

| 评估项 | 结论 |
|--------|------|
| **框架支持** | ✅ `deepagents` v0.3.8 的 `create_deep_agent()` 原生支持 `middleware` 参数 |
| **底层兼容** | ✅ deepagents 内部调用 LangChain `create_agent()`，使用相同的 `AgentMiddleware` 基类 |
| **改动范围** | 最小改动：`agent_factory.py` 增加 middleware 参数透传 + `assembler.py` 增加 `build_middleware_chain()` |
| **风险评估** | 低风险：自定义 middleware 追加在 deepagents 内置栈末尾，不影响已有行为 |
| **渐进迁移** | ✅ 可逐个迁移，不需要一次性重构 |

### 3.4 改造核心原则

> **assembler 只管"用什么造 Agent"（组装时），中间件管"Agent 跑起来时的行为"（运行时）。这条线划清楚，后续新功能都不用改 assembler。**
