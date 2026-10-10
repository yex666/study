# 从零构建 Coding Agent (1)：EventBus 事件总线

> **学习目标**：理解 Coding Agent 内部模块间如何通信，掌握发布/订阅模式在异步系统中的应用，学会处理背压和事件丢失问题。
>
> **对应源码**：`src/chrys/foundation/events/bus.py`（144 行）+ `types.py`

---

## 0. 为什么需要 EventBus？

### 场景重现

假设你正在写一个简单的 Coding Agent：

```python
class MyAgent:
    def __init__(self):
        self.llm = OpenAI()
        self.tools = [ReadFile(), WriteFile(), Shell()]
    
    async def run(self, user_input: str):
        # 1. 调用 LLM
        response = await self.llm.chat(user_input)
        
        # 2. 如果有工具调用
        while response.has_tool_calls():
            for call in response.tool_calls:
                result = await self.execute_tool(call)
                
                # 3. 需要通知 UI 更新进度
                self.ui.show_progress(f"Executing {call.name}...")
                
                # 4. 需要记录日志
                self.logger.log(f"Tool called: {call.name}")
                
                # 5. 需要保存到历史
                self.history.save(call, result)
                
                # 6. 需要追踪变更（用于撤销）
                self.mutations.track(call)
            
            response = await self.llm.chat(messages)
        
        return response.text
```

### 问题：紧耦合

上面的代码有几个严重问题：

1. **Agent 知道太多细节**：它必须知道有 UI、日志、历史、变更追踪这些模块，还要知道怎么调用它们。
2. **难以扩展**：如果想加一个新功能（比如发送通知到 Slack），就要改 Agent 代码。
3. **测试困难**：测试 Agent 时必须 mock 所有这些依赖。
4. **顺序依赖**：如果 `ui.show_progress()` 抛异常，后面的日志和历史就不会执行。

### 解法：事件驱动

让 Agent 只负责"发生了一件事"，不关心谁在乎：

```python
class MyAgent:
    def __init__(self, event_bus: EventBus):
        self.event_bus = event_bus
    
    async def run(self, user_input: str):
        response = await self.llm.chat(user_input)
        
        while response.has_tool_calls():
            for call in response.tool_calls:
                result = await self.execute_tool(call)
                
                # 只发布事件，不关心谁监听
                await self.event_bus.publish(
                    ToolCallEvent(tool_name=call.name, result=result)
                )
            
            response = await self.llm.chat(messages)
        
        return response.text
```

其他模块自己订阅：

```python
# UI 模块
event_bus.subscribe(ToolCallEvent, lambda e: ui.show_progress(e.tool_name))

# 日志模块
event_bus.subscribe(ToolCallEvent, lambda e: logger.info(f"Called {e.tool_name}"))

# 历史模块
event_bus.subscribe(ToolCallEvent, lambda e: history.save(e))
```

**好处**：
- Agent 不知道有哪些监听者（解耦）
- 加新功能只需订阅事件（可扩展）
- 测试时只需 mock EventBus（易测试）

---

## 1. 第一版：最简单的 EventBus

### 如果你来写，第一版会是什么样？

```python
# version_1.py - 最朴素的实现

class EventBus:
    def __init__(self):
        self.handlers = {}  # event_type -> [handler_fn]
    
    def subscribe(self, event_type, handler):
        if event_type not in self.handlers:
            self.handlers[event_type] = []
        self.handlers[event_type].append(handler)
    
    def publish(self, event):
        event_type = type(event)
        for handler in self.handlers.get(event_type, []):
            handler(event)  # 直接调用
```

### 测试一下

```python
import asyncio

# 定义一个事件
class ToolCallEvent:
    def __init__(self, tool_name: str):
        self.tool_name = tool_name

# 使用
async def main():
    bus = EventBus()
    
    # 订阅
    bus.subscribe(ToolCallEvent, lambda e: print(f"Tool called: {e.tool_name}"))
    
    # 发布
    bus.publish(ToolCallEvent("read_file"))

asyncio.run(main())
# 输出：Tool called: read_file
```

**看起来工作正常。但有个大问题。**

---

## 2. 第二版：异步 Handler 的问题

