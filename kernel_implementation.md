# Kernel 层详细实现解析

> **TL;DR**: Kernel 层是 Chrys 的核心创新所在。本文深入解析四个关键实现的源码细节：Exchange Grammar、Echo Removal、Invocation Order Stamps、LoopRecorder。每个都包含完整的源码注释和设计决策分析。

---

## 1. Exchange Grammar（交换语法）—— 如何判断工具调用已回答

### 问题背景

考虑这个对话历史：

```python
messages = [
    {"role": "user", "content": "帮我重构这个函数"},
    
    # Assistant 发起了两个工具调用
    {"role": "assistant", "content": [
        {"type": "function_call", "call_id": "C1", "name": "read_file", "args": {"path": "main.py"}},
        {"type": "function_call", "call_id": "C2", "name": "grep", "args": {"pattern": "def refactor_me"}},
    ]},
    
    # Tool results
    {"role": "tool", "content": [
        {"type": "function_result", "call_id": "C1", "result": "def refactor_me(): ..."},
    ]},
    {"role": "tool", "content": [
        {"type": "function_result", "call_id": "C2", "result": "Found 1 match"},
    ]},
    
    # Assistant 继续
    {"role": "assistant", "content": "我看到了这个函数..."},
]
```

**问题**：如何知道 C1 和 C2 都被回答了？如果压缩后 message ID 变了怎么办？

### Exchange Grammar 的核心思想

不依赖 ID，而是用**结构化的语法规则**来判断。

### 源码实现详解

#### Step 1: 定义 Accessor Protocol

```python
# kernel/exchanges.py

class MessageAccessor(Protocol):
    """访问者模式：抽象出消息的读取方式"""
    
    def role(self, message: Any) -> str: ...
    def contents(self, message: Any) -> Sequence[Any]: ...
    def content_type(self, content: Any) -> str | None: ...
    def raw_id(self, content: Any) -> object: ...
    def is_informational(self, content: Any) -> bool: ...
    def error_kind(self, content: Any) -> str | None: ...
    def has_marker(self, message: Any) -> bool: ...
    def call_types(self) -> frozenset[str]: ...
    def result_types(self) -> frozenset[str]: ...
```

**为什么用 Protocol？**
- 可以处理两种不同的消息表示：
  - `LiveAccessor`：处理实时的 `Message`/`Content` 对象
  - `DictAccessor`：处理序列化的 dict（用于恢复旧会话）
- 同一套配对逻辑，不同的数据源

#### Step 2: 实现 LiveAccessor

```python
class LiveAccessor:
    """Accessor over live kernel Message/Content objects."""
    
    def role(self, message: Any) -> str:
        return message.role
    
    def contents(self, message: Any) -> Sequence[Any]:
        return message.contents
    
    def content_type(self, content: Any) -> str | None:
        return content.type
    
    def raw_id(self, content: Any) -> object:
        # 图像生成用 image_id，其他用 call_id
        if content.type in _IMAGE_CONTENT_TYPES:
            return content.image_id
        return content.call_id
    
    def is_informational(self, content: Any) -> bool:
        return bool(content.informational_only)
    
    def error_kind(self, content: Any) -> str | None:
        kind = content.additional_properties.get(TOOL_ERROR_KIND_METADATA_KEY)
        return kind if isinstance(kind, str) else None
    
    def has_marker(self, message: Any) -> bool:
        # KEY presence, not value truthiness
        # 即使 marker 值是 falsy，也算有标记
        return HistoryMarkerKind.KEY in message.additional_properties
    
    def call_types(self) -> frozenset[str]:
        return TOOL_CALL_CONTENT_TYPES
    
    def result_types(self) -> frozenset[str]:
        return TOOL_RESULT_CONTENT_TYPES
```

**关键设计点**：

1. **命名空间隔离**：图像生成用 `image_id`，其他用 `call_id`
   ```python
   PairingKey = tuple[Literal["call_id", "image_id"], str]
   ```
   这样即使 `image_id == "abc"` 和 `call_id == "abc"`，也不会误配。

