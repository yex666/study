# S14-S17: acp_client/analytics/todos/trajectory — 辅助模块

> **文件路径**: 
> - `src/chrys/service/acp_client/` (ACP 客户端)
> - `src/chrys/service/analytics/` (分析)
> - `src/chrys/service/todos/` (TODO 追踪)
> - `src/chrys/service/trajectory/` (Service 层轨迹)
> 
> **一句话**: Service 层的辅助模块——ACP 通信、分析、TODO 管理、轨迹增强。

---

## S14: acp_client — ACP 客户端

### 核心文件

```
service/acp_client/
├── ...
```

### 是什么

ACP（Agent Client Protocol）客户端的 Service 层实现。当 Chrys 的子 Agent 配置为 ACP 模式时，通过 ACP 协议与外部 Agent 进程通信。

```python
class AcpSubAgentClient:
    """Client for communicating with ACP sub-agents."""
    
    def __init__(self, session: AcpSession):
        self.session = session
    
    async def send_message(self, text: str) -> AsyncIterator[Event]:
        """Send a message and stream the response."""
        ...
    
    async def cancel(self) -> None:
        """Cancel the current operation."""
        ...
```

---

## S15: analytics — 分析

### 是什么

收集 Agent 的使用数据（不是遥测遥测，是使用统计）：

```python
class Analytics:
    """Collect usage analytics."""
    
    def __init__(self):
        self.events: list[AnalyticsEvent] = []
    
    async def record(self, event: AnalyticsEvent) -> None:
        """Record an analytics event."""
        ...
```

---

## S16: todos — TODO 追踪

### 是什么

管理 Agent 执行过程中的 TODO 列表，让 Agent 能跟踪任务进度：

```python
class TodoManager:
    """Manage a TODO list for the current task."""
    
    def __init__(self):
        self.items: list[TodoItem] = []
    
    async def add(self, title: str, priority: str = "medium") -> TodoItem:
        """Add a new todo item."""
        item = TodoItem(
            id=str(uuid4())[:8],
            title=title,
            status="pending",
            priority=priority,
            created_at=now(),
        )
        self.items.append(item)
        return item
    
    async def update(self, item_id: str, status: str) -> bool:
        """Update a todo item's status."""
        ...
    
    async def remove(self, item_id: str) -> bool:
        """Remove a todo item."""
        ...
    
    def to_prompt(self) -> str:
        """Format the todo list for the LLM."""
        ...
```

---

## S17: trajectory（Service 层）— 轨迹增强

### 是什么

Service 层对 Foundation trajectory 的增强，主要包含：
- **preparation.py**：准备阶段的轨迹记录（准入、hook 评估等）
- **waits.py**：等待阶段的轨迹记录
- 与 Orchestration 层的集成

```python
class PreparationTrace:
    """Tracks pre-admission work for trajectory recording."""
    
    def __init__(self, scope, phase):
        self.scope = scope
        self.phase = phase
        self.started_at = monotonic_ns()
    
    def finish(self, outcome: PreparationOutcome) -> None:
        """Mark the preparation as finished."""
        ...
```

---

## 📊 Service 层完成总结

| 序号 | 模块 | 文档 |
|------|------|------|
| S01 | llm | `S01_S03_core.md` |
| S02 | profiles | `S01_S03_core.md` |
| S03 | tools | `S01_S03_core.md` |
| S04 | mcp | `S04_S08_integration.md` |
| S05 | skills | `S04_S08_integration.md` |
| S06 | hooks | `S04_S08_integration.md` |
| S07 | approval | `S04_S08_integration.md` |
| S08 | context | `S04_S08_integration.md` |
| S09 | agent_middleware | `S09_S13_middleware.md` |
| S10 | mutations | `S09_S13_middleware.md` |
| S11 | session | `S09_S13_middleware.md` |
| S12 | state | `S09_S13_middleware.md` |
| S13 | workflows | `S09_S13_middleware.md` |
| S14 | acp_client | `S14_S17_auxiliary.md` |
| S15 | analytics | `S14_S17_auxiliary.md` |
| S16 | todos | `S14_S17_auxiliary.md` |
| S17 | trajectory | `S14_S17_auxiliary.md` |

---

## 下一步

- **上一篇**: S09-S13 中间件与状态模块
- **下一篇**: O01-O06（Orchestration 层）——引擎、执行、子代理、工作流
