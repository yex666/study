# K02: loop — 工具循环层（Tool Loop Layer）

> **文件路径**: `src/chrys/kernel/loop.py` (~3700 行)
> **一句话**: Chrys 最核心的创新——工具调用循环、回声消除、中断恢复、续接令牌管理。

---

## 这是什么？

**ToolLoopLayer** 是 Kernel 层的心脏。它在 ChatMiddlewareLayer 之上，负责：

1. **驱动工具循环**：LLM 返回工具调用 → 执行 → 把结果发回 LLM → 重复
2. **管理回声消除**：追踪对话中已有的 Content 对象，移除 LLM 回显的历史
3. **处理中断恢复**：记录每次 wire 调用的状态，支持从崩溃中恢复
4. **管理续接令牌**：适配有状态（服务端存储）和无状态（本地历史）提供者
5. **集成轨迹追踪**：记录每个 model cycle 和 tool operation 的时间线

---

## 架构位置

```
AgentTelemetryLayer (遥测)
    ↓
_AgentCore (编排)
    ↓
ToolLoopLayer ⭐ (工具循环 + 回声消除 + 中断恢复)
    ↓
ChatMiddlewareLayer (中间件管道)
    ↓
Instrumented Raw Client (实际 LLM 调用)
```

---

## 核心设计：五个关键机制

### 机制 1: 主迭代循环

```python
async def _get_response() -> ChatResponse:
    """Non-streaming implementation."""
    
    # ... 准备上下文 ...
    
    errors_in_a_row = 0
    next_tool_ordinal = 0
    fcc_messages: list[Message] = []  # FCC = function call context
    
    for _attempt_idx in range(self.max_iterations):
        # 1. 调用 LLM（带重试）
        response = await _blocking_response_with_retry(
            prepped_messages,
            request_message_observer=wire_request_observer,
        )
        
        # 2. 落地响应内容（打序数戳、注册到 echo registry）
        next_tool_ordinal = _land_response_contents(
            response, 
            next_tool_ordinal, 
            echo_registry,
        )
        
        # 3. 记录到 LoopRecorder（用于中断恢复）
        if recorder is not None:
            recorder.record_response(response)
        
        # 4. 提取工具调用
        function_calls = _extract_function_calls(response)
        
        # 5. 如果没有工具调用，返回响应
        if not (function_calls and mutable_options.get("tools")):
            _prepend_fcc_messages(response, fcc_messages)
            return response
        
        # 6. 执行工具调用（并发）
        results, should_terminate, had_errors = await execute_batch(
            function_calls,
            result_carrier_item_id=new_analytics_id(),
        )
        
        # 7. 处理工具结果
        action, errors_in_a_row = _handle_function_call_results(
            response=response,
            function_call_results=results,
            fcc_messages=fcc_messages,
            echo_registry=echo_registry,
            errors_in_a_row=errors_in_a_row,
            had_errors=had_errors,
            max_errors=self.max_consecutive_errors,
        )
        
        # 8. 检查终止条件
        if should_terminate:
            # 中间件要求终止
            return response
        
        if action == "stop":
            # 连续错误达到上限，强制最后一次无工具调用
            mutable_options["tool_choice"] = "none"
        
        # 9. 准备下一轮的消息
        prepped_messages.extend(response.messages)
    
    # 10. 循环耗尽：最后一次无工具调用
    mutable_options["tool_choice"] = "none"
    response = await _blocking_response_with_retry(prepped_messages)
    _strip_unexecutable_calls_from_response(response)
    return response
```

**关键点**：
- **max_iterations**：默认 40 次迭代，防止无限循环
- **max_consecutive_errors**：默认 3 次连续错误后强制停止
- **FCC messages**：累积的工具调用上下文，最终会 prepend 到响应中

---

### 机制 2: 回声消除（Echo Removal）

```python
def _land_response_contents(
    response: ChatResponse,
    next_ordinal: int,
    echo_registry: WeakIdentityRegistry,
) -> int:
    """Landing happens at response arrival — before recording, dispatch, validation."""
    
    ordinal = next_ordinal
    
    for message in response.messages:
        for content in message.contents:
            # 1. 给 function_call 打上序数戳
            if content.type == "function_call" and not content.informational_only:
                content.additional_properties[TOOL_INVOCATION_ORDER_KEY] = ordinal
                ordinal += 1
            
            # 2. 注册到 echo registry（除了 usage）
            if content.type != "usage":
                echo_registry.register(content)
    
    return ordinal


def _strip_echoed_update(
    update: ChatResponseUpdate,
    echo_registry: WeakIdentityRegistry,
) -> ChatResponseUpdate:
    """Filter echoed content objects from stream updates BEFORE assembly."""
    
    if any(item in echo_registry for item in update.contents):
        # 浅拷贝 update，避免修改原对象
        update = copy(update)
        update._chrys_echo_content_stripped = True
        
        # 过滤掉已在对话中的 content
        update.contents = [
            item for item in update.contents 
            if item not in echo_registry
        ]
    
    return update
```