2. **Marker 检测用 key presence**：
   ```python
   return HistoryMarkerKind.KEY in message.additional_properties
   ```
   而不是 `bool(message.additional_properties.get(KEY))`。因为即使 marker 值是 `False`，也要视为硬边界。

#### Step 3: 定义 Exchange 数据结构

```python
@dataclass(frozen=True)
class Occurrence:
    """一个调用或结果的坐标"""
    message_index: int      # 消息在 history 中的索引
    content_index: int      # content 在消息中的索引
    raw_id: str | None      # 原始 ID（可能是 None、空字符串、或非字符串）
    content_type: str       # 内容类型
    error_kind: str | None  # 错误类型（如果有）

@dataclass(frozen=True)
class Exchange:
    """一个交换：响应运行 + 回答它的工具输出"""
    response_indices: tuple[int, ...]    # assistant 消息的索引
    output_indices: tuple[int, ...]       # tool result 消息的索引
    embedded_results: tuple[Occurrence, ...]  # 嵌入式结果（在 assistant 消息中的 result）
```

**为什么要区分 `response_indices` 和 `output_indices`？**

考虑这个场景：

```
msg[0]: user "hi"
msg[1]: assistant [call C1]     ← response
msg[2]: tool [result R1]        ← output
msg[3]: assistant [call C2]     ← 新的 response（新 exchange 开始）
msg[4]: tool [result R2]        ← output
```

Exchange 1: `response_indices=(1,)`, `output_indices=(2,)`
Exchange 2: `response_indices=(3,)`, `output_indices=(4,)`

#### Step 4: 遍历所有 Exchange

```python
def iter_exchanges(accessor: MessageAccessor, messages: Sequence[Any]) -> Iterator[Exchange]:
    """遍历所有交换"""
    
    i = 0
    while i < len(messages):
        msg = messages[i]
        
        # 跳过硬边界（history marker）
        if accessor.has_marker(msg):
            i += 1
            continue
        
        # 如果是 assistant 消息且包含工具调用
        if accessor.role(msg) == "assistant" and _has_tool_calls(accessor, msg):
            # 收集连续的 assistant 兄弟消息
            response_start = i
            response_indices = []
            
            while i < len(messages):
                current = messages[i]
                
                # 遇到硬边界就停
                if accessor.has_marker(current):
                    break
                
                # 只收集 assistant 消息
                if accessor.role(current) != "assistant":
                    break
                
                # 检查是否有工具调用
                if _has_tool_calls(accessor, current):
                    response_indices.append(i)
                    i += 1
                else:
                    # 纯文本的 assistant 也属于这个 exchange
                    response_indices.append(i)
                    i += 1
            
            # 收集后续的 tool result
            output_start = i
            output_indices = []
            
            while i < len(messages):
                current = messages[i]
                
                # 遇到硬边界就停
                if accessor.has_marker(current):
                    break
                
                # 只收集 tool result
                if accessor.role(current) == "tool" or _is_result_only_assistant(accessor, current):
                    output_indices.append(i)
                    i += 1
                else:
                    break
            
            # 收集嵌入式结果（在 assistant 消息中的 result）
            embedded_results = []
            for idx in response_indices:
                msg = messages[idx]
                for content_idx, content in enumerate(accessor.contents(msg)):
                    if accessor.content_type(content) in accessor.result_types():
                        embedded_results.append(Occurrence(
                            message_index=idx,
                            content_index=content_idx,
                            raw_id=accessor.raw_id(content),
                            content_type=accessor.content_type(content) or "",
                            error_kind=accessor.error_kind(content),
                        ))
            
            yield Exchange(
                response_indices=tuple(response_indices),
                output_indices=tuple(output_indices),
                embedded_results=tuple(embedded_results),
            )
        else:
            i += 1
```

**关键设计点**：

