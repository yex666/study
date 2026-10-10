# K06: middleware — 中间件管道系统

> **文件路径**: `src/chrys/kernel/middleware.py` (~600 行)
> **一句话**: 提供 ChatMiddleware 和 FunctionMiddleware 两种中间件抽象，支持在 LLM 调用前后和工具执行前后插入自定义逻辑。

---

## 这是什么？

Chrys 的中间件系统分为两类：

1. **ChatMiddleware**：包装每次 LLM 调用（在 ChatMiddlewareLayer 中执行）
2. **FunctionMiddleware**：包装每次工具调用（在 ToolLoopLayer 中执行）

```
Agent.run()
    ↓
ToolLoopLayer (执行 FunctionMiddleware)
    ↓
ChatMiddlewareLayer (执行 ChatMiddleware)
    ↓
Raw LLM Client
```

---

## 核心设计

### 1. 中间件基类

```python
class ChatMiddleware(ABC):
    """Intercepts one model call."""
    
    @abstractmethod
    async def __call__(
        self, 
        context: ChatContext, 
        call_next: Callable[[ChatContext], Awaitable[None]]
    ) -> None:
        """Process the chat context, then call call_next to continue the chain."""
        ...


class FunctionMiddleware(ABC):
    """Intercepts one tool invocation."""
    
    @abstractmethod
    async def __call__(
        self,
        context: FunctionInvocationContext,
        call_next: Callable[[FunctionInvocationContext], Awaitable[None]]
    ) -> None:
        """Process the function invocation, then call call_next to continue."""
        ...
```

### 2. 上下文对象

```python
class ChatContext:
    """Mutable context for one chat middleware invocation."""
    
    messages: list[Message]           # 可修改的消息列表
    options: dict[str, Any]           # LLM 调用选项
    result: ChatResponse | ResponseStream | None  # 最终结果
    
    async def call_next(self) -> None:
        """Continue to the next middleware or the actual LLM call."""
        ...


class FunctionInvocationContext:
    """Context for one tool invocation through the function middleware pipeline."""
    
    function: FunctionTool            # 被调用的工具
    arguments: BaseModel | dict       # 工具参数
    metadata: dict[str, Any]          # 元数据
    result: Any                       # 工具执行结果
    session: AgentSession | None      # 会话上下文
    tools: list[ToolTypes] | None     # 当前可用工具列表
    
    def add_tools(self, tools) -> None:
        """动态添加工具到下一次 LLM 调用"""
        ...
    
    def remove_tools(self, tools) -> None:
        """从下一次 LLM 调用中移除工具"""
        ...
```

### 3. 中间件管道

```python
class ChatMiddlewarePipeline:
    """Executes chat middleware as a nested chain."""
    
    def __init__(self, *middleware: ChatMiddleware):
        self._middleware = list(middleware)
    
    async def execute(self, context: ChatContext, final_handler: Callable) -> None:
        """Execute the pipeline, ending with final_handler."""
        
        async def call_next(ctx: ChatContext, index: int) -> None:
            if index >= len(self._middleware):
                await final_handler(ctx)
            else:
                await self._middleware[index](ctx, lambda c: call_next(c, index + 1))
        
        await call_next(context, 0)


class FunctionMiddlewarePipeline:
    """Executes function middleware as a nested chain."""
    
    def __init__(self, *middleware: FunctionMiddleware):
        self._middleware = list(middleware)
    
    async def execute(self, context: FunctionInvocationContext) -> Any:
        """Execute the pipeline and return the result."""
        
        async def call_next(ctx: FunctionInvocationContext, index: int) -> None:
            if index >= len(self._middleware):
                # 实际执行工具
                ctx.result = await ctx.function.invoke(ctx.arguments, **ctx.kwargs)
            else:
                await self._middleware[index](ctx, lambda c: call_next(c, index + 1))
        
        await call_next(context, 0)
        return context.result
```

### 4. 终止机制

