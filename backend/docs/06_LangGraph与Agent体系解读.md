# LangGraph 与 Agent 体系解读

> 覆盖 `src/agents/lead_agent/agent.py`、`src/agents/thread_state.py`、`src/agents/middlewares/`、`src/sandbox/middleware.py`

---

## 0. 先回答：`create_agent` 到底是啥？

```python
# src/agents/lead_agent/agent.py
from langchain.agents import create_agent

return create_agent(
    model=create_chat_model(...),
    tools=get_available_tools(...),
    middleware=_build_middlewares(...),
    system_prompt=apply_prompt_template(...),
    state_schema=ThreadState,
)
```

**一句话**：`create_agent` 是 **LangChain v1 新版 API**，它帮你把 LangGraph 的底层图构建细节全部封装好，你只需要声明"用什么模型、有什么工具、插什么中间件"，框架自动组装成一个可运行的 ReAct 循环图。

这是从旧版到新版的演进：

| 时代 | 函数 | 包路径 |
|---|---|---|
| v0（旧） | `create_react_agent` | `langgraph.prebuilt` |
| v1（新，本项目用） | `create_agent` | `langchain.agents` |

---

## 1. LangGraph 是什么？

LangGraph 是一个**有向图（Directed Graph）运行时**，专门用来描述和执行 AI Agent 的多步推理流程。

### 1.1 为什么需要它？

LLM 调用一次只能"推理一步"。但 Agent 需要：
- 调用工具，拿到结果，再继续推理
- 条件分支（有 tool call → 执行工具；没有 → 输出最终答案）
- 跨轮次保留上下文（多轮对话）

这些都需要一个**状态机**来驱动循环。LangGraph 就是这个状态机框架。

### 1.2 核心概念

```
StateGraph
├── State（状态）   ← 整个图共享的数据快照，类似 Java 里一个线程持有的 Context 对象
├── Node（节点）    ← 每一步操作（调用 LLM、执行工具），类似 Java 方法
└── Edge（边）      ← 节点之间的跳转规则（条件边 = if/else 分支）
```

**类比 Java**：
- `StateGraph` ≈ Spring StateMachine
- `State` ≈ 线程安全的 `ConcurrentHashMap`，每个节点执行后可以往里写数据
- `Node` ≈ `@Service` 方法，接收 State，返回 State 的增量更新
- `Edge` ≈ 路由逻辑（类似策略模式的路由器）

---

## 2. `create_agent` 内部构建的图

`create_agent` 自动搭建了如下 LangGraph 结构：

```
┌─────────────────────────────────────────────────────────┐
│  START                                                   │
│    ↓                                                     │
│  [before_agent middleware]  ← 每次 invoke 前执行         │
│    ↓                                                     │
│  ┌──────────────────────────────────────────────┐       │
│  │           ReAct 循环                          │       │
│  │                                               │       │
│  │  [before_model middleware]                    │       │
│  │    ↓                                          │       │
│  │  [model 节点]  ← 调用 LLM                    │       │
│  │    ↓                                          │       │
│  │  [after_model middleware]                     │       │
│  │    ↓                                          │       │
│  │  有 tool call? ──YES──→ [tools 节点]          │       │
│  │       ↑                    ↓ wrap_tool_call   │       │
│  │       └────────────────────┘                 │       │
│  │  NO（最终答案）→ 退出循环                     │       │
│  └──────────────────────────────────────────────┘       │
│    ↓                                                     │
│  [after_agent middleware]  ← Agent 完整回答后执行        │
│    ↓                                                     │
│  END                                                     │
└─────────────────────────────────────────────────────────┘
```

这一切都由 `create_agent` 内部自动搭建，你不需要手写一行 LangGraph 代码。

---

## 3. `create_agent` 的五个参数

### 3.1 `model` — 推理引擎

```python
model=create_chat_model(
    name=model_name,               # "deepseek-v3" / "claude-sonnet-4-6" 等
    thinking_enabled=thinking_enabled,
    reasoning_effort=reasoning_effort,
)
```

传入一个 `BaseChatModel` 实例。框架把它挂在图的 **model 节点**，每轮循环都会调用它。

支持动态切换（通过 middleware 的 `wrapModelCall` 钩子在运行时替换模型）。

---

### 3.2 `tools` — Agent 的能力武器库

```python
tools=get_available_tools(
    model_name=model_name,
    groups=agent_config.tool_groups if agent_config else None,
    subagent_enabled=subagent_enabled,
)
```