1. **硬边界检测**：`has_marker` 检查 key 是否存在，而不是值是否为真
2. **嵌入式结果**：某些 MCP server 会以 assistant-role 返回结果，这些结果嵌入在 assistant 消息中
3. **连续 assistant 兄弟**：多个 assistant 消息如果没有被 tool result 打断，属于同一个 exchange

#### Step 5: 配对策略

```python
@dataclass(frozen=True)
class PairingPolicy:
    """配对策略：决定如何处理各种边界情况"""
    
    call_types: frozenset[str]              # 哪些 content type 算调用
    include_informational_calls: bool       # 是否包含信息性调用
    result_types: frozenset[str]            # 哪些 content type 算结果
    none_id: NoneIdPolicy                   # 如何处理 None ID
    empty_id: EmptyIdPolicy                 # 如何处理空字符串 ID
    malformed_id: Literal["stringify", "treat_as_none"]  # 如何处理畸形 ID

class NoneIdPolicy(Enum):
    IGNORE = "ignore"                       # 忽略
    UNPAIRABLE_OCCURRENCE = "unpairable"    # 标记为无法配对
    POSITIONAL_FAILURES = "positional_failures"  # 按位置匹配失败
    POSITIONAL = "positional"               # 按位置匹配

def pair_results(exchange: Exchange, policy: PairingPolicy, accessor: MessageAccessor, messages: Sequence[Any]) -> ExchangePairing:
    """根据策略配对调用和结果"""
    
    pairing = ExchangePairing()
    
    # Phase 1: 注册所有调用
    for msg_idx in exchange.response_indices:
        msg = messages[msg_idx]
        for content_idx, content in enumerate(accessor.contents(msg)):
            content_type = accessor.content_type(content)
            
            if content_type not in policy.call_types:
                continue
            
            if accessor.is_informational(content) and not policy.include_informational_calls:
                continue
            
            # 获取并规范化 ID
            raw_id = accessor.raw_id(content)
            normalized_id = _normalize_id(raw_id, policy.malformed_id)
            
            if normalized_id is None:
                # 处理 None ID
                if policy.none_id == NoneIdPolicy.IGNORE:
                    continue
                elif policy.none_id == NoneIdPolicy.UNPAIRABLE_OCCURRENCE:
                    pairing.unpairable_calls.append(...)
                    continue
            
            # 注册调用
            key = (content_type, normalized_id)
            pairing.register_call(key, Occurrence(...))
    
    # Phase 2: 消费所有结果
    for msg_idx in exchange.output_indices:
        msg = messages[msg_idx]
        for content_idx, content in enumerate(accessor.contents(msg)):
            content_type = accessor.content_type(content)
            
            if content_type not in policy.result_types:
                continue
            
            raw_id = accessor.raw_id(content)
            normalized_id = _normalize_id(raw_id, policy.malformed_id)
            
            # 尝试消费调用
            key = (content_type, normalized_id)
            if not pairing.consume_result(key, Occurrence(...)):
                # 没有匹配的调用，是孤儿结果
                pairing.unconsumed_results[key].append(...)
    
    return pairing
```

**关键设计点**：

1. **策略驱动**：不同消费者可以声明不同的规则
2. **ID 规范化**：`_normalize_id` 根据策略处理畸形 ID
3. **两阶段处理**：先注册所有调用，再消费结果

### 实际使用示例

```python
# 压缩模块使用 Exchange Grammar

def compact_messages(messages: list[Message], budget: int) -> list[Message]:
    """压缩消息到预算内"""
    
    # 遍历所有 exchange
    for exchange in iter_exchanges(LiveAccessor(), messages):
        # 配对调用和结果
        pairing = pair_results(exchange, COMPACT_POLICY, LiveAccessor(), messages)
        
        # 找到未回答的调用
        unanswered_calls = [
            call for key, calls in pairing.truthy_assignments.items()
            for call, result in calls
            if result is None
        ]
        
        # 保留未回答的调用及其对应的结果
        # 压缩已完成的 exchange
        ...
```

