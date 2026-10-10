# Orchestration 层详细实现解析

> **TL;DR**: Orchestration 层是 Chrys 的"编排层"，负责协调多个组件协同工作。本文深入解析四个核心实现的源码细节：ExecutionLease（单一互斥原语）、三条构建路径、Turn Pass 机制、Sub-agent Shell。

---

## 1. ExecutionLease —— 单一互斥原语

### 问题背景

Chrys 有三种顶层工作：
1. **Turn**：用户驱动的 agent pass（主对话）
2. **Workflow run**：工作流驱动的运行
3. **Session operations**：会话操作（new/clear/fork/restore/delete/rollback）

**问题**：这三者之间需要互斥。比如：
- 工作流运行时，不能开始新的 turn
- 会话恢复时，不能运行工作流
- Turn 运行时，不能切换会话

**天真做法**：为每种组合写一个 `if` 检查。

```python
async def start_turn():
    if workflow_running:
        raise Error("Workflow is running")
    if session_operation_in_progress:
        raise Error("Session operation in progress")
    # ... 更多检查
```

**问题**：组合爆炸，容易遗漏。

### Chrys 的做法：单一互斥原语

**核心思想**：用一个 `ExecutionLease` 对象统一管理所有互斥逻辑。

### 源码实现详解

#### Step 1: 数据结构定义

```python
# orchestration/engine/execution.py

@dataclass
class WorkflowExecution:
    """The one workflow run the lease admits at a time."""
    
    run_id: str
    request_id: str
    task: asyncio.Task[None] | None = None

@dataclass(frozen=True)
class PromptAdmissionScope:
    """Exact prompt/retry admission token."""
    
    admission_id: int
    kind: Literal["fresh", "retry"]
    session_generation: int      # 代际围栏
    build_generation: int        # 代际围栏
    preparation_trace: PreparationTrace | None

@dataclass(frozen=True)
class CurrentRunScope:
    """Orchestration-owned identity for one logical main-agent run."""
    
    run_generation: int          # 代际围栏
    session_generation: int      # 代际围栏
    build_generation: int        # 代际围栏
    owner_admission_id: int
    reminder_scope: CurrentRunReminderScope
```

**关键设计点**：

1. **代际围栏（Generation Fences）**：
   - `session_generation`：会话转换时递增
   - `build_generation`：成功构建时递增
   - `run_generation`：每运行递增
   
   **作用**：确保在 `await` 之前捕获的准入在 `await` 期间发生的重建/会话切换后无效。

2. **准入令牌（Admission Token）**：
   - `PromptAdmissionScope` 是一个冻结的 dataclass
   - 包含 `session_generation` 和 `build_generation`
   - 在提升为运行任务时会验证这些代际是否仍然匹配

#### Step 2: ExecutionLease 核心类