```python
class MiddlewareTermination(Exception):
    """Control-flow exception that terminates middleware execution early."""
    
    def __init__(self, message: str = "", *, result: Any = None) -> None:
        super().__init__(message)
        self.result = result


# ChatMiddleware 中的终止
async def my_chat_middleware(context: ChatContext, call_next):
    if some_condition:
        # 设置结果并终止
        context.result = ChatResponse(...)
        raise MiddlewareTermination("Early exit", result=context.result)
    
    await call_next(context)


# FunctionMiddleware 中的终止
async def my_function_middleware(context: FunctionInvocationContext, call_next):
    if should_skip_tool:
        # 设置结果并终止（传播到 ToolLoopLayer）
        raise MiddlewareTermination("Skip tool", result="skipped")
    
    await call_next(context)
```

**关键区别**：
- **ChatMiddleware 终止**：ChatMiddlewarePipeline 捕获异常，返回 `context.result`
- **FunctionMiddleware 终止**：异常传播到 ToolLoopLayer，由它处理

---

## 实际应用示例

### 示例 1: 审批中间件

```python
class ApprovalMiddleware(FunctionMiddleware):
    """Ask user for approval before executing dangerous tools."""
    
    def __init__(self, approval_policy: ApprovalPolicy):
        self.policy = approval_policy
    
    async def __call__(self, context, call_next):
        tool_name = context.function.name
        args = context.arguments
        
        # 检查是否需要审批
        if await self.policy.requires_approval(tool_name, args):
            # 询问用户
            approved = await ask_user(f"Execute {tool_name}({args})?")
            
            if not approved:
                raise MiddlewareTermination(
                    "User denied",
                    result="Tool execution denied by user"
                )
        
        # 继续执行工具
        await call_next(context)
```

### 示例 2: 日志中间件

```python
class LoggingMiddleware(ChatMiddleware):
    """Log all LLM calls."""
    
    async def __call__(self, context, call_next):
        logger.info(f"LLM call: {len(context.messages)} messages")
        
        start_time = time.time()
        await call_next(context)
        duration = time.time() - start_time
        
        if isinstance(context.result, ChatResponse):
            tokens = context.result.usage.total_tokens
            logger.info(f"LLM response: {tokens} tokens in {duration:.2f}s")
```

### 示例 3: 压缩中间件

```python
class CompactionMiddleware(ChatMiddleware):
    """Compress message history if it exceeds token budget."""
    
    def __init__(self, strategy: CompactionStrategy, tokenizer: Tokenizer):
        self.strategy = strategy
        self.tokenizer = tokenizer
    
    async def __call__(self, context, call_next):
        # 计算当前 token 数
        total_tokens = sum(
            self.tokenizer.count_tokens(msg.text)
            for msg in context.messages
        )
        
        # 如果超出预算，执行压缩
        if total_tokens > self.strategy.budget:
            context.messages = await self.strategy.compact(context.messages)
        
        await call_next(context)
```

### 示例 4: 渐进式工具暴露

```python
class ProgressiveToolMiddleware(FunctionMiddleware):
    """Dynamically add/remove tools based on context."""
    
    async def __call__(self, context, call_next):
        # 根据工具名称动态添加相关工具
        if context.function.name == "read_file":
            # 添加相关的搜索工具
            context.add_tools([grep_tool, glob_tool])
        
        # 执行工具
        await call_next(context)
        
        # 根据结果决定是否保留某些工具
        if "error" in str(context.result).lower():
            context.remove_tools(["dangerous_tool"])
```

---

## 与其他模块的关系

```
middleware.py 依赖：
├── _types.py (ChatResponse, Message, etc.)
├── client.py (SupportsAclose)
└── tools.py (FunctionTool)

被以下模块使用：
├── agent.py (组合中间件到 Agent)
├── loop.py (ToolLoopLayer 执行 FunctionMiddleware)
└── service/agent_middleware/ (各种具体中间件实现)
```

---

## 下一步

- **上一篇**: K02 loop — 工具循环层
- **下一篇**: K07 compaction — 上下文压缩策略