传入工具列表（`BaseTool` 实例），框架把它们绑定到图的 **tools 节点**。

LLM 返回 `tool_call` 时，框架自动路由到 tools 节点执行，结果以 `ToolMessage` 形式回填到消息历史里。

---

### 3.3 `system_prompt` — 静态系统提示

```python
system_prompt=apply_prompt_template(
    subagent_enabled=subagent_enabled,
    max_concurrent_subagents=max_concurrent_subagents,
    agent_name=agent_name,
)
```

每次调用 LLM 前，框架自动把这段 prompt 作为 `SystemMessage` 拼进消息列表最前面。

> **注意**：v1 里这个参数是**静态**的（字符串）。如果需要根据对话状态动态生成 prompt，要用 Middleware 的 `@dynamic_prompt` 钩子。

---

### 3.4 `state_schema` — 自定义状态 Schema

```python
state_schema=ThreadState
```

**源码**：[`src/agents/thread_state.py`](../src/agents/thread_state.py)

```python
from langchain.agents import AgentState

class ThreadState(AgentState):
    sandbox: NotRequired[SandboxState | None]      # 沙箱 ID
    thread_data: NotRequired[ThreadDataState | None] # 线程目录路径
    title: NotRequired[str | None]                 # 对话标题
    artifacts: Annotated[list[str], merge_artifacts] # 产出文件列表
    todos: NotRequired[list | None]                # Plan mode 任务列表
    uploaded_files: NotRequired[list[dict] | None] # 上传文件信息
    viewed_images: Annotated[dict[str, ...], merge_viewed_images]
```

`AgentState` 是框架内置基类，已经包含了 `messages: list[BaseMessage]`。

`ThreadState` 在它基础上**扩展**了沙箱、标题、文件等项目专有字段。

**类比 Java**：就像一个 `ThreadLocal` 持有的 Context 对象，贯穿整个 Agent 执行周期，所有节点和 Middleware 都能读写它。

> **Reducer（合并函数）**：带 `Annotated` 的字段有特殊合并逻辑。例如 `artifacts` 用 `merge_artifacts` 函数——追加去重而非直接覆盖。这是 LangGraph 的特性：不同节点可以同时修改 State，框架负责合并。

---

### 3.5 `middleware` — 中间件链（v1 最大的新特性）

```python
middleware=_build_middlewares(config, model_name=model_name, agent_name=agent_name)
```

这是整个 v1 架构最核心的新增能力。下一节重点讲。

---

## 4. Middleware 体系

### 4.1 它解决什么问题？

在旧版（`create_react_agent`）里，想在 Agent 前后插入逻辑很麻烦：要直接改 LangGraph 的图结构，添加节点和边，侵入性强。

v1 引入了 **Middleware（中间件）** 机制：定义好钩子点，业务代码只需实现对应的钩子方法，框架自动在正确的时机调用。

**类比 Java**：就像 Spring MVC 的 `HandlerInterceptor`，或者 Servlet Filter Chain，按顺序执行，每个 Middleware 各司其职。

### 4.2 可用的钩子点

```python
from langchain.agents.middleware import AgentMiddleware

class MyMiddleware(AgentMiddleware):

    def before_agent(self, state, runtime) -> dict | None:
        """整个 Agent 开始前执行（每次 invoke 调用一次）"""
        # 常用于：初始化资源（沙箱、路径）、注入上下文
        # ⚠️ 「这个 Agent」指的是注册了本 Middleware 的那个 agent 实例

    def before_model(self, state, runtime) -> dict | None:
        """每次调用 LLM 前执行（ReAct 循环每轮调用）"""
        # 常用于：消息摘要、裁剪 context window、动态 prompt 注入
        # ⚠️ 指的是 create_agent(model=...) 传入的那个 LLM，每轮 ReAct 循环触发一次

    def after_model(self, state, runtime) -> dict | None:
        """每次 LLM 返回后执行（在判断是否有 tool call 之前）"""
        # 常用于：拦截并修改 LLM 的输出、Human-in-the-loop 审批

    def after_agent(self, state, runtime) -> dict | None:
        """整个 Agent 完成后执行（最终回答输出后）"""
        # 常用于：生成标题、写入记忆、触发异步任务
        # ⚠️ 同 before_agent，只对注册了本 Middleware 的那个 agent 实例触发

    def wrap_tool_call(self, call, handler):
        """包装工具调用（在 tools 节点执行时）"""
        # 常用于：工具调用重试、错误处理、审计日志
        # ⚠️ 对该 agent 本轮触发的【所有】工具调用均生效，无法在注册时指定只拦截某个工具
        # 若要针对特定工具，在方法内部用 request.tool_call["name"] 判断
```

