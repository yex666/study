# S09-S13: agent_middleware/mutations/session/state/workflows — 中间件与状态模块

> **文件路径**: 
> - `src/chrys/service/agent_middleware/` (Agent 中间件实现)
> - `src/chrys/service/mutations/` (变更追踪)
> - `src/chrys/service/session/` (会话持久化)
> - `src/chrys/service/state/` (状态序列化)
> - `src/chrys/service/workflows/` (工作流 SDK)
> 
> **一句话**: Service 层的核心中间件实现与状态管理。

---

## S09: agent_middleware — Agent 中间件实现

### 核心文件

```
service/agent_middleware/
├── system_reminder.py     ← System Reminder 注入（核心）
├── response_validation.py ← 响应验证（畸形响应重试）
├── injection.py           ← 中途消息注入
├── events/                ← 事件辅助（hosted_tools 等）
└── control/               ← 控制流（approval、ask_user、interrupt 等）
```

### SystemReminderMiddleware

**作用**：在每次 LLM 调用前，注入 `<system-reminder>` 标签到消息中。

```python
class SystemReminderMiddleware(ChatMiddleware):
    """Injects <system-reminder> tags per LLM call."""
    
    def __init__(self, instructions: str):
        self.instructions = instructions
    
    async def __call__(self, context, call_next):
        # 替换最后一个 user 消息为 enriched 版本
        if context.messages and context.messages[-1].role == "user":
            original = context.messages[-1]
            
            enriched = Message(
                role="user",
                contents=[
                    *original.contents,
                    Content(
                        type="system_reminder",
                        text=f"<system-reminder>Turn instructions:\n{self.instructions}\n</system-reminder>",
                    ),
                ],
            )
            
            context.messages[-1] = enriched
        
        await call_next(context)
```

**为什么**：
- **轮次级指令**：每条 user 消息携带当前轮次的指令（如"你正在重构 xxx"）
- **不走 system prompt**：避免 KV cache 失效
- **调用级**：每次调用都注入，但内容是快照的（不随时间变化）

### ResponseValidationMiddleware

**作用**：检测并重试畸形响应（空内容、空白文本、泄漏的工具调用标记）。

```python
class ResponseValidationMiddleware(ChatMiddleware):
    """Retries on malformed agent responses."""
    
    async def __call__(self, context, call_next):
        attempt = 0
        
        while attempt < self.max_attempts:
            await call_next(context)
            
            response = context.result
            
            # 验证响应
            validation = self._validate(response)
            
            if validation.is_valid:
                return
            
            # 重试（带退避）
            await asyncio.sleep(self.backoff(attempt))
            attempt += 1
        
        # 放弃：剥离 function_call，让工具循环正常退出
        self._strip_tool_calls(context.result)
```

### InjectionMiddleware

**作用**：处理用户在 Agent 工作途中发送的消息。

```python
class InjectionMiddleware(ChatMiddleware):
    """Delivers user messages sent mid-tool-loop."""
    
    def __init__(self):
        self._pending: list[Message] = []
    
    def inject(self, message: Message) -> None:
        """Queue a user message for delivery."""
        self._pending.append(message)
    
    async def __call__(self, context, call_next):
        # 在调用 LLM 前，把 pending 消息追加到 messages
        if self._pending:
            context.messages.extend(self._pending)
            self._pending.clear()
        
        await call_next(context)
```

---

## S10: mutations — 变更追踪

### 核心文件

```
service/mutations/
├── pipeline.py       ← 变更检测管道（写/编辑钩子 + shell 扫描）
├── tracker.py        ← 变更跟踪器
├── store.py          ← 文件快照存储
├── git_calibrator.py ← Git 变更校准
├── coordination.py   ← 管道协调
└── trace.py          ← shell 命令追踪
```

### 作用

记录 Agent 对文件系统的所有修改，用于：
- **撤销**：用户可以回滚到之前的某个状态
- **审计**：追踪谁在什么时候改了什么
- **快照**：记录每个文件的 before/after 状态

### 核心设计

```python
class MutationTracker:
    """Track file edits made by tools."""
    
    def __init__(self, snapshot_store: SnapshotStore):
        self.store = snapshot_store
        self.mutations: list[Mutation] = []
    
    async def record_edit(self, path: str, before: bytes, after: bytes) -> Mutation:
        """Record a file edit."""
        
        mutation = Mutation(
            path=path,
            before_digest=sha256(before),
            after_digest=sha256(after),
            timestamp=now(),
        )
        
        # 保存快照
        await self.store.save(path, before, mutation.id)
        
        self.mutations.append(mutation)
        return mutation
    
    async def rollback_to(self, mutation_id: str) -> list[Path]:
        """Rollback files to the state at a given mutation."""
        ...
```

