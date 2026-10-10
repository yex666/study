# F07: trajectory — 执行轨迹追踪

> **模块路径**: `src/chrys/foundation/trajectory/`
> **核心文件**: `context.py`, `writer.py`, `envelope.py`, `event_types.py`
> **一句话**: 记录 Agent 执行的每一步（LLM 调用、工具执行、等待时间），用于调试、性能分析和审计。

---

## 这是什么？

一个**追加写的执行日志系统**。把 Agent 的每一步操作记录到 `<session>/trajectory/events.jsonl`，格式是一行一个 JSON 事件。

```jsonl
{"event_type":"model.cycle.started","actor":{"role":"main"},"turn_id":"turn_123","operation_id":"op_456",...}
{"event_type":"tool.operation.started","actor":{"role":"main"},"payload":{"tool_name":"read_file"},...}
{"event_type":"tool.operation.finished","actor":{"role":"main"},"payload":{"duration_ms":42},...}
{"event_type":"model.cycle.finished","actor":{"role":"main"},"payload":{"input_tokens":1500,"output_tokens":200},...}
```

---

## 为什么需要？

### 如果不做会怎样？

Agent 运行完了，但你不知道：
- **时间花在哪了？** LLM 调用慢？还是工具执行慢？还是网络延迟？
- **为什么做了这个决定？** Agent 看到了什么？调用了哪些工具？
- **Token 消耗多少？** 每个 LLM 调用花了多少钱？
- **出了什么问题？** 重试了几次？为什么失败？

```python
# ❌ 只有简单的日志
logger.info("Calling LLM")
response = await llm.chat(messages)
logger.info(f"Got response with {len(response.tool_calls)} tool calls")

for tool_call in response.tool_calls:
    logger.info(f"Executing {tool_call.name}")
    result = await execute_tool(tool_call)
    logger.info(f"Got result: {result[:100]}...")
```

**问题**：
1. 没有时间戳，无法计算耗时
2. 没有结构化数据，难以分析
3. 没有关联信息（哪个 turn、哪个 operation）
4. 无法回放整个执行过程

### 用了 trajectory 之后

```python
# ✅ 结构化的执行轨迹
async def run_turn():
    # 绑定轨迹上下文
    context = current_trajectory()
    
    # 记录 LLM 调用开始
    cycle_op = context.start_operation("model.cycle")
    await context.emit(cycle_op.started_event(
        input_tokens=count_tokens(messages),
    ))
    
    try:
        response = await llm.chat(messages)
        
        # 记录 LLM 调用结束
        await context.emit(cycle_op.finished_event(
            output_tokens=response.usage.output_tokens,
            duration_ms=cycle_op.elapsed_ms(),
        ))
        
        # 记录工具调用
        for tool_call in response.tool_calls:
            tool_op = context.start_operation("tool.operation")
            await context.emit(tool_op.started_event(
                tool_name=tool_call.name,
            ))
            
            result = await execute_tool(tool_call)
            
            await context.emit(tool_op.finished_event(
                result_preview=result[:100],
                duration_ms=tool_op.elapsed_ms(),
            ))
```

生成的 `events.jsonl`：
```jsonl
{"seq":1,"ts":"2026-10-10T10:00:00Z","event_type":"model.cycle.started","actor":{"role":"main"},"turn_id":"turn_123","operation_id":"op_456","payload":{"input_tokens":1500}}
{"seq":2,"ts":"2026-10-10T10:00:05Z","event_type":"model.cycle.finished","actor":{"role":"main"},"turn_id":"turn_123","operation_id":"op_456","payload":{"output_tokens":200,"duration_ms":5000}}
{"seq":3,"ts":"2026-10-10T10:00:05Z","event_type":"tool.operation.started","actor":{"role":"main"},"turn_id":"turn_123","operation_id":"op_789","parent_operation_id":"op_456","payload":{"tool_name":"read_file"}}
{"seq":4,"ts":"2026-10-10T10:00:05.042Z","event_type":"tool.operation.finished","actor":{"role":"main"},"turn_id":"turn_123","operation_id":"op_789","payload":{"duration_ms":42}}
```

**好处**：
1. 可以可视化成时间线（Perfetto UI）
2. 可以统计分析（哪个工具最慢？）
3. 可以回放整个执行过程
4. 可以计算成本（token 消耗 × 单价）

---

## 核心设计：四个关键决策

### 决策 1: ContextVar 传播上下文

