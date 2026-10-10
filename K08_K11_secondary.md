# K08-K11: tools/sessions/exceptions/instrumentation — 辅助模块

> **文件路径**: 
> - `src/chrys/kernel/tools.py` (~500 行)
> - `src/chrys/kernel/sessions.py` (~400 行)
> - `src/chrys/kernel/exceptions.py` (~100 行)
> - `src/chrys/kernel/instrumentation.py` (~300 行)
> 
> **一句话**: 工具定义与会话管理的基础设施，异常处理和遥测层的实现。

---

## K08: tools.py — 工具定义与规范化

### 核心类

```python
class FunctionTool:
    """A tool that can be invoked by the agent."""
    
    def __init__(
        self,
        name: str,
        description: str,
        parameters: dict[str, Any] | type[BaseModel],
        fn: Callable[..., Any] | None = None,
    ):
        self.name = name
        self.description = description
        self.parameters = parameters
        self._fn = fn
    
    async def invoke(
        self,
        arguments: BaseModel | dict[str, Any],
        **kwargs: Any,
    ) -> Content:
        """Execute the tool and return a result content."""
        
        # 参数验证
        if isinstance(self.parameters, type):
            validated_args = self.parameters(**arguments)
        else:
            validated_args = arguments
        
        # 执行函数
        if self._fn is None:
            raise RuntimeError(f"Tool {self.name} has no implementation")
        
        try:
            result = await self._fn(**validated_args)
            
            return Content.from_function_result(
                call_id=kwargs.get("call_id"),
                result=result,
            )
        
        except Exception as e:
            # 工具异常转为错误结果
            return Content.from_function_result(
                call_id=kwargs.get("call_id"),
                result=f"Error: {e}",
                is_error=True,
            )


def normalize_tools(tools: Any) -> list[FunctionTool]:
    """Normalize various tool representations to a list of FunctionTool.
    
    Accepts:
    - Single FunctionTool
    - List of FunctionTool
    - Dict with 'name', 'description', 'parameters'
    - Callable (auto-generates schema from signature)
    """
    
    if tools is None:
        return []
    
    if isinstance(tools, FunctionTool):
        return [tools]
    
    if callable(tools) and not isinstance(tools, list):
        # 单个 callable，自动包装
        return [FunctionTool.from_callable(tools)]
    
    result = []
    for tool in tools:
        if isinstance(tool, FunctionTool):
            result.append(tool)
        elif isinstance(tool, dict):
            result.append(FunctionTool(**tool))
        elif callable(tool):
            result.append(FunctionTool.from_callable(tool))
    
    return result
```

---

## K09: sessions.py — Agent 会话管理

### 核心类

```python
@dataclass
class AgentSession:
    """Session state for one agent run."""
    
    conversation_id: str | None = None
    continuation_token: Any = None
    history: list[Message] = field(default_factory=list)
    metadata: dict[str, Any] = field(default_factory=dict)
    
    before_providers: list[SessionProvider] = field(default_factory=list)
    after_providers: list[SessionProvider] = field(default_factory=list)


class SessionProvider(Protocol):
    """Hook that runs before or after an agent run."""
    
    async def __call__(
        self,
        context: SessionContext,
        response: ChatResponse | None = None,
    ) -> None: ...


@dataclass
class SessionContext:
    """Read-only view of session state passed to providers."""
    
    session: AgentSession
    messages: Sequence[Message]
    options: Mapping[str, Any]
    _response: ChatResponse | None = None
    
    @property
    def response(self) -> ChatResponse | None:
        return self._response


class HistoryProvider(Protocol):
    """Load and save conversation history."""
    
    async def load_history(self, session: AgentSession) -> list[Message]: ...
    
    async def save_history(
        self,
        session: AgentSession,
        messages: Sequence[Message],
    ) -> None: ...


class InMemoryHistoryProvider:
    """Simple in-memory history storage."""
    
    def __init__(self):
        self._histories: dict[str, list[Message]] = {}
    
    async def load_history(self, session: AgentSession) -> list[Message]:
        return self._histories.get(session.conversation_id or "", [])
    
    async def save_history(
        self,
        session: AgentSession,
        messages: Sequence[Message],
    ) -> None:
        key = session.conversation_id or ""
        self._histories[key] = list(messages)
```