返回值是 `dict`（State 的增量更新）或 `None`（不修改 State）。

#### before_agent / after_agent 指的是「哪个 Agent」？

Middleware 是**绑定在某个具体 agent 实例上**的，不是全局的。`before_agent` / `after_agent` 只对注册了它的那个 agent 实例触发，每次 `invoke` / `astream` 调用触发一次。

本项目有 Lead Agent 和 Subagent 两种角色：

```
Lead Agent 的 before_agent  ← 触发（Lead Agent 的 Middleware）
    ↓
  ReAct 循环...
    ↓ LLM 决定调用 create_task 工具
    ↓ 派生出 Subagent 实例（独立运行，有自己的 Middleware，或没有）
    ↓ Subagent 执行完毕，返回 ToolMessage 给 Lead Agent
    ↓
Lead Agent 的 after_agent   ← 触发（Lead Agent 的 Middleware）
```

Subagent 的执行不会触发 Lead Agent Middleware 里的 `before_agent` / `after_agent`，两者是独立的。

#### wrap_tool_call 指的是「哪个 tool」？

`wrap_tool_call` **对该 agent 所有工具调用均生效**，无法在注册 Middleware 时指定只拦截某个工具。如果 LLM 一次性生成了多个并行 tool_call，`wrap_tool_call` 会对每个 tool_call 分别触发一次。

若要针对特定工具做不同处理，在方法体内判断工具名：

```python
def wrap_tool_call(self, request, handler):
    if request.tool_call["name"] == "bash":
        # 只对 bash 工具做重试
        for attempt in range(3):
            try:
                return handler(request)
            except Exception:
                if attempt == 2: raise
    # 其他工具走正常流程
    return handler(request)
```

### 4.3 自定义 State Schema

每个 Middleware 可以声明自己需要用的 State 字段：

```python
class TitleMiddlewareState(AgentState):
    title: NotRequired[str | None]  # 声明我需要 title 字段

class TitleMiddleware(AgentMiddleware[TitleMiddlewareState]):
    state_schema = TitleMiddlewareState  # 绑定

    def after_agent(self, state: TitleMiddlewareState, runtime) -> dict | None:
        if self._should_generate_title(state):
            title = self._generate_title(state)
            return {"title": title}  # 写入 State
        return None
```

框架会把所有 Middleware 声明的 `state_schema` **自动合并**到最终的 State 里。

---

## 5. 本项目的中间件链

**源码**：[`src/agents/lead_agent/agent.py`](../src/agents/lead_agent/agent.py) `_build_middlewares()`

```python
middlewares = [
    ThreadDataMiddleware(),          # 1. 计算线程专属目录路径 → thread_data
    UploadsMiddleware(),             # 2. 把上传文件信息注入 State
    SandboxMiddleware(),             # 3. 懒加载沙箱（lazy_init=True）
    DanglingToolCallMiddleware(),    # 4. 修复缺失的 ToolMessage（LLM 故障容错）
    SummarizationMiddleware(...),    # 5. [可选] 消息历史超长时自动摘要
    TodoListMiddleware(...),         # 6. [可选] Plan mode 任务列表工具
    TitleMiddleware(),               # 7. 第一轮后自动生成对话标题
    MemoryMiddleware(...),           # 8. 对话结束后异步写入长期记忆
    ViewImageMiddleware(),           # 9. [可选] 把图片转 base64 注入消息
    SubagentLimitMiddleware(...),    # 10.[可选] 限制并发 subagent 数量
    ClarificationMiddleware(),       # 11.拦截澄清请求（永远最后）
]
```

**顺序的意义非常重要**，注释里有明确说明：

```python
# ThreadDataMiddleware 必须在 SandboxMiddleware 之前（沙箱初始化需要线程路径）
# DanglingToolCallMiddleware 修复历史消息，必须在 model 看到历史之前处理
# SummarizationMiddleware 应尽早裁剪 context，减少后续处理的消息量
# ClarificationMiddleware 必须最后，拦截完整的 LLM 输出
```