```python
from contextvars import ContextVar

_current_trajectory: ContextVar[TrajectoryContext | None] = ContextVar(
    "current_trajectory", 
    default=None
)

def current_trajectory() -> TrajectoryContext | None:
    """Get the current trajectory context."""
    return _current_trajectory.get()

def bind_trajectory(context: TrajectoryContext) -> Token:
    """Bind a new trajectory context, return a reset token."""
    return _current_trajectory.set(context)
```

**为什么？**

- **异步安全**：`asyncio.Task` 会自动继承父任务的 ContextVar
- **隔离性**：子代理可以绑定自己的 actor，不会污染父代理
- **透明性**：所有中间件、工具都可以访问 `current_trajectory()`

```python
# 主 agent 绑定上下文
main_context = TrajectoryContext(
    sink=writer,
    session_id="session_123",
    actor=main_actor("session_123"),
    turn_id="turn_456",
)
token = bind_trajectory(main_context)

try:
    # 所有操作都在这个上下文中记录
    
    # 启动子代理时，子代理会继承这个上下文
    # 但子代理可以重新绑定自己的 actor
    sub_agent_context = main_context.with_actor(sub_agent_actor(...))
    sub_token = bind_trajectory(sub_agent_context)
    
    try:
        # 子代理的操作记录在自己的 actor 下
        await sub_agent.run(task)
    finally:
        _current_trajectory.reset(sub_token)
finally:
    _current_trajectory.reset(token)
```

---

### 决策 2: Write-through + Written-ack

```python
class TrajectoryWriter:
    """Single-writer, write-through trajectory event log with written-ack."""
    
    async def emit(self, draft: EventDraft) -> EmitResult:
        """Emit an event; returns only after the line reached the file."""
        
        sequence = self._next_sequence()
        data = encode_event_line(draft, sequence)
        
        # 提交给 worker 线程
        future = self._worker.submit(sequence, data)
        
        # 等待确认（最多 3 秒）
        try:
            result = await asyncio.wait_for(
                asyncio.wrap_future(future),
                timeout=self.write_ack_timeout,
            )
            return result
        except asyncio.TimeoutError:
            # 超时降级
            self._state = WriterState.TERMINAL_DEGRADED
            return EmitResult.DEGRADED
```

**为什么？**

- **Write-through**：`emit()` 返回时，事件已经写入磁盘（或在队列中）
- **Written-ack**：调用者知道事件是否成功持久化
- **Bounded wait**：超时后降级，不阻塞主流程
- **Ordered**：worker 线程按 sequence 顺序写入，保证事件顺序

**两种结果**：
- `WRITTEN`：事件已持久化（或确定会持久化）
- `DEGRADED`：写入失败或超时，事件可能丢失

---

### 决策 3: 内容最小化

```python
@dataclass(frozen=True)
class EventDraft:
    """One immutable event drafted by a producer."""
    
    event_type: str
    actor: Actor
    turn_id: str | None
    operation_id: str | None
    parent_operation_id: str | None
    payload: Mapping[str, Any]
    
    # 时间戳在构造时捕获，但在 emit 时才分配 sequence
    timestamp_ns: int = field(default_factory=monotonic_now_ns)
    wall_clock: str = field(default_factory=utc_now_rfc3339)

def encode_event_line(draft: EventDraft, sequence: int) -> bytes:
    """Encode one event as a JSON line, content-minimized."""
    
    event = {
        "seq": sequence,
        "ts": draft.wall_clock,
        "event_type": draft.event_type,
        "actor": {
            "kind": draft.actor.kind.value,
            "role": draft.actor.role,
            "actor_id": draft.actor.actor_id,
        },
    }
    
    # 可选字段
    if draft.turn_id:
        event["turn_id"] = draft.turn_id
    if draft.operation_id:
        event["operation_id"] = draft.operation_id
    if draft.parent_operation_id:
        event["parent_operation_id"] = draft.parent_operation_id
    if draft.payload:
        event["payload"] = draft.payload
    
    # 编码为 JSON 行
    return (json.dumps(event, separators=(",", ":")) + "\n").encode("utf-8")
```

**为什么？**

- **紧凑**：每行一个事件，无多余空格
- **可解析**：标准 JSON，易于处理
- **完整**：包含所有必要信息（时间、actor、operation、payload）
- **可追溯**：sequence 保证顺序，timestamp 记录真实时间

---

### 决策 4: Actor 身份派生