### 场景：Handler 需要等数据库写入

```python
# 历史记录器需要异步写入数据库
async def save_to_db(event: ToolCallEvent):
    await database.insert(event.tool_name)  # 可能需要 100ms
    print(f"Saved to DB: {event.tool_name}")

bus.subscribe(ToolCallEvent, save_to_db)

# 发布事件
bus.publish(ToolCallEvent("read_file"))
print("Done")
```

### 实际输出

```
Done
# （save_to_db 根本没执行！）
```

### 问题原因

第一版的 `publish()` 是同步的：

```python
def publish(self, event):
    for handler in self.handlers.get(type(event), []):
        handler(event)  # 如果 handler 是 async 函数，这里只是创建了协程对象，没有 await！
```

`handler(event)` 返回一个 coroutine 对象，但没有 `await`，所以协程根本没有运行。

### 修复：改成异步

```python
# version_2.py

class EventBus:
    async def publish(self, event):
        event_type = type(event)
        for handler in self.handlers.get(event_type, []):
            await handler(event)  # ← 关键：await
```

### 现在可以工作了

```python
async def main():
    bus = EventBus()
    
    async def save_to_db(event):
        await asyncio.sleep(0.1)  # 模拟数据库写入
        print(f"Saved to DB: {event.tool_name}")
    
    bus.subscribe(ToolCallEvent, save_to_db)
    
    await bus.publish(ToolCallEvent("read_file"))
    print("Done")

asyncio.run(main())
# 输出：
# Saved to DB: read_file
# Done
```

**看起来解决了问题。但还有更深的坑。**

---

## 3. 第三版：Fire-and-Forget 的灾难

### 场景：有人觉得 await 太慢，想优化

```python
# version_3.py - 危险的优化

class EventBus:
    async def publish(self, event):
        event_type = type(event)
        for handler in self.handlers.get(event_type, []):
            # "优化"：创建任务就跑，不等完成
            asyncio.create_task(handler(event))
```

### 这会出什么问题？

#### 问题 A：程序退出时任务没完成

```python
async def main():
    bus = EventBus()
    
    async def save_to_db(event):
        await asyncio.sleep(0.1)
        print(f"Saved to DB: {event.tool_name}")
    
    bus.subscribe(ToolCallEvent, save_to_db)
    
    await bus.publish(ToolCallEvent("read_file"))
    print("Done")
    # 函数立刻返回，create_task 的任务还没执行完！

asyncio.run(main())
# 可能输出：
# Done
# （Save to DB 可能打印，也可能不打印——竞态条件）
```

#### 问题 B：错误被吞掉

```python
async def bad_handler(event):
    raise ValueError("Something went wrong!")

bus.subscribe(ToolCallEvent, bad_handler)

await bus.publish(ToolCallEvent("read_file"))
# 没有任何错误提示！异常在后台任务里被吞掉了
```

#### 问题 C：顺序无法保证

```python
async def handler_a(event):
    await asyncio.sleep(0.1)
    print("A done")

async def handler_b(event):
    print("B done")

bus.subscribe(ToolCallEvent, handler_a)
bus.subscribe(ToolCallEvent, handler_b)

await bus.publish(ToolCallEvent("test"))
# 可能输出：
# B done
# A done
# （B 先完成了，因为它不需要 sleep）
```

**在某些场景下，顺序很重要**。比如 Chrys 的 TUI 依赖"tool start 事件一定在 tool result 之前处理完毕"。

---

## 4. Chrys 的做法：带背压的同步等待

### 核心原则

> **"Never fire-and-forget"** —— 永远不要发了就走。每个 handler 都必须 await 完成。

### 源码解析

```python
# src/chrys/foundation/events/bus.py

class EventBus:
    async def publish(self, event: Event, *, raise_handler_errors: bool = False) -> None:
        """Publish an event to all subscribers."""
        event_type = type(event)
        failures: list[Exception] = []
        
        # 关键 1：遍历所有 handler
        for handler in list(self._handlers.get(event_type, [])):
            try:
                # 关键 2：await handler，同步等待完成
                await handler(event)
            except Exception as exc:
                # 关键 3：收集失败，但不中断其他 handler
                failures.append(exc)
                logger.exception("EventBus handler failed for %s", event_type.__name__)
        
        # 关键 4：分发给 stream 消费者（后面讲）
        for stream in list(self._streams):
            stream._deliver(event)
        
        # 关键 5：可选地抛出异常
        if raise_handler_errors and failures:
            if len(failures) == 1:
                raise failures[0]
            raise ExceptionGroup(f"EventBus handlers failed", failures)
```