```python
@dataclass
class ExecutionLease:
    """One top-level task truth, its generation fences, admission and save evidence."""
    
    # 代际围栏
    _session_generation: int = 0
    _build_generation: int = 0
    _load_generation: int = 0
    _run_generation: int = 0
    _conversation_revision: int = 0
    
    # Turn 状态
    _run_task: asyncio.Task | None = None
    _current_run_scope: CurrentRunScope | None = None
    _pending_retry: PendingRetry | None = None
    
    # Workflow 状态
    workflow: WorkflowExecution | None = None
    
    # 会话操作状态
    _session_operations: int = 0
    
    # 准入控制
    _prompt_admission_closed: bool = False
    
    # 事件总线
    _event_bus: EventBus | None = None
    
    # 保存证据
    _successfully_saved_run_tasks: WeakSet[asyncio.Task] = field(default_factory=WeakSet)
    
    # --------------------------------------------------------------------------
    # Workflow 互斥
    # --------------------------------------------------------------------------
    
    async def begin_workflow(self, run_id: str, request_id: str) -> None:
        """Begin a workflow run. Refused if a turn or session operation is live."""
        
        # 检查是否有活跃的 turn
        if self._run_task is not None and not self._run_task.done():
            raise Error(code="turn_active", message="Cannot start workflow while turn is active")
        
        # 检查是否有会话操作
        if self._session_operations > 0:
            raise Error(code="session_operation_active", message="Cannot start workflow during session operation")
        
        # 检查是否已有工作流运行
        if self.workflow is not None:
            raise Error(code="workflow_already_active", message="Another workflow is already running")
        
        # 创建工作流执行
        self.workflow = WorkflowExecution(run_id=run_id, request_id=request_id)
        
        # 发布事件
        await self._publish_execution变更()
    
    async def end_workflow(self) -> None:
        """End the current workflow run."""
        
        if self.workflow is None:
            return
        
        self.workflow = None
        await self._publish_execution变更()
    
    def refuse_while_workflow_active(self) -> None:
        """Refuse an operation if a workflow is active."""
        
        if self.workflow is not None:
            raise Error(code="workflow_active", message="Operation refused while workflow is running")
    
    # --------------------------------------------------------------------------
    # Session 操作互斥
    # --------------------------------------------------------------------------
    
    @contextlib.contextmanager
    def session_operation(self, target_session_id: str):
        """Context manager for session operations. Refused if workflow is active."""
        
        # 检查工作流
        if self.workflow is not None:
            raise Error(
                code="workflow_active",
                message="Session operation refused while workflow is running",
                addressed_to=target_session_id,
            )
        
        self._session_operations += 1
        
        try:
            yield
        finally:
            self._session_operations -= 1
    
    # --------------------------------------------------------------------------
    # Turn 准入
    # --------------------------------------------------------------------------
    
    def reserve_prompt_admission(
        self,
        kind: Literal["fresh", "retry"],
        preparation_trace: PreparationTrace | None = None,
    ) -> PromptAdmissionScope:
        """Reserve a prompt admission slot. Refused if admissions are closed."""
        
        # 检查准入是否关闭（重建期间会关闭）
        if self._prompt_admission_closed:
            raise Error(code="admission_closed", message="Prompt admission is currently closed")
        
        #  mint 准入令牌
        admission = PromptAdmissionScope(
            admission_id=self._mint_admission_id(),
            kind=kind,
            session_generation=self._session_generation,
            build_generation=self._build_generation,
            preparation_trace=preparation_trace,
        )
        
        return admission
    
    def promote_fresh_admission_to_run(
        self, 
        admission: PromptAdmissionScope,
    ) -> PromptPromotionResult:
        """Promote a fresh admission into a run task. Returns success status."""
        
        # 验证身份
        if not self._admission_matches(admission):
            return PromptPromotionResult(promoted=False, stale=True)
        
        # 检查是否已有活跃运行
        if self._current_run_scope is not None:
            return PromptPromotionResult(promoted=False, conflict=True)
        
        # 检查运行任务是否已完成
        if self._run_task is not None and not self._run_task.done():
            return PromptPromotionResult(promoted=False, conflict=True)
        
        # 创建运行作用域
        self._run_generation += 1
        self._current_run_scope = CurrentRunScope(
            run_generation=self._run_generation,
            session_generation=admission.session_generation,
            build_generation=admission.build_generation,
            owner_admission_id=admission.admission_id,
            reminder_scope=CurrentRunReminderScope(),
        )
        
        return PromptPromotionResult(promoted=True)
    
    def _admission_matches(self, admission: PromptAdmissionScope) -> bool:
        """Check if an admission's generations still match current state."""
        
        return (
            admission.session_generation == self._session_generation
            and admission.build_generation == self._build_generation
        )
    
    # --------------------------------------------------------------------------
    # 代际管理
    # --------------------------------------------------------------------------
    
    def bump_session_generation(self) -> None:
        """Increment session generation. Invalidates outstanding admissions."""
        self._session_generation += 1
        self._prompt_admission_closed = False
    
    def bump_build_generation(self) -> None:
        """Increment build generation after successful build."""
        self._build_generation += 1
    
    def close_prompt_admission(self) -> None:
        """Close prompt admission (during rebuild)."""
        self._prompt_admission_closed = True
    
    # --------------------------------------------------------------------------
    # 快照和证据
    # --------------------------------------------------------------------------
    
    def execution(self) -> ExecutionSnapshot:
        """Return a snapshot of current execution state."""
        
        if self.workflow is not None:
            return ExecutionSnapshot(
                kind="workflow",
                workflow_run_id=self.workflow.run_id,
                cancellable=True,
            )
        
        if self._run_task is not None and not self._run_task.done():
            return ExecutionSnapshot(
                kind="turn",
                task=self._run_task,
                cancellable=True,
            )
        
        return ExecutionSnapshot(kind="idle")
    
    def record_current_run_final_save(self) -> None:
        """Record that the current run task was finally saved."""
        
        task = asyncio.current_task()
        if task is not None:
            self._successfully_saved_run_tasks.add(task)
    
    def was_run_task_finally_saved(self, task: asyncio.Task) -> bool:
        """Check if a run task was finally saved."""
        return task in self._successfully_saved_run_tasks
```

