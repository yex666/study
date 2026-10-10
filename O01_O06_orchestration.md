# O01-O06: Orchestration 层全览

> **文件路径**: `src/chrys/orchestration/`
> **一句话**: 编排层协调所有组件——启动引导、引擎装配、turn 生命周期、子代理、工作流。

---

## 模块总览

```
orchestration/
├── startup.py           ← O01 启动引导
├── engine/              ← O02 引擎
│   ├── assembly.py      ←   引擎装配（唯一工厂）
│   ├── engine.py        ←   AgentEngine 类
│   ├── execution.py     ←   ExecutionLease（互斥原语）
│   ├── build/builder.py ←   主 Agent 构建
│   ├── run/             ←   Turn 运行（coordinator/runner/resume）
│   └── state/           ←   引擎状态（active_session/current_agent/FSM）
├── invoker/             ← O03 调用器
│   ├── contracts.py     ←   InvocationConversation 契约
│   ├── kernel.py        ←   KernelConversation（Kernel 后端）
│   ├── acp.py           ←   AcpConversation（ACP 后端）
│   └── origin.py        ←   BoundEmitter（身份绑定）
├── sub_agents/          ← O04 子代理
│   ├── shell.py         ←   SubAgentToolShell（pause/Retry/Abort）
│   └── tools.py         ←   SubAgentTools（构建子代理工具）
├── session_host.py      ← O05 会话宿主
└── workflows/           ← O06 工作流
    ├── coordinator.py   ←   工作流协调器
    ├── agent_node.py    ←   WorkflowAgentShell
    └── agent_node_build.py ← 工作流节点构建
```

---

## O01: startup — 启动引导

### 核心函数

```python
def bootstrap_runtime() -> RuntimeBootstrap:
    """Bootstrap the runtime environment for all entrypoints."""
    
    # 1. 应用第三方库补丁（在导入之前）
    from chrys.foundation.patches import apply_all
    apply_all()
    
    # 2. 加载设置
    loaded = load_settings(...)
    
    # 3. 解析 i18n
    localizer = Localizer(loaded.settings.locale)
    
    # 4. 设置会话根目录
    settle_session_root(loaded.settings)
    
    # 5. 安装进程级设置
    install_process_settings(loaded)
    
    # 6. 应用危险设置的警告
    warnings = _dangerous_setting_warnings(loaded)
    
    return RuntimeBootstrap(loaded=loaded, warnings=warnings)
```

### 设计要点

- **入口统一**：所有 entrypoint 都调用 `bootstrap_runtime()`
- **豁免**：`serve` 和 `install` 豁免（守卫测试保证）
- **不可重复**：每个 entrypoint 只调用一次

---

## O02: engine — 引擎

### 引擎装配（assembly.py）

```python
async def assemble_agent_engine(event_bus, settings=None, ...) -> AgentEngine:
    """The sole engine factory."""
    
    engine = AgentEngine()
    
    # 装配 12 个组件
    engine.active_session = ActiveSession(...)
    engine.current_agent = CurrentAgent(...)
    engine.turn_state = TurnRuntimeState(...)
    engine.permits = LifecyclePermits(...)
    engine.session_writer = SessionWriter(...)
    engine.agent_loader = AgentLoader(...)
    engine.session_lifecycle = SessionLifecycle(...)
    engine.rollback = RollbackController(...)
    engine.turn_coordinator = TurnCoordinator(...)
    engine.controls = RuntimeControls(...)
    engine.workflow_coordinator = WorkflowCoordinator(...)
    engine.usage = UsagePublisher(...)
    engine.fsm = EngineStateMachine(...)
    
    # 订阅事件（在第一次构建前）
    engine._subscribe_to_events()
    
    return engine
```

### ExecutionLease（execution.py）

**核心**：单一互斥原语，管理 Turn / Workflow run / Session operation 之间的互斥。

```python
class ExecutionLease:
    """One top-level task truth, its generation fences, admission and save evidence."""
    
    # 代际围栏
    session_generation: int     # 会话转换时递增
    build_generation: int       # 成功构建时递增
    load_generation: int        # agent-load 时递增
    run_generation: int         # 每运行递增
    
    # 互斥方法
    async def begin_workflow(self, run_id, request_id): ...
    async def end_workflow(self): ...
    def refuse_while_workflow_active(self): ...
    def reserve_prompt_admission(self, kind): ...
    def promote_fresh_admission_to_run(self, admission): ...
```

### 三条构建路径

| 路径 | 位置 | 所有权 |
|------|------|--------|
| **主构建** | `build/builder.py::build_agent` | ExecutionLease |
| **子代理构建** | `sub_agents/tools.py::SubAgentTools` | 父工具循环 |
| **工作流节点** | `workflows/agent_node_build.py::build_kernel_node` | 调度器 |

### 📚 详细解析

→ **orchestration_implementation.md** 第 1 节（ExecutionLease）和第 2 节（三条构建路径）

---

## O03: invoker — 调用器

### 契约

```python
class InvocationConversation(Protocol):
    """One contract, two backends."""
    
    async def run(self, request: RunRequest) -> Ok | Failed | Aborted: ...
```

### 两个后端

