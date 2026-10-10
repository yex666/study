# Service 层详细实现解析

> **TL;DR**: Service 层是 Chrys 的"集成层"，负责把 Kernel 连接到真实世界。本文深入解析四个核心实现的源码细节：LLM Client Factory、MCP Connection Cache、四阶段压缩、双进程工作流 SDK。

---

## 1. LLM Client Factory —— 如何支持多个 LLM 提供者

### 问题背景

你想支持多个 LLM 提供者：
- OpenAI（GPT-4, GPT-3.5）
- Anthropic（Claude）
- DeepSeek（兼容 OpenAI API）
- GLM（智谱 AI，也兼容 OpenAI API）

**天真做法**：为每个 Provider 写一个独立的 Client 类。

```python
class OpenAIClient: ...
class AnthropicClient: ...
class DeepSeekClient: ...  # 和 OpenAIClient 90% 相同
class GLMClient: ...       # 和 OpenAIClient 90% 相同
```

**问题**：大量重复代码，难以维护。

### Chrys 的做法：api_style 正交轴

**核心思想**：把"wire transport"和"request shape"解耦。

```python
# Provider 是传输协议（OpenAI / Anthropic）
# api_style 是请求形状（chat_completions / responses）

Provider × api_style = 具体的 Client
```

### 源码实现详解

#### Step 1: 定义 api_style 常量

```python
# service/profiles/models/schema.py

API_STYLE_CHAT_COMPLETIONS = "chat_completions"
API_STYLE_RESPONSES = "responses"
```

#### Step 2: ModelProfile 定义

```python
@dataclass
class ModelProfile:
    provider: str              # "openai", "anthropic", "deepseek-openai", "glm-openai"
    api_style: str             # "chat_completions" or "responses"
    model_id: str              # "gpt-4", "claude-3-opus", etc.
    base_url: str | None       # 自定义 endpoint
    api_key: str | None        # API key
    http_headers: dict         # 自定义 headers
    # ... 更多配置
```

#### Step 3: create_client 工厂函数

```python
# service/llm/clients.py

def create_client(profile: ModelProfile) -> ChatClient:
    """Create a chat client for the given profile."""
    
    provider = profile.provider
    
    # 根据 provider 选择 wire client
    if provider == "anthropic":
        return _create_anthropic_client(profile)
    
    elif provider in {"openai", "deepseek-openai", "glm-openai"}:
        # 这三个都走 OpenAI wire transport
        return _create_openai_flavored_client(profile)
    
    else:
        raise ValueError(f"Unknown provider: {provider}")

def _create_openai_flavored_client(profile: ModelProfile) -> ChatClient:
    """Create an OpenAI-flavored client (includes DeepSeek and GLM)."""
    
    # 确定 base_url
    base_url = profile.base_url
    if not base_url:
        if profile.provider == "openai":
            base_url = OPENAI_DEFAULT_BASE_URL
        elif profile.provider == "deepseek-openai":
            base_url = "https://api.deepseek.com/v1"
        elif profile.provider == "glm-openai":
            base_url = "https://open.bigmodel.cn/api/paas/v4"
    
    # 确定 API key
    api_key = profile.api_key
    if not api_key:
        env_var = _PROVIDER_API_KEY_ENVS[profile.provider]
        api_key = os.environ.get(env_var)
    
    # 验证 wire charset（防止 header 中有非法字符）
    _validate_wire_charset(api_key, profile.http_headers)
    
    # 创建 HTTP client
    http_client = _build_profile_http_client(profile)
    
    # 根据 api_style 选择不同的 client
    if profile.api_style == API_STYLE_CHAT_COMPLETIONS:
        # 标准的 Chat Completions API
        raw_client = RawOpenAIChatCompletionClient(
            api_key=api_key,
            model=profile.model_id,
            base_url=base_url,
            http_client=http_client,
        )
        
        # 如果是 DeepSeek 或 GLM，用特化的子类
        if profile.provider == "deepseek-openai":
            from .deepseek import DeepSeekChatCompletionClient
            raw_client = DeepSeekChatCompletionClient(raw_client)
        elif profile.provider == "glm-openai":
            from .glm import GLMChatCompletionClient
            raw_client = GLMChatCompletionClient(raw_client)
    
    elif profile.api_style == API_STYLE_RESPONSES:
        # OpenAI Responses API（实验性）
        raw_client = RawOpenAIResponsesClient(
            api_key=api_key,
            model=profile.model_id,
            base_url=base_url,
            http_client=http_client,
        )
    
    # 包装成 instrumented client（添加遥测、中间文本捕获等）
    return create_instrumented_openai_client(raw_client, profile)
```

**关键设计点**：

1. **api_style 正交轴**：
   - `deepseek-openai` 不是独立 provider，而是 `openai` provider + `chat_completions` style
   - 如果未来 DeepSeek 有自己的原生 API，可以加一个 `deepseek` provider
   - 新增变体只需子类化 wire client，不需要重写整个 factory