**关键设计点**：

1. **双向互斥**：
   ```python
   # 工作流拒绝 turn
   def refuse_while_workflow_active(self):
       if self.workflow is not None:
           raise Error(...)
   
   # 会话操作拒绝工作流
   @contextmanager
   def session_operation(self, target_session_id):
       if self.workflow is not None:
           raise Error(..., addressed_to=target_session_id)
   ```

2. **代际围栏**：
   ```python
   def _admission_matches(self, admission):
       return (
           admission.session_generation == self._session_generation
           and admission.build_generation == self._build_generation
       )
   ```
   确保在 `await` 期间发生的重建会使之前的准入失效。

3. **保存证据**：
   ```python
   def record_current_run_final_save(self):
       task = asyncio.current_task()
       self._successfully_saved_run_tasks.add(task)
   ```
   用 `WeakSet` 追踪哪些任务已成功保存，用于 GC freeze 等场景。

### 实际使用流程

```python
# 启动工作流
async def start_workflow(workflow_request):
    lease = engine.lease
    
    # 预留租约
    await lease.begin_workflow(workflow_request.run_id, workflow_request.request_id)
    
    try:
        # 运行工作流
        await run_workflow(workflow_request)
    
    finally:
        # 释放租约
        await lease.end_workflow()

# 开始 turn
async def on_user_message(message):
    lease = engine.lease
    
    # 检查工作流
    lease.refuse_while_workflow_active()
    
    # 预留准入
    admission = lease.reserve_prompt_admission(kind="fresh")
    
    # ... 准备用户内容 ...
    
    # 提升为运行任务
    result = lease.promote_fresh_admission_to_run(admission)
    if not result.promoted:
        if result.stale:
            raise Error("Admission became stale (session rebuilt)")
        if result.conflict:
            raise Error("Another run is already active")
    
    # 创建运行任务
    lease._run_task = asyncio.create_task(run_and_save(message))

# 会话恢复
async def restore_session(session_id):
    lease = engine.lease
    
    with lease.session_operation(session_id):
        # 恢复会话
        await do_restore(session_id)
        
        # 递增代际
        lease.bump_session_generation()
```

### 如果你要自己实现

**最小版本**：
```python
class SimpleLease:
    def __init__(self):
        self.running = False
    
    async def acquire(self):
        if self.running:
            raise Error("Already running")
        self.running = True
    
    async def release(self):
        self.running = False
```

**进阶版本**：
- 加代际围栏
- 加双向互斥
- 加准入令牌

---

## 2. 三条构建路径 —— 为什么需要三个独立的构建器

### 问题背景

Chrys 有三个地方需要构建 Agent：
1. **主构建**：主 Agent（用户直接对话）
2. **子 Agent 构建**：子 Agent（被父 Agent 调用）
3. **工作流节点构建**：工作流中的 Agent 节点

**天真做法**：写一个通用的 `build_agent()` 函数，三处都调用它。

**问题**：三者的所有权模型和策略不同：
- 主 Agent 拥有 `ExecutionLease`
- 子 Agent 是父工具循环拥有的工具
- 工作流节点由调度器拥有

混合在一起会导致：
- 子 Agent 意外获得 `ExecutionLease`
- 工作流节点显示 per-child card（但没有 UI 支持）
- 重试策略混乱

### Chrys 的做法：三条独立路径

### 源码实现详解

#### Path 1: 主构建

