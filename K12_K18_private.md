# K12-K18: 私有类型与辅助模块

> **文件路径**: `src/chrys/kernel/_*.py`（7 个私有模块）
> **一句话**: Kernel 层的数据类型定义（Message/Content）、序列化、参数验证、结果截断等基础设施。

---

## 为什么叫"私有"？

`_` 开头的模块是 **kernel 内部私有**的：
- **外部（kernel 之外）**：必须通过 `from chrys.kernel import ...` 导入公开 API
- **内部**：`_*.py` 之间可以互相导入
- **例外**：`_KERNEL_PRIVATE_IMPORT_ALLOWLIST` 中有白名单的少量导入

```python
# ✅ 正确的导入方式
from chrys.kernel import Message, Content, ChatResponse

# ❌ 错误（在 kernel 之外）
from chrys.kernel._types import Message
```

---

## K12: _types.py (~2500 行) — 核心类型定义

### Message

```python
class Message(SerializationMixin):
    """A single message in the conversation history."""
    
    def __init__(
        self,
        role: str,                    # "user" / "assistant" / "tool" / "system"
        contents: Sequence[Content] | None = None,
        additional_properties: dict[str, Any] | None = None,
        ...
    ):
        self.role = role
        self.contents = ContentList(contents or [])
        # ↑ 强制转换为 ContentList（支持弱引用）
        self.additional_properties = additional_properties or {}
    
    @property
    def text(self) -> str | None:
        """Concatenate all text contents."""
        texts = [
            c.text for c in self.contents
            if c.type == "text" and c.text
        ]
        return "\n".join(texts) if texts else None
    
    def to_dict(self) -> dict:
        """Serialize to dict (wire format)."""
        ...
    
    @classmethod
    def from_dict(cls, data: dict) -> "Message":
        """Deserialize from dict."""
        ...
```

### ChatResponse

```python
@dataclass
class ChatResponse(SerializationMixin):
    """A complete chat response from the LLM."""
    
    role: str = "assistant"
    contents: list[Content] = field(default_factory=list)
    finish_reason: str | None = None
    usage: UsageDetails | None = None
    messages: list[Message] = field(default_factory=list)
    conversation_id: str | None = None
    
    @classmethod
    def from_updates(cls, updates: Sequence[ChatResponseUpdate]) -> "ChatResponse":
        """Assemble a response from stream updates."""
        ...
```

### ChatResponseUpdate

```python
@dataclass
class ChatResponseUpdate:
    """A single stream update (delta) from the LLM."""
    
    type: str = "text"                    # "text" / "function_call" / ...
    delta: str = ""                       # 文本增量
    contents: list[Content] = field(default_factory=list)
    ...
```

### ResponseStream

```python
class ResponseStream(Generic[UpdateT, ResponseT]):
    """Async iterable that yields updates and produces a final response."""
    
    @classmethod
    def from_awaitable(
        cls,
        awaitable: Awaitable[ResponseT],
    ) -> "ResponseStream[ChatResponseUpdate, ChatResponse]":
        """Create a stream from an awaitable (non-streaming source)."""
        ...
    
    def with_update_filter(
        self,
        filter_fn: Callable[[UpdateT], UpdateT],
    ) -> "ResponseStream[UpdateT, ResponseT]":
        """Add a filter applied to every update (used for echo removal)."""
        ...
```

---

## K13: _content.py (~1800 行) — Content 类型

### Content 基类

```python
@dataclass(frozen=True)
class Content:
    """A single content item within a message."""
    
    type: str                                    # "text" / "function_call" / ...
    text: str | None = None                      # 文本内容
    call_id: str | None = None                   # 工具调用 ID
    name: str | None = None                      # 工具名称
    arguments: dict | None = None                # 工具参数
    result: Any = None                           # 工具结果
    additional_properties: dict[str, Any] = field(default_factory=dict)
    
    # --- 工厂方法 ---
    
    @classmethod
    def from_text(cls, text: str, **kwargs) -> "Content":
        return cls(type="text", text=text, **kwargs)
    
    @classmethod
    def from_function_call(cls, name: str, arguments: dict, call_id: str) -> "Content":
        return cls(
            type="function_call",
            name=name,
            arguments=arguments,
            call_id=call_id,
        )
    
    @classmethod
    def from_function_result(cls, call_id: str, result: Any, **kwargs) -> "Content":
        return cls(
            type="function_result",
            call_id=call_id,
            result=result,
            **kwargs,
        )
```