---

## 2. Echo Removal（回声消除）—— 如何移除 LLM 回显的历史内容

### 问题背景

某些 LLM（特别是自部署的模型）会在响应中包含历史内容：

```python
# 发送给 LLM
messages = [
    {"role": "user", "content": "hi"},
    {"role": "assistant", "content": "hello"},  # 历史
    {"role": "user", "content": "how are you?"},
]

# LLM 返回（错误地包含了历史）
response = {
    "role": "assistant",
    "content": [
        {"type": "text", "text": "hello"},  # ← 这是历史的回声！
        {"type": "text", "text": "I'm fine, thanks!"},
    ]
}
```

### Echo Removal 的核心思想

用**对象身份追踪**，在流式更新的最内层移除对话已持有的 Content。

### 源码实现详解

#### Step 1: 初始化 Echo Registry

```python
# kernel/loop.py

async def _process_streaming_response(
    self, 
    stream: ResponseStream,
    history: list[Message],
):
    # 初始化回声记忆
    echo_registry = WeakIdentityRegistry(ignore_usage=True)
    
    # 注册历史中的所有 content
    for msg in history:
        for content in msg.contents:
            echo_registry.register(content)
    
    # 注册 update filter（在最内层执行）
    stream = stream.with_update_filter(
        lambda update: _strip_echoed_update(update, echo_registry)
    )
    
    # 处理流式更新
    async for update in stream:
        # 此时 update 已经被过滤过
        ...
```

**关键点**：
- `ignore_usage=True`：usage content 不参与回声消除
- `with_update_filter`：在流的最内层过滤，防止组装时洗白身份

#### Step 2: `_strip_echoed_update` 实现

```python
def _strip_echoed_update(
    update: ChatResponseUpdate,
    echo_registry: WeakIdentityRegistry,
) -> ChatResponseUpdate:
    """移除对话已持有的 content 对象"""
    
    # 检查是否有任何 content 在 registry 中
    if any(item in echo_registry for item in update.contents):
        # 浅拷贝 update（不修改原对象）
        update = copy(update)
        
        # 标记此 update 被剥离过（用于溯源）
        update._chrys_echo_content_stripped = True
        
        # 过滤掉在 registry 中的 content
        update.contents = [
            item for item in update.contents 
            if item not in echo_registry
        ]
    
    return update
```

**关键设计点**：

1. **浅拷贝而非深拷贝**：
   ```python
   update = copy(update)
   ```
   只拷贝 update 对象本身，contents 列表是新创建的，但 content 对象是共享的。

2. **溯源标记**：
   ```python
   update._chrys_echo_content_stripped = True
   ```
   私有属性，不会序列化，用于后续清理空消息壳。

3. **对象身份检查**：
   ```python
   if item not in echo_registry
   ```
   这里用的是 `WeakIdentityRegistry.__contains__`，检查的是对象身份（`ref() is obj`），不是相等性。

#### Step 3: 清理空消息壳

```python
def _remove_echo_emptied_message_shells(response: ChatResponse) -> None:
    """移除因回声剥离而变空的消息"""
    
    response.messages = [
        message for message in response.messages 
        if message.contents or not message._chrys_echo_content_stripped
    ]
```

**为什么需要这一步？**

如果一个 update 的所有 content 都是回声，过滤后 `update.contents` 为空。组装后会得到一个空消息。这个空消息应该被移除。

**为什么不直接检查 `not message.contents`？**

因为有些消息本来就是空的（合法的空消息）。只有因为回声剥离而变空的消息才应该被移除。`_chrys_echo_content_stripped` 标记区分了这两种情况。

#### Step 4: 注册新的 Content

```python
def _record_stream_fragment_identities(
    stream: ResponseStream,
    echo_registry: WeakIdentityRegistry,
) -> None:
    """记录每个接受的 stream content 的身份"""
    
    # 当 stream 被消费完后，注册所有接受的 content
    async def on_complete():
        for msg in stream.response.messages:
            for content in msg.contents:
                echo_registry.register(content)
    
    stream.on_complete(on_complete)
```

