# DeepAgents 内置工具剥离方案

> 调研 deepagents `create_deep_agent()` 强制注入的内置工具来源、注入机制，以及三种剥离方案的对比。

---

## 一、问题描述

`deepagents` v0.3.8 的 `create_deep_agent()` 内部硬编码了一套中间件栈，每个中间件会通过 `self.tools` 属性向 Agent 注册工具，并通过 `wrap_model_call` 向 system prompt 注入使用说明。这些工具和 prompt **没有提供参数去禁用**。

我们的场景不需要这些内置工具（工具由管理后台配置、MCP 动态加载），它们的存在会：
- 污染 LLM 的工具列表，增加 token 消耗
- 注入无关的 system prompt 片段，干扰 Agent 行为
- 引入不需要的文件系统虚拟层（StateBackend）

---

## 二、内置工具来源分析

### 2.1 `create_deep_agent` 内部的中间件栈

`graph.py:189-218` 硬编码构建：

```
deepagent_middleware = [
    TodoListMiddleware(),                    ← 注册 write_todos 工具
    MemoryMiddleware(...),                   ← 条件启用（需传 memory 参数）
    SkillsMiddleware(...),                   ← 条件启用（需传 skills 参数）
    FilesystemMiddleware(backend=backend),   ← 注册 7 个文件系统工具
    SubAgentMiddleware(...),                 ← 注册 task 工具
    SummarizationMiddleware(...),            ← 无工具，上下文摘要
    AnthropicPromptCachingMiddleware(...),   ← 无工具，Anthropic 缓存优化
    PatchToolCallsMiddleware(),              ← 无工具，修复工具调用格式
    ...用户传入的 middleware 追加在末尾...
]
```

### 2.2 三个注册工具的中间件

| 中间件 | 注册的工具 | 注入的 system prompt | 源文件 |
|--------|-----------|---------------------|--------|
| `TodoListMiddleware` | `write_todos` | 任务管理说明 | `langchain.agents.middleware` |
| `FilesystemMiddleware` | `ls`, `read_file`, `write_file`, `edit_file`, `glob`, `grep`, `execute`（7 个） | 文件系统使用说明 + 沙箱执行说明 | `deepagents/middleware/filesystem.py` |
| `SubAgentMiddleware` | `task` | 子代理调度说明（约 250 行） | `deepagents/middleware/subagents.py` |

**合计**：9 个内置工具 + 约 300 行 system prompt 注入。

### 2.3 工具注入机制

中间件通过两个机制注入工具和 prompt：

**机制一：`self.tools` 属性**

LangChain 的 `create_agent()` 在构建图时，会自动收集所有中间件的 `self.tools` 列表，合并到 Agent 的工具列表中。中间件只需在 `__init__` 中设置 `self.tools = [...]`。

**机制二：`wrap_model_call` 拦截器**

中间件在每次调 LLM 前，通过 `request.override(system_message=...)` 向 system prompt 追加内容。例如 `FilesystemMiddleware` 追加文件系统使用说明，`SubAgentMiddleware` 追加子代理调度说明。

```python
# FilesystemMiddleware.wrap_model_call 核心逻辑（filesystem.py:935-981）
def wrap_model_call(self, request, handler):
    # 如果 backend 不支持执行，从工具列表移除 execute
    if not backend_supports_execution:
        filtered_tools = [t for t in request.tools if t.name != "execute"]
        request = request.override(tools=filtered_tools)

    # 追加文件系统 system prompt
    new_system_message = append_to_system_message(request.system_message, FILESYSTEM_SYSTEM_PROMPT)
    request = request.override(system_message=new_system_message)
    return handler(request)
```

---

## 三、剥离方案对比

### 方案 1：绕过 `create_deep_agent`，直接用 `create_agent`（推荐）

`create_deep_agent` 本质上就是帮你组装中间件然后调 LangChain 的 `create_agent`。既然内置中间件栈不是我们要的，直接调底层接口：

```python
from langchain.agents import create_agent

agent = create_agent(
    model=llm,
    tools=your_tools,              # 只放你需要的工具
    system_prompt=system_prompt,
    middleware=your_middlewares,    # 只放你需要的中间件
    checkpointer=checkpointer,
)
```

**改造后的 `agent_factory.py`**：

```python
def create_deep_agent(
    self,
    model, tools, system_prompt, subagents,
    middleware=None,
    checkpointer=None,
    name="main_agent",
):
    from langchain.agents import create_agent

    final_middleware = list(middleware or [])

    # 按需加回 deepagents 的某些能力
    # from deepagents.middleware.summarization import SummarizationMiddleware
    # final_middleware.append(SummarizationMiddleware(...))

    # 子代理支持（可选）
    if subagents:
        from deepagents.middleware.subagents import SubAgentMiddleware
        final_middleware.append(SubAgentMiddleware(
            default_model=model,
            default_tools=tools,
            subagents=subagents,
            general_purpose_agent=False,
        ))

    return create_agent(
        model=model,
        tools=tools,
        system_prompt=system_prompt,
        middleware=final_middleware,
        checkpointer=checkpointer,
        name=name,
    )
```