```python
# orchestration/engine/build/builder.py

async def build_agent(
    engine: AgentEngine,
    profile: AgentProfile,
) -> AgentBuildResult:
    """Build the main agent. Owns ExecutionLease."""
    
    # 1. 解析 model profile
    model_profile = resolve_model_profile(profile, engine.settings)
    
    # 2. 捕获 SessionEnvironment
    runtime = engine.active_session.capture_environment()
    
    # 3. 加载 memory 文件
    memory_files = await load_memory_files(profile.memory_files)
    
    # 4. 构建 LLM client
    llm_client = create_client(model_profile)
    
    # 5. 构建工具注册表
    tool_registry = build_tool_registry(
        builtins=get_builtin_tools(profile),
        web=assemble_web_tools(profile.web),
        sub_agents=build_sub_agent_tools(engine, profile.sub_agents),
        mcp=await connect_mcp_servers(profile.mcp),
        skills=load_skills(profile.skills),
    )
    
    # 6. 构建上下文管理器
    context_manager = ContextManager(profile.compaction)
    
    # 7. 构建 system reminder middleware
    reminder_middleware = SystemReminderMiddleware(profile.instructions)
    
    # 8. 构建 response validation middleware
    validation_middleware = ResponseValidationMiddleware()
    
    # 9. 构建 approval middleware
    approval_middleware = ApprovalMiddleware(engine.active_session.approval_mode)
    
    # 10. 构建 ask_user middleware
    ask_user_middleware = AskUserMiddleware()
    
    # 11. 组装 TurnBindings
    bindings = TurnBindings(
        agent=Agent(
            client=llm_client,
            tools=tool_registry,
            middleware=[
                reminder_middleware,
                validation_middleware,
                approval_middleware,
                ask_user_middleware,
            ],
        ),
        context_manager=context_manager,
        loop_recorder=LoopRecorder(),
    )
    
    # 12. 准备 Agent
    prepared = await PreparedAgent.open(bindings)
    
    return AgentBuildResult(
        prepared=prepared,
        conversation=prepared.conversation,
        agent=prepared.agent,
        bindings=bindings,
        runtime=runtime,
        loop_recorder=bindings.loop_recorder,
        reminder_middleware=reminder_middleware,
        # ... 更多字段
    )
```

#### Path 2: 子 Agent 构建

```python
# orchestration/sub_agents/tools.py

class SubAgentTools:
    """Build sub-agents as FunctionTools. No ExecutionLease ownership."""
    
    def __init__(
        self,
        engine: AgentEngine,
        max_total_concurrency: int = 5,
    ):
        self.engine = engine
        self.max_total_concurrency = max_total_concurrency
        self.active_count = 0
        self.semaphore = asyncio.Semaphore(max_total_concurrency)
    
    async def build_child(
        self,
        parent: Agent,
        child_profile: AgentProfile,
    ) -> FunctionTool:
        """Build a child agent as a FunctionTool."""
        
        # 1. 解析 model profile
        model_profile = resolve_model_profile(child_profile, self.engine.settings)
        
        # 2. 构建 LLM client（共享父 Agent 的 HTTP client）
        llm_client = create_client(model_profile, http_client=parent.http_client)
        
        # 3. 构建工具（不含 sub_agents，防止递归）
        tools = build_tool_registry(
            builtins=get_builtin_tools(child_profile),
            web=assemble_web_tools(child_profile.web),
            # 不构建 sub_agents
            # 不构建 ask_user（子 Agent 不能问用户）
        )
        
        # 4. 构建中间件
        middleware = [
            SystemReminderMiddleware(child_profile.instructions),
            ResponseValidationMiddleware(),
            # 无 approval middleware（继承父 Agent 的模式）
            # 无 ask_user middleware
        ]
        
        # 5. 创建 Agent
        agent = Agent(
            client=llm_client,
            tools=tools,
            middleware=middleware,
        )
        
        # 6. 包装成 FunctionTool
        async def call_sub_agent(**kwargs):
            async with self.semaphore:
                self.active_count += 1
                try:
                    task = kwargs.get("task", "")
                    result = await agent.run(task)
                    return result.text
                finally:
                    self.active_count -= 1
        
        return FunctionTool(
            name=child_profile.name,
            description=child_profile.description,
            parameters={"task": str},
            fn=call_sub_agent,
        )
```

#### Path 3: 工作流节点构建

```python
# orchestration/workflows/agent_node_build.py

async def build_kernel_node(
    scheduler: Scheduler,
    node_spec: NodeSpec,
) -> KernelConversation:
    """Build a kernel conversation for a workflow agent node. No ExecutionLease."""
    
    # 1. 解析 profile
    profile = node_spec.agent_profile
    
    # 2. 构建 LLM client
    llm_client = create_client(profile.model)
    
    # 3. 构建工具
    tools = build_tool_registry(
        builtins=get_builtin_tools(profile),
        web=assemble_web_tools(profile.web),
        sub_agents=build_sub_agent_tools(scheduler.engine, profile.sub_agents),
        mcp=await connect_mcp_servers(profile.mcp),
        # headless 模式移除 ask_user
    )
    
    # 移除 ask_user（headless 模式）
    tools = [t for t in tools if t.name != "ask_user"]
    
    # 4. 构建中间件
    middleware = [
        SystemReminderMiddleware(profile.instructions),
        ResponseValidationMiddleware(),
        # 无 approval middleware
        # 无 ask_user middleware
    ]
    
    # 5. 创建 Conversation
    conversation = KernelConversation(
        client=llm_client,
        tools=tools,
        middleware=middleware,
    )
    
    return conversation
```

