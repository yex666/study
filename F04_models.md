# F04: models — 核心数据模型

> **模块路径**: `src/chrys/foundation/models/`
> **核心文件**: `invocations.py`, `workspace.py`, `history_markers.py`, `turns.py`, `execution.py`, `session_env.py`
> **一句话**: 定义整个系统共用的核心数据结构。

---

## 这是什么？

一组**不可变的数据类**，定义了系统中流转的核心概念：
- **InvocationOrigin** — 一个逻辑操作的唯一身份
- **Workspace** — 会话的工作目录配置
- **History Markers** — 消息的特殊标记（中断、注入、续接）
- **Execution Snapshot** — 当前执行状态的快照

---

## 为什么需要？

### 如果不做会怎样？

到处用字符串和字典表示这些概念：

```python
# ❌ 散落的字符串和字典
event = {
    "type": "tool_call",
    "origin": "turn",              # ← 魔法字符串
    "session_id": "abc123",
    "invocation_id": "def456",
    "parent": None,
}

workspace = {
    "cwd": "/path/to/project",
    "extra_dirs": ["/another/dir"],  # ← 没有类型检查
}

# 判断是否是中途注入的消息
if "_injected" in message.additional_properties:  # ← 魔法字符串
    ...
```

**问题**：
1. 魔法字符串容易拼错（`"turrn"` vs `"turn"`）
2. 没有类型检查（IDE 无法自动补全）
3. 结构不清晰（哪些字段是必需的？）
4. 重构困难（改了字段名要全局搜索替换）

### 用了数据模型之后

```python
# ✅ 强类型的数据类
origin = InvocationOrigin(
    kind="turn",
    session_id="abc123",
    invocation_id="def456",
    parent=None,
)

workspace = Workspace.from_cwd("/path/to/project")
workspace.working_dirs.append(WorkingDir(path="/another/dir"))

# IDE 自动补全 + 类型检查
if is_injected_message(message):
    ...
```

---

## 核心数据模型详解

### 1. InvocationOrigin — 调用身份标识

**是什么**：标识一个逻辑操作（一次 turn、一次子代理调用、一次工作流节点执行）。

```python
@dataclass(frozen=True, slots=True)
class InvocationOrigin:
    """A logical caller operation, including its explicitly bound ancestry."""
    
    kind: Literal["turn", "sub_agent", "workflow_node"]
    session_id: str
    invocation_id: str
    parent: InvocationOrigin | None
    attempt: int = 0  # 工作流调度器的重试次数
    
    def same_invocation(self, other: InvocationOrigin | None) -> bool:
        """Whether *other* names this logical invocation, in any of its attempts."""
        return (
            other is not None
            and other.kind == self.kind
            and other.session_id == self.session_id
            and other.invocation_id == self.invocation_id
            and other.parent == self.parent
        )
    
    @property
    def root(self) -> InvocationOrigin:
        """The outermost bound ancestor."""
        origin = self
        while origin.parent is not None:
            origin = origin.parent
        return origin
```

**为什么需要？**

考虑这个场景：
1. 用户在 TUI 中发起一轮对话（turn）
2. Agent 调用了一个子代理（sub_agent）
3. 子代理又调用了 MCP 工具

这三个操作都有各自的上下文（事件、日志、轨迹），但要能追踪它们的父子关系。

```python
# Turn 的 origin
turn_origin = InvocationOrigin(
    kind="turn",
    session_id="session_123",
    invocation_id="inv_456",
    parent=None,
)

# 子代理的 origin（parent 指向 turn）
sub_agent_origin = InvocationOrigin(
    kind="sub_agent",
    session_id="session_123",
    invocation_id="inv_789",
    parent=turn_origin,
)

# 发布事件时携带 origin
await event_bus.publish(ToolCallEvent(
    tool_name="read_file",
    origin=sub_agent_origin,  # ← 知道这是子代理的调用
))
```

**关键设计**：
- **frozen=True**：不可变，防止意外修改
- **slots=True**：内存优化（不创建 `__dict__`）
- **same_invocation()**：比较时忽略 `attempt`（同一次调用的不同重试算同一个）
- **root**：找到最外层的祖先（总是 turn 或 workflow_node）

---

### 2. Workspace — 工作区配置

**是什么**：一个会话的工作目录配置。

```python
@dataclass
class WorkingDir:
    """A single working directory within a workspace."""
    
    path: str           # Absolute path
    label: str = ""     # Optional user label (e.g. "frontend", "api")
    is_primary: bool = False  # Exactly one must be True per workspace
    
    @property
    def exists(self) -> bool:
        """Check if the directory still exists on disk."""
        return os.path.isdir(self.path)

@dataclass
class Workspace:
    """Workspace — working directory configuration for a session."""
    
    primary_cwd: str                           # 主工作目录
    working_dirs: list[WorkingDir] = field(default_factory=list)  # 额外工作目录
    reference_files: list[str] = field(default_factory=list)      # 参考文件
    
    @classmethod
    def from_cwd(cls, cwd: str | None = None) -> Workspace:
        """Create from a working directory (defaults to os.getcwd())."""
        raw = cwd or os.getcwd()
        return cls(primary_cwd=os.path.abspath(os.path.expanduser(raw)))
```

**为什么需要？**

用户可能同时处理多个相关项目：
- 前端项目（React）
- 后端项目（Node.js）
- 共享的类型定义包

Workspace 允许在一个会话中添加多个工作目录，Agent 可以在它们之间切换。

```python
# 创建 workspace
workspace = Workspace.from_cwd("/path/to/frontend")

# 添加额外工作目录
workspace.working_dirs.append(WorkingDir(
    path="/path/to/backend",
    label="backend",
))

workspace.working_dirs.append(WorkingDir(
    path="/path/to/shared-types",
    label="shared",
))

# Agent 可以读取任意工作目录的文件
for working_dir in workspace.working_dirs:
    files = list_files(working_dir.path)
```