**为什么要在 stream 完成后才注册？**

因为 assembly 会把多个 fragment 合并成新的 content 对象。如果在 fragment 级别注册，合并后的新对象不会被识别。

**流程**：
1. Stream yield fragment A（旧的，在 registry 中）
2. `_strip_echoed_update` 移除 A
3. Stream yield fragment B（新的，不在 registry 中）
4. Assembly 合并 B 和其他 fragment 成新的 content C
5. Stream 完成后，注册 C 到 registry
6. 下一轮如果 LLM 回显 C，会被正确移除

### 实际效果示例

```python
# 第 1 轮
history = [
    Message(role="user", contents=[Content(type="text", text="hi")]),
    Message(role="assistant", contents=[Content(type="text", text="hello")]),
]

# 第 2 轮，发送给 LLM
provider_messages = [
    {"role": "user", "content": "hi"},
    {"role": "assistant", "content": "hello"},  # 历史
    {"role": "user", "content": "how are you?"},
]

# LLM 返回（流式）
stream = [
    ChatResponseUpdate(contents=[Content(text="hello")]),  # 回声！
    ChatResponseUpdate(contents=[Content(text="I'm fine")]),
]

# 经过 _strip_echoed_update
filtered_stream = [
    ChatResponseUpdate(contents=[]),  # 被清空
    ChatResponseUpdate(contents=[Content(text="I'm fine")]),
]

# 经过 _remove_echo_emptied_message_shells
final_response = ChatResponse(
    messages=[
        Message(role="assistant", contents=[Content(text="I'm fine")])
    ]
)
```

---

## 3. Invocation Order Stamps（调用顺序戳）—— 如何保证调用顺序不被破坏

### 问题背景

LLM 可能返回重复的 call_id：

```python
response = ChatResponse(
    contents=[
        Content(type="function_call", call_id="C1", name="read_file"),
        Content(type="function_call", call_id="C2", name="grep"),
        Content(type="function_call", call_id="C1", name="read_file"),  # 重复！
    ]
)
```

如果分发过滤器丢弃了第三个调用，但后面的模块按出现顺序编号 0, 1，就会出错。

### Invocation Order Stamps 的核心思想

Kernel 循环是 `_chrys_tool_invocation_order` 的**唯一生产者**。在响应到达时立即打戳，包括被过滤掉的调用。

### 源码实现详解

#### `_land_response_contents` 实现

```python
def _land_response_contents(
    response: ChatResponse,
    next_ordinal: int,
    echo_registry: WeakIdentityRegistry,
    *,
    exchange_operation_id: str | None = None,
) -> int:
    """Normalize a landed response's contents and stamp fresh function calls.
    
    This loop is the single authoritative producer of 
    _chrys_tool_invocation_order.
    
    Landing happens at response arrival — before recording, dispatch 
    extraction, tool lookup, and argument validation — over every 
    non-informational function_call content in model emission order.
    
    Calls the dispatch filter drops (already-answered, duplicate ids, 
    no-tools batches) and calls that fail pre-pipeline validation still 
    consume ordinals, so persisted numbering can never diverge from 
    pipeline numbering.
    """
    
    ordinal = next_ordinal
    
    for message in response.messages:
        for content in message.contents:
            # 只对非信息性的 function_call 打戳
            if content.type == "function_call" and not content.informational_only:
                # 无论分发过滤器是否丢弃，都消费序数
                content.additional_properties[TOOL_INVOCATION_ORDER_KEY] = ordinal
                
                # 注册到 echo registry（用于后续回声消除）
                echo_registry.register(content)
                
                ordinal += 1
            
            # 其他类型的 content 不打戳，但要注册到 echo registry
            elif content.type != "usage":
                echo_registry.register(content)
    
    return ordinal
```

**关键设计点**：