### 三条路径的对比

| 特性 | 主构建 | 子 Agent | 工作流节点 |
|------|--------|----------|------------|
| **Ownership** | ExecutionLease | 父工具循环 | 调度器 |
| **ask_user** | ✅ 有 | ❌ 无 | ❌ 无（headless） |
| **approval** | ✅ 有 | 继承父级 | ❌ 无 |
| **sub_agents** | ✅ 有 | ❌ 无（防递归） | ✅ 有 |
| **失败处理** | 用户重试 | 返回错误给父 | 调度器决策 |
| **UI 卡片** | ✅ 有 | ❌ 无 | ❌ 无 |

### 如果你要自己实现

**最小版本**（只有一条路径）：
```python
def build_agent(profile):
    # 通用构建
    return Agent(...)
```

**进阶版本**：
- 分离主构建和子 Agent 构建
- 加并发控制
- 加 headless 模式

---

## 3. Turn Pass 机制 —— 一次完整的 turn 是如何运行的

### 问题背景

一个 turn 的生命周期很复杂：
1. 用户发送消息
2. 准入检查（工作流、会话操作）
3. 准备用户内容（图片发现、压缩）
4. 评估 hooks
5. 预留准入
6. 提升为运行任务
7. 调用 LLM
8. 执行工具循环
9. 保存状态
10. 释放租约

**问题**：如何保证这些步骤的正确顺序？如何处理中断和重试？

### Chrys 的做法：TurnCoordinator + TurnRunner + TurnBindings

### 源码实现详解

#### Step 1: TurnCoordinator（面向事件的门面）

```python
# orchestration/engine/run/coordinator.py

class TurnCoordinator:
    """Event-facing facade for one main-agent turn lifecycle."""
    
    def __init__(
        self,
        engine: AgentEngine,
        runner: TurnRunner,
        finalizer: TurnFinalizer,
        injector: ActiveTurnInjector,
    ):
        self.engine = engine
        self.runner = runner
        self.finalizer = finalizer
        self.injector = injector
    
    async def on_user_message(self, event: UserMessage) -> None:
        """Handle a user message event. Entry point for a fresh turn."""
        
        lease = self.engine.lease
        
        # 1. 检查工作流
        try:
            lease.refuse_while_workflow_active()
        except Error as e:
            await self.engine.event_bus.publish(Error(
                code="workflow_active",
                message="Cannot start turn while workflow is running",
            ))
            return
        
        # 2. 跟踪 inflight injection
        self.engine.begin_inflight_injection(event.injection_id)
        
        # 3. 开启 PreparationTrace
        preparation = PreparationTrace(
            scope=PreparationScope.PRE_TURN,
            phase=InputAdmissionPhase.START,
        )
        
        # 4. 等待 agent load idle
        await self.engine.wait_for_agent_load_idle()
        
        # 5. 检查 current.loaded
        if self.engine.current.loaded is None:
            await self.engine.event_bus.publish(Error(
                code="not_ready",
                message="Agent is not ready yet",
            ))
            return
        
        # 6. 如果 FSM is_running()，注入到当前 turn
        if self.engine.fsm.is_running():
            await self.injector.inject(event.text)
            return
        
        # 7. 等待前次运行任务完成
        if lease._run_task is not None and not lease._run_task.done():
            await lease._run_task
        
        # 8. 预留准入
        admission = lease.reserve_prompt_admission(
            kind="fresh",
            preparation_trace=preparation,
        )
        
        # 9. 评估 user_prompt_submit hooks
        hook_decision = await self.engine.hook_manager.fire(
            HookEvent.USER_PROMPT_SUBMIT,
            text=event.text,
        )
        
        if hook_decision.blocked:
            await self.engine.event_bus.publish(Error(
                code="hook_blocked",
                message=f"User prompt blocked by hook: {hook_decision.reason}",
            ))
            return
        
        # 10. 准备用户内容
        user_contents = await self.prepare_user_contents(event)
        
        # 11. 最终取消检查
        if self.engine.cancel_requested:
            return
        
        # 12. 提升为运行任务
        promotion = lease.promote_fresh_admission_to_run(admission)
        if not promotion.promoted:
            await self.engine.event_bus.publish(Error(
                code="promotion_failed",
                message=f"Failed to promote admission: {promotion}",
            ))
            return
        
        # 13. 设置当前输入
        lease.set_current_input(user_contents, task=None)
        
        # 14. FSM 转换
        self.engine.fsm.try_transition(Trigger.USER_MESSAGE)
        
        # 15. 创建运行任务
        lease._run_task = asyncio.create_task(
            self.run_and_save(user_contents, admission)
        )
    
    async def run_and_save(
        self,
        user_contents: list[Content],
        admission: PromptAdmissionScope,
    ) -> None:
        """Run the turn and save state."""
        
        try:
            # 绑定操作到 Conversation
            self.engine.current.loaded.conversation.bind_operation(
                TurnTaskBinding(
                    task=asyncio.current_task(),
                    latch_abort=self.engine.current.loaded.conversation.latch_abort,
                )
            )
            
            # 运行 turn
            outcome = await self.runner.run_fresh(user_contents)
            
            # 保存状态
            await self.finalizer.save(outcome)
            
            # 记录保存证据
            self.engine.lease.record_current_run_final_save()
        
        except Exception as e:
            # 错误处理
            await self.finalizer.handle_error(e)
        
        finally:
            # FSM 转换
            self.engine.fsm.try_transition(Trigger.TURN_COMPLETE)
```