2. **Wire Charset Validation**：
   ```python
   def _validate_wire_charset(api_key: str, headers: dict):
       """Validate that API key and headers can be encoded in HTTP"""
       # 防止手误在 YAML 中写了不可见字符
       try:
           api_key.encode("ascii")
       except UnicodeEncodeError as e:
           raise api_key_charset_error(api_key, e)
       
       for name, value in headers.items():
           try:
               name.encode("ascii")
               value.encode("ascii")
           except UnicodeEncodeError as e:
               raise header_value_charset_error(name, value, e)
   ```

3. **HTTP Client Ownership**：
   ```python
   def _build_profile_http_client(profile: ModelProfile) -> httpx.AsyncClient:
       """Build an HTTP client with proper pool, proxy, and TLS settings."""
       
       # 使用 SDK 的默认配置作为基准
       sdk_defaults = openai.AsyncOpenAI()._client
       
       # 叠加 Chrys 的配置
       return httpx.AsyncClient(
           limits=sdk_defaults._limits,
           proxy=sdk_defaults._proxy_url,
           verify=sdk_defaults._verify,
           timeout=sdk_defaults._timeout,
           # 叠加 route-fact hooks
           event_hooks={"request": [route_facts_hook]},
       )
   ```

#### Step 4: Instrumented Shims

```python
# service/llm/instrumented.py

def create_instrumented_openai_client(
    raw_client: RawOpenAIChatCompletionClient,
    profile: ModelProfile,
) -> ChatClient:
    """Wrap raw client with instrumentation."""
    
    # 组合最终的 client stack
    return ToolLoopLayer(                      # chrys-owned tool loop
        ChatMiddlewareLayer(                   # chrys-owned per-call chat middleware
            _InstrumentedOpenAIClient(         # mixin + telemetry + raw wire client
                ChatTelemetryLayer(
                    raw_client
                )
            )
        )
    )

class _InstrumentedOpenAIClient(RawOpenAIChatCompletionClient):
    """Mixin that captures intermediate text."""
    
    async def get_response(self, messages, tools):
        stream = await super().get_response(messages, tools)
        
        async for update in stream:
            # 检查是否有"中间文本"（工具调用前的说明）
            if update.type == "text" and not update.is_final:
                # 实时显示给用户
                await self.on_intermediate_text(update.text)
            
            yield update
```

### 实际效果

```python
# 用户配置
profile = ModelProfile(
    provider="deepseek-openai",
    api_style="chat_completions",
    model_id="deepseek-coder",
    api_key="sk-xxx",
)

# 创建 client
client = create_client(profile)

# 返回的是：
# ToolLoopLayer(
#     ChatMiddlewareLayer(
#         _InstrumentedDeepSeekClient(      ← DeepSeek 特化
#             ChatTelemetryLayer(
#                 RawOpenAIChatCompletionClient  ← OpenAI wire transport
#             )
#         )
#     )
# )
```

### 如果你要自己实现

**最小版本**：
```python
def create_llm_client(provider: str, api_key: str, model: str):
    if provider == "openai":
        return OpenAIClient(api_key, model)
    elif provider == "anthropic":
        return AnthropicClient(api_key, model)
    # ... 其他 provider
```

**进阶版本**：
- 加 api_style 抽象
- 加 HTTP client ownership
- 加 wire charset validation
- 加 instrumented shims

---

## 2. MCP Connection Cache —— 连接缓存与渐进式披露

### 问题背景

你的 Agent 可能连接多个 MCP server：
- GitHub MCP server（10+ 工具）
- Jira MCP server（20+ 工具）
- Slack MCP server（15+ 工具）

**问题 1**：每次重建 Agent 都要重新连接 MCP server，很慢。

**问题 2**：大型 MCP server 有 50+ 工具，全部加载会撑爆上下文窗口。

### Chrys 的做法

**两个创新**：
1. **连接缓存**：相同配置的 MCP server 共享一个连接
2. **渐进式披露**：按需加载工具，而不是一次性全部加载

### 源码实现详解

#### Part 1: 连接缓存

