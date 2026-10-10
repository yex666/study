# K01: agent — Agent 入口与编排

> **文件路径**: `src/chrys/kernel/agent.py` (~800 行)
> **一句话**: Agent 的核心类，负责准备请求、驱动 LLM 客户端、运行工具循环、聚合响应、执行会话提供者。

---

## 这是什么？

Agent 是 Chrys kernel 的**顶层编排器**。它把多个组件组合在一起：

```
Agent = AgentTelemetryLayer (遥测) 
      + _AgentCore (核心逻辑)
      
Client Stack = ToolLoopLayer (工具循环)
             + ChatMiddlewareLayer (中间件)
             + Instrumented Raw Client (实际的 LLM 调用)
```

### 核心职责

1. **准备请求**：合并历史消息、instructions、tools、options
2. **驱动客户端**：调用 LLM，处理流式和非流式响应
3. **运行工具循环**：如果 LLM 返回工具调用，执行它们并继续
4. **聚合响应**：把多个 chat response 合并成一个 agent response
5. **会话管理**：运行 before/after 会话提供者（如历史持久化）

---

## 为什么需要？

### 如果不做会怎样？

```python
# ❌ 简单的 while 循环
class SimpleAgent:
    async def run(self, messages, tools):
        while True:
            response = await self.llm.chat(messages, tools)
            
            if not response.tool_calls:
                return response.text
            
            for tool_call in response.tool_calls:
                result = await execute_tool(tool_call)
                messages.append(result)
```

**问题**：
1. **没有会话管理**：无法恢复历史、无法持久化
2. **没有中间件**：无法插入审批、日志、压缩等逻辑
3. **没有遥测**：无法追踪性能、成本
4. **没有存储模式处理**：不知道是用本地历史还是服务端存储
5. **没有选项合并**：无法灵活配置每次运行

### 用了 Agent 之后

```python
# ✅ 完整的 Agent
agent = Agent(
    client=instrumented_client,
    session=my_session,
    history_provider=MyHistoryProvider(),
    context_providers=[MyContextProvider()],
    compaction_strategy=my_compaction,
    tokenizer=my_tokenizer,
)

response = await agent.run(
    messages=[{"role": "user", "content": "Help me refactor"}],
    instructions="You are a helpful assistant.",
    tools=[read_file, write_file],
    store=True,  # 使用服务端存储
)

# Agent 自动处理：
# - 从 session 加载历史
# - 运行 context providers
# - 应用 compaction
# - 调用 LLM（带中间件）
# - 执行工具循环
# - 持久化结果到 session
# - 记录遥测数据
```

---

## 核心设计：四个关键决策

### 决策 1: 分层架构（MRO 模式）

```python
class Agent(AgentTelemetryLayer, _AgentCore):
    """Public Agent interface combining telemetry and core logic."""
    pass

class AgentTelemetryLayer:
    """Adds OpenTelemetry spans around _AgentCore methods."""
    
    async def run(self, *args, **kwargs):
        with tracer.start_as_current_span("agent.run"):
            return await super().run(*args, **kwargs)

class _AgentCore:
    """Core logic: prepare request, drive client, finalize response."""
    
    async def run(self, ...):
        # 1. 准备上下文
        context = await self._prepare_run_context(...)
        
        # 2. 运行 before 会话提供者
        for provider in self.session.before_providers:
            await provider(context)
        
        # 3. 调用客户端（可能多次，如果有工具调用）
        response = await self.client.get_response(...)
        
        # 4. 运行 after 会话提供者
        for provider in self.session.after_providers:
            await provider(context, response)
        
        # 5. 返回响应
        return response
```

**为什么？**

- **关注点分离**：遥测逻辑和核心逻辑分开
- **可组合**：可以轻松添加新的层（如日志层、缓存层）
- **可测试**：可以单独测试 `_AgentCore`，不需要遥测开销

---

### 决策 2: 惰性求值（Lazy Evaluation）

