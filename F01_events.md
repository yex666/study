# F01: events — 事件总线

> **模块路径**: `src/chrys/foundation/events/`
> **核心文件**: `bus.py`（144 行）, `types.py`（~400 行）
> **一句话**: 让 Agent 内部各个模块互相通信，不丢消息、不乱序。

---

## 这是什么？

一个**进程内的发布/订阅系统**。模块 A 发布事件，模块 B/C/D 订阅事件，互相不认识。

```
Agent Loop: "工具调用开始了！" → publish(ToolCallStartEvent)
                                ↓
UI 模块: subscribe(ToolCallStartEvent) → 显示进度条
日志模块: subscribe(ToolCallStartEvent) → 记录日志
历史模块: subscribe(ToolCallStartEvent) → 保存到历史
```

Agent Loop 不需要知道有 UI、日志、历史这些模块的存在。它只管喊一嗓子。

---

## 为什么需要？

### 如果不做会怎样？

```python
class Agent:
    async def run(self, user_input):
        response = await self.llm.chat(user_input)
        
        for tool_call in response.tool_calls:
            result = await self.execute_tool(tool_call)
            
            # 要通知 UI 更新
            self.ui.show_progress(tool_call.name)      # ← 紧耦合
            # 要记录日志
            self.logger.log(tool_call)                 # ← 紧耦合
            # 要保存历史
            self.history.save(tool_call, result)      # ← 紧耦合
            # 要追踪变更
            self.mutations.track(tool_call)           # ← 紧耦合
```

**问题**：
1. Agent 知道太多细节（每加一个监听者就要改 Agent 代码）
2. 测试困难（要 mock 所有依赖）
3. 如果 `self.ui.show_progress()` 抛异常，后面的日志和历史就不会执行

### 用了 EventBus 之后

```python
class Agent:
    def __init__(self, event_bus: EventBus):
        self.event_bus = event_bus
    
    async def run(self, user_input):
        response = await self.llm.chat(user_input)
        
        for tool_call in response.tool_calls:
            result = await self.execute_tool(tool_call)
            await self.event_bus.publish(ToolCallEvent(...))  # 只喊一嗓子
```

其他模块自己订阅：
```python
event_bus.subscribe(ToolCallEvent, ui.show_progress)
event_bus.subscribe(ToolCallEvent, logger.log)
event_bus.subscribe(ToolCallEvent, history.save)
```

---

## 核心设计：两个关键决策

### 决策 1: await handler（背压），不是 fire-and-forget

**问题**：handler 可能是异步的（比如写数据库要 100ms），如果 fire-and-forget，可能丢事件。

```python
# ❌ 危险的方式
async def publish(self, event):
    for handler in self.handlers:
        asyncio.create_task(handler(event))  # 发了就走
        # 问题1: 程序退出时任务可能没完成
        # 问题2: 异常被吞掉
        # 问题3: 顺序无法保证
```

```python
# ✅ Chrys 的方式
async def publish(self, event):
    for handler in self.handlers:
        await handler(event)  # 等它完成
        # 好处1: 不丢事件
        # 好处2: 保证顺序
        # 好处3: 错误能被捕获
```

**代价**：如果 handler 慢，publisher 会被阻塞。这是故意的——叫做**背压**（backpressure），让发布者意识到"下游处理不过来了"。

### 决策 2: 精确类型匹配，不是 isinstance

```python
class ToolCallEvent(Event): pass
class ShellToolCallEvent(ToolCallEvent): pass

bus.subscribe(ToolCallEvent, handler_a)
bus.subscribe(ShellToolCallEvent, handler_b)

# 发布 ShellToolCallEvent
await bus.publish(ShellToolCallEvent())
```

**isinstance 匹配**（Chrys 不用）：handler_a 和 handler_b 都被调用（因为 ShellToolCallEvent 是 ToolCallEvent 的子类）

**精确类型匹配**（Chrys 的做法）：只有 handler_b 被调用

```python
# Chrys 的实现
event_type = type(event)  # 获取精确类型
for handler in self._handlers.get(event_type, []):  # 只查这个类型
    await handler(event)
```