```python
# service/mcp/cache.py

@dataclass
class MCPConnectionEntry:
    """Internal cache entry for one canonical MCP server config."""
    
    key: str                          # 缓存键（基于配置生成）
    config: MCPServerConfig           # 原始配置
    mcp_tool: MCPTool | None          # 已连接的 MCPTool
    refcount: int                     # 引用计数
    last_used: datetime               # 最后使用时间
    connect_task: asyncio.Task | None # 正在进行的连接任务

class MCPConnectionCache:
    """Shared live MCP connection cache."""
    
    def __init__(self):
        self._entries: dict[str, MCPConnectionEntry] = {}
        self._lock = asyncio.Lock()
    
    async def acquire(
        self, 
        config: MCPServerConfig,
        *,
        stdio_cwd: str | None = None,
    ) -> MCPConnectionLease:
        """Acquire a lease on a cached MCP connection."""
        
        # 解析有效配置（包括环境变量模板）
        effective = resolve_mcp_effective_config(config, stdio_cwd=stdio_cwd)
        key = effective.key
        
        async with self._lock:
            entry = self._entries.get(key)
            
            if entry is None:
                # 首次连接
                entry = MCPConnectionEntry(
                    key=key,
                    config=config,
                    mcp_tool=None,
                    refcount=0,
                    last_used=datetime.now(UTC),
                    connect_task=None,
                )
                self._entries[key] = entry
            
            # 增加引用计数
            entry.refcount += 1
            entry.last_used = datetime.now(UTC)
            
            # 如果还没连接，启动连接
            if entry.mcp_tool is None:
                if entry.connect_task is None:
                    entry.connect_task = asyncio.create_task(
                        self._connect(effective)
                    )
                
                # 等待连接完成
                entry.mcp_tool = await entry.connect_task
                entry.connect_task = None
            
            # 创建 FunctionTool 包装器
            functions = self._wrap_mcp_tools(entry.mcp_tool, config)
            
            return MCPConnectionLease(
                key=key,
                server_name=config.name,
                config=config,
                functions=functions,
                mcp_tool=entry.mcp_tool,
            )
    
    async def release(self, lease: MCPConnectionLease) -> None:
        """Release a lease on a cached MCP connection."""
        
        async with self._lock:
            entry = self._entries.get(lease.key)
            if entry:
                entry.refcount -= 1
                
                # 如果没人用了，标记为空闲
                if entry.refcount == 0:
                    entry.last_used = datetime.now(UTC)
                    
                    # 可选：空闲超时后关闭连接
                    asyncio.create_task(self._schedule_cleanup(lease.key))
    
    async def _connect(self, effective: MCPEffectiveConfig) -> MCPTool:
        """Connect to an MCP server."""
        
        config = effective.config
        
        try:
            if config.type == "stdio":
                # 启动子进程
                transport = await _inherited_stdio_environment(
                    command=config.command,
                    args=config.args,
                    env=effective.stdio_env,
                    cwd=effective.resolved_stdio_cwd,
                )
            
            elif config.type == "http":
                # HTTP 连接
                transport = HttpTransport(
                    url=config.url,
                    headers=effective.http_headers,
                )
            
            # 连接并列出工具
            mcp_tool = await MCPTool.connect(transport)
            
            return mcp_tool
        
        except Exception as e:
            logger.error(f"Failed to connect to MCP server {config.name}: {e}")
            raise MCPConnectionError(f"MCP connection failed: {e}") from e
    
    def _wrap_mcp_tools(
        self, 
        mcp_tool: MCPTool, 
        config: MCPServerConfig,
    ) -> list[FunctionTool]:
        """Wrap MCP tools as FunctionTools."""
        
        functions = []
        
        for tool_spec in mcp_tool.list_tools():
            # 创建 FunctionTool 包装器
            fn_tool = FunctionTool(
                name=tool_spec.name,
                description=tool_spec.description,
                parameters=tool_spec.inputSchema,
                fn=self._make_mcp_caller(mcp_tool, tool_spec.name),
            )
            
            # 设置 tool kind
            set_tool_kind(fn_tool, KIND_MCP)
            
            functions.append(fn_tool)
        
        return functions
    
    def _make_mcp_caller(self, mcp_tool: MCPTool, tool_name: str):
        """Create a caller function for an MCP tool."""
        
        async def caller(**kwargs):
            result = await mcp_tool.call_tool(tool_name, kwargs)
            return truncate_mcp_result(result)  # 截断过长的结果
        
        return caller
```

**关键设计点**：

1. **缓存键生成**：
   ```python
   def _compute_cache_key(config: MCPEffectiveConfig) -> str:
       """Generate a stable cache key from config."""
       
       payload = {
           "type": config.type,
           "command": config.command,
           "args": config.args,
           "env": _stable_mapping(config.stdio_env or {}),
           "cwd": config.resolved_stdio_cwd,
           "url": config.url,
           "headers": _stable_mapping(config.http_headers or {}),
       }
       
       # SHA256 哈希
       json_str = json.dumps(payload, sort_keys=True)
       return hashlib.sha256(json_str.encode()).hexdigest()[:16]
   ```
   
   - 相同的配置 → 相同的 key → 共享连接
   - 不同的配置 → 不同的 key → 独立连接

2. **引用计数**：
   - 每次 `acquire()` 增加 refcount
   - 每次 `release()` 减少 refcount
   - refcount == 0 时可以清理连接

3. **异步连接**：
   - 第一次 `acquire()` 时启动连接任务
   - 后续的 `acquire()` 等待同一个任务
   - 避免重复连接

#### Part 2: 渐进式披露

