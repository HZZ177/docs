# 替换 create_deep_agent 为 create_agent — 需求文档

> 将主 Agent 创建方式从 `deepagents.create_deep_agent` 替换为 `langchain.agents.create_agent` +
> 按需引入中间件，彻底解决内置工具栈和系统提示词污染问题。

---

## 一、背景与问题

### 1.1 现状

当前主 Agent 通过 `deepagents==0.3.8` 的 `create_deep_agent()` 创建。该函数内部硬编码了 **8 个中间件**，最终调用
`langchain.agents.create_agent()` 构建 Agent。

调用链路：

```
assembler.assemble()
  → agent_factory.create_deep_agent()
    → deepagents.create_deep_agent()    # 硬编码 8 个中间件
      → langchain.agents.create_agent() # 实际构建 Agent
```

### 1.2 问题

`create_deep_agent` 强制注入的中间件栈带来三类问题：

**问题一：不需要的工具注入（9 个）**

| 来源中间件                  | 注入的工具                                                                   | 影响                                  |
|------------------------|-------------------------------------------------------------------------|-------------------------------------|
| `TodoListMiddleware`   | `write_todos`                                                           | 污染工具列表                              |
| `FilesystemMiddleware` | `ls`, `read_file`, `write_file`, `edit_file`, `glob`, `grep`, `execute` | 污染工具列表，引入虚拟文件系统层                    |
| `SubAgentMiddleware`   | `task`（含通用子 agent）                                                      | 通用子 agent 与我们的 `run_sub_agent` 功能重复 |

当前通过 `ToolFilterMiddleware` 黑名单屏蔽了其中 8 个工具，但只是"注入后再过滤"，中间件本身仍在运行。

**问题二：无法删除的 system_prompt 注入**

每个中间件通过 `wrap_model_call` 的 `request.override(system_message=...)` 向每次 LLM 调用追加提示词，包括：

- `FilesystemMiddleware`：文件系统操作说明
- `SubAgentMiddleware`：子代理调度说明（约 250 行）
- `TodoListMiddleware`：任务管理说明
- `BASE_AGENT_PROMPT`：通用工具使用提示

这些提示词**无法通过 ToolFilterMiddleware 屏蔽**，每轮对话都会消耗额外 token 并干扰 Agent 行为。

**问题三：不需要的运行时开销**

- `FilesystemMiddleware`：维护 StateBackend 虚拟文件系统，执行大文件驱逐逻辑
- `SummarizationMiddleware`：在不受我们控制的阈值下触发摘要
- `AnthropicPromptCachingMiddleware`：针对 Anthropic 模型优化，我们使用 OpenAI 兼容接口无效
- `MemoryMiddleware` / `SkillsMiddleware`：从 AGENTS.md / SKILL.md 加载（我们不使用）

---

## 二、方案概述

### 2.1 核心思路

绕过 `create_deep_agent`，直接调用 `langchain.agents.create_agent`，自行组装精简的中间件栈。

### 2.2 可行性依据

1. `create_deep_agent` 最终调用的就是 `create_agent`（`deepagents/graph.py:239`），返回类型均为 `CompiledStateGraph`
2. `create_agent` 原生支持 `middleware`、`checkpointer`、`store` 参数，接口完全兼容
3. `astream_events(version="v2")` 调用方式不变，event_handler 无需修改
4. deepagents 的所有中间件均实现 langchain 的 `AgentMiddleware` 接口，可单独 import 使用

### 2.3 中间件取舍决策

| 中间件                                | 来源                  | 决策            | 理由                                                                         |
|------------------------------------|---------------------|---------------|----------------------------------------------------------------------------|
| `SubAgentMiddleware`               | deepagents          | **保留，调整参数**   | 提供 `task` 工具实现配置子 agent 调度；设置 `general_purpose_agent=False` 关闭多余的通用子 agent |
| `PatchToolCallsMiddleware`         | deepagents          | **保留**        | 修补悬空工具调用（AI 发出 tool_call 但无对应 ToolMessage 时自动补充取消消息），提升稳定性                 |
| `SummarizationMiddleware`          | deepagents          | **暂不引入，后续按需** | 当前对话长度可控，且该中间件依赖 Backend 体系；如需引入需要单独配置触发阈值                                 |
| `TodoListMiddleware`               | langchain           | **移除**        | 不需要 todo 功能                                                                |
| `FilesystemMiddleware`             | deepagents          | **移除**        | 不需要文件操作工具，我们的工具通过管理后台配置 + MCP 动态加载                                         |
| `MemoryMiddleware`                 | deepagents          | **移除**        | 我们有自己的 `UserProfileMiddleware` + `MemoryUpdateMiddleware`                  |
| `SkillsMiddleware`                 | deepagents          | **移除**        | 我们有自己的 `skill_assembler`                                                   |
| `AnthropicPromptCachingMiddleware` | langchain_anthropic | **移除**        | 使用 OpenAI 兼容接口，此中间件无效                                                      |