### GitDiffCalibrator

```python
class GitDiffCalibrator:
    """Per-shell-command implicit change detection using Git state."""
    
    def collect_observations(self, cwd: Path) -> GitObservations:
        """Collect Git state WITHOUT mutating anything."""
        
        # 记录当前 Git 状态
        status = git_status(cwd)
        diff = git_diff(cwd)
        
        return GitObservations(
            files_changed=set(diff.files),
            untracked=set(status.untracked),
            ...
        )
```

---

## S11: session — 会话持久化

### 核心文件

```
service/session/
├── persistence.py       ← 会话保存/恢复
├── history.py           ← 历史管理器
└── sub_agent_transcript.py ← 子代理转录
```

### 存储结构

```
~/.chrys/
└── sessions/
    └── <short-id>/
        ├── session.json           ← 主文件
        ├── session.json.bak       ← 备份
        ├── session.recovery.json  ← 崩溃恢复边车
        └── trajectory/
            └── events.jsonl       ← 执行轨迹
```

### 恢复边车

```python
def load_session(session_dir: Path) -> SessionState:
    """Load session with recovery sidecar support."""
    
    main_path = session_dir / "session.json"
    recovery_path = session_dir / "session.recovery.json"
    
    # 优先使用恢复边车（如果存在且更新）
    if recovery_path.exists():
        main_ts = main_path.stat().st_mtime if main_path.exists() else 0
        recovery_ts = recovery_path.stat().st_mtime
        
        if recovery_ts > main_ts:
            return json.loads(recovery_path.read_text())
    
    return json.loads(main_path.read_text())
```

---

## S12: state — 状态序列化

### 核心文件

```
service/state/
├── serializers.py      ← 状态键表（哪个键保存，哪个不保存）
├── store.py            ← StateStore（JSON 文件存储）
├── locks.py            ← 文件锁
├── session_migration.py ← 会话迁移
├── session_mru.py      ← 最近使用记录
└── _fork.py            ← 会话复制
```

### 键表设计

```python
# 哪些键按 dict 复制
_COPY_AS_DICT_KEYS = frozenset({
    "last_usage",           # token 使用统计
    "context_calibration",  # 上下文校准
})

# 哪些键只在真值时保存
_PASS_THROUGH_OPTIONAL_KEYS = frozenset({
    "chrys_mutations",        # 变更追踪
    "chrys_workspace_baseline", # 工作区基线
    "chrys_todos",            # TODO 列表
    "last_words",             # 最后的话
    "trajectory_state",       # 轨迹状态
})

# 哪些键保存（None 和 0 有语义区别）
_PASS_THROUGH_NULLABLE_KEYS = frozenset({
    "catalog_pointer_record_count_state",
    "total_session_cache_hit_tokens",
})
```

**为什么需要键表**：
- **防止静默丢失**：新状态键不在表中 → save 会静默丢弃
- **语义区分**：`None`（未报告）vs `0`（报告为零）是不同的
- **向后兼容**：旧会话必须能继续加载

---

## S13: workflows — 工作流 SDK

### 核心文件

```
service/workflows/
├── sdk/                ← 用户可用的 SDK（纯标准库，Python 3.9）
├── scheduler.py        ← 调度器（纯确定性，无 I/O）
├── graph.py            ← 图规范
├── admission.py        ← 准入控制
├── records.py          ← 运行记录
├── store.py            ← 持久化存储
├── journal.py          ← 日志
├── worker_host.py      ← Worker 进程（在用户解释器运行）
└── builtins/           ← 内置工作流示例
```

### 双进程架构

```
主进程（Chrys）                      Worker 进程（用户解释器）
┌────────────────────┐              ┌─────────────────────┐
│  WorkflowCoordinator│──JSON-RPC──▶│  worker_host.py     │
│  (调度器)           │              │  (执行用户工作流)    │
└────────────────────┘              └─────────────────────┘
```

### SDK 示例（用户编写）

```python
# user_workflow.py
from chrys.workflows import Workflow, WorkflowStep

async def analyze_code(context):
    """使用用户的依赖来分析代码"""
    import my_custom_linter  # ← 用户的依赖
    return {"errors": my_custom_linter.check(context["code"])}

workflow = Workflow([
    WorkflowStep("analyze", analyze_code),
])
```

### 📚 详细解析

→ **service_implementation.md** 第 4 节：双进程工作流 SDK（含完整源码注释）

---

## 下一步

- **上一篇**: S04-S08 集成模块
- **下一篇**: S14-S17 合并文档（acp_client/analytics/todos/trajectory）