#### Step 2: TurnRunner（执行 turn）

```python
# orchestration/engine/run/runner.py

class TurnRunner:
    """Execute a turn via the Conversation backend."""
    
    def __init__(
        self,
        engine: AgentEngine,
        retry_coordinator: RetryCoordinator,
    ):
        self.engine = engine
        self.retry_coordinator = retry_coordinator
    
    async def run_fresh(
        self,
        user_contents: list[Content],
    ) -> TurnOutcome:
        """Run a fresh turn (not a retry)."""
        
        conversation = self.engine.current.loaded.conversation
        
        # 构建 RunRequest
        request = RunRequest(
            intent=RunIntent.FRESH,
            messages=[Message(role="user", contents=user_contents)],
            stream=True,
        )
        
        # 运行 Conversation
        result = await conversation.run(request)
        
        # 处理结果
        if isinstance(result, Ok):
            return TurnOutcome(
                success=True,
                text=result.segments[-1].text if result.segments else "",
                response=result.backend_payload,
            )
        
        elif isinstance(result, Failed):
            return TurnOutcome(
                success=False,
                error=result.error,
                continuation=result.continuation,
            )
        
        elif isinstance(result, Aborted):
            return TurnOutcome(
                success=False,
                aborted=True,
                cause=result.cause,
            )
```

#### Step 3: KernelConversation（Kernel 后端）