```python
# service/mcp/_progressive.py

class ProgressiveMCPExposure:
    """Progressive disclosure of MCP tools to avoid context window bloat."""
    
    def __init__(
        self,
        mcp_tool: MCPTool,
        always_load: list[str] | None = None,
    ):
        self.mcp_tool = mcp_tool
        self.always_load = set(always_load or [])
        self.loaded_tools: set[str] = set(self.always_load)
        
        # 控制工具
        self.control_tools = [
            FunctionTool(
                name="list_mcp_tools",
                description="List all available MCP tools",
                parameters={},
                fn=self.list_tools,
            ),
            FunctionTool(
                name="load_mcp_tool",
                description="Load a specific MCP tool",
                parameters={"name": str},
                fn=self.load_tool,
            ),
            FunctionTool(
                name="unload_mcp_tool",
                description="Unload a specific MCP tool",
                parameters={"name": str},
                fn=self.unload_tool,
            ),
        ]
    
    def get_exposed_tools(self) -> list[FunctionTool]:
        """Get currently exposed tools."""
        
        exposed = list(self.control_tools)
        
        # 添加已加载的工具
        for tool_name in self.loaded_tools:
            tool_spec = self.mcp_tool.get_tool_spec(tool_name)
            if tool_spec:
                exposed.append(self._wrap_tool(tool_spec))
        
        return exposed
    
    async def list_tools(self) -> str:
        """List all available MCP tools."""
        
        all_tools = self.mcp_tool.list_tools()
        
        lines = ["Available MCP tools:"]
        for tool in all_tools:
            status = "✓ loaded" if tool.name in self.loaded_tools else "○ not loaded"
            lines.append(f"- {tool.name}: {tool.description} [{status}]")
        
        return "\n".join(lines)
    
    async def load_tool(self, name: str) -> str:
        """Load a specific MCP tool."""
        
        if name in self.loaded_tools:
            return f"Tool {name} is already loaded"
        
        tool_spec = self.mcp_tool.get_tool_spec(name)
        if not tool_spec:
            return f"Tool {name} not found"
        
        self.loaded_tools.add(name)
        return f"Loaded tool {name}"
    
    async def unload_tool(self, name: str) -> str:
        """Unload a specific MCP tool."""
        
        if name not in self.loaded_tools:
            return f"Tool {name} is not loaded"
        
        if name in self.always_load:
            return f"Tool {name} is always loaded and cannot be unloaded"
        
        self.loaded_tools.remove(name)
        return f"Unloaded tool {name}"
    
    def _wrap_tool(self, tool_spec: ToolSpec) -> FunctionTool:
        """Wrap an MCP tool spec as a FunctionTool."""
        
        async def caller(**kwargs):
            result = await self.mcp_tool.call_tool(tool_spec.name, kwargs)
            return truncate_mcp_result(result)
        
        return FunctionTool(
            name=tool_spec.name,
            description=tool_spec.description,
            parameters=tool_spec.inputSchema,
            fn=caller,
        )
```

### 实际使用流程

```python
# 1. Agent 启动时
cache = MCPConnectionCache()

# 2. 获取 MCP 工具
lease = await cache.acquire(github_config)
mcp_tool = lease.mcp_tool

# 3. 创建渐进式暴露
exposure = ProgressiveMCPExposure(
    mcp_tool=mcp_tool,
    always_load=["search_issues", "get_issue"],  # 始终加载的工具
)

# 4. Agent 看到的初始工具集
initial_tools = exposure.get_exposed_tools()
# [
#     list_mcp_tools,
#     load_mcp_tool,
#     unload_mcp_tool,
#     search_issues,      ← always_load
#     get_issue,          ← always_load
# ]

# 5. Agent 需要使用其他工具时
# Agent 调用 list_mcp_tools
tools_list = await agent.call_tool("list_mcp_tools", {})
print(tools_list)
# Available MCP tools:
# - create_issue: Create a new issue [○ not loaded]
# - update_issue: Update an existing issue [○ not loaded]
# - search_issues: Search issues [✓ loaded]
# ...

# Agent 决定加载 create_issue
await agent.call_tool("load_mcp_tool", {"name": "create_issue"})
# Loaded tool create_issue

# 6. 现在 create_issue 可用了
exposed_tools = exposure.get_exposed_tools()
# [..., create_issue]
```

### 如果你要自己实现

**最小版本**（只加连接缓存）：
```python
class SimpleMCPCache:
    def __init__(self):
        self.connections = {}
    
    async def get_connection(self, config):
        key = hash_config(config)
        if key not in self.connections:
            self.connections[key] = await connect_mcp(config)
        return self.connections[key]
```

**进阶版本**（加渐进式披露）：
- 加控制工具（list/load/unload）
- 加 always_load 机制
- 加工具截断

---

## 3. 四阶段压缩 —— 如何处理上下文窗口满的问题

### 问题背景

LLM 有上下文窗口限制（如 GPT-4 是 128k tokens）。随着对话变长，你需要压缩历史。

**天真做法**：简单截断。

```python
def truncate(messages, max_tokens):
    while count_tokens(messages) > max_tokens:
        messages.pop(1)  # 移除最早的消息
    return messages
```

**问题**：丢失重要信息，Agent 表现下降。

### Chrys 的做法：四阶段渐进压缩

从细粒度到粗粒度逐步压缩。

### 源码实现详解