```python
class _AgentCore:
    @overload
    async def run(self, *, stream: Literal[False] = False, ...) -> AgentResponse: ...
    
    @overload
    def run(self, *, stream: Literal[True], ...) -> ResponseStream: ...
    
    def run(self, *, stream: bool = False, ...):
        """Run the agent, returning either a coroutine or a stream."""
        
        if stream:
            # 返回 ResponseStream，不立即执行
            return ResponseStream.from_awaitable(
                self._run_streaming(...)
            )
        else:
            # 返回协程，等待时才执行
            return self._run_non_streaming(...)
    
    async def _run_non_streaming(self, ...):
        """Actually execute the agent (non-streaming)."""
        # ... 实际执行逻辑
    
    async def _run_streaming(self, ...):
        """Actually execute the agent (streaming)."""
        # ... 实际执行逻辑
```

**为什么？**

- **灵活性**：调用者可以选择何时开始执行
- **资源控制**：可以在执行前设置超时、取消等
- **组合性**：可以把 Agent run 作为另一个异步任务的一部分

---

### 决策 3: 选项合并策略

```python
def _merge_options(base: dict, override: dict) -> dict:
    """Merge base and override chat options.
    
    Rules:
    - None values are skipped (treated as "unset")
    - tools are unioned (deduplicated by name)
    - logit_bias/metadata are dict-merged
    - instructions are newline-concatenated
    - everything else is replaced
    """
    
    result = dict(base)
    
    for key, value in override.items():
        if value is None:
            continue
        
        if key == "tools":
            # 合并工具列表（去重）
            base_tools = normalize_tools(result.get("tools"))
            override_tools = normalize_tools(value)
            result["tools"] = _append_unique_tools(
                list(base_tools),
                override_tools,
            )
        
        elif key == "logit_bias":
            # 字典合并
            result["logit_bias"] = {**result.get("logit_bias", {}), **value}
        
        elif key == "metadata":
            # 字典合并
            result["metadata"] = {**result.get("metadata", {}), **value}
        
        elif key == "instructions":
            # 换行连接
            if result.get("instructions"):
                result["instructions"] = f"{result['instructions']}\n{value}"
            else:
                result["instructions"] = value
        
        else:
            # 直接替换
            result[key] = value
    
    # 移除所有 None 值
    return {k: v for k, v in result.items() if v is not None}
```

**为什么？**

- **灵活性**：Agent 可以有默认的工具和 instructions，每次 run 可以添加更多
- **安全性**：工具名称必须唯一，防止冲突
- **可预测**：明确的合并规则，不会意外覆盖重要配置

---

### 决策 4: 存储模式感知

```python
class _AgentCore:
    async def _prepare_run_context(self, ...):
        """Prepare the context for one run."""
        
        # 解析存储模式
        storage_resolution = resolve_storage_mode_and_handles(
            client=self.client,
            options=chat_options,
            client_kwargs=client_kwargs,
        )
        
        # 根据存储模式选择不同的行为
        if storage_resolution.service_side:
            # 服务端存储：不需要本地历史
            session_messages = []
            continuation_handle = collect_conversation_handle(chat_options)
        else:
            # 本地存储：需要从 session 加载历史
            session_messages = await self.session.load_history()
            continuation_handle = None
        
        # 清理失效的句柄
        if continuation_handle:
            strip_invalidated_conversation_handles(chat_options)
        
        return _RunContext(
            session=self.session,
            session_context=SessionContext(...),
            input_messages=input_messages,
            session_messages=session_messages,
            chat_options=chat_options,
            # ...
        )
```

**为什么？**

- **适配不同场景**：有些模型支持服务端存储（如 OpenAI Responses API），有些只支持本地
- **避免重复**：服务端存储时不需要发送完整历史
- **容错性**：如果服务端存储失败，可以回退到本地存储

---

## 源码解析

### Agent 类定义