**类比 Java**：就像 Spring Security 的 Filter Chain，顺序错误会导致逻辑 bug。

---

## 6. 以 TitleMiddleware 为例的完整解析

### 6.1 功能

第一轮对话结束后（用户发了消息，Agent 给了回答），自动调用轻量级 LLM 生成一个对话标题，写入 State。

### 6.2 代码走读

**源码**：[`src/agents/middlewares/title_middleware.py`](../src/agents/middlewares/title_middleware.py)

```python
class TitleMiddleware(AgentMiddleware[TitleMiddlewareState]):
    state_schema = TitleMiddlewareState   # 声明需要 title 字段

    def after_agent(self, state, runtime) -> dict | None:
        # 挂在 after_agent 钩子 → 整个 Agent 回答完毕后触发
        if self._should_generate_title(state):
            title = self._generate_title(state)
            return {"title": title}       # 写入 State["title"]
        return None

    def _should_generate_title(self, state) -> bool:
        # 已有标题 → 跳过
        if state.get("title"):
            return False
        messages = state.get("messages", [])
        user_msgs = [m for m in messages if m.type == "human"]
        ai_msgs   = [m for m in messages if m.type == "ai"]
        # 精确在第一轮结束时触发（1条用户消息 + 至少1条 AI 回复）
        return len(user_msgs) == 1 and len(ai_msgs) >= 1
```

### 6.3 执行时机

```
用户发第一条消息
    ↓ before_agent（各 Middleware 初始化）
    ↓ before_model → model 调用 LLM → after_model
    ↓ （如有工具调用，循环几轮）
    ↓ LLM 输出最终答案，退出 ReAct 循环
    ↓ after_agent ← TitleMiddleware 在这里触发
    ↓ 调用轻量级 LLM 生成标题，写入 State["title"]
    ↓ 框架把更新后的 State 持久化到 Checkpointer
```

---

## 7. ThreadState 与 AgentState 的关系

```
langchain.agents.AgentState（框架基类）
└── messages: list[BaseMessage]  ← 完整对话历史

        继承
        ↓

ThreadState（本项目的扩展）
├── messages          （继承自父类）
├── sandbox           → 沙箱 ID，供工具调用时查找
├── thread_data       → 线程专属目录路径
├── title             → 对话标题
├── artifacts         → Agent 产出的文件列表（带去重 Reducer）
├── todos             → Plan mode 任务列表
├── uploaded_files    → 用户上传文件元信息
└── viewed_images     → 已加载的图片（base64）
```

`ThreadState` 把"对话历史"之外所有的**运行时上下文**都装在里面，通过 Checkpointer 持久化，实现多轮对话的状态延续。

---

## 8. Checkpointer：State 是怎么持久化的

LangGraph 有 **Checkpointer** 机制：每次 State 更新后自动持久化到存储后端。

```
用户第 1 轮：invoke(messages=[HumanMessage("你好")])
    → State 保存到 Checkpointer（thread_id="abc123"）

用户第 2 轮：invoke(messages=[HumanMessage("继续")])
    → 框架自动从 Checkpointer 加载 thread_id="abc123" 的 State
    → State 里已有第一轮的 messages，直接追加
    → 多轮对话上下文自动延续
```

本项目支持两种 Checkpointer 后端（在 config.yaml 里配置）：

| 类型 | 场景 |
|---|---|
| `MemoryCheckpointer` | 开发调试（进程重启后丢失） |
| `PostgresCheckpointer` | 生产环境（跨进程持久化） |

**类比 Java**：Checkpointer 就像 Spring Session 里的 `SessionRepository`——每个 `thread_id` 对应一个 Session，框架自动序列化/反序列化。

---

## 9. 整体调用链

```
HTTP 请求到达 FastAPI
    ↓
gateway/routers/agents.py
    ↓ 调用 client.py 的 invoke/astream
    ↓
make_lead_agent(config)
    ↓ 从 config 解析 model_name、is_plan_mode、subagent_enabled 等
    ↓ create_agent(model, tools, middleware, system_prompt, state_schema)
    ↓ 返回一个 LangGraph Compiled Graph（可 invoke / astream）

图执行：
    ↓ Checkpointer 加载历史 State
    ↓ before_agent 中间件链（初始化沙箱路径、上传文件等）
    ↓ ┌── ReAct 循环 ────────────────────────────────────┐
    ↓ │  before_model → LLM 推理 → after_model           │
    ↓ │  有 tool call? → tools 节点 → 回到 before_model  │
    ↓ └──────────────────────────────────────────────────┘
    ↓ after_agent 中间件链（生成标题、更新记忆等）
    ↓ Checkpointer 保存更新后的 State
    ↓ 通过 SSE 流式返回给前端
```