1. **唯一生产者**：
   ```python
   # This loop is the single authoritative producer
   ```
   只有这一个地方会设置 `_chrys_tool_invocation_order`。

2. **在响应到达时打戳**：
   ```python
   # Landing happens at response arrival — before recording, dispatch 
   # extraction, tool lookup, and argument validation
   ```
   在任何过滤之前。所以即使后面的过滤器丢弃了这个调用，序数已经被消费了。

3. **覆盖已有戳记**：
   
   如果一个 content 已经有 `_chrys_tool_invocation_order`，但不是当前 run 设置的（比如从缓存中恢复的），会被覆盖。
   
   ```python
   # Sole-authority consequences: a pre-existing stamp on a content this 
   # run has NOT numbered is foreign data (a bridged or cached response 
   # replaying another run's history) and is overwritten
   ```

4. **Usage content 豁免**：
   ```python
   elif content.type != "usage":
       echo_registry.register(content)
   ```
   Usage content 不参与回声消除，也不打序数戳。

### 实际效果示例

```python
# LLM 返回
response = ChatResponse(
    contents=[
        Content(type="function_call", call_id="C1", name="read_file"),
        Content(type="function_call", call_id="C2", name="grep"),
        Content(type="function_call", call_id="C1", name="read_file"),  # 重复
    ]
)

# 经过 _land_response_contents
# 假设 next_ordinal = 5

for content in response.contents:
    if content.type == "function_call":
        content.additional_properties["_chrys_tool_invocation_order"] = ordinal
        ordinal += 1

# 结果：
# contents[0]._chrys_tool_invocation_order = 5
# contents[1]._chrys_tool_invocation_order = 6
# contents[2]._chrys_tool_invocation_order = 7  # 即使重复也有序数

# 后续分发过滤器可能丢弃 contents[2]（因为 call_id 重复）
# 但序数 7 已经被消费了，下一个响应的第一个调用会从 8 开始
```

### 为什么这么重要？

后续模块依赖这个序数：

```python
# 压缩模块
for content in message.contents:
    order = read_tool_invocation_order(content)
    # 如果序数不连续，压缩可能会出错

# 审计重放
for event in trajectory_events:
    if event.type == "tool_call":
        order = event.metadata["_chrys_tool_invocation_order"]
        # 必须保证顺序正确

# 轨迹分析
for tool_call in tool_calls:
    order = tool_call.order
    # 用于统计和分析
```

---

## 4. LoopRecorder（中断恢复）—— 如何处理中断恢复

### 问题背景

工具执行过程中，用户可能按下 Ctrl+C：

```python
# 工具执行
try:
    result = await execute_tool("delete_file", {"path": "important.txt"})
except KeyboardInterrupt:
    # 用户取消了
    # 但工具可能已经执行了一半
    raise
```

下次重试时，不知道从哪里开始。

### LoopRecorder 的核心思想

在每次 wire 调用前记录快照，在响应后密封。支持 snapshot/restore 用于 whole-run 重试。

### 源码实现详解

#### 数据结构

```python
@dataclass
class _PendingExchange:
    """待处理的交换"""
    call: FunctionCall
    result: FunctionResult | None = None
    sealed: bool = False
    
    def seal(self, result: FunctionResult) -> None:
        self.result = result
        self.sealed = True

@dataclass
class _SealedExchange:
    """已密封的交换"""
    call: FunctionCall
    result: FunctionResult

class LoopRecorderSnapshot:
    """重试快照"""
    initial_count: int | None
    captured: tuple[Message, ...] | None
    service_loop_messages: tuple[Message, ...]
    last_checkpoint_key: tuple[int, str] | None
```

#### LoopRecorder 类