---

## K10: exceptions.py — 异常定义

### 核心异常

```python
class ChrysKernelException(Exception):
    """Base exception for kernel-layer errors."""
    pass


class ModelVisibleToolError(ChrysKernelException):
    """Tool error whose message is safe to show to the model.
    
    Unlike generic exceptions (which become "Error: Function failed."),
    this exception's message is forwarded verbatim to the LLM so it can
    reason about what went wrong.
    """
    
    def __init__(self, message: str):
        super().__init__(message)
        self.message = message


class AgentInvalidResponseException(ChrysKernelException):
    """LLM returned a malformed response."""
    
    def __init__(self, response: ChatResponse, reason: str):
        super().__init__(f"Invalid response: {reason}")
        self.response = response
        self.reason = reason


GENERIC_TOOL_ERROR_TEXT = "Error: Function failed."
"""Fallback text when a tool raises a non-ModelVisibleToolError."""
```

**关键设计**：
- **ModelVisibleToolError**：工具可以抛出这个异常，让 LLM 看到具体的错误信息
- **其他异常**：统一转为 "Error: Function failed."，防止泄露敏感信息

---

## K11: instrumentation.py — 遥测层

### 核心类

```python
class AgentTelemetryLayer:
    """Adds OpenTelemetry spans around _AgentCore methods."""
    
    AGENT_PROVIDER_NAME: ClassVar[str] = "chrys"
    
    async def run(self, *args, **kwargs):
        """Wrap agent.run() with telemetry."""
        
        with tracer.start_as_current_span(
            "agent.run",
            attributes={
                "gen_ai.system": self.AGENT_PROVIDER_NAME,
                "gen_ai.agent.name": getattr(self, "agent_name", "unknown"),
            },
        ) as span:
            try:
                result = await super().run(*args, **kwargs)
                
                # 记录使用统计
                if hasattr(result, "usage"):
                    span.set_attribute("gen_ai.usage.input_tokens", result.usage.input_tokens)
                    span.set_attribute("gen_ai.usage.output_tokens", result.usage.output_tokens)
                
                return result
            
            except Exception as e:
                span.set_attribute("error", True)
                span.set_attribute("error.message", str(e))
                raise


class ChatTelemetryLayer:
    """Adds OpenTelemetry spans around raw LLM calls."""
    
    def __init__(self, inner: BaseChatClient):
        self.inner = inner
    
    async def get_response(self, messages, **kwargs):
        """Wrap get_response() with telemetry."""
        
        model = kwargs.get("model", "unknown")
        
        with tracer.start_as_current_span(
            "chat.completion",
            attributes={
                "gen_ai.system": self.inner.OTEL_PROVIDER_NAME,
                "gen_ai.request.model": model,
            },
        ) as span:
            result = await self.inner.get_response(messages, **kwargs)
            
            # 记录响应统计
            if hasattr(result, "usage"):
                span.set_attribute("gen_ai.usage.input_tokens", result.usage.input_tokens)
                span.set_attribute("gen_ai.usage.output_tokens", result.usage.output_tokens)
            
            return result
```

---

## 📊 总结

| 模块 | 职责 | 行数 |
|------|------|------|
| **tools.py** | 工具定义、参数验证、结果封装 | ~500 |
| **sessions.py** | 会话状态、历史提供者 | ~400 |
| **exceptions.py** | 异常定义、错误消息控制 | ~100 |
| **instrumentation.py** | OpenTelemetry 集成 | ~300 |

这些模块相对独立，功能明确，没有太多复杂的交互逻辑。

---

## 下一步

- **上一篇**: K07 compaction — 上下文压缩策略
- **下一篇**: K12-K18 合并文档（私有数据类型和辅助模块）