### 类型常量

```python
# 所有工具调用类型
TOOL_CALL_CONTENT_TYPES = frozenset({
    "function_call",
    "code_interpreter_tool_call",
    "image_generation_tool_call",
    "mcp_server_tool_call",
    "search_tool_call",
    "shell_tool_call",
})

# 所有工具结果类型
TOOL_RESULT_CONTENT_TYPES = frozenset({
    "function_result",
    "hosted_tool_result",
    "code_interpreter_tool_result",
    "image_generation_tool_result",
    "mcp_server_tool_result",
    "search_tool_result",
    "shell_tool_result",
})
```

### UsageDetails

```python
@dataclass
class UsageDetails:
    """Token usage statistics."""
    
    input_tokens: int = 0
    output_tokens: int = 0
    total_tokens: int = 0
    cache_read_tokens: int = 0
    cache_creation_tokens: int = 0
    reasoning_tokens: int = 0
    
    def add(self, other: "UsageDetails") -> None:
        """Aggregate another usage into this one."""
        self.input_tokens += other.input_tokens
        self.output_tokens += other.output_tokens
        ...
```

---

## K14: _serialization.py (~500 行) — 序列化

```python
class SerializationMixin:
    """Shared to_dict/from_dict contract."""
    
    def to_dict(self) -> dict[str, Any]:
        """Serialize to dict (wire format)."""
        ...
    
    @classmethod
    def from_dict(cls, data: dict[str, Any]) -> "Self":
        """Deserialize from dict."""
        ...
```

### 序列化规则

```python
# CamelCase → snake_case 转换
camel_to_snake = lambda s: re.sub(r"(?<!^)(?=[A-Z])", "_", s).lower()

# 例如：
# "functionCall" → "function_call"
# "toolUseId" → "tool_use_id"
# "finishReason" → "finish_reason"
```

---

## K15: _null_overlay.py (~1200 行) — Null 参数恢复

### 解决的问题

Pydantic 的 `model_dump(exclude_none=True)` 会**丢弃显式设置的 `None` 值**。

```python
class ReadFileArgs(BaseModel):
    path: str
    encoding: str | None = None

# 用户显式设置了 encoding=None（表示"使用默认编码"）
args = ReadFileArgs(path="main.py", encoding=None)

# Pydantic 序列化时丢弃 None
dumped = args.model_dump(exclude_none=True)
# → {"path": "main.py"}  ← encoding 丢了！

# LLM 不知道 encoding 参数存在，下次可能传错
```

### 解决方案：静态分类

```python
class NullOverlay:
    """Restore caller-explicit None tool arguments."""
    
    # 构建时将注解图分类为 FULL/TOP_LEVEL/OFF
    # - FULL: 所有层都恢复 None
    # - TOP_LEVEL: 只有顶层恢复 None
    # - OFF: 不恢复（保留 Pydantic 输出）
    
    def overlay(
        self,
        dumped: dict[str, Any],
        original: BaseModel,
    ) -> dict[str, Any]:
        """Restore explicit None values into the dumped dict."""
        
        for field_name, field_value in vars(original).items():
            if field_value is None and field_name not in dumped:
                # 根据分类决定是否恢复
                if self._policy_for(field_name) in (FULL, TOP_LEVEL):
                    dumped[field_name] = None
        
        return dumped
```

---

## K16: _tool_arg_errors.py (~2000 行) — 参数错误诊断

### 解决的问题

LLM 传了错误的工具参数时，要生成**模型可读**的错误信息：