```python
# orchestration/invoker/kernel.py

class KernelConversation(InvocationConversation):
    """Kernel-backed conversation. Runs the agent loop directly."""
    
    def __init__(
        self,
        client: ChatClient,
        tools: list[FunctionTool],
        middleware: list[ChatMiddleware],
    ):
        self.client = client
        self.tools = tools
        self.middleware = middleware
        self._generation = 0
        self._ticket: ContinuationTicket | None = None
        self._ticket_origin: InvocationOrigin | None = None
    
    async def run(self, request: RunRequest) -> Ok | Failed | Aborted:
        """Run one pass of the conversation."""
        
        # 1. 验证请求
        self.validate(request)
        
        # 2. 分配 PassHandle
        invocation_id = new_analytics_id()
        handle = PassHandle(invocation_id, f"{invocation_id}:0")
        
        # 3. 创建 ActivePass
        active = _ActivePass(handle=handle, invocation_id=invocation_id)
        self._active = active
        
        # 4. 设置 origin
        origin = InvocationOrigin(
            kind="turn",
            session_id=self.session_id,
            invocation_id=invocation_id,
        )
        self._current_invocation_origin = origin
        
        # 5. 重置 LoopRecorder
        self.loop_recorder.reset()
        
        # 6. 运行 AttemptRunner
        attempt_runner = AttemptRunner(self.client, self.tools, self.middleware)
        
        try:
            response = await attempt_runner.run(
                messages=request.messages,
                stream=request.stream,
            )
            
            # 7. 构建 PassEvidence
            evidence = PassEvidence(
                handle=handle,
                pass_id=handle.pass_id,
                committed_count=self.loop_recorder._committed_count,
            )
            
            # 8. 检查 abort
            if active.cause is not None:
                return Aborted(cause=active.cause)
            
            # 9. 返回 Ok
            return Ok(
                segments=response.segments,
                backend_payload=response,
                evidence=evidence,
            )
        
        except CancelledError:
            # 取消
            cause = self.abort_cause()
            active.cancelled(cause)
            raise
        
        except Exception as e:
            # 错误
            active.failed(e)
            
            # 铸造 ContinuationTicket
            ticket = ContinuationTicket(
                conversation_id=self.conversation_id,
                failed_pass_id=handle.pass_id,
                state_generation=self._generation,
                capability=Capability.CONTINUE_HISTORY,
            )
            
            return Failed(
                error=str(e),
                continuation=ticket,
                disposition=Disposition.TERMINAL,
            )
        
        finally:
            self._active = None
    
    def validate(self, request: RunRequest) -> None:
        """Validate the request. No mutation, no hooks, no history access."""
        
        if self._owner.closing:
            raise Error("Conversation is closing")
        
        if self._active is not None:
            raise Error("Overlapping run")
        
        # 检查 continuation ticket（如果是重试）
        if request.intent == RunIntent.RETRY:
            if not self.continuation_is_live(request.continuation, request.origin):
                raise Error("Continuation ticket is not live")
    
    def continuation_is_live(
        self,
        ticket: ContinuationTicket,
        origin: InvocationOrigin,
    ) -> bool:
        """Check if a continuation ticket is still valid."""
        
        return (
            not self._owner.closing
            and ticket is self._ticket  # 身份检查
            and ticket.state_generation == self._generation
            and origin.same_invocation(self._ticket_origin)
        )
    
    def invalidate_continuation(self) -> None:
        """Invalidate the current continuation ticket."""
        self._generation += 1
        self._ticket = None
```

### 实际流程

```
User: "帮我重构这个函数"
  ↓
TurnCoordinator.on_user_message()
  ↓
  ├─ refuse_while_workflow_active()
  ├─ reserve_prompt_admission()
  ├─ prepare_user_contents()
  └─ promote_fresh_admission_to_run()
  ↓
TurnCoordinator.run_and_save()
  ↓
TurnRunner.run_fresh()
  ↓
KernelConversation.run()
  ├─ validate()
  ├─ AttemptRunner.run()
  │   ├─ LLM 调用
  │   ├─ 工具执行
  │   └─ 循环直到完成
  └─ 返回 Ok/Failed/Aborted
  ↓
TurnFinalizer.save()
  ↓
FSM transition to IDLE
```

---

## 4. Sub-agent Shell —— 子 Agent 的唯一 pause/Retry/Abort 循环

### 问题背景

子 Agent 可能失败。如何处理？

**选项 A**：立即返回错误给父 Agent。
**选项 B**：暂停，让用户选择 Retry 或 Abort。

**问题**：什么时候选 A，什么时候选 B？

### Chrys 的做法：根据 human_failure_decisions 决定

### 源码实现详解

```python
# orchestration/sub_agents/shell.py

class SubAgentToolShell:
    """The child's sole OperationBinding and the only pause/Retry/Abort loop."""
    
    def __init__(
        self,
        engine: AgentEngine,
        child_profile: AgentProfile,
        *,
        human_failure_decisions: bool = True,
    ):
        self.engine = engine
        self.child_profile = child_profile
        self.human_failure_decisions = human_failure_decisions
        
        # 状态
        self._cascade_latched = False
        self._terminal_writer: TerminalWriter | None = None
    
    async def run(self, task: str) -> str:
        """Run the child agent. Returns the result text."""
        
        conversation = self.conversation
        
        # 构建 RunRequest
        request = RunRequest(
            intent=RunIntent.FRESH,
            messages=[Message(role="user", contents=[Content(type="text", text=task)])],
        )
        
        # 运行循环
        while True:
            result = await conversation.run(request)
            
            if isinstance(result, Ok):
                # 成功，返回文本
                return FinalSegment.of(result).text
            
            elif isinstance(result, Failed):
                # 失败
                if self.human_failure_decisions:
                    # TUI/ACP 模式：暂停让用户决策
                    decision = await self.pause_for_human_decision(result)
                    
                    if decision == "retry":
                        # 重试
                        request = self.build_retry_request(result)
                        continue
                    
                    elif decision == "abort":
                        # 中止
                        raise CancelledError("User aborted sub-agent")
                
                else:
                    # Headless/workflow 模式：立即返回错误
                    return f"Error: sub-agent '{self.child_profile.name}' failed — {result.error}"
            
            elif isinstance(result, Aborted):
                # 被中止
                if result.cause in {Cascade.CASCADE, Cascade.OWNER_CLOSE}:
                    self._cascade_latched = True
                
                raise CancelledError(f"Sub-agent aborted: {result.cause}")
    
    async def pause_for_human_decision(self, failure: Failed) -> Literal["retry", "abort"]:
        """Pause and ask the user what to do."""
        
        # 发送事件到 TUI
        await self.engine.event_bus.publish(SubAgentFailureEvent(
            agent_name=self.child_profile.name,
            error=failure.error,
            can_retry=failure.continuation is not None,
        ))
        
        # 等待用户决策
        decision = await self.engine.event_bus.wait_for(SubAgentDecisionEvent)
        
        return decision.choice
```