```python
# service/context/compaction/strategy.py

class FourPhaseCompactionStrategy:
    """Four-phase progressive compaction strategy."""
    
    def __init__(self, budget_calculator: BudgetCalculator):
        self.budget_calculator = budget_calculator
    
    def compact(self, messages: list[Message], context: CompactionContext) -> bool:
        """Compact messages to fit within budget. Returns True if changed."""
        
        budget = self.budget_calculator.calculate(context)
        current_tokens = sum(count_tokens(m) for m in messages)
        
        if current_tokens <= budget:
            return False  # 不需要压缩
        
        # Phase 1: 旧轮次工具结果摘要化
        if self._phase1_summarize_tool_results(messages, budget):
            return True
        
        # Phase 2: 旧轮次工具调用组完全移除
        if self._phase2_remove_tool_groups(messages, budget):
            return True
        
        # Phase 3: 文本密集的已完成轮次压缩
        if self._phase3_compress_text_turns(messages, budget):
            return True
        
        # Phase 4: 当前轮次丢弃 + LLM 生成累积 LAST_WORDS
        if self._phase4_drop_current_turn_with_accumulating_note(messages, budget):
            return True
        
        return False
    
    def _phase1_summarize_tool_results(
        self, 
        messages: list[Message], 
        budget: int,
    ) -> bool:
        """Phase 1: Summarize old turn tool results.
        
        Keep call arguments, replace results with one-line summaries.
        Oldest-turn-first.
        """
        
        # 找到最早的已完成轮次
        exchanges = list(iter_exchanges(LiveAccessor(), messages))
        
        for exchange in exchanges[:-1]:  # 跳过最后一个（可能还在进行）
            if count_tokens(messages) <= budget:
                break
            
            # 对每个工具结果生成摘要
            for msg_idx in exchange.output_indices:
                msg = messages[msg_idx]
                
                for content in msg.contents:
                    if content.type in TOOL_RESULT_CONTENT_TYPES:
                        # 用一行摘要替换完整结果
                        summary = self._summarize_result(content)
                        content.text = f"[Result summary: {summary}]"
                        
                        # 标记为已压缩
                        content.additional_properties["compacted"] = True
            
            if count_tokens(messages) <= budget:
                return True
        
        return False
    
    def _phase2_remove_tool_groups(
        self, 
        messages: list[Message], 
        budget: int,
    ) -> bool:
        """Phase 2: Remove old turn tool-call groups entirely.
        
        Remove both args and results. Only user/assistant text survives per turn.
        """
        
        exchanges = list(iter_exchanges(LiveAccessor(), messages))
        
        for exchange in exchanges[:-1]:
            if count_tokens(messages) <= budget:
                break
            
            # 移除所有工具调用和结果消息
            indices_to_remove = set(exchange.response_indices) | set(exchange.output_indices)
            
            # 保留有文本内容的 assistant 消息
            for msg_idx in exchange.response_indices:
                msg = messages[msg_idx]
                if any(c.type == "text" for c in msg.contents):
                    indices_to_remove.discard(msg_idx)
            
            # 标记为已移除
            for idx in sorted(indices_to_remove, reverse=True):
                messages[idx].additional_properties["compacted_removed"] = True
            
            if count_tokens(messages) <= budget:
                return True
        
        return False
    
    def _phase3_compress_text_turns(
        self, 
        messages: list[Message], 
        budget: int,
    ) -> bool:
        """Phase 3: Compress completed text-heavy turns.
        
        Oldest-turn-first when Phase 1/2 can't reach budget.
        """
        
        # 找到纯文本的轮次（没有工具调用）
        turns = self._extract_text_turns(messages)
        
        for turn in turns[:-1]:
            if count_tokens(messages) <= budget:
                break
            
            # 用 LLM 生成摘要
            summary = self._summarize_turn_with_llm(turn)
            
            # 替换整个轮次为一个摘要消息
            replacement = Message(
                role="assistant",
                contents=[Content(type="text", text=f"[Turn summary: {summary}]")],
            )
            replacement.additional_properties["compacted_summary"] = True
            
            # 替换原轮次
            start_idx = turn[0].index
            end_idx = turn[-1].index + 1
            messages[start_idx:end_idx] = [replacement]
            
            if count_tokens(messages) <= budget:
                return True
        
        return False
    
    def _phase4_drop_current_turn_with_accumulating_note(
        self, 
        messages: list[Message], 
        budget: int,
    ) -> bool:
        """Phase 4: Drop current turn tool calls + accumulate LAST_WORDS note.
        
        All current-turn tool-call and inline assistant-text messages dropped
        in a single shot, replaced by a <system-reminder>[LAST_WORDS]…</system-reminder>
        block appended to the user message's content array.
        
        The note is LLM-generated (no programmatic fallback); transient generator
        failures retry the same compaction input with backoff. On subsequent Phase 4
        triggers within the same turn, the previous note + newly-dropped work feed
        back to the generator so the note accumulates insight without re-summarising
        from scratch.
        """
        
        # 找到当前轮次
        current_turn = self._find_current_turn(messages)
        if not current_turn:
            return False
        
        # 读取上一轮的 LAST_WORDS note（如果有）
        prev_note = self._find_last_words_note(messages)
        
        # 收集要丢弃的工作
        dropped_work = []
        
        for msg in current_turn:
            for content in msg.contents:
                if content.type in TOOL_CALL_CONTENT_TYPES:
                    dropped_work.append({
                        "type": "tool_call",
                        "name": content.name,
                        "args_preview": str(content.args)[:100],
                    })
                elif content.type == "text" and msg.role == "assistant":
                    dropped_work.append({
                        "type": "assistant_text",
                        "text_preview": content.text[:100],
                    })
        
        # 生成新的 LAST_WORDS note
        new_note = self._generate_last_words_note(
            previous_note=prev_note,
            newly_dropped=dropped_work,
        )
        
        # 丢弃当前轮次的工具调用和内联助手文本
        for msg in current_turn:
            msg.contents = [
                c for c in msg.contents
                if c.type not in TOOL_CALL_CONTENT_TYPES
                and not (c.type == "text" and msg.role == "assistant")
            ]
        
        # 注入 LAST_WORDS note 到用户消息
        user_msg = current_turn[0]
        user_msg.contents.append(
            Content(
                type="system_reminder",
                text=f"[LAST_WORDS]{new_note}[/LAST_WORDS]",
            )
        )
        
        return True
    
    def _generate_last_words_note(
        self,
        previous_note: str | None,
        newly_dropped: list[dict],
    ) -> str:
        """Generate a LAST_WORDS note using LLM."""
        
        prompt = f"""You are summarizing work that was dropped due to context window limits.

Previous summary: {previous_note or 'None'}

Newly dropped work:
{json.dumps(newly_dropped, indent=2)}

Generate a concise summary that accumulates insight from both the previous summary and the newly dropped work. Focus on what was learned, what was attempted, and what remains to be done.

Keep it under 200 words."""
        
        # 调用 LLM 生成摘要
        response = self.llm.generate(prompt)
        
        return response.text
```