**为什么在组装前过滤？**

流式更新会被组装器合并（例如连续的文本片段合并成一个）。如果在组装后过滤，合并产生的新对象身份不同于原始对象，无法被 echo_registry 识别。所以在最内层（组装前）过滤。

---

### 机制 3: 中断恢复（LoopRecorder）

```python
class LoopRecorder:
    """Records tool-loop messages for interrupt recovery."""
    
    def __init__(self):
        self._pending_exchange: _PendingExchange | None = None
        self._sealed_exchanges: list[_SealedExchange] = []
        self._committed_count = 0
    
    def record_pre_call(self, call: FunctionCall, ordinal: int) -> None:
        """Record before each wire call."""
        self._pending_exchange = _PendingExchange(call=call)
    
    def record_response(self, response: ChatResponse) -> None:
        """Record after each parsed response."""
        for content in response.contents:
            if content.type == "function_result":
                if self._pending_exchange and content.call_id == self._pending_exchange.call.call_id:
                    self._pending_exchange.seal(content)
                    self._sealed_exchanges.append(_SealedExchange(...))
                    self._pending_exchange = None
                    self._committed_count += 1
    
    def snapshot(self) -> LoopRecorderSnapshot:
        """Capture state for whole-run retry."""
        return LoopRecorderSnapshot(
            sealed_exchanges=tuple(self._sealed_exchanges),
            committed_count=self._committed_count,
        )
    
    def restore(self, snapshot: LoopRecorderSnapshot) -> None:
        """Restore state while preserving answered slots."""
        self._sealed_exchanges = list(snapshot.sealed_exchanges)
        self._committed_count = snapshot.committed_count
```

**关键设计**：
- **PendingExchange**：工具调用已发出，等待结果
- **SealedExchange**：结果已收到，不可撤销
- **committed_count**：已完成的结果数，whole-run 重试时不会重放

---

### 机制 4: 续接令牌管理

```python
def _update_continuation_state(
    client_kwargs: dict,
    response: ChatResponse,
    session: AgentSession | None,
    options: dict,
) -> None:
    """Mirror the live continuation token into the retry owner's request state."""
    
    # 从响应中提取续接令牌
    token = extract_continuation_token(response)
    
    if token is not None:
        # 存储到 options 中，下次请求时使用
        options["continuation_token"] = token
        
        # 如果是会话存储模式，还要更新 session
        if session is not None:
            session.continuation_token = token
```

**两种存储模式**：
- **Service-side**：OpenAI Responses API 等服务端存储历史，使用 continuation_token
- **Client-side**：本地存储历史，不使用 continuation_token

---

### 机制 5: 工具执行与结果处理

```python
async def _execute_function_calls(
    *,
    function_calls: Sequence[Content],
    pipeline: FunctionMiddlewarePipeline,
    tool_map: dict[str, FunctionTool],
    ...
) -> tuple[list[Content], bool, bool]:
    """Execute a batch concurrently and collect results."""
    
    # 并发执行所有工具调用
    tasks = [
        asyncio.create_task(
            _invoke_function_call(
                function_call,
                tool_map=tool_map,
                pipeline=pipeline,
            )
        )
        for function_call in function_calls
    ]
    
    settled = await asyncio.gather(*tasks, return_exceptions=True)
    
    # 处理结果
    contents = []
    should_terminate = False
    had_errors = False
    
    for outcome in settled:
        if isinstance(outcome, BaseException):
            if isinstance(outcome, MiddlewareTermination):
                # 中间件要求终止
                contents.append(outcome.result)
                should_terminate = True
            else:
                raise outcome
        else:
            result_content = outcome
            if result_content.exception is not None:
                had_errors = True
            contents.append(result_content)
    
    return contents, should_terminate, had_errors


def _handle_function_call_results(
    *,
    response: ChatResponse,
    function_call_results: list[Content],
    fcc_messages: list[Message],
    echo_registry: WeakIdentityRegistry,
    errors_in_a_row: int,
    had_errors: bool,
    max_errors: int,
) -> tuple[str, int]:
    """Fold tool results back into the response.
    
    Returns (action, errors_in_a_row) where action is "continue" or "stop".
    """
    
    # 创建工具结果消息
    result_message = Message(
        role="tool",
        contents=function_call_results,
    )
    
    # 注册到 echo registry
    for content in function_call_results:
        echo_registry.register(content)
    
    # 追加到 FCC 消息列表
    fcc_messages.append(result_message)
    
    # 更新连续错误计数
    if had_errors:
        errors_in_a_row += 1
    else:
        errors_in_a_row = 0
    
    # 检查是否达到上限
    if errors_in_a_row >= max_errors:
        return "stop", errors_in_a_row
    
    return "continue", errors_in_a_row
```