### 关键设计点

#### 1. `await handler(event)` —— 背压（Backpressure）

```python
# 伪代码演示背压效果

async def slow_handler(event):
    await asyncio.sleep(1)  # 模拟慢操作
    print("Slow handler done")

async def fast_handler(event):
    print("Fast handler done")

bus.subscribe(MyEvent, slow_handler)
bus.subscribe(MyEvent, fast_handler)

start = time.time()
await bus.publish(MyEvent())
elapsed = time.time() - start

print(f"Elapsed: {elapsed:.2f}s")
# 输出：
# Slow handler done
# Fast handler done
# Elapsed: 1.00s  ← publish 等了 1 秒！
```

**背压的效果**：如果某个 handler 很慢（比如写数据库），`publish()` 就会变慢。这迫使发布者意识到"下游处理不过来了"，从而可以做出调整（比如减少发布频率）。

**对比 fire-and-forget**：
```python
# Fire-and-forget 版本
start = time.time()
await bus.publish(MyEvent())  # 立即返回
elapsed = time.time() - start

print(f"Elapsed: {elapsed:.2f}s")
# 输出：
# Elapsed: 0.00s  ← 太快了，但任务可能还没完成
# Slow handler done  ← 稍后才打印
# Fast handler done
```

#### 2. 错误隔离 —— 一个 handler 失败不影响其他

```python
async def good_handler(event):
    print("Good handler running")

async def bad_handler(event):
    raise ValueError("Oops!")
    print("This won't be printed")

bus.subscribe(MyEvent, good_handler)
bus.subscribe(MyEvent, bad_handler)
bus.subscribe(MyEvent, good_handler)  # 再订阅一次，验证是否继续

await bus.publish(MyEvent(), raise_handler_errors=False)
# 输出：
# Good handler running
# ERROR: EventBus handler failed for MyEvent  ← 记录了错误
# Good handler running  ← 第三个 handler 还是执行了！
```

**为什么这样设计？**
- 不能让一个 bug 的 handler 阻塞整个系统
- 但要记录错误，方便排查
- 提供 `raise_handler_errors=True` 选项给需要严格模式的场景（比如测试）

#### 3. `list(...)` 拷贝 —— 防止迭代时修改

```python
for handler in list(self._handlers.get(event_type, [])):
#                 ^^^^ 关键：拷贝一份列表
```

**为什么需要拷贝？**

```python
# 假设不拷贝，直接迭代原始列表
for handler in self._handlers.get(event_type, []):
    await handler(event)
    # 如果 handler 内部调用了 bus.unsubscribe()，修改了列表
    # 会导致 RuntimeError: dictionary changed size during iteration
```

**实际场景**：某些 handler 可能在执行后自动取消订阅（一次性监听器）。

---

## 5. 两种分发机制：Subscribe vs Stream

### 问题：不同场景有不同需求

Chrys 发现有两大类消费者：

| 类型 | 例子 | 特点 |
|------|------|------|
| **回调型** | UI 更新、数据库保存 | 必须同步完成，不能丢 |
| **流式型** | 实时显示 LLM 输出 | 可以慢，但不能丢 |

### Subscribe：回调型，带背压

```python
# 适合：必须同步完成的场景

async def save_to_db(event: ToolCallEvent):
    await database.insert(event.tool_name)
    # 必须等写入完成才能继续

bus.subscribe(ToolCallEvent, save_to_db)

await bus.publish(ToolCallEvent("read_file"))
# ↑ 这里会等到 save_to_db 完成
```

**特点**：
- ✅ 不会丢事件
- ✅ 保证顺序
- ❌ 如果 handler 慢，publisher 会被阻塞

### Stream：流式型，无界队列