**为什么**：避免继承关系导致的意外触发。如果你订阅了 `ToolCallEvent`，你不希望收到所有子类的事件。

---

## 两种消费模式

Chrys 提供了两种订阅方式，适用于不同场景：

### 模式 1: subscribe（回调型，带背压）

适合：必须同步完成的操作（保存到数据库、写日志）

```python
async def save_to_db(event: ToolCallEvent):
    await database.insert(event.tool_name)
    # 必须等写入完成才能继续

await bus.subscribe(ToolCallEvent, save_to_db)

await bus.publish(ToolCallEvent(...))
# ↑ 这里会等 save_to_db 完成
```

### 模式 2: stream（流式型，无界队列）

适合：实时消费，允许慢的操作（UI 更新、流式输出）

```python
async def ui_loop():
    async with bus.stream(InvocationMessage) as stream:
        async for event in stream:
            print(event.text, end="")  # 实时显示 LLM 输出
```

**为什么 stream 用无界队列？**
- 有界队列满了要么丢事件，要么阻塞 publisher
- Chrys 选择：宁可内存暴涨，也不丢 LLM 输出
- 有高水位告警（10000 事件时警告一次）

---

## 源码解析

### EventBus 类（`bus.py` 第 29-91 行）

```python
class EventBus:
    def __init__(self) -> None:
        # handler 注册表：event_type -> [handler]
        self._handlers: dict[type[Event], list[Callable]] = defaultdict(list)
        # stream 消费者列表
        self._streams: list[_EventStream] = []
    
    async def publish(self, event: Event, *, raise_handler_errors: bool = False) -> None:
        """发布事件"""
        event_type = type(event)  # 精确类型
        failures: list[Exception] = []
        
        # 1. 通知所有 subscribe 的 handler（带背压）
        for handler in list(self._handlers.get(event_type, [])):
            # ↑ list() 拷贝：防止 handler 内部 unsubscribe 导致迭代错误
            try:
                await handler(event)  # ← 关键：await，不是 fire-and-forget
            except Exception as exc:
                failures.append(exc)
                logger.exception("EventBus handler failed for %s", event_type.__name__)
                # ↑ 记录错误但不中断其他 handler
        
        # 2. 通知所有 stream 消费者（无背压）
        for stream in list(self._streams):
            stream._deliver(event)  # ← 非阻塞，入队
        
        # 3. 可选：抛出错误（测试用）
        if raise_handler_errors and failures:
            if len(failures) == 1:
                raise failures[0]
            raise ExceptionGroup(f"EventBus handlers failed", failures)
    
    async def subscribe(self, event_type: type[E], handler: Callable[[E], Awaitable[None]]) -> None:
        """订阅事件"""
        self._handlers[event_type].append(handler)
    
    async def unsubscribe(self, event_type: type[E], handler: Callable) -> None:
        """取消订阅"""
        handlers = self._handlers.get(event_type, [])
        if handler in handlers:
            handlers.remove(handler)
    
    def stream(self, *event_types: type[Event]) -> _EventStream:
        """创建流式消费者"""
        return _EventStream(self, event_types)
```

### _EventStream 类（`bus.py` 第 94-144 行）

```python
class _EventStream:
    """Async iterator adapter for EventBus streaming."""
    
    def __init__(self, bus: EventBus, event_types: tuple[type[Event], ...]) -> None:
        self._bus = bus
        self._event_types = event_types
        # 无界队列：永不丢弃事件
        self._queue: asyncio.Queue[Event] = asyncio.Queue()
        self._warned_high_water = False
    
    def _deliver(self, event: Event) -> None:
        """入队事件（生产者侧过滤）"""
        # 在进入队列前就过滤，避免无关事件占内存
        if self._event_types and not isinstance(event, self._event_types):
            return  # 不匹配，不入队
        
        self._queue.put_nowait(event)  # 无界队列，永远不会满
        
        # 高水位告警（只告警一次）
        if not self._warned_high_water and self._queue.qsize() >= 10_000:
            self._warned_high_water = True
            logger.warning("Stream backlog exceeded 10000 events; consumer is falling behind")
    
    async def __aenter__(self) -> _EventStream:
        """进入 async with 时注册到 bus"""
        self._bus._streams.append(self)
        return self
    
    async def __aexit__(self, *exc: object) -> None:
        """退出 async with 时从 bus 移除"""
        self._bus._streams.remove(self)
    
    def __aiter__(self) -> AsyncIterator[Event]:
        return self
    
    async def __anext__(self) -> Event:
        """从队列取事件（阻塞直到有事件）"""
        return await self._queue.get()
```