```python
class Agent(AgentTelemetryLayer, _AgentCore):
    """Public Agent interface.
    
    Combines telemetry (AgentTelemetryLayer) with core logic (_AgentCore).
    Uses cooperative MRO to chain method calls.
    """
    
    DEFAULT_MAX_ITERATIONS: ClassVar[int] = 40
    DEFAULT_MAX_CONSECUTIVE_ERRORS: ClassVar[int] = 3
    
    def __init__(
        self,
        client: BaseChatClient,
        *,
        session: AgentSession | None = None,
        history_provider: HistoryProvider | None = None,
        context_providers: Sequence[ContextProvider] = (),
        compaction_strategy: Any = None,
        tokenizer: Any = None,
        max_iterations: int = DEFAULT_MAX_ITERATIONS,
        max_consecutive_errors: int = DEFAULT_MAX_CONSECUTIVE_ERRORS,
        middleware: Sequence[ChatMiddleware | FunctionMiddleware] = (),
    ):
        # 分割中间件
        chat_middleware, function_middleware = split_middleware(middleware)
        
        # 初始化核心
        super().__init__(
            client=client,
            session=session or AgentSession(),
            history_provider=history_provider or InMemoryHistoryProvider(),
            context_providers=context_providers,
            compaction_strategy=compaction_strategy,
            tokenizer=tokenizer,
            max_iterations=max_iterations,
            max_consecutive_errors=max_consecutive_errors,
            chat_middleware=chat_middleware,
            function_middleware=function_middleware,
        )
```

### run() 方法（非流式）

```python
async def _run_non_streaming(
    self,
    *,
    messages: Sequence[Message],
    instructions: str | None = None,
    tools: ToolTypes | None = None,
    **chat_options: Any,
) -> AgentResponse:
    """Run the agent non-streaming, returning a complete response."""
    
    # 1. 准备上下文
    context = await self._prepare_run_context(
        messages=messages,
        instructions=instructions,
        tools=tools,
        chat_options=chat_options,
        stream=False,
    )
    
    try:
        # 2. 运行 before 会话提供者
        for provider in context["session"].before_providers:
            await provider(context)
        
        # 3. 调用客户端（ToolLoopLayer 会处理工具循环）
        chat_response = await self.client.get_response(
            messages=context["input_messages"],
            **context["chat_options"],
        )
        
        # 4. 构建 agent response
        agent_response = _build_agent_response_from_chat_response(chat_response)
        
        # 5. 保存到 session context
        context["session_context"]._response = agent_response
        
        # 6. 运行 after 会话提供者
        for provider in context["session"].after_providers:
            await provider(context, agent_response)
        
        return agent_response
    
    except Exception as exc:
        # 错误处理
        logger.exception("Agent run failed")
        raise
```

### 工具去重逻辑

```python
def _append_unique_tools(
    existing_tools: list[Any],
    new_tools: Sequence[Any],
    *,
    duplicate_error_message: str | None = None,
) -> list[Any]:
    """Append tools while preserving uniqueness by name.
    
    Rules:
    - Nameless tools always append
    - Same object (or clones of same foreign source) seen twice is skipped
    - Different object under existing name raises ValueError
    """
    
    seen_by_name: dict[str, Any] = {}
    
    # 索引现有工具
    for tool_item in existing_tools:
        if tool_name := _get_tool_name(tool_item):
            seen_by_name[tool_name] = tool_item
    
    # 添加新工具
    for tool_item in new_tools:
        tool_name = _get_tool_name(tool_item)
        
        # 无名称的工具直接添加
        if tool_name is None:
            existing_tools.append(tool_item)
            continue
        
        # 检查是否已存在
        existing_tool = seen_by_name.get(tool_name)
        
        if existing_tool is None:
            # 新工具，添加
            seen_by_name[tool_name] = tool_item
            existing_tools.append(tool_item)
            continue
        
        # 检查是否是同一个工具（或克隆）
        if existing_tool is tool_item or _tool_identity(existing_tool) is _tool_identity(tool_item):
            # 相同工具，跳过
            continue
        
        # 不同工具但同名，报错
        message = duplicate_error_message or "Tool names must be unique."
        raise ValueError(f"Duplicate tool name '{tool_name}'. {message}")
    
    return existing_tools
```