```python
# 适合：实时消费，允许慢的场景

async def display_llm_output(bus: EventBus):
    async with bus.stream(LLMTextChunk) as stream:
        async for event in stream:
            print(event.text, end="", flush=True)
```

**特点**：
- ✅ publisher 不会被阻塞
- ✅ 不会丢事件（无界队列）
- ✅ 支持过滤（只订阅特定事件类型）
- ❌ 如果消费者太慢，内存会增长

### Stream 的源码实现

```python
class _EventStream:
    """Async iterator adapter for EventBus streaming."""
    
    def __init__(self, bus: EventBus, event_types: tuple[type[Event], ...]) -> None:
        self._bus = bus
        self._event_types = event_types
        # 关键 1：无界队列
        self._queue: asyncio.Queue[Event] = asyncio.Queue()  # maxsize=0，无界
        self._warned_high_water = False
    
    def _deliver(self, event: Event) -> None:
        """Enqueue an event — never drops."""
        # 关键 2：生产者侧过滤
        if self._event_types and not isinstance(event, self._event_types):
            return  # 不匹配，直接丢弃
        
        # 关键 3：入队（无界队列，永远不会满）
        self._queue.put_nowait(event)
        
        # 关键 4：高水位告警（只是警告，不丢弃）
        if not self._warned_high_water and self._queue.qsize() >= 10_000:
            self._warned_high_water = True
            logger.warning("Stream backlog exceeded 10000 events")
    
    async def __aenter__(self) -> _EventStream:
        # 注册到 bus
        self._bus._streams.append(self)
        return self
    
    async def __aexit__(self, *exc: object) -> None:
        # 从 bus 移除
        self._bus._streams.remove(self)
    
    def __aiter__(self) -> AsyncIterator[Event]:
        return self
    
    async def __anext__(self) -> Event:
        # 从队列取事件
        return await self._queue.get()
```

### 关键设计点

#### 1. 无界队列 —— 永不丢弃

```python
self._queue: asyncio.Queue[Event] = asyncio.Queue()  # maxsize=0
```

**为什么不用有界队列？**

```python
# 假设有界队列
self._queue = asyncio.Queue(maxsize=100)

# 当队列满了
self._queue.put_nowait(event)  # 抛出 QueueFull！
# 或者
await self._queue.put(event)  # 阻塞，导致 publisher 被卡住
```

**权衡**：
- 无界队列 → 可能内存暴涨
- 有界队列 → 要么丢事件，要么阻塞 publisher

Chrys 的选择：**宁可内存暴涨，也不丢事件**。因为 LLM 输出的流式文本一旦丢了就没法恢复。

#### 2. 生产者侧过滤 —— 减少无效入队

```python
def _deliver(self, event: Event) -> None:
    # 在进入队列前就过滤
    if self._event_types and not isinstance(event, self._event_types):
        return  # 不匹配，不入队
    
    self._queue.put_nowait(event)
```

**为什么不在消费者侧过滤？**

```python
# 反例：消费者侧过滤

async def consumer():
    async for event in bus.stream():  # 订阅所有事件
        if isinstance(event, LLMTextChunk):  # 消费者侧过滤
            process(event)
```

**问题**：
- 所有事件都会入队，浪费内存
- 消费者要处理大量无关事件

**正确做法**：在 `_deliver` 时就过滤，只有匹配的事件才入队。

#### 3. 高水位告警 —— 可观测性

```python
if not self._warned_high_water and self._queue.qsize() >= 10_000:
    self._warned_high_water = True
    logger.warning("Stream backlog exceeded 10000 events")
```

**为什么需要告警？**

虽然不丢事件，但如果队列一直涨，说明消费者跟不上。告警帮助发现问题。

**为什么只告警一次？**

避免日志爆炸。`_warned_high_water` 标志确保只告警一次。

---

## 6. Event 基类设计

### 源码

```python
# src/chrys/foundation/events/types.py

@dataclass
class Event:
    """Base class for all events."""
    
    event_id: str = field(default_factory=lambda: uuid4().hex[:12])
    timestamp: datetime = field(default_factory=lambda: datetime.now(tz=UTC))
    session_id: str | None = None
```

### 设计要点