### 2.4 替换后的中间件栈

```
替换前（create_deep_agent 硬编码 + 我们追加）：
┌──────────────────────────────────────────────┐
│ 1. TodoListMiddleware          ← 不需要       │
│ 2. MemoryMiddleware            ← 不需要       │  deepagents
│ 3. SkillsMiddleware            ← 不需要       │  内置（不可控）
│ 4. FilesystemMiddleware        ← 不需要       │
│ 5. SubAgentMiddleware          ← 需要但参数不对 │
│ 6. SummarizationMiddleware     ← 暂不需要     │
│ 7. AnthropicPromptCaching      ← 不需要       │
│ 8. PatchToolCallsMiddleware    ← 需要         │
├──────────────────────────────────────────────┤
│ 9.  ToolFilterMiddleware       ← 为了屏蔽上面  │  我们追加
│ 10. ToolCallLimitMiddleware    ← 需要         │  （条件启用）
│ 11. UserProfileMiddleware      ← 需要         │
│ 12. MemoryUpdateMiddleware     ← 需要         │
└──────────────────────────────────────────────┘

替换后（全部由我们控制）：
┌──────────────────────────────────────────────┐
│ 1. SubAgentMiddleware          ✅ 精简配置     │  从 deepagents 单独引入
│ 2. PatchToolCallsMiddleware    ✅ 稳定性保障   │  从 deepagents 单独引入
├──────────────────────────────────────────────┤
│ 3. ToolCallLimitMiddleware     ✅ 条件启用     │  我们的自定义中间件
│ 4. UserProfileMiddleware       ✅ 条件启用     │
│ 5. MemoryUpdateMiddleware      ✅ 条件启用     │
└──────────────────────────────────────────────┘
```

注意：`ToolFilterMiddleware` **不再需要**，因为不再有需要屏蔽的内置工具。

---

## 三、详细设计

### 3.1 改造 agent_factory.py

**改动点**：`create_deep_agent` 方法内部改为调用 `langchain.agents.create_agent`

```python
# agent_backend/engine/agent_factory.py

@staticmethod
def create_deep_agent(
        model: ChatOpenAI,
        tools: List[Any],
        system_prompt: str,
        subagents: List[Any],
        name: str = "main_agent",
        checkpointer: Any = None,
        middleware: Any = (),
) -> Any:
    """创建主 Agent（使用 create_agent + 精简中间件栈）。"""
    from langchain.agents import create_agent as _create_agent
    from deepagents.middleware.subagents import SubAgentMiddleware
    from deepagents.middleware.patch_tool_calls import PatchToolCallsMiddleware

    # 构建精简的中间件栈
    agent_middleware = []

    # 1. SubAgentMiddleware：配置子 agent 调度
    if subagents:
        agent_middleware.append(
            SubAgentMiddleware(
                default_model=model,
                default_tools=tools,
                subagents=subagents,
                default_middleware=[PatchToolCallsMiddleware()],
                general_purpose_agent=False,  # 关闭通用子 agent
            )
        )

    # 2. PatchToolCallsMiddleware：修补悬空工具调用
    agent_middleware.append(PatchToolCallsMiddleware())

    # 3. 追加自定义中间件（ToolCallLimit / UserProfile / MemoryUpdate 等）
    if middleware:
        agent_middleware.extend(middleware)

    agent = _create_agent(
        model=model,
        tools=tools,
        system_prompt=system_prompt,
        middleware=agent_middleware,
        checkpointer=checkpointer,
        name=name,
    )
    return agent
```

**说明**：

- 方法签名保持不变，assembler 调用处无需修改
- `SubAgentMiddleware` 的 `default_middleware` 参数控制子 agent 执行时的中间件栈，只保留 `PatchToolCallsMiddleware`
- `general_purpose_agent=False` 关闭 deepagents 自带的通用子 agent（我们有自己的 `run_sub_agent` 工具）

