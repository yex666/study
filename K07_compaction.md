# K07: compaction — 上下文压缩策略与分组注解

> **文件路径**: `src/chrys/kernel/compaction.py` (~1200 行)
> **一句话**: 定义压缩策略接口、消息分组逻辑、token 估算、投影原子性保证。四阶段压缩算法在 Service 层实现。

---

## 这是什么？

Kernel 层提供压缩的**原语和基础设施**：

1. **CompactionStrategy Protocol**：压缩策略接口（具体实现在 Service 层）
2. **group_messages()**：计算消息分组（哪些消息应该一起压缩）
3. **annotate_message_groups()**：给消息打上组标签
4. **annotate_token_counts()**：估算每条消息的 token 数
5. **project_included_messages()**：计算压缩后的消息列表
6. **CompactionProjectionAtomicityError**：投影原子性错误

**注意**：四阶段压缩算法（Phase 1-4）在 `service/context/compaction/strategy.py` 中实现，本文档只讲 Kernel 层的原语。

---

## 核心设计

### 1. CompactionStrategy Protocol

```python
class CompactionStrategy(Protocol):
    """Compression strategy that mutates messages in place."""
    
    async def __call__(
        self,
        messages: list[Message],
        *,
        context: "CompactionCallContext",
    ) -> bool:
        """Compress messages to fit within token budget.
        
        Returns True if any compression was performed.
        """
        ...


@dataclass(frozen=True)
class CompactionCallContext:
    """Minimal context passed to the compression strategy."""
    
    last_words_completer: LastWordsCompleter | None
    tool_definition_tokens: int
    request_overhead_tokens: int
    next_exchange_operation_id: str | None
```

**关键设计**：
- **最小化上下文**：策略不需要知道 client、options 等细节
- **LastWordsCompleter**：用于 Phase 4 生成"最后的话"摘要
- **token 估算**：工具定义和请求开销的 token 数

### 2. 消息分组

```python
def group_messages(messages: Sequence[Message]) -> list[MessageGroup]:
    """Compute message groups for atomic compression.
    
    Rules:
    - Exchange boundaries come from iter_exchanges()
    - Spans are a secondary partition of each exchange
    - Calls and results must share ONE group
      (otherwise partial exclusion could send a result without its call)
    """
    
    groups = []
    current_group = []
    
    for i, msg in enumerate(messages):
        # 检查是否是交换边界
        if _is_exchange_boundary(msg, i, messages):
            if current_group:
                groups.append(MessageGroup(current_group))
            current_group = [msg]
        else:
            current_group.append(msg)
    
    if current_group:
        groups.append(MessageGroup(current_group))
    
    return groups


@dataclass(frozen=True)
class MessageGroup:
    """A group of messages that must be compressed together."""
    
    messages: tuple[Message, ...]
    
    @property
    def total_tokens(self) -> int:
        return sum(msg.token_count or 0 for msg in self.messages)
    
    def annotate_group_id(self, group_id: str) -> None:
        """Stamp all messages with the same group ID."""
        for msg in self.messages:
            msg.additional_properties[GROUP_ID_KEY] = group_id
```

### 3. Token 估算