#### 1. `event_id` —— 唯一标识

```python
event_id: str = field(default_factory=lambda: uuid4().hex[:12])
```

**为什么需要？**
- 追踪事件（日志、调试）
- 去重（某些场景需要）
- 关联（多个事件属于同一逻辑操作）

**为什么用 12 位 hex 而不是完整 UUID？**
- 短一点，日志好看
- 12 位 hex = 48 bit = 2^48 ≈ 281 万亿种组合，足够用

#### 2. `timestamp` —— 发生时间

```python
timestamp: datetime = field(default_factory=lambda: datetime.now(tz=UTC))
```

**为什么带时区？**
- 避免时区混乱
- 跨时区系统也能正确排序

#### 3. `session_id` —— 会话归属

```python
session_id: str | None = None
```

**为什么需要？**
- 多会话共存时，区分事件属于哪个会话
- Headless/ACP 模式下，只转发带正确 `session_id` 的事件

---

## 7. 精确类型匹配 vs isinstance 匹配

### 问题：继承关系怎么处理？

```python
class ToolCallEvent(Event): pass
class ShellToolCallEvent(ToolCallEvent): pass

bus.subscribe(ToolCallEvent, handler_a)
bus.subscribe(ShellToolCallEvent, handler_b)

# 发布 ShellToolCallEvent
await bus.publish(ShellToolCallEvent())
# handler_a 应该被调用吗？
```

### Chrys 的选择：精确类型匹配

```python
# bus.py 第 58 行
event_type = type(event)  # 获取精确类型
for handler in list(self._handlers.get(event_type, [])):  # 只查这个类型的 handler
```

**效果**：
- 发布 `ShellToolCallEvent` → 只调用 `handler_b`
- 发布 `ToolCallEvent` → 只调用 `handler_a`

### 为什么不用 isinstance？

```python
# 反例：isinstance 匹配

for registered_type, handlers in self._handlers.items():
    if isinstance(event, registered_type):  # 包括子类
        for handler in handlers:
            await handler(event)
```

**问题**：
- 意外触发：发布子类事件时，父类的 handler 也被调用
- 难以预测：开发者不确定 handler 会不会被调用
- 调试困难：出了问题不知道是哪个 handler 触发的

### 如果需要监听多种类型怎么办？

```python
# 显式订阅多个类型
bus.subscribe(ToolCallEvent, handler_a)
bus.subscribe(ShellToolCallEvent, handler_a)  # 显式再订阅一次
```

**Stream 支持 isinstance**：

```python
# stream 用的是 isinstance
async with bus.stream(ToolCallEvent, ShellToolCallEvent) as stream:
    async for event in stream:
        # 这里用的是 isinstance
        pass
```

**为什么 stream 可以用 isinstance？**
- stream 通常用于消费端，允许宽泛匹配
- subscribe 用于生产端，需要精确控制

---

## 8. 实际使用场景

### 场景 1：TUI 实时更新

```python
# app/tui/screens/main/screen.py

class MainScreen:
    async def on_mount(self):
        # 订阅工具调用事件，更新 UI
        self.app.event_bus.subscribe(
            InvocationToolCallStart,
            self._on_tool_call_start
        )
    
    async def _on_tool_call_start(self, event: InvocationToolCallStart):
        # 更新工具卡片
        self.tool_card.update(event.tool_name, event.args)
```

### 场景 2：Headless 模式流式输出

```python
# orchestration/session_host.py

class ChrysSessionHost:
    async def iter_run_events(self, message: str):
        """Yield events until the run finishes."""
        
        # 创建 stream，只订阅特定事件类型
        async with self.event_bus.stream(*_HEADLESS_RUN_EVENT_TYPES) as stream:
            # 启动运行
            run_task = asyncio.create_task(self.engine.run(message))
            
            # 流式产出事件
            async for event in stream:
                yield event
                
                # 检查运行是否结束
                if run_task.done():
                    break
```

### 场景 3：日志记录