### 3.2 改造 chain_builder.py

**改动点**：移除 `ToolFilterMiddleware` 的注入

```python
# agent_backend/middleware/chain_builder.py

def build_middleware_chain(scene_config: Any) -> List[AgentMiddleware]:
    middlewares: List[AgentMiddleware] = []
    scene = scene_config.scene
    memory_cfg = scene_config.memory_config or {}

    # ToolFilterMiddleware 已移除 — 不再有需要屏蔽的内置工具

    # 1. 工具调用次数限制
    max_iterations = scene.max_iterations or 0
    if max_iterations > 0:
        from langchain.agents.middleware import ToolCallLimitMiddleware
        middlewares.append(
            ToolCallLimitMiddleware(run_limit=max_iterations, exit_behavior="continue")
        )

    # 2. 用户画像注入
    if memory_cfg.get("user_profile_enabled", False):
        from agent_backend.middleware.user_profile import UserProfileMiddleware
        middlewares.append(UserProfileMiddleware(memory_config=memory_cfg))

    # 3. 记忆更新
    if memory_cfg.get("memory_enabled", False):
        from agent_backend.middleware.memory_update import MemoryUpdateMiddleware
        middlewares.append(MemoryUpdateMiddleware(memory_config=memory_cfg))

    return middlewares
```

### 3.3 assembler.py — 无需修改

`assembler.py` 的代码（`assemble` 函数、`_create_sub_agent` 函数）**不需要任何修改**，因为：

- `agent_factory.create_deep_agent` 的方法签名保持不变
- `from deepagents import CompiledSubAgent` 导入保持不变
- 工具解析、Prompt 构建、中间件链构建逻辑不受影响

### 3.4 删除 tool_filter.py

`agent_backend/middleware/tool_filter.py` 整个文件可以删除，不再有需要屏蔽的内置工具。

同时清理 `chain_builder.py` 中对它的导入：

```python
# 删除这行
from agent_backend.middleware.tool_filter import ToolFilterMiddleware
```

---

## 四、影响范围

### 4.1 需要修改的文件

| 文件                                          | 改动类型 | 改动内容                                             |
|---------------------------------------------|------|--------------------------------------------------|
| `agent_backend/engine/agent_factory.py`     | 修改   | `create_deep_agent` 内部改用 `create_agent` + 精简中间件栈 |
| `agent_backend/middleware/chain_builder.py` | 修改   | 移除 `ToolFilterMiddleware` 注入，更新模块注释              |
| `agent_backend/middleware/tool_filter.py`   | 删除   | 不再需要                                             |

### 4.2 不需要修改的文件

| 文件                                          | 理由                                                                           |
|---------------------------------------------|------------------------------------------------------------------------------|
| `agent_backend/engine/assembler.py`         | 不变，`create_deep_agent` 方法签名未变，`CompiledSubAgent` 导入不变                        |
| `agent_backend/engine/event_handler.py`     | `task` 工具名不变（SubAgentMiddleware 保留），SubAgent 识别逻辑（`tool_name == "task"`）无需改动 |
| `agent_backend/services/chat_service.py`    | `astream_events()` 调用方式不变，`run_config` 配置不变                                  |
| `agent_backend/tools/run_sub_agent.py`      | 独立工具，不受中间件栈变化影响                                                              |
| `agent_backend/middleware/user_profile.py`  | 自定义中间件，仍通过 chain_builder 注入                                                  |
| `agent_backend/middleware/memory_update.py` | 自定义中间件，仍通过 chain_builder 注入                                                  |
| `admin_frontend/*`                          | WebSocket 推送事件格式不变，前端无需改动                                                    |

### 4.3 依赖变化

- `deepagents==0.3.8` **仍需保留**（使用 `SubAgentMiddleware`、`CompiledSubAgent`、`PatchToolCallsMiddleware`）
- 如果后续将 SubAgent 调度也自行实现，可完全移除 `deepagents` 依赖

---

## 五、替换后的收益