```python
class LoopRecorder:
    """Records tool-loop messages for interrupt recovery."""
    
    def __init__(
        self,
        *,
        capture_service_loop_messages: bool = False,
        on_checkpoint: Callable[[], Coroutine[Any, Any, None]] | None = None,
        on_pre_wire_barrier: Callable[[], Awaitable[RecoveryPersistOutcome]] | None = None,
        on_result_checkpoint: Callable[[], Coroutine[Any, Any, None]] | None = None,
        message_hasher: Callable[[Message], str] | None = None,
    ) -> None:
        self._capture_service_loop_messages = capture_service_loop_messages
        self._on_result_checkpoint = on_result_checkpoint or on_checkpoint
        self._on_pre_wire_barrier = on_pre_wire_barrier
        self._message_hasher = message_hasher or repr
        
        # 状态
        self._initial_count: int | None = None
        self._captured: list[Message] | None = None
        self._service_loop_messages: list[Message] = []
        self._last_checkpoint_key: tuple[int, str] | None = None
        self._sealed_exchanges: list[_SealedExchange] = []
        self._pending_exchange: _PendingExchange | None = None
        self._landed_response: tuple[Message, ...] | None = None
        self._committed_count = 0
        self._barrier_degraded = False
        self._barrier_unconfigured = False
        self._barrier_warned = False
    
    def reset(self) -> None:
        """Clear captured state for a new run."""
        self._initial_count = None
        self._captured = None
        self._service_loop_messages = []
        self._last_checkpoint_key = None
        self._sealed_exchanges = []
        self._pending_exchange = None
        self._landed_response = None
        self._committed_count = 0
        self._barrier_degraded = False
        self._barrier_warned = False
    
    def snapshot(self) -> LoopRecorderSnapshot:
        """Capture recorder state for an outer provider retry."""
        return LoopRecorderSnapshot(
            initial_count=self._initial_count,
            captured=None if self._captured is None else tuple(self._captured),
            service_loop_messages=tuple(self._service_loop_messages),
            last_checkpoint_key=self._last_checkpoint_key,
        )
    
    def restore(self, snapshot: LoopRecorderSnapshot) -> None:
        """Restore capture state while preserving every answered journal slot."""
        self._initial_count = snapshot.initial_count
        self._captured = None if snapshot.captured is None else list(snapshot.captured)
        self._service_loop_messages = list(snapshot.service_loop_messages)
        self._last_checkpoint_key = snapshot.last_checkpoint_key
        # The retried attempt sends its request again.
        # Sealed exchanges are preserved across retries.
```

#### 关键方法

```python
def record_pre_call(self, call: FunctionCall, ordinal: int) -> None:
    """Record before a wire call."""
    
    # 创建 pending exchange
    self._pending_exchange = _PendingExchange(call=call)
    
    # 触发 pre-wire barrier（如果有）
    if self._on_pre_wire_barrier:
        # 异步回调，用于持久化状态
        asyncio.create_task(self._on_pre_wire_barrier())

def record_response(self, response: ChatResponse) -> None:
    """Record after a parsed response."""
    
    # 如果有 pending exchange，密封它
    if self._pending_exchange:
        for content in response.contents:
            if content.type == "function_result":
                if content.call_id == self._pending_exchange.call.call_id:
                    self._pending_exchange.seal(content)
                    break
        
        # 移动到 sealed 列表
        if self._pending_exchange.sealed:
            self._sealed_exchanges.append(_SealedExchange(
                call=self._pending_exchange.call,
                result=self._pending_exchange.result,
            ))
            self._pending_exchange = None
            self._committed_count += 1
            
            # 触发 checkpoint 回调
            if self._on_result_checkpoint:
                asyncio.create_task(self._on_result_checkpoint())
    
    # 记录 landed response
    self._landed_response = tuple(response.messages)

@property
def loop_messages(self) -> list[Message]:
    """Get all recorded messages."""
    messages = []
    
    # 添加 sealed exchanges
    for exchange in self._sealed_exchanges:
        messages.append(exchange.call.to_message())
        messages.append(exchange.result.to_message())
    
    # 添加 landed response
    if self._landed_response:
        messages.extend(self._landed_response)
    
    return messages
```