```python
def derive_actor_id(session_id: str, role: str, *, invocation_id: str | None = None) -> str:
    """Derive the stable actor_id of one logical actor."""
    
    material = f"chrys.trajectory.actor\x00{session_id}\x00{role}\x00{invocation_id or ''}"
    return hashlib.sha256(material.encode("utf-8")).hexdigest()[:ANALYTICS_ID_HEX_LENGTH]

def main_actor(session_id: str) -> Actor:
    return Actor(
        kind=ActorKind.AGENT,
        role=ActorRole.MAIN,
        actor_id=derive_actor_id(session_id, ActorRole.MAIN),
    )

def sub_agent_actor(session_id: str, invocation_id: str) -> Actor:
    return Actor(
        kind=ActorKind.AGENT,
        role=ActorRole.SUB_AGENT,
        actor_id=derive_actor_id(session_id, ActorRole.SUB_AGENT, invocation_id=invocation_id),
        invocation_id=invocation_id,
    )
```

**为什么？**

- **确定性**：相同的 session + role 总是生成相同的 actor_id
- **无需存储**：不需要在某个地方维护 actor_id 映射表
- **唯一性**：不同 session 或不同子代理调用的 actor_id 不同
- **可验证**：可以重新计算来验证 actor_id 的正确性

---

## 源码解析

### TrajectoryContext

```python
@dataclass(frozen=True, slots=True)
class TrajectoryContext:
    """The ambient recording scope a producer emits under."""
    
    sink: TrajectorySink              # 写入目标（writer）
    session_id: str                    # 会话 ID
    actor: Actor                       # 当前执行者
    turn_id: str | None = None         # 当前 turn
    run_operation_id: str | None = None      # 当前 run operation
    cycle_operation_id: str | None = None    # 当前 model cycle
    exchange_operation_id: str | None = None # 当前 exchange
    
    def draft(
        self,
        event_type: str,
        *,
        payload: Mapping[str, Any] | None = None,
        operation_id: str | None = None,
        parent_operation_id: str | None = None,
    ) -> EventDraft:
        """Build a draft stamped with this context's actor and turn."""
        return EventDraft(
            event_type=event_type,
            actor=self.actor,
            turn_id=self.turn_id,
            operation_id=operation_id,
            parent_operation_id=parent_operation_id,
            payload=payload if payload is not None else {},
        )
    
    def with_cycle(self, cycle_operation_id: str | None) -> TrajectoryContext:
        """Return a copy with a new cycle operation."""
        return replace(self, cycle_operation_id=cycle_operation_id, exchange_operation_id=None)
    
    def with_exchange(self, exchange_operation_id: str | None) -> TrajectoryContext:
        """Return a copy with a new exchange operation."""
        return replace(self, exchange_operation_id=exchange_operation_id)
```

### EventDraft

```python
@dataclass(frozen=True, slots=True)
class EventDraft:
    """One immutable event drafted by a producer."""
    
    event_type: str                              # "model.cycle.started"
    actor: Actor                                  # 执行者
    turn_id: str | None                          # 所属 turn
    operation_id: str | None                     # 当前操作
    parent_operation_id: str | None              # 父操作
    payload: Mapping[str, Any]                   # 事件数据
    
    # 时间戳（在构造时捕获）
    timestamp_ns: int = field(default_factory=monotonic_now_ns)
    wall_clock: str = field(default_factory=utc_now_rfc3339)
```

### TrajectoryWriter

```python
class TrajectoryWriter:
    """Single-writer, write-through trajectory event log."""
    
    def __init__(
        self,
        session_dir: Path,
        *,
        write_ack_timeout: float = DEFAULT_WRITE_ACK_TIMEOUT_SECONDS,
    ):
        self.session_dir = session_dir
        self.write_ack_timeout = write_ack_timeout
        
        # 打开事件日志文件
        self._fd = secure_open_owner_only_append(session_dir / "trajectory" / "events.jsonl")
        self._backend = FdWriteBackend(self._fd)
        
        # 启动 worker 线程
        self._worker = self._start_worker()
        self._sequence = 0
        self._state = WriterState.ACTIVE
    
    async def emit(self, draft: EventDraft) -> EmitResult:
        """Emit an event; returns only after the line reached the file."""
        
        if self._state != WriterState.ACTIVE:
            return EmitResult.DEGRADED
        
        # 分配 sequence
        sequence = self._sequence
        self._sequence += 1
        
        # 编码为 JSON 行
        data = encode_event_line(draft, sequence)
        
        # 提交给 worker
        future = self._worker.submit(sequence, data)
        
        # 等待确认
        try:
            result = await asyncio.wait_for(
                asyncio.wrap_future(future),
                timeout=self.write_ack_timeout,
            )
            return result
        except asyncio.TimeoutError:
            self._state = WriterState.TERMINAL_DEGRADED
            return EmitResult.DEGRADED
    
    def _start_worker(self) -> _TrajectoryWorker:
        """Start the dedicated daemon thread for writing."""
        worker = _TrajectoryWorker(self._backend)
        worker.start()
        return worker
```