---

## 流式 vs 非流式

### 非流式路径

```python
async def _get_response() -> ChatResponse:
    """Blocking path: returns complete response after all iterations."""
    
    for _attempt_idx in range(self.max_iterations):
        # 一次性获取完整响应
        response = await _blocking_response_with_retry(prepped_messages)
        
        # 处理工具调用...
        
        if no_tool_calls:
            return response
    
    # 最后一次无工具调用
    return await _blocking_response_with_retry(prepped_messages, tool_choice="none")
```

### 流式路径

```python
async def _stream() -> AsyncIterable[ChatResponseUpdate]:
    """Streaming path: yields updates as they arrive."""
    
    final_messages: list[Message] = []
    
    for _attempt_idx in range(self.max_iterations):
        # 流式获取响应
        logical_stream = _streaming_response_with_retry(
            prepped_messages,
            stream_update_filter=echo_filter,  # ← 回声消除在这里
        )
        
        # 组装流式更新为完整响应
        async for update in logical_stream:
            yield update  # ← 实时产出给调用者
        
        response = _assemble_from_updates(updates)
        
        # 处理工具调用...
        
        if no_tool_calls:
            return
    
    # 最后一次无工具调用
    async for update in _streaming_response_with_retry(...):
        yield update
```

**关键区别**：
- 非流式：等待完整响应后才处理
- 流式：实时更新产出给调用者，但最终也要组装成完整响应来处理工具调用

---

## 与其他模块的关系

```
loop.py 依赖：
├── _types.py (ChatResponse, ChatResponseUpdate, Message, Content)
├── identity.py (WeakIdentityRegistry)
├── exchanges.py (TOOL_CALL_CONTENT_TYPES, TOOL_RESULT_CONTENT_TYPES)
├── tools.py (FunctionTool, normalize_tools)
├── middleware.py (FunctionMiddlewarePipeline, MiddlewareTermination)
├── sessions.py (AgentSession)
└── exceptions.py (ModelVisibleToolError)

被以下模块使用：
├── agent.py (Agent 组合 ToolLoopLayer)
└── orchestration/engine/build/builder.py (构建 Agent 时创建 ToolLoopLayer)
```

---

## 如果你要自己实现

### 最小版本（50 行）

```python
class SimpleToolLoop:
    def __init__(self, client, max_iterations=10):
        self.client = client
        self.max_iterations = max_iterations
    
    async def run(self, messages, tools=None):
        for iteration in range(self.max_iterations):
            # 调用 LLM
            response = await self.client.get_response(messages, tools=tools)
            
            # 提取工具调用
            tool_calls = [c for c in response.contents if c.type == "function_call"]
            
            if not tool_calls:
                return response
            
            # 执行工具调用
            for tool_call in tool_calls:
                result = await self.execute_tool(tool_call)
                messages.append(Message(role="tool", contents=[result]))
        
        # 最后一次无工具调用
        return await self.client.get_response(messages, tool_choice="none")
```

### 进阶版本需要的功能

1. **回声消除**：WeakIdentityRegistry 追踪对话中的 Content 对象
2. **中断恢复**：LoopRecorder 记录 pending/sealed exchanges
3. **续接令牌**：支持有状态和无状态提供者
4. **错误处理**：连续错误计数、中间件终止
5. **轨迹追踪**：记录每个 cycle 和 operation 的时间线
6. **流式支持**：实时更新产出 + 最终组装

参考 Chrys 的 `loop.py` 实现。

---

## 📚 延伸阅读

- **KERNEL_DEEP_DIVE.md**：概念性的解释，适合理解"为什么需要这些机制"
- **kernel_implementation.md**：更详细的源码解析，包含具体的代码注释

本文档侧重于"实际代码是怎么组织的"，配合上述文档可以获得完整的理解。

---

## 下一步

- **上一篇**: K01 agent — Agent 入口点
- **下一篇**: K03 client — Wire client 抽象（简短版）
