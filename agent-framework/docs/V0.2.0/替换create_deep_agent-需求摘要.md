# 替换 create_deep_agent 为 create_agent — 需求摘要

> 面向 Coding 平台的需求文档，仅保留需求与约束，不含实现细节。

---

## 一、背景与问题

### 1.1 现状

主 Agent 当前通过 `deepagents.create_deep_agent()` 创建，其内部硬编码约 8 个中间件，再调用 `langchain.agents.create_agent()` 构建 Agent。我们无法控制中间件栈的组成与顺序。

### 1.2 问题

**问题一：不需要的工具被注入**

- TodoListMiddleware 注入 `write_todos`
- FilesystemMiddleware 注入文件类工具（ls、read_file、write_file 等）及虚拟文件系统
- SubAgentMiddleware 注入通用子 agent（与现有 `run_sub_agent` 功能重叠）

当前通过 ToolFilterMiddleware 黑名单屏蔽部分工具，属于「先注入再过滤」，中间件仍在运行。

**问题二：无法删除的 system_prompt 注入**

各中间件通过 override 向每次 LLM 调用追加提示词（文件操作、子代理调度、todo、BASE_AGENT_PROMPT 等），无法通过工具过滤屏蔽，每轮多消耗 token 并干扰行为。

**问题三：不必要的运行时开销**

- 虚拟文件系统、大文件驱逐、不受控的摘要触发、Anthropic 专用缓存、AGENTS.md/SKILL.md 加载等，与当前业务无关却持续占用资源。

---

## 二、方案目标

- **核心目标**：不再使用 `create_deep_agent`，改为直接使用 `langchain.agents.create_agent`，由项目**自行组装中间件栈**，从源头去掉不需要的工具与提示词。
- **兼容性**：对外接口（assembler 调用方式、astream_events、run_config、事件格式）保持不变，前端与定时任务等无需改动。

---

## 三、中间件取舍要求

### 3.1 保留并调整

| 中间件 | 要求 |
|--------|------|
| **SubAgentMiddleware** | 保留，用于配置子 agent 调度（task 工具）；须关闭「通用子 agent」，仅保留配置子 agent，避免与 run_sub_agent 重复 |
| **PatchToolCallsMiddleware** | 保留，用于修补悬空工具调用（AI 发出 tool_call 但无对应 ToolMessage 时补取消消息），保障稳定性 |

### 3.2 移除

| 中间件 | 理由 |
|--------|------|
| TodoListMiddleware | 不需要 todo 功能 |
| FilesystemMiddleware | 不需要文件操作工具，工具由管理后台配置 + MCP 动态加载 |
| MemoryMiddleware | 已有 UserProfileMiddleware + MemoryUpdateMiddleware |
| SkillsMiddleware | 已有 skill_assembler |
| AnthropicPromptCachingMiddleware | 使用 OpenAI 兼容接口，无效 |
| SummarizationMiddleware | 本期暂不引入；若后续需要需单独配置触发与保留策略 |

### 3.3 替换后中间件栈（需求侧）

- **从 deepagents 按需引入**：SubAgentMiddleware（精简配置）、PatchToolCallsMiddleware。
- **项目自定义中间件**：ToolCallLimitMiddleware、UserProfileMiddleware、MemoryUpdateMiddleware，按场景配置条件启用。
- **ToolFilterMiddleware**：**不再需要**，因不再注入需屏蔽的内置工具。

---

## 四、子 Agent 中间件栈

- 子 agent 执行时的中间件栈需可配置；替换后仅保留必要项（如 PatchToolCallsMiddleware）。
- 若未来子 agent 对话过长需要摘要，再按需引入 SummarizationMiddleware 并单独配置。

---

## 五、对外行为与兼容

- **assembler**：调用创建 Agent 的入口方法签名保持不变，无需改调用方。
- **事件与流式**：astream_events(version="v2")、run_config、事件类型与格式不变；SubAgent 仍通过 `task` 工具识别，event_handler 逻辑可不变。
- **工具列表**：仅包含业务配置的工具 + task（SubAgent），无多余内置工具。
- **system_prompt**：仅包含业务提示词，不再追加 BASE_AGENT_PROMPT 及各类中间件的内置说明；若出现「不主动调用工具」可考虑在业务 prompt 中加引导语。

---

## 六、依赖与清理

- **deepagents**：仍保留依赖，用于 SubAgentMiddleware、PatchToolCallsMiddleware、CompiledSubAgent；若未来自研子 agent 调度与 patch 逻辑，可再考虑移除。
- **删除**：ToolFilterMiddleware 及其在中间件链中的注册与导入；不再维护工具黑名单。

---

## 七、预期收益（需求侧）

| 维度 | 期望 |
|------|------|
| 工具列表 | 仅业务工具，无 9 个内置工具注入 |
| system_prompt | 无文件/子代理/todo 等内置大段提示词 |
| token | 每轮节省与内置提示词及多余工具 schema 相当的 token（量级可评估） |
| 运行时 | 仅运行 2～5 个必要中间件，无虚拟文件系统等无关逻辑 |
| 可控性 | 中间件栈完全由项目控制，可调整顺序与参数 |

---

## 八、风险与注意事项

- **recursion_limit**：若原 create_deep_agent 有默认配置，替换后需保证由 run_config 或等价方式统一设置（如 chat_service 中已有则保持不变）。
- **子 agent 中间件**：仅保留必要项，避免再次带入不需要的中间件；后续摘要需求单独设计。
- **BASE_AGENT_PROMPT**：不再自动追加；若 Agent 行为异常可业务侧补充引导语。

---

## 九、回归验证要点

替换后需验证至少以下场景：

- 基础对话：流式响应、token 统计正常
- 工具调用：配置工具与 MCP 工具正常，工具列表中无多余项
- 配置子 agent：task 工具与 SubAgent 事件正常
- 动态子 agent：run_sub_agent 正常
- Skill：手动触发与索引正常
- 记忆：UserProfileMiddleware、MemoryUpdateMiddleware 正常
- 工具调用限制：max_iterations > 0 时限制生效
- 多轮与持久化：Checkpointer 与历史恢复正常
- 定时任务：执行与回写正常
- 调试：debug_context、调用链、llm_calls 等完整
- 取消：cancel_event 能正确中断执行