| 后端 | 用途 | 特点 |
|------|------|------|
| **KernelConversation** | 主 turn、Kernel 子代理 | 直接运行 Agent 循环 |
| **AcpConversation** | ACP 子代理 | 通过 ACP 协议通信 |

### 续接令牌

```python
@dataclass(frozen=True)
class ContinuationTicket:
    """A frozen, single-use, identity-bearing token."""
    
    conversation_id: str
    failed_pass_id: str
    state_generation: int
    capability: Literal["CONTINUE_HISTORY", "FRESH_SESSION"]

def continuation_is_live(ticket, origin) -> bool:
    """Check if a continuation ticket is still valid."""
    return (
        not owner.closing
        and ticket is self._ticket        # 身份检查
        and ticket.state_generation == self._generation
        and origin.same_invocation(self._ticket_origin)
    )
```

### 📚 详细解析

→ **orchestration_implementation.md** 第 3 节（Turn Pass 机制）

---

## O04: sub_agents — 子代理

### SubAgentToolShell

```python
class SubAgentToolShell:
    """The child's sole OperationBinding and the only pause/Retry/Abort loop."""
    
    def __init__(self, engine, child_profile, *, human_failure_decisions=True):
        self.human_failure_decisions = human_failure_decisions
    
    async def run(self, task: str) -> str:
        while True:
            result = await conversation.run(request)
            
            if isinstance(result, Ok):
                return FinalSegment.of(result).text
            
            elif isinstance(result, Failed):
                if self.human_failure_decisions:
                    # TUI/ACP 模式：暂停让用户决策
                    decision = await self.pause_for_human_decision(result)
                    if decision == "retry":
                        continue
                    raise CancelledError("User aborted")
                else:
                    # Headless/workflow 模式：立即返回错误
                    return f"Error: sub-agent failed — {result.error}"
```

### SubAgentTools

```python
class SubAgentTools:
    """Build sub-agents as FunctionTools."""
    
    def __init__(self, engine, max_total_concurrency=5):
        self.semaphore = asyncio.Semaphore(max_total_concurrency)
    
    async def build_child(self, parent, child_profile) -> FunctionTool:
        # 构建子 Agent，包装为 FunctionTool
        # 父 Agent 通过工具调用子 Agent
        ...
```

### 📚 详细解析

→ **orchestration_implementation.md** 第 4 节（Sub-agent Shell）

---

## O05: session_host — 会话宿主

```python
class ChrysSessionHost:
    """One bus + engine + session for non-TUI callers."""
    
    def __init__(self, agent_profile="Code", model_profile=None, cwd=None):
        self.approval_mode = ApprovalMode.BYPASS  # Headless 默认
        self.allow_user_interaction = False       # 移除 ask_user
    
    async def start(self):
        await bootstrap_runtime()
        self.engine = await assemble_agent_engine(...)
        await self.engine.start()
    
    async def iter_run_events(self, prompt):
        """Run a prompt and yield events until completion."""
        async with self.event_bus.stream(*_HEADLESS_RUN_EVENT_TYPES) as stream:
            run_task = asyncio.create_task(self.engine.run(prompt))
            async for event in stream:
                yield event
                if run_task.done():
                    break
    
    async def run_until_final(self, prompt) -> str:
        """Run a prompt and return the final text."""
        ...
```

---

## O06: workflows — 工作流编排

### WorkflowCoordinator

```python
class WorkflowCoordinator:
    """Coordinate workflow execution."""
    
    async def run_workflow(self, target, input_text, timeout):
        """Run a workflow."""
        
        # 1. 预留执行租约
        await lease.begin_workflow(run_id, request_id)
        
        try:
            # 2. 解析工作流文件
            # 3. 加载到 worker 进程
            # 4. 验证确认
            # 5. 准入清单
            # 6. 开启运行记录
            # 7. 启动 runner
            
            await runner.execute()
        
        finally:
            await lease.end_workflow()
```

### WorkflowAgentShell

```python
class WorkflowAgentShell:
    """The third caller of the invoker contract (after main turn and sub-agent)."""
    
    def __init__(self, node_spec, scheduler):
        ...
    
    async def open(self):
        """Build a private kernel or ACP conversation."""
        self.conversation = await build_kernel_node(self.scheduler, self.node_spec)
    
    async def run(self, attempt) -> Ok | Failed | Aborted:
        """Execute one pass per scheduler attempt."""
        ...
    
    async def close(self):
        """Unbind and release everything in reverse."""
        ...
```

---

## 📊 Orchestration 层完成总结

| 序号 | 模块 | 文档 |
|------|------|------|
| O01 | startup | `O01_O06_orchestration.md` |
| O02 | engine | `O01_O06_orchestration.md` |
| O03 | invoker | `O01_O06_orchestration.md` |
| O04 | sub_agents | `O01_O06_orchestration.md` |
| O05 | session_host | `O01_O06_orchestration.md` |
| O06 | workflows | `O01_O06_orchestration.md` |

---

## 下一步

- **上一篇**: S14-S17 辅助模块（Service 层完成！）
- **下一篇**: A01-A04（App 层）——CLI、TUI、ACP、Features