---

## 10. Tool 体系：工具是怎么给 Agent 用的

### 10.1 疑问的起点

你在 [`src/tools/tools.py`](../src/tools/tools.py) 看到这行代码：

```python
from langchain.tools import BaseTool

loaded_tools = [resolve_variable(tool.use, BaseTool) for tool in config.tools ...]
```

然后在 [`src/community/tavily/tools.py`](../src/community/tavily/tools.py) 看到：

```python
from langchain.tools import tool

@tool("web_search", parse_docstring=True)
def web_search_tool(query: str) -> str:
    """Search the web.

    Args:
        query: The query to search for.
    """
    ...
```

三个问题：
1. `BaseTool` 是什么？
2. `@tool` 注解做了什么？`parse_docstring` 是什么？
3. 函数加了 `@tool` 之后，变成了什么？

---

### 10.2 `BaseTool` — 工具的基类

`BaseTool` 是 LangChain 里**所有工具的统一抽象基类**，类比 Java 里：
- `BaseTool` ≈ `java.lang.Runnable` 接口

但比 `Runnable` 富得多，它包含：

```
BaseTool（抽象类）
├── name: str            ← 工具名（LLM 调用时使用）
├── description: str     ← 工具整体描述（LLM 依此决定是否用这个工具）
├── args_schema: Schema  ← 参数的 JSON Schema（LLM 依此填参数）
└── _run(...)            ← 抽象方法，子类实现具体逻辑
```

**LLM 为什么需要这三个字段？**

当 Agent 调用 LLM 时，框架会把所有工具的 `name + description + args_schema` 序列化成 JSON 发给 LLM：

```json
[
  {
    "name": "web_search",
    "description": "Search the web.",
    "parameters": {
      "type": "object",
      "properties": {
        "query": {
          "type": "string",
          "description": "The query to search for."
        }
      },
      "required": ["query"]
    }
  }
]
```

LLM 读了这个 JSON schema，自己决定：**"我需要搜索网页，所以调用 `web_search`，参数是用户的问题"**。

然后 LLM 在输出里生成一个 `tool_call`，框架识别到后路由到 tools 节点执行。

---

### 10.3 `@tool` 装饰器 — 把函数变成 `BaseTool` 实例

`@tool` 是 LangChain 提供的**工厂装饰器**，核心作用是：

> **把一个普通 Python 函数"包装"成一个 `BaseTool` 的子类实例**

类比 Java：就像 Spring 的 `@Bean`——加了注解，方法就变成了容器管理的对象。

**加上 `@tool` 前后的本质变化：**

```python
# 加之前：只是一个普通函数
def web_search_tool(query: str) -> str:
    ...

print(type(web_search_tool))  # <class 'function'>

# 加之后：变成了 BaseTool 的子类实例！
@tool("web_search", parse_docstring=True)
def web_search_tool(query: str) -> str:
    ...

print(type(web_search_tool))  # <class 'langchain_core.tools.structured.StructuredTool'>
print(isinstance(web_search_tool, BaseTool))  # True
print(web_search_tool.name)        # "web_search"
print(web_search_tool.description) # "Search the web."
```

**框架自动做了什么：**
1. 创建一个 `StructuredTool`（`BaseTool` 的子类）实例
2. 把函数的**类型注解（type hints）**转成 `args_schema`（Pydantic schema）
3. 把函数体封装成 `_run()` 方法
4. 把 `@tool("web_search")` 的第一个参数设为 `tool.name`

---

### 10.4 `parse_docstring=True` — 从注释里提取参数描述

这是最关键的细节。

**不用 `parse_docstring` 时（参数描述为空）：**

```python
@tool("web_search")
def web_search_tool(query: str) -> str:
    """Search the web."""
    ...
```

生成的 schema：
```json
{
  "name": "web_search",
  "description": "Search the web.",
  "parameters": {
    "properties": {
      "query": {"type": "string"}   // ← 没有 description！
    }
  }
}
```

LLM 看到 `query` 但不知道它具体是什么，容易填错。

**用了 `parse_docstring=True` 时（从 Args 段提取）：**