### 关键设计点

1. **human_failure_decisions 标志**：
   - `True`（TUI/ACP）：暂停让用户决策
   - `False`（headless/workflow）：立即返回错误

2. **FinalSegment.of**：
   ```python
   class FinalSegment:
       @staticmethod
       def of(result: Ok) -> Content:
           """Get the final text segment from an Ok result."""
           # 返回最后一个工具/hosted 输出之后的文本
           # 而不是 response.text
           ...
   ```
   子 Agent 只返回最后一个工具输出之后的文本，不是整个响应。

3. **Cascade 传播**：
   ```python
   if result.cause in {Cascade.CASCADE, Cascade.OWNER_CLOSE}:
       self._cascade_latched = True
   ```
   如果父 Agent 被取消，子 Agent 也会级联取消。

---

## 5. 总结：Orchestration 层的核心价值

### 四大创新的协同

```
┌─────────────────────────────────────────────────────────┐
│                 Orchestration 层架构                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  用户输入                                                │
│       ↓                                                 │
│  ① ExecutionLease                                       │
│     - 单一互斥原语                                      │
│     - 代际围栏                                          │
│       ↓                                                 │
│  ② 三条构建路径                                         │
│     - 主构建 / 子 Agent / 工作流节点                    │
│       ↓                                                 │
│  ③ Turn Pass 机制                                       │
│     - TurnCoordinator → TurnRunner → KernelConversation │
│       ↓                                                 │
│  ④ Sub-agent Shell                                      │
│     - 唯一的 pause/Retry/Abort 循环                     │
│       ↓                                                 │
│  返回结果给用户                                          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 学习建议

1. **先看 ExecutionLease**：理解互斥和代际围栏
2. **再看三条构建路径**：理解不同 ownership 模型
3. **然后看 Turn Pass 机制**：理解完整的 turn 生命周期
4. **最后看 Sub-agent Shell**：理解失败处理策略

### 如果你要自己实现

**优先级排序**：

1. **ExecutionLease**（必需）
   - 最小版本：简单的 running flag
   - 进阶版本：加代际围栏和双向互斥

2. **三条构建路径**（推荐）
   - 最小版本：一条通用路径
   - 进阶版本：分离主构建和子 Agent

3. **Turn Pass 机制**（可选）
   - 最小版本：简单的 while 循环
   - 进阶版本：加 Coordinator 和 Runner

4. **Sub-agent Shell**（高级）
   - 最小版本：直接返回错误
   - 进阶版本：加 human decision 暂停

---

## 附录：关键源码文件索引

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| `orchestration/engine/execution.py` | ~600 | ExecutionLease |
| `orchestration/engine/build/builder.py` | ~400 | 主构建 |
| `orchestration/sub_agents/tools.py` | ~300 | 子 Agent 构建 |
| `orchestration/workflows/agent_node_build.py` | ~200 | 工作流节点构建 |
| `orchestration/engine/run/coordinator.py` | ~500 | TurnCoordinator |
| `orchestration/engine/run/runner.py` | ~300 | TurnRunner |
| `orchestration/invoker/kernel.py` | ~600 | KernelConversation |
| `orchestration/sub_agents/shell.py` | ~400 | SubAgentToolShell |

---

**下一步**：想看 App 层的实现吗？那里有 TUI 性能优化、Headless CLI、ACP Server 等用户直接接触的部分。