| 维度                       | 替换前                                                 | 替换后                                        |
|--------------------------|-----------------------------------------------------|--------------------------------------------|
| **工具列表**                 | 9 个内置工具被注入后再通过黑名单屏蔽                                 | 只有业务配置的工具，无多余工具                            |
| **system_prompt**        | 被追加约 300 行内置提示词（文件操作、子代理调度、todo 管理等）                | 干净的业务提示词，无多余内容                             |
| **每轮 token 消耗**          | 内置提示词 + 9 个工具 schema 消耗额外 token                     | 节省约 2000-3000 token/轮                      |
| **运行时开销**                | 8 个中间件全部运行（含虚拟文件系统、大文件驱逐等）                          | 只运行 2-5 个必要中间件                             |
| **通用子 agent**            | SubAgentMiddleware 自带通用子 agent，与 run_sub_agent 功能重复 | `general_purpose_agent=False`，只保留配置子 agent |
| **ToolFilterMiddleware** | 需要维护黑名单列表，deepagents 升级可能引入新工具需更新                   | 不再需要，从源头解决                                 |
| **可控性**                  | 中间件栈由 deepagents 硬编码，无法调整顺序或参数                      | 完全自主控制                                     |

---

## 六、风险与注意事项

### 6.1 SubAgentMiddleware 的 default_middleware

`SubAgentMiddleware` 的 `default_middleware` 参数决定了**子 agent 执行时的中间件栈**。`create_deep_agent` 原来给子 agent
配了 6 个中间件：

```
子 agent 原中间件栈：
1. TodoListMiddleware       ← 不需要
2. SkillsMiddleware         ← 不需要
3. FilesystemMiddleware     ← 不需要
4. SummarizationMiddleware  ← 暂不需要
5. AnthropicPromptCaching   ← 不需要
6. PatchToolCallsMiddleware ← 需要
```

替换后只保留 `PatchToolCallsMiddleware`。如果子 agent 的对话较长需要摘要，后续可按需加入 `SummarizationMiddleware`。

### 6.2 recursion_limit

`create_deep_agent` 源码末尾有 `.with_config({"recursion_limit": 1000})`。替换后此配置丢失，但 `chat_service.py` 中已通过
`run_config["recursion_limit"]` 动态设置（根据 `max_iterations` 计算），**不受影响**。

### 6.3 BASE_AGENT_PROMPT

`create_deep_agent` 会在 system_prompt 末尾追加：

```
"In order to complete the objective that the user asks of you, you have access to a number of standard tools."
```

替换后不再追加。如果 Agent 行为出现异常（不主动调用工具），可在我们自己的 system_prompt 中添加类似引导语。

### 6.4 回归测试要点

| 测试场景      | 验证内容                                                |
|-----------|-----------------------------------------------------|
| 基础对话      | 流式响应正常，token 统计正确                                   |
| 工具调用      | 配置的内置工具和 MCP 工具正常调用，工具列表中无多余工具                      |
| 配置子 agent | `task` 工具正常工作，SubAgent 启动/结束事件正确推送                  |
| 动态子 agent | `run_sub_agent` 工具正常工作                              |
| Skill 触发  | 手动 Skill 触发和索引功能正常                                  |
| 记忆功能      | UserProfileMiddleware 和 MemoryUpdateMiddleware 正常工作 |
| 工具调用限制    | ToolCallLimitMiddleware 在 max_iterations > 0 时正常限制  |
| 多轮对话      | Checkpointer 持久化正常，历史消息正确恢复                         |
| 定时任务      | 定时任务触发的 Agent 执行正常，回写工具正常注入                         |
| 调试模式      | debug_context、call_chain_nodes、llm_calls 数据完整       |
| 取消执行      | cancel_event 正常中断 Agent 执行                          |

---

## 七、后续演进（可选）

### 7.1 完全移除 deepagents 依赖

如果后续希望完全移除 `deepagents` 包，需要自行实现：

1. **SubAgent 调度工具**：自定义 `task` 工具替代 `SubAgentMiddleware`，核心逻辑约 50 行
2. **PatchToolCallsMiddleware**：自定义悬空工具调用修补，核心逻辑约 30 行

### 7.2 按需引入 SummarizationMiddleware

当对话长度成为问题时（接近模型 context window），可引入 `SummarizationMiddleware`：

```python
from deepagents.middleware.summarization import SummarizationMiddleware

agent_middleware.append(
    SummarizationMiddleware(
        model=model,
        backend=lambda rt: StateBackend(rt),
        trigger=("tokens", 100000),  # 触发阈值
        keep=("messages", 6),  # 保留最近消息数
    )
)
```

需要配置合适的 `trigger` 和 `keep` 参数，并提供 Backend 实例。