```python
@tool("web_search", parse_docstring=True)
def web_search_tool(query: str) -> str:
    """Search the web.

    Args:
        query: The query to search for.   ← 这行被解析了！
    """
    ...
```

生成的 schema：
```json
{
  "name": "web_search",
  "description": "Search the web.",
  "parameters": {
    "properties": {
      "query": {
        "type": "string",
        "description": "The query to search for."   // ← 有了！
      }
    }
  }
}
```

**这就是为什么项目里所有 `@tool` 都加 `parse_docstring=True`，并且 docstring 的 Args 段写得非常详细**——这些描述直接影响 LLM 理解参数的准确度。

---

### 10.5 `@tool` 的其他参数

项目里出现过的参数：

```python
@tool("ask_clarification", parse_docstring=True, return_direct=True)
def ask_clarification_tool(...):
    ...
```

| 参数 | 含义 |
|---|---|
| 第一个字符串 `"web_search"` | 工具名（LLM 看到的 `name`），不传则用函数名 |
| `parse_docstring=True` | 从 docstring Args 段提取参数描述 |
| `return_direct=True` | 工具返回结果后**直接输出给用户**，不再交回 LLM 推理 |

`return_direct=True` 用于 `ask_clarification_tool`，因为澄清问题需要立刻中断 Agent 返回给用户，不能让 LLM 再"加工"一遍。

---

### 10.6 `ToolRuntime` — 工具如何访问 Agent 的 State

沙箱相关工具（`bash_tool`、`ls_tool` 等）有一个特殊参数：

```python
# src/sandbox/tools.py
from langchain.tools import ToolRuntime

@tool("bash", parse_docstring=True)
def bash_tool(
    runtime: ToolRuntime[ContextT, ThreadState],  # ← 特殊！
    description: str,
    command: str,
) -> str:
    ...
    sandbox = ensure_sandbox_initialized(runtime)  # 从 runtime 取 sandbox
    thread_data = get_thread_data(runtime)         # 从 runtime 取线程路径
```

`ToolRuntime` 是框架注入的**上下文对象**，包含：
- `runtime.state` → 当前 `ThreadState`（可以读写 sandbox_id、thread_data 等）
- `runtime.context` → 调用时的 context（thread_id 等）

**关键**：`runtime` 参数的值**由框架自动注入**，LLM 永远不会看到它，也不会尝试给它传值。框架在生成 args_schema 时会自动排除 `ToolRuntime` 类型的参数。

**类比 Java**：就像 Spring MVC 里 Controller 方法可以接受 `HttpServletRequest` 参数，框架自动注入，不在 URL 参数里体现。

---

### 10.7 工具注册：从 config.yaml 到 `BaseTool` 实例

完整的工具加载链路：

```
config.yaml（声明配置）
    ↓
tools:
  - name: web_search
    group: web
    use: src.community.tavily.tools:web_search_tool  ← 字符串引用
    max_results: 5
    ↓
get_available_tools()
    ↓
resolve_variable("src.community.tavily.tools:web_search_tool", BaseTool)
    ↓ 反射：importlib.import_module + getattr
    ↓ 拿到 web_search_tool 这个 BaseTool 实例
    ↓
[web_search_tool, web_fetch_tool, bash_tool, ..., ask_clarification_tool]
    ↓
create_agent(tools=[...])
    ↓
框架把所有工具的 schema 序列化发给 LLM
```

`resolve_variable("src.community.tavily.tools:web_search_tool", BaseTool)` 这行：
- `:` 前面是模块路径
- `:` 后面是模块里的变量名（这里是 `web_search_tool`，一个 `BaseTool` 实例）
- 第二个参数 `BaseTool` 是类型断言（确保反射拿到的是 `BaseTool` 的实例）

---

### 10.8 完整总结：一个工具的一生