---

### 3. History Markers — 历史标记

**是什么**：在消息的 `additional_properties` 中标记特殊状态。

```python
class HistoryMarkerKind:
    """String constants stored in Message.additional_properties."""
    
    KEY: Final[str] = "_chrys_kind"
    
    # Mid-turn user-message flags
    INJECTED_KEY: Final[str] = "_injected"        # 用户中途注入的消息
    CONTINUATION_KEY: Final[str] = "_continuation"  # 系统续接消息
    MID_TURN_USER_KEYS: Final[frozenset[str]] = frozenset({
        INJECTED_KEY, 
        CONTINUATION_KEY,
    })
    
    # Status codes
    STATUS_EXECUTION_INTERRUPTED: Final[str] = "execution_interrupted"
    STATUS_EXECUTION_FAILED: Final[str] = "execution_failed"
    STATUS_SESSION_CLOSED: Final[str] = "session_closed"
    STATUS_AWAITING_SUB_AGENTS: Final[str] = "awaiting_sub_agents"
    
    # Marker kinds
    TURN: Final[str] = "turn"
    SUMMARY: Final[str] = "summary"
    INTERRUPTED: Final[str] = "interrupted"
    AWAITING_SUB_AGENTS: Final[str] = "awaiting_sub_agents"
```

**为什么需要？**

对话历史中的消息不都是普通的用户输入或 AI 回复。有些有特殊含义：

```python
# 普通用户消息
user_msg = Message(role="user", contents=[...])

# 用户中途注入的消息（Agent 正在工作时，用户插了一句话）
injected_msg = Message(
    role="user",
    contents=[Content(text="等等，先别改那个文件")],
    additional_properties={
        "_injected": True,  # ← 标记为中途注入
        "_injection_id": "inj_123",
    },
)

# 系统续接消息（Agent 被中断后恢复时，系统自动发的 "continue"）
continuation_msg = Message(
    role="user",
    contents=[Content(text="continue")],
    additional_properties={
        "_continuation": True,  # ← 标记为系统续接
    },
)

# 执行中断标记
interrupted_msg = Message(
    role="assistant",
    contents=[Content(text="[Execution interrupted]")],
    additional_properties={
        "_chrys_kind": "interrupted",
        "_status_code": "execution_interrupted",
    },
)
```

**关键用途**：

1. **轮次边界判定**（`turns.py`）：
   ```python
   def opens_turn(msg: Message) -> bool:
       """User message that opens a real turn (unflagged user input)."""
       return msg.role == "user" and not is_mid_turn_user_message(msg)
   
   def is_mid_turn_user_message(msg: Message) -> bool:
       """User message that continues the current task instead of opening a turn."""
       return msg.role == "user" and any(
           msg.additional_properties.get(key, False)
           for key in HistoryMarkerKind.MID_TURN_USER_KEYS
       )
   ```

2. **压缩排除**：某些标记的消息不参与压缩
   ```python
   LAST_WORDS_EXCLUDED: Final[frozenset[str]] = frozenset({
       TURN, SUMMARY, INTERRUPTED,
   })
   ```

3. **会话计数排除**：某些标记的消息不计入统计
   ```python
   SESSION_COUNT_EXCLUDED: Final[frozenset[str]] = frozenset({
       TURN, INTERRUPTED, AWAITING_SUB_AGENTS,
   })
   ```

---

### 4. Execution Snapshot — 执行状态快照

**是什么**：当前执行状态的快照（idle / turn / workflow）。

```python
@dataclass(frozen=True)
class ExecutionSnapshot:
    """Snapshot of current execution state."""
    
    kind: Literal["idle", "turn", "workflow"]
    
    # Turn-specific fields
    task: asyncio.Task | None = None
    cancellable: bool = False
    
    # Workflow-specific fields
    workflow_run_id: str | None = None
```

**为什么需要？**

UI 需要根据当前执行状态显示不同的界面：
- idle：显示输入框，等待用户输入
- turn：显示进度条，允许取消
- workflow：显示工作流进度，允许取消

```python
snapshot = engine.lease.execution()

if snapshot.kind == "idle":
    ui.show_input_box()
elif snapshot.kind == "turn":
    ui.show_progress_bar()
    if snapshot.cancellable:
        ui.show_cancel_button()
elif snapshot.kind == "workflow":
    ui.show_workflow_progress(snapshot.workflow_run_id)
```

---

## 和其他模块的关系

```
models 被以下模块使用：
├── events（Event 基类包含 session_id）
├── kernel（Message/Content 使用 history markers）
├── orchestration（InvocationOrigin 贯穿整个编排层）
├── service（Workspace 在会话管理中）
└── app（UI 根据 ExecutionSnapshot 更新界面）
```

---

## 如果你要自己实现

### 最小版本（只保留核心）

```python
from dataclasses import dataclass
from typing import Literal

@dataclass(frozen=True)
class SimpleInvocationOrigin:
    kind: Literal["turn", "sub_agent"]
    session_id: str
    invocation_id: str

@dataclass
class SimpleWorkspace:
    primary_cwd: str
    
    @classmethod
    def from_cwd(cls, cwd: str | None = None) -> "SimpleWorkspace":
        import os
        return cls(primary_cwd=os.path.abspath(cwd or os.getcwd()))
```

### 进阶版本

加 parent chain、attempt、history markers。参考 Chrys 的实现。

---

## 下一步

- **上一篇**: F03 platform — 跨平台抽象
- **下一篇**: F05 errors — 错误分类和处理