---

## 实际使用场景

### 场景 1: 简单的非流式调用

```python
from chrys.kernel import Agent
from chrys.service.llm import create_client

# 创建客户端
client = create_client(model_profile)

# 创建 Agent
agent = Agent(client=client)

# 运行
response = await agent.run(
    messages=[{"role": "user", "content": "What is 2+2?"}],
    instructions="You are a math tutor.",
)

print(response.text)  # "2+2 equals 4."
```

### 场景 2: 流式调用

```python
# 创建 Agent（同上）
agent = Agent(client=client)

# 流式运行
stream = agent.run(
    messages=[{"role": "user", "content": "Tell me a story."}],
    stream=True,
)

async for update in stream:
    if update.delta:
        print(update.delta, end="", flush=True)
```

### 场景 3: 带工具的调用

```python
from chrys.kernel.tools import FunctionTool

# 定义工具
def read_file(path: str) -> str:
    with open(path) as f:
        return f.read()

read_file_tool = FunctionTool(
    name="read_file",
    description="Read a file",
    parameters={"path": {"type": "string"}},
    fn=read_file,
)

# 创建 Agent（带工具）
agent = Agent(
    client=client,
    tools=[read_file_tool],
)

# 运行（Agent 会自动处理工具循环）
response = await agent.run(
    messages=[{"role": "user", "content": "Read README.md and summarize it."}],
)

print(response.text)
```

### 场景 4: 带会话历史的调用

```python
from chrys.kernel.sessions import AgentSession

# 创建会话
session = AgentSession(conversation_id="conv_123")

# 创建 Agent（带会话）
agent = Agent(
    client=client,
    session=session,
)

# 第一次运行
response1 = await agent.run(
    messages=[{"role": "user", "content": "My name is Alice."}],
)

# 第二次运行（自动包含历史）
response2 = await agent.run(
    messages=[{"role": "user", "content": "What's my name?"}],
)

print(response2.text)  # "Your name is Alice."
```

---

## 和其他模块的关系

```
agent.py 依赖：
├── _types.py (AgentResponse, Message, etc.)
├── client.py (BaseChatClient, storage mode resolution)
├── sessions.py (AgentSession, HistoryProvider)
├── instrumentation.py (AgentTelemetryLayer)
├── middleware.py (split_middleware)
└── tools.py (normalize_tools)

被以下模块使用：
├── orchestration/engine/build/builder.py (构建主 Agent)
├── orchestration/sub_agents/tools.py (构建子 Agent)
└── orchestration/workflows/agent_node_build.py (构建工作流节点 Agent)
```

---

## 如果你要自己实现

### 最小版本（50 行）

```python
class SimpleAgent:
    def __init__(self, client):
        self.client = client
    
    async def run(self, messages, tools=None):
        iteration = 0
        max_iterations = 10
        
        while iteration < max_iterations:
            iteration += 1
            
            # 调用 LLM
            response = await self.client.get_response(
                messages=messages,
                tools=tools,
            )
            
            # 如果没有工具调用，返回文本
            if not response.tool_calls:
                return response.text
            
            # 执行工具调用
            for tool_call in response.tool_calls:
                result = await execute_tool(tool_call)
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": result,
                })
        
        raise RuntimeError("Max iterations reached")
```

### 进阶版本

加会话管理、中间件、遥测、存储模式处理。参考 Chrys 的 `agent.py`。

---

## 下一步

- **上一篇**: F14 vendor — 第三方二进制打包（Foundation 层完成！）
- **下一篇**: K02 loop — 工具循环（Kernel 层的核心）