```python
class Tokenizer(Protocol):
    """Token counting interface."""
    
    def count_tokens(self, text: str) -> int: ...


def annotate_token_counts(
    messages: Sequence[Message],
    tokenizer: Tokenizer,
) -> None:
    """Estimate token counts for each message and stamp them."""
    
    for msg in messages:
        if msg.token_count is not None:
            continue  # Already annotated
        
        total = 0
        
        # 角色开销
        total += _count_role_overhead(msg.role)
        
        # 内容 token
        for content in msg.contents:
            total += _count_content_tokens(content, tokenizer)
        
        msg.token_count = total


def _count_content_tokens(content: Content, tokenizer: Tokenizer) -> int:
    """Estimate tokens for one content item."""
    
    if content.type == "text":
        return tokenizer.count_tokens(content.text or "")
    
    elif content.type == "function_call":
        # 函数名 + 参数 JSON
        name_tokens = tokenizer.count_tokens(content.name or "")
        args_tokens = tokenizer.count_tokens(json.dumps(content.arguments or {}))
        return name_tokens + args_tokens + 10  # 结构开销
    
    elif content.type == "function_result":
        return tokenizer.count_tokens(str(content.result or ""))
    
    elif content.type == "image":
        # 图像按固定大小估算
        return _estimate_image_tokens(content)
    
    else:
        return 0


def _estimate_image_tokens(content: Content) -> int:
    """Estimate tokens for an image based on dimensions."""
    
    width = content.width or 512
    height = content.height or 512
    
    # OpenAI Vision API 的估算公式
    # https://platform.openai.com/docs/guides/vision/calculating-costs
    tiles_w = math.ceil(width / 512)
    tiles_h = math.ceil(height / 512)
    total_tiles = tiles_w * tiles_h
    
    return 85 + 170 * total_tiles
```

### 4. 投影原子性

```python
class CompactionProjectionAtomicityError(Exception):
    """Raised when a partial projection would break call-result pairing."""
    
    def __init__(self, group_id: str, missing_calls: list[str]):
        super().__init__(
            f"Cannot exclude group {group_id}: missing calls {missing_calls}. "
            f"This would leave results without their calls."
        )
        self.group_id = group_id
        self.missing_calls = missing_calls


def project_included_messages(
    messages: Sequence[Message],
    excluded_group_ids: set[str],
) -> list[Message]:
    """Compute the projected message list after excluding certain groups.
    
    Raises CompactionProjectionAtomicityError if exclusion would break pairing.
    """
    
    # 收集所有被排除的调用
    excluded_calls = set()
    for msg in messages:
        group_id = msg.additional_properties.get(GROUP_ID_KEY)
        if group_id in excluded_group_ids:
            for content in msg.contents:
                if content.type == "function_call":
                    excluded_calls.add(content.call_id)
    
    # 检查是否有结果引用了被排除的调用
    for msg in messages:
        group_id = msg.additional_properties.get(GROUP_ID_KEY)
        if group_id not in excluded_group_ids:
            # 这个消息会被保留
            for content in msg.contents:
                if content.type == "function_result":
                    if content.call_id in excluded_calls:
                        raise CompactionProjectionAtomicityError(
                            group_id,
                            [content.call_id],
                        )
    
    # 安全，执行投影
    return [
        msg for msg in messages
        if msg.additional_properties.get(GROUP_ID_KEY) not in excluded_group_ids
    ]
```

### 5. LastWordsCompleter

```python
class LastWordsCompleter(Protocol):
    """Generate a 'last words' summary before compressing away the current turn."""
    
    async def complete(
        self,
        messages: Sequence[Message],
        *,
        dropped_work: Sequence[Message],
        previous_note: str | None = None,
    ) -> str:
        """Generate a natural-language summary of dropped work.
        
        Args:
            messages: Current conversation history
            dropped_work: Messages about to be compressed away
            previous_note: Previous last-words note (for accumulation)
        
        Returns:
            Natural-language summary text
        """
        ...
```

**用途**：Phase 4 压缩时，用 LLM 生成一段"最后的话"，保留关键信息。

---

## 与其他模块的关系

```
compaction.py 依赖：
├── _types.py (Message, Content)
├── exchanges.py (iter_exchanges)
└── foundation/text/tokenizer.py (MixedLanguageTokenizer)

被以下模块使用：
├── client.py (BaseChatClient 调用压缩策略)
├── service/context/compaction/strategy.py (四阶段压缩实现)
└── loop.py (ToolLoopLayer 检查是否需要压缩)
```

---

## 📚 延伸阅读

- **service_implementation.md** 第 3 节：四阶段压缩的详细解析
- **KERNEL_DEEP_DIVE.md**：压缩的整体设计思路

---

## 下一步

- **上一篇**: K04+K05 合并概述
- **下一篇**: K08-K11 合并文档（tools/sessions/exceptions/instrumentation）