### 关键设计点

1. **渐进式压缩**：
   - Phase 1：最轻量，只替换结果为摘要
   - Phase 2：中度，移除整个工具调用组
   - Phase 3：重度，用 LLM 摘要整轮对话
   - Phase 4：最激进，丢弃当前轮次 + 累积 note

2. **累积 LAST_WORDS**：
   - 第一次触发：生成初始 note
   - 第二次触发：把上次的 note + 新丢弃的工作回馈给 LLM
   - 这样 note 会越来越丰富，而不会从头重摘要

3. **Oldest-turn-first**：
   - 总是从最早的轮次开始压缩
   - 保留最近的上下文

### 如果你要自己实现

**最小版本**：
```python
def simple_compact(messages, max_tokens):
    # 简单截断
    while count_tokens(messages) > max_tokens:
        messages.pop(1)
    return messages
```

**进阶版本**：
- 加 Phase 1（结果摘要化）
- 加 Phase 4（LAST_WORDS note）

**生产级版本**：
- 完整的四阶段压缩
- 与 Exchange Grammar 集成
- 排除 ledger（防止丢弃仍在引用的内容）

---

## 4. 双进程工作流 SDK —— 如何在用户解释器中运行工作流

### 问题背景

你想让用户定义工作流：

```yaml
# my_workflow.yaml
steps:
  - name: analyze_code
    agent: Code
    instructions: "Analyze the code structure"
  
  - name: write_tests
    agent: QA
    instructions: "Write tests based on the analysis"
  
  - name: run_tests
    tool: shell
    command: "pytest tests/"
```

**问题**：工作流可能需要特定的 Python 版本或依赖。如果在 Chrys 的进程中运行，可能会冲突。

### Chrys 的做法：双进程架构

**主进程**：Chrys 本体，负责调度和协调。

**Worker 进程**：用户的 Python 解释器，运行工作流逻辑。

### 源码实现详解

#### Part 1: Worker Host（用户环境）

```python
# service/workflows/worker_host.py
# 这个文件在用户的 Python 解释器中运行
# 必须是纯标准库，Python 3.9 语法

import sys
import json
import asyncio
from pathlib import Path

def main():
    """Worker host entry point."""
    
    # 1. 从命令行参数获取 SDK 路径
    sdk_dir = sys.argv[1]
    
    # 2. dup(1) 创建私有协议写入 fd
    protocol_fd = os.dup(1)
    
    # 3. dup2 fd 1 和 2 到捕获管道
    # 使 os.write(1, ...) 和 C 扩展无法触及协议通道
    capture_r, capture_w = os.pipe()
    os.dup2(capture_w, 1)
    os.dup2(capture_w, 2)
    
    # 4. 替换 sys.stdout/stderr 为上下文本地捕获
    sys.stdout = ContextLocalWriter(capture_w)
    sys.stderr = ContextLocalWriter(capture_w)
    
    # 5. 将 SDK 前置到 sys.path
    sys.path.insert(0, sdk_dir)
    
    # 6. 导入 chrys.workflows
    import chrys.workflows as workflows
    
    # 7. 服务请求
    asyncio.run(serve_requests(protocol_fd, workflows))

async def serve_requests(protocol_fd: int, workflows):
    """Serve JSON-RPC requests from the main process."""
    
    reader = asyncio.StreamReader()
    protocol = asyncio.StreamReaderProtocol(reader)
    transport, _ = await asyncio.get_event_loop().connect_read_pipe(
        lambda: protocol, 
        os.fdopen(protocol_fd, 'r')
    )
    
    while True:
        # 读取请求
        line = await reader.readline()
        if not line:
            break
        
        request = json.loads(line)
        
        # 处理请求
        method = request["method"]
        params = request["params"]
        
        if method == "load_workflow":
            workflow_file = params["file"]
            workflow = workflows.load(workflow_file)
            result = {"workflow_id": id(workflow)}
        
        elif method == "execute_step":
            workflow_id = params["workflow_id"]
            step_name = params["step_name"]
            context = params["context"]
            
            workflow = workflows.get(workflow_id)
            result = await workflow.execute_step(step_name, context)
        
        elif method == "close":
            break
        
        # 发送响应
        response = {"result": result}
        transport.write((json.dumps(response) + "\n").encode())

if __name__ == "__main__":
    main()
```