---

## 实际使用场景

### 场景 1: 记录 LLM 调用

```python
async def call_llm_with_trajectory(messages):
    context = current_trajectory()
    if context is None:
        # 轨迹未启用，直接调用
        return await llm.chat(messages)
    
    # 开始 model cycle
    cycle_op = context.start_operation("model.cycle")
    await context.sink.emit(cycle_op.started_event(
        input_tokens=count_tokens(messages),
        model=context.exchange_facts.get("model_id"),
    ))
    
    try:
        response = await llm.chat(messages)
        
        # 完成 model cycle
        await context.sink.emit(cycle_op.finished_event(
            output_tokens=response.usage.output_tokens,
            cache_read_tokens=response.usage.cache_read_tokens,
            duration_ms=cycle_op.elapsed_ms(),
            outcome=ExchangeOutcome.OK.value,
        ))
        
        return response
    
    except Exception as e:
        # 记录失败
        classification = classify_error(e)
        await context.sink.emit(cycle_op.finished_event(
            duration_ms=cycle_op.elapsed_ms(),
            outcome=ExchangeOutcome.ERROR.value,
            error_kind=classification.kind.value,
            retryable=classification.retryable,
        ))
        raise
```

### 场景 2: 记录工具调用

```python
async def execute_tool_with_trajectory(tool_name, args):
    context = current_trajectory()
    if context is None:
        return await execute_tool(tool_name, args)
    
    # 开始 tool operation
    tool_op = context.start_operation("tool.operation")
    await context.sink.emit(tool_op.started_event(
        tool_name=tool_name,
        tool_args_preview=json.dumps(args)[:200],
    ))
    
    start_time = time.monotonic()
    
    try:
        result = await execute_tool(tool_name, args)
        
        # 完成 tool operation
        await context.sink.emit(tool_op.finished_event(
            duration_ms=int((time.monotonic() - start_time) * 1000),
            result_preview=str(result)[:200],
            outcome=ToolOutcome.SUCCESS.value,
        ))
        
        return result
    
    except Exception as e:
        await context.sink.emit(tool_op.finished_event(
            duration_ms=int((time.monotonic() - start_time) * 1000),
            error_message=str(e)[:200],
            outcome=ToolOutcome.ERROR.value,
        ))
        raise
```

### 场景 3: 可视化和分析

```bash
# 导出为 Perfetto 格式
uv run python scripts/trajectory_export.py --session abc123 --format perfetto

# 输出：session_abc123/trajectory/perfetto_trace.json

# 用 Perfetto UI 打开
# https://ui.perfetto.dev/

# 统计分析
uv run python scripts/trajectory_stats.py --session abc123

# 输出：
# Total duration: 45.2s
# Model calls: 12 (avg 3.2s, total 38.4s)
# Tool calls: 25 (avg 0.3s, total 7.5s)
# Token usage: 15,420 input, 3,280 output
# Estimated cost: $0.12
```

---

## 和其他模块的关系

```
trajectory 被以下模块使用：
├── kernel/loop.py（记录 model cycle、tool operation）
├── service/llm/instrumented.py（记录 LLM 调用详情）
├── orchestration/engine/run/（记录 turn、retry）
├── orchestration/sub_agents/（记录子代理调用）
└── app/tui/screens/trajectory/（UI 展示轨迹）
```

---

## 如果你要自己实现

### 最小版本（30 行）

```python
import json
import time
from pathlib import Path
from dataclasses import dataclass, asdict

@dataclass
class SimpleTrajectoryEvent:
    event_type: str
    timestamp: float
    payload: dict

class SimpleTrajectoryWriter:
    def __init__(self, log_file: Path):
        self.log_file = log_file
        self.log_file.parent.mkdir(parents=True, exist_ok=True)
    
    def emit(self, event_type: str, **payload):
        event = SimpleTrajectoryEvent(
            event_type=event_type,
            timestamp=time.time(),
            payload=payload,
        )
        with open(self.log_file, "a", encoding="utf-8") as f:
            f.write(json.dumps(asdict(event)) + "\n")

# 使用
writer = SimpleTrajectoryWriter(Path("trajectory.jsonl"))
writer.emit("llm_call_started", model="gpt-4", input_tokens=1500)
writer.emit("llm_call_finished", output_tokens=200, duration_ms=5000)
```

### 进阶版本

加 ContextVar 传播、write-through ack、actor 派生。参考 Chrys 的 `context.py` + `writer.py`。

---

## 下一步

- **上一篇**: F06 i18n — 国际化
- **下一篇**: F08 text — 文本处理