```
① 开发者写函数 + 加 @tool 注解
   ↓
   @tool("bash", parse_docstring=True)
   def bash_tool(runtime, description, command) -> str:
       """Execute a bash command..."""
   ↓
② Python 加载模块时，@tool 把函数变成 BaseTool 实例
   bash_tool = StructuredTool(name="bash", description="...", args_schema=...)
   ↓
③ config.yaml 里声明引用路径
   use: src.sandbox.tools:bash_tool
   ↓
④ get_available_tools() 通过反射加载，返回 [BaseTool, ...]
   ↓
⑤ create_agent(tools=[...]) 把工具绑到 LangGraph 的 tools 节点
   ↓
⑥ 每次调用 LLM 前，框架把所有工具的 name+description+args_schema
   序列化成 JSON，附在 LLM 请求里
   ↓
⑦ LLM 决定调用 "bash"，生成 tool_call:
   {"name": "bash", "arguments": {"description": "run script", "command": "python a.py"}}
   ↓
⑧ 框架路由到 tools 节点，执行 bash_tool._run(description=..., command=...)
   框架自动注入 runtime（LLM 不感知）
   ↓
⑨ 执行结果包装成 ToolMessage，追加到 State["messages"]
   ↓
⑩ 下一轮 LLM 推理看到工具结果，继续思考
```

---

## 11. 动手实验

以下实验不需要启动完整服务，直接写脚本验证核心概念。

### 实验 1：理解 ThreadState 的字段结构

```python
import sys
sys.path.insert(0, "src")

from src.agents.thread_state import ThreadState
from langchain.agents import AgentState
from langchain_core.messages import HumanMessage, AIMessage

# ThreadState 继承自 AgentState
print("ThreadState 的字段：")
for key, val in ThreadState.__annotations__.items():
    print(f"  {key}: {val}")

# 构造一个 ThreadState 实例（TypedDict，直接当 dict 用）
state: ThreadState = {
    "messages": [HumanMessage("hello"), AIMessage("hi!")],
    "title": None,
    "sandbox": None,
    "thread_data": None,
    "artifacts": [],
    "todos": None,
    "uploaded_files": None,
    "viewed_images": {},
}
print("\n消息数量：", len(state["messages"]))
print("第一条消息类型：", state["messages"][0].type)
```

---

### 实验 2：验证 Reducer 的合并行为

```python
import sys
sys.path.insert(0, "src")

from src.agents.thread_state import merge_artifacts, merge_viewed_images

# artifacts：追加 + 去重
existing = ["file_a.py", "file_b.txt"]
new_files = ["file_b.txt", "file_c.csv"]  # file_b 重复
result = merge_artifacts(existing, new_files)
print("合并后 artifacts：", result)
# 期望：["file_a.py", "file_b.txt", "file_c.csv"]（去重，保序）

# viewed_images：空 dict 表示清空
images = {"img1.png": {"base64": "abc", "mime_type": "image/png"}}
cleared = merge_viewed_images(images, {})   # 传空 dict 表示清空
print("清空后 viewed_images：", cleared)    # 期望：{}
```

---

### 实验 3：直接实例化一个最小 Agent

```python
import sys
sys.path.insert(0, "src")
import os
os.environ["OPENAI_API_KEY"] = "your-key"  # 或从 config.yaml 读

from langchain.agents import create_agent
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool

@tool
def add(a: int, b: int) -> int:
    """把两个数相加"""
    return a + b

agent = create_agent(
    model=ChatOpenAI(model="gpt-4o-mini"),
    tools=[add],
    system_prompt="你是一个数学助手。",
)

result = agent.invoke({"messages": [{"role": "user", "content": "3 + 5 等于多少？"}]})
print("最终消息：", result["messages"][-1].content)
```

**预期**：Agent 会调用 `add(3, 5)` 工具，然后输出 "8"。

---

## 12. 设计模式总结

| 模式 | 体现位置 | 说明 |
|---|---|---|
| **模板方法模式** | `create_agent` | 框架定义了图的骨架（节点+边），用户只填 model/tools/middleware 三个槽位 |
| **责任链模式** | Middleware Chain | 中间件按序执行，每个只关心自己的钩子点，互不干扰 |
| **工厂装饰器模式** | `@tool` | 把普通函数变成 `BaseTool` 实例，自动生成 LLM 所需的 JSON Schema |
| **依赖注入** | `ToolRuntime` | 框架自动向工具函数注入运行时上下文，LLM 不感知 |
| **TypedDict + Reducer** | `ThreadState` | State 字段通过注解声明合并策略，框架自动处理并发写入 |
| **策略模式** | Checkpointer | `MemoryCheckpointer` / `PostgresCheckpointer` 实现相同接口，配置切换 |
| **懒初始化** | `SandboxMiddleware(lazy_init=True)` | 资源在第一次真正需要时才创建，避免浪费 |
| **切面（AOP）** | `before_agent` / `after_agent` 钩子 | 横切关注点（标题生成、记忆更新）与业务逻辑解耦 |