```python
def argument_validation_message(
    exc: ValidationError,
    tool_name: str,
    *,
    max_items: int = 256,        # 最多比较多少项
    max_payload_scan: int = 10_000,  # 最多扫描多少 payload
) -> str:
    """Generate a compact, model-actionable diagnostic."""
    
    issues = []
    
    for error in exc.errors():
        loc = ".".join(str(x) for x in error["loc"])
        msg = error["msg"]
        
        issues.append(f"  - {loc}: {msg}")
    
    # 截断过长列表
    if len(issues) > 20:
        issues = issues[:20] + [f"  ... and {len(issues) - 20} more"]
    
    return (
        f"Argument validation failed for {tool_name}. "
        f"Correct the following:\n"
        + "\n".join(issues)
    )
```

**关键设计**：
- **有界预算**：防止敌意或指数级参数图导致死循环
- **安全指导**：只给 schema 建议，不回显完整参数 payload（防泄露）

---

## K17: _result_ceiling.py (~200 行) — 结果截断

```python
def apply_result_ceiling(
    content: Content,
    max_tokens: int | None,
) -> Content:
    """Idempotently truncate oversized tool results."""
    
    if max_tokens is None or content.type != "function_result":
        return content
    
    result_text = str(content.result or "")
    estimated_tokens = len(result_text) // 4  # 粗略估算
    
    if estimated_tokens <= max_tokens:
        return content
    
    # 保留开头和结尾
    head_tokens = max_tokens // 2
    tail_tokens = max_tokens - head_tokens
    
    head = result_text[: head_tokens * 4]
    tail = result_text[-tail_tokens * 4 :]
    
    truncated = Content.from_function_result(
        call_id=content.call_id,
        result=f"{head}\n\n[...truncated {estimated_tokens} tokens...]\n\n{tail}",
    )
    
    # 记录截断元数据
    truncated.additional_properties[TOOL_RESULT_METADATA_KEY] = {
        "truncated": True,
        "original_tokens": estimated_tokens,
        "max_tokens": max_tokens,
    }
    
    return truncated
```

---

## K18: _tool_expansion.py (~50 行) — 工具扩展缝隙

```python
# 进程全局缝隙，kernel 不导入 service 层 MCP 类型

_tool_expander: Callable | None = None

def _set_tool_expander(expander: Callable | None) -> None:
    """Set the process-global tool expander (called by service layer)."""
    global _tool_expander
    _tool_expander = expander


def expand_tools(tools: list[Any]) -> list[Any]:
    """Expand tool declarations using the registered expander."""
    if _tool_expander is None:
        return tools
    return _tool_expander(tools) or tools
```

---

## 📊 Kernel 层完成总结

| 序号 | 文件 | 状态 | 文档 |
|------|------|------|------|
| K01 | agent.py | ✅ | `K01_agent.md` |
| K02 | loop.py | ✅ | `K02_loop.md` |
| K03 | client.py | ✅ | `K03_client.md` |
| K04 | exchanges.py | ✅ | `K04_K05_overview.md` |
| K05 | identity.py | ✅ | `K04_K05_overview.md` |
| K06 | middleware.py | ✅ | `K06_middleware.md` |
| K07 | compaction.py | ✅ | `K07_compaction.md` |
| K08 | tools.py | ✅ | `K08_K11_secondary.md` |
| K09 | sessions.py | ✅ | `K08_K11_secondary.md` |
| K10 | exceptions.py | ✅ | `K08_K11_secondary.md` |
| K11 | instrumentation.py | ✅ | `K08_K11_secondary.md` |
| K12 | _types.py | ✅ | `K12_K18_private.md` |
| K13 | _content.py | ✅ | `K12_K18_private.md` |
| K14 | _serialization.py | ✅ | `K12_K18_private.md` |
| K15 | _null_overlay.py | ✅ | `K12_K18_private.md` |
| K16 | _tool_arg_errors.py | ✅ | `K12_K18_private.md` |
| K17 | _result_ceiling.py | ✅ | `K12_K18_private.md` |
| K18 | _tool_expansion.py | ✅ | `K12_K18_private.md` |

---

## 下一步

- **上一篇**: K08-K11 辅助模块
- **下一篇**: S01-S17（Service 层）——LLM 集成、MCP、Skills、Hooks 等