### Event 基类（`types.py` 第 52-58 行）

```python
@dataclass
class Event:
    """Base class for all events."""
    
    event_id: str = field(default_factory=lambda: uuid4().hex[:12])
    # ↑ 唯一标识，用于追踪和去重
    
    timestamp: datetime = field(default_factory=lambda: datetime.now(tz=UTC))
    # ↑ 发生时间，带时区
    
    session_id: str | None = None
    # ↑ 会话归属，多会话共存时区分事件属于哪个会话
```

### 主要事件类型（`types.py`）

```python
@dataclass(kw_only=True)
class ExecutionChanged(Event):
    """执行状态变化（idle → turn → workflow）"""
    snapshot: ExecutionSnapshot

@dataclass(kw_only=True)
class InvocationEvent(Event):
    """一个执行/展示事实，带明确的逻辑所有者"""
    origin: InvocationOrigin  # 不可变的调用身份

@dataclass(kw_only=True)
class UserMessage(Event):
    """用户发送的消息"""
    text: str
    injection_id: str | None = None  # 中途注入的消息有特殊 ID

# ... 还有 Error、AgentProfileSwitch、SessionNew/Restore/Delete 等
```

---

## 实际使用场景

### 场景 1: TUI 实时更新

```python
class MainScreen:
    async def on_mount(self):
        await self.app.event_bus.subscribe(
            InvocationToolCallStart,
            self._on_tool_call_start
        )
    
    async def _on_tool_call_start(self, event):
        self.tool_card.update(event.tool_name, event.args)
```

### 场景 2: Headless 流式输出

```python
class ChrysSessionHost:
    async def iter_run_events(self, message: str):
        async with self.event_bus.stream(*_HEADLESS_RUN_EVENT_TYPES) as stream:
            run_task = asyncio.create_task(self.engine.run(message))
            async for event in stream:
                yield event
                if run_task.done():
                    break
```

### 场景 3: 日志记录

```python
class TrajectoryWriter:
    async def start(self):
        await self.event_bus.subscribe(InvocationMessage, self._on_message)
        await self.event_bus.subscribe(InvocationToolCallStart, self._on_tool_start)
    
    async def _on_message(self, event):
        self.file.write(f"[MESSAGE] {event.text}\n")
```

---

## 和其他模块的关系

```
EventBus 被以下模块使用：
├── AgentEngine（发布 Turn 开始/结束事件）
├── TurnCoordinator（发布 UserMessage、ExecutionChanged 事件）
├── ToolLoopLayer（发布 InvocationToolCallStart、InvocationToolResult 事件）
├── TUI MainScreen（订阅事件，更新 UI）
├── TrajectoryWriter（订阅事件，记录轨迹）
├── ChrysSessionHost（stream 事件，转发给 headless 消费者）
└── WorkflowCoordinator（发布 WorkflowRunAccepted 等事件）
```

---

## 如果你要自己实现

### 最小版本（20 行）

```python
class SimpleEventBus:
    def __init__(self):
        self.handlers = {}
    
    async def subscribe(self, event_type, handler):
        self.handlers.setdefault(event_type, []).append(handler)
    
    async def publish(self, event):
        for handler in list(self.handlers.get(type(event), [])):
            try:
                await handler(event)
            except Exception as e:
                print(f"Handler failed: {e}")
```

### 进阶版本（加 stream 支持）

参考 Chrys 的 `_EventStream`，重点关注：
- 无界队列
- 生产者侧过滤
- `async with` 生命周期管理

---

## 下一步

- **上一篇**: 无（这是第一个模块）
- **下一篇**: F02 config — 配置系统