**关键设计点**：

1. **协议通道隔离**：
   ```python
   protocol_fd = os.dup(1)  # 复制 stdout fd
   os.dup2(capture_w, 1)    # 重定向 stdout 到捕获管道
   ```
   - 协议通过 `protocol_fd` 通信
   - 用户的 `print()` 输出到捕获管道，不会干扰协议

2. **纯标准库**：
   - 不导入任何第三方库
   - 确保在任何 Python 环境中都能运行

3. **Python 3.9 语法**：
   - 不使用 3.10+ 的特性（如 match/case）
   - 确保兼容性

#### Part 2: 主进程调度器

```python
# orchestration/workflows/coordinator.py

class WorkflowCoordinator:
    """Coordinate workflow execution across processes."""
    
    def __init__(self):
        self.workers = {}  # workflow_id -> WorkerProcess
    
    async def run_workflow(
        self,
        workflow_file: Path,
        interpreter: str = "python3",
    ):
        """Run a workflow in a separate process."""
        
        # 1. 启动 worker 进程
        worker = await self._spawn_worker(workflow_file, interpreter)
        
        try:
            # 2. 加载工作流
            workflow_id = await worker.call("load_workflow", {"file": str(workflow_file)})
            
            # 3. 执行步骤
            context = {}
            
            for step in workflow.steps:
                if step.type == "agent":
                    # Agent 步骤在主进程中执行
                    result = await self._run_agent_step(step, context)
                
                elif step.type == "python":
                    # Python 步骤在 worker 中执行
                    result = await worker.call(
                        "execute_step",
                        {
                            "workflow_id": workflow_id,
                            "step_name": step.name,
                            "context": context,
                        }
                    )
                
                context[step.name] = result
            
            return context
        
        finally:
            # 4. 关闭 worker
            await worker.call("close", {})
            await worker.terminate()
    
    async def _spawn_worker(self, workflow_file: Path, interpreter: str):
        """Spawn a worker process."""
        
        # 获取 worker_host.py 的路径
        worker_host = Path(__file__).parent / "worker_host.py"
        
        # 获取 SDK 目录
        sdk_dir = Path(__file__).parent.parent / "workflows" / "sdk"
        
        # 启动子进程
        proc = await asyncio.create_subprocess_exec(
            interpreter,
            str(worker_host),
            str(sdk_dir),
            stdin=asyncio.subprocess.DEVNULL,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )
        
        return WorkerProcess(proc)

class WorkerProcess:
    """Wrapper around a worker subprocess."""
    
    def __init__(self, proc: asyncio.subprocess.Process):
        self.proc = proc
        self.request_id = 0
    
    async def call(self, method: str, params: dict) -> Any:
        """Call a method in the worker process."""
        
        self.request_id += 1
        request = {
            "id": self.request_id,
            "method": method,
            "params": params,
        }
        
        # 发送请求
        self.proc.stdin.write((json.dumps(request) + "\n").encode())
        await self.proc.stdin.drain()
        
        # 读取响应
        line = await self.proc.stdout.readline()
        response = json.loads(line)
        
        if "error" in response:
            raise WorkflowError(response["error"])
        
        return response["result"]
    
    async def terminate(self):
        """Terminate the worker process."""
        self.proc.terminate()
        await self.proc.wait()
```

#### Part 3: SDK（用户编写工作流）