```python
# service/trajectory/writer.py

class TrajectoryWriter:
    def __init__(self, event_bus: EventBus):
        self.event_bus = event_bus
    
    async def start(self):
        # 订阅所有感兴趣的事件
        await self.event_bus.subscribe(InvocationMessage, self._on_message)
        await self.event_bus.subscribe(InvocationToolCallStart, self._on_tool_start)
        await self.event_bus.subscribe(InvocationToolResult, self._on_tool_result)
    
    async def _on_message(self, event: InvocationMessage):
        self.file.write(f"[MESSAGE] {event.text}\n")
    
    async def _on_tool_start(self, event: InvocationToolCallStart):
        self.file.write(f"[TOOL START] {event.tool_name}\n")
```

---

## 9. 如果你要自己实现，建议的步骤

### Step 1：最小可用版本（10 分钟）

```python
class SimpleEventBus:
    def __init__(self):
        self.handlers = {}
    
    async def subscribe(self, event_type, handler):
        self.handlers.setdefault(event_type, []).append(handler)
    
    async def publish(self, event):
        for handler in self.handlers.get(type(event), []):
            await handler(event)
```

**测试**：
```python
async def test_basic():
    bus = SimpleEventBus()
    results = []
    
    await bus.subscribe(str, lambda e: results.append(e))
    await bus.publish("hello")
    
    assert results == ["hello"]
```

### Step 2：添加错误处理（20 分钟）

```python
class BetterEventBus(SimpleEventBus):
    async def publish(self, event, *, raise_errors=False):
        failures = []
        for handler in self.handlers.get(type(event), []):
            try:
                await handler(event)
            except Exception as e:
                failures.append(e)
                logger.exception(f"Handler failed: {e}")
        
        if raise_errors and failures:
            raise ExceptionGroup("Handler failures", failures)
```

### Step 3：添加 Stream 支持（30 分钟）

参考 Chrys 的 `_EventStream` 实现，重点关注：
- 无界队列
- 生产者侧过滤
- async context manager（`__aenter__` / `__aexit__`）

### Step 4：添加高级特性（可选）

- 优先级（某些 handler 先执行）
- 一次性监听器（执行后自动取消订阅）
- 事件拦截器（在发布前修改事件）
- 性能监控（统计每个 handler 的耗时）

---

## 10. 关键 Takeaways

### ✅ 必须记住的原则

1. **永不 fire-and-forget**：所有 handler 必须 await 完成
2. **错误隔离**：一个 handler 失败不影响其他
3. **精确类型匹配**：subscribe 用 `type(event)`，不用 `isinstance`
4. **无界队列 + 告警**：stream 不丢事件，但要监控积压

### ⚠️ 常见陷阱

1. **忘记 await handler**：导致协程没执行
2. **迭代时修改列表**：用 `list(...)` 拷贝
3. **有界队列阻塞 publisher**：用无界队列
4. **isinstance 意外触发**：用精确类型匹配

### 🎯 设计决策的 trade-off

| 选择 | 优点 | 缺点 |
|------|------|------|
| await handler（背压） | 不丢事件，保证顺序 | publisher 可能被阻塞 |
| 无界队列 | 永不丢事件 | 内存可能暴涨 |
| 精确类型匹配 | 行为可预测 | 需要显式订阅多个类型 |
| 错误隔离 | 系统鲁棒 | 可能隐藏 bug |

---

## 11. 下一步

EventBus 是整个系统的神经系统。理解了它，你就知道：
- ✅ 模块之间如何松耦合通信
- ✅ 如何处理异步系统的背压问题
- ✅ 如何设计可扩展的事件系统

**接下来建议学习**：
- `foundation/config/settings.py` —— 配置系统（如何管理各种设置）
- `kernel/identity.py` —— WeakIdentityRegistry（如何追踪对象身份）

或者你可以尝试：
- 自己实现一个 EventBus，跑通上面的测试用例
- 给你的 Agent 加上 EventBus，替换掉直接调用

---

## 附录：完整源码注释版

如果你想逐行研读，我推荐打开 `src/chrys/foundation/events/bus.py`，对照本文的解析看。重点关注：

- 第 56-71 行：`publish()` 的核心逻辑
- 第 94-144 行：`_EventStream` 的实现
- 第 108-128 行：`_deliver()` 的过滤和告警逻辑

有任何不明白地方，随时问我具体某一行或某个概念。