### 实际使用流程

```python
# 主循环
recorder = LoopRecorder()

for iteration in range(max_iterations):
    # 调用 LLM
    response = await llm.chat(messages, tools)
    
    # 记录响应
    recorder.record_response(response)
    
    # 如果没有工具调用，结束
    if not response.tool_calls:
        break
    
    # 执行工具调用
    for tool_call in response.tool_calls:
        # 记录预调用
        recorder.record_pre_call(tool_call, ordinal)
        
        try:
            # 执行工具
            result = await execute_tool(tool_call)
            
            # 添加工具结果到消息
            messages.append(result.to_message())
        
        except KeyboardInterrupt:
            # 用户取消
            # recorder 保存了当前状态
            raise

# 如果需要重试
snapshot = recorder.snapshot()

# ... 稍后 ...

recorder.restore(snapshot)
# 从上次中断的地方继续
```

### 关键设计点

1. **Pending vs Sealed**：
   - Pending：调用已发出，等待结果
   - Sealed：结果已收到，不可撤销

2. **Snapshot/Restore**：
   - Snapshot：捕获当前状态，用于重试
   - Restore：恢复状态，但保留已密封的交换

3. **Checkpoint 回调**：
   - `on_pre_wire_barrier`：在 wire 调用前触发，用于持久化
   - `on_result_checkpoint`：在结果密封后触发，用于持久化

4. **Barrier 降级**：
   - 如果 barrier 回调失败，标记为 degraded
   - 后续跳过 barrier，但不影响核心功能

---

## 5. 总结：Kernel 层的核心价值

### 四大创新的协同

```
┌─────────────────────────────────────────────────────────┐
│                   Kernel 层架构                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  LLM Response 到达                                      │
│       ↓                                                 │
│  ① _land_response_contents                              │
│     - 打上调用顺序戳                                    │
│     - 注册到 echo registry                              │
│       ↓                                                 │
│  ② _strip_echoed_update (流式)                          │
│     - 在最内层移除回声                                  │
│     - 防止组装时洗白身份                                │
│       ↓                                                 │
│  ③ iter_exchanges + pair_results                        │
│     - 识别 exchange 边界                                │
│     - 策略驱动的配对                                    │
│       ↓                                                 │
│  ④ LoopRecorder                                         │
│     - 记录 pending/sealed exchanges                     │
│     - 支持 snapshot/restore                             │
│       ↓                                                 │
│  继续工具循环或返回最终响应                             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 学习建议

1. **先理解 Exchange Grammar**：这是基础，其他都依赖它
2. **再看 Echo Removal**：理解对象身份追踪的重要性
3. **然后看 Invocation Order Stamps**：理解为什么需要唯一生产者
4. **最后看 LoopRecorder**：理解中断恢复的机制

### 如果你要自己实现

**最小版本**（1-2 天）：
- 简单的 while 循环 + 按 `call_id` 配对
- 基本的错误处理

**进阶版本**（1-2 周）：
- 加 Exchange Grammar（结构化配对）
- 加 Echo Removal（对象身份追踪）
- 加 Invocation Order Stamps（唯一生产者）

**生产级版本**（几个月）：
- 完整的 LoopRecorder（中断恢复）
- 策略驱动的配对
- 与压缩、审计、轨迹的深度集成

---

## 附录：关键源码文件索引

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| `kernel/exchanges.py` | ~800 | Exchange Grammar 定义和实现 |
| `kernel/identity.py` | 96 | WeakIdentityRegistry |
| `kernel/loop.py` | ~3700 | 工具循环、回声消除、调用顺序戳、LoopRecorder |
| `kernel/client.py` | ~1500 | Wire client 抽象、存储模式解析 |
| `kernel/compaction.py` | ~1200 | 分组注解、LastWordsCompleter |

---

**下一步**：想看 Service 层的实现吗？那里有更实用的东西：LLM Client Factory、MCP Cache、四阶段压缩、双进程工作流 SDK。