```python
# service/workflows/sdk/__init__.py
# 纯标准库，Python 3.9 语法

from dataclasses import dataclass
from typing import Callable, Any

@dataclass
class WorkflowStep:
    """A single step in a workflow."""
    name: str
    type: str  # "agent", "python", "shell"
    fn: Callable | None = None
    agent: str | None = None
    instructions: str | None = None
    command: str | None = None

class Workflow:
    """A user-defined workflow."""
    
    def __init__(self, steps: list[WorkflowStep]):
        self.steps = steps
    
    async def execute_step(self, step_name: str, context: dict) -> Any:
        """Execute a single step."""
        
        step = next(s for s in self.steps if s.name == step_name)
        
        if step.type == "python":
            return await step.fn(context)
        
        elif step.type == "shell":
            import subprocess
            result = subprocess.run(step.command, shell=True, capture_output=True, text=True)
            return {"stdout": result.stdout, "stderr": result.stderr, "returncode": result.returncode}
        
        else:
            raise ValueError(f"Unknown step type: {step.type}")

def load(workflow_file: str) -> Workflow:
    """Load a workflow from a file."""
    
    # 动态加载用户的工作流模块
    import importlib.util
    spec = importlib.util.spec_from_file_location("user_workflow", workflow_file)
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)
    
    # 期望用户定义了 workflow 变量
    return module.workflow
```

### 实际使用示例

```python
# user_workflow.py（用户编写）

from chrys.workflows.sdk import Workflow, WorkflowStep

async def analyze_code(context: dict) -> dict:
    """Analyze code structure."""
    # 可以使用用户的任何依赖
    import ast
    import my_custom_linter
    
    code = context["code"]
    tree = ast.parse(code)
    
    return {
        "functions": [node.name for node in ast.walk(tree) if isinstance(node, ast.FunctionDef)],
        "lint_errors": my_custom_linter.check(code),
    }

workflow = Workflow([
    WorkflowStep(name="analyze", type="python", fn=analyze_code),
    WorkflowStep(name="format", type="shell", command="black ."),
])
```

```bash
# 运行工作流
icode workflow run user_workflow.py --interpreter python3.9
```

### 关键设计点

1. **双进程隔离**：
   - 主进程：Chrys 本体，负责调度
   - Worker 进程：用户环境，运行工作流逻辑
   - 通过 JSON-RPC over pipe 通信

2. **协议通道隔离**：
   - `os.dup(1)` 创建私有协议 fd
   - `dup2` 重定向 stdout/stderr 到捕获管道
   - 用户的 `print()` 不会干扰协议

3. **纯标准库 SDK**：
   - 不依赖任何第三方库
   - 确保在任何 Python 环境中都能运行

4. **Python 3.9 语法**：
   - 不使用现代 Python 特性
   - 确保最大兼容性

### 如果你要自己实现

**最小版本**（单进程）：
```python
import subprocess

def run_workflow(workflow_file):
    # 直接 exec 用户的工作流
    exec(open(workflow_file).read())
```

**进阶版本**（双进程）：
- 加 JSON-RPC 通信
- 加协议通道隔离
- 加错误处理

**生产级版本**：
- 完整的 SDK
- 多种步骤类型（agent、python、shell）
- 并发控制和超时

---

## 5. 总结：Service 层的核心价值

### 四大创新的协同

```
┌─────────────────────────────────────────────────────────┐
│                   Service 层架构                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  用户请求                                                │
│       ↓                                                 │
│  ① LLM Client Factory                                   │
│     - api_style 正交轴                                  │
│     - 多 Provider 支持                                  │
│       ↓                                                 │
│  ② MCP Connection Cache                                 │
│     - 连接复用                                          │
│     - 渐进式披露                                        │
│       ↓                                                 │
│  ③ 四阶段压缩                                           │
│     - 渐进式上下文管理                                  │
│     - 累积 LAST_WORDS                                   │
│       ↓                                                 │
│  ④ 双进程工作流 SDK                                     │
│     - 用户环境隔离                                      │
│     - 纯标准库 SDK                                      │
│       ↓                                                 │
│  返回结果给用户                                          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 学习建议

1. **先看 LLM Client Factory**：理解 api_style 正交轴的设计
2. **再看 MCP Cache**：理解连接缓存和渐进式披露
3. **然后看四阶段压缩**：理解渐进式上下文管理
4. **最后看双进程工作流**：理解进程隔离和协议设计

### 如果你要自己实现

**优先级排序**：

1. **LLM Client Factory**（必需）
   - 最小版本：直接调用 OpenAI SDK
   - 进阶版本：加 api_style 抽象

2. **MCP Cache**（推荐）
   - 最小版本：简单的连接字典
   - 进阶版本：加引用计数和渐进式披露

3. **四阶段压缩**（可选）
   - 最小版本：简单截断
   - 进阶版本：加 Phase 1 和 Phase 4

4. **双进程工作流**（高级）
   - 最小版本：单进程 exec
   - 进阶版本：加 JSON-RPC 和进程隔离

---

## 附录：关键源码文件索引

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| `service/llm/clients.py` | ~300 | LLM Client Factory |
| `service/mcp/cache.py` | ~400 | MCP Connection Cache |
| `service/mcp/_progressive.py` | ~150 | Progressive Disclosure |
| `service/context/compaction/strategy.py` | ~500 | Four-Phase Compaction |
| `service/workflows/worker_host.py` | ~200 | Worker Host (纯标准库) |
| `service/workflows/sdk/__init__.py` | ~100 | Workflow SDK (纯标准库) |

---

**下一步**：想看 Orchestration 层的实现吗？那里有更复杂的编排逻辑：ExecutionLease、三条构建路径、Turn Pass 机制。