### 方案 2：保留 `create_deep_agent`，用中间件过滤工具

写一个 `ToolFilterMiddleware`，在每次调 LLM 前从工具列表中移除不需要的工具：

```python
class ToolFilterMiddleware(AgentMiddleware):
    """从 LLM 可见的工具列表中移除指定工具。"""

    BLOCKED_TOOLS = {
        "ls", "read_file", "write_file", "edit_file",
        "glob", "grep", "execute", "write_todos",
    }

    def wrap_model_call(self, request, handler):
        filtered = [t for t in request.tools if t.name not in self.BLOCKED_TOOLS]
        return handler(request.override(tools=filtered))

    async def awrap_model_call(self, request, handler):
        filtered = [t for t in request.tools if t.name not in self.BLOCKED_TOOLS]
        return await handler(request.override(tools=filtered))
```

使用方式：

```python
agent = create_deep_agent(
    model=llm,
    tools=your_tools,
    middleware=[ToolFilterMiddleware()],
    ...
)
```

**原理**：`wrap_model_call` 的 `request.override(tools=...)` 可以替换 LLM 看到的工具列表。`FilesystemMiddleware` 自身就用了这个机制来条件移除 `execute` 工具（`filesystem.py:959-961`），所以这是框架支持的标准做法。

### 方案 3：Monkey-patch `create_deep_agent`

在项目启动时替换 `deepagents.graph.create_deep_agent` 的内部中间件构建逻辑。

```python
# 不推荐，仅说明可行性
import deepagents.graph as _graph
_original = _graph.create_deep_agent

def _patched_create_deep_agent(*args, **kwargs):
    # 篡改内部行为...
    pass

_graph.create_deep_agent = _patched_create_deep_agent
```

---

## 四、方案对比

| 维度 | 方案 1：直接用 `create_agent` | 方案 2：ToolFilterMiddleware | 方案 3：Monkey-patch |
|------|------|------|------|
| **干净程度** | 完全干净，无多余对象 | 工具对象仍在内存，LLM 看不到 | 取决于 patch 质量 |
| **system prompt** | 无多余注入 | 仍被注入文件系统/子代理说明 | 取决于 patch 范围 |
| **改动量** | 改 `agent_factory.py` 一个方法 | 新增一个中间件类 | 新增 patch 模块 |
| **框架耦合** | 低（只依赖 LangChain） | 中（依赖 deepagents + LangChain） | 高（依赖 deepagents 内部实现） |
| **升级风险** | 低 | 低 | 高（版本升级可能失效） |
| **按需加回能力** | 自由选择任意中间件 | 只能过滤工具，无法去除中间件 | 理论上可以但复杂 |
| **子代理支持** | 按需引入 SubAgentMiddleware | 保留 task 或过滤掉 | 取决于 patch |
| **多余开销** | 无 | Filesystem 的大文件驱逐、prompt 注入仍在运行 | 取决于 patch |

---

## 五、推荐方案

**方案 1（直接用 `create_agent`）是正解。**

理由：

1. **`create_deep_agent` 的价值在于帮你组装内置中间件栈**，但这个栈恰好不是我们要的，就没必要用它
2. **`create_agent` 是 LangChain 的标准接口**，稳定性和兼容性比 deepagents 封装层更好
3. **完全的控制权**：每一个中间件、每一个工具、每一段 system prompt 都由我们决定
4. **deepagents 的能力仍可按需引入**：`SummarizationMiddleware`、`SubAgentMiddleware`、`PatchToolCallsMiddleware` 等都可以单独 import 使用，不需要通过 `create_deep_agent` 打包

### 可按需保留的 deepagents 中间件

| 中间件 | 功能 | 是否建议保留 |
|--------|------|-------------|
| `SummarizationMiddleware` | 上下文接近 token 上限时自动摘要 | 建议保留 |
| `PatchToolCallsMiddleware` | 修复某些 LLM 的工具调用格式问题 | 建议保留 |
| `AnthropicPromptCachingMiddleware` | Anthropic 模型的 prompt 缓存优化 | 按需（不用 Anthropic 则不需要） |
| `SubAgentMiddleware` | 子代理调度（task 工具） | 按需（有子代理需求时引入） |
| `FilesystemMiddleware` | 文件操作工具 | 不保留（我们有自己的工具体系） |
| `TodoListMiddleware` | write_todos 工具 | 不保留 |
| `MemoryMiddleware` | 记忆加载 | 不保留（我们有自己的记忆方案） |
| `SkillsMiddleware` | 技能加载 | 不保留（我们有自己的 Skill 体系） |
