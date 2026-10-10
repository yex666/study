# S04-S08: mcp/skills/hooks/approval/context — 集成与策略模块

> **文件路径**: 
> - `src/chrys/service/mcp/` (MCP 协议集成)
> - `src/chrys/service/skills/` (技能系统)
> - `src/chrys/service/hooks/` (钩子系统)
> - `src/chrys/service/approval/` (审批系统)
> - `src/chrys/service/context/` (上下文压缩)
> 
> **一句话**: Service 层的集成模块——连接外部工具、执行钩子、控制审批、管理上下文。

---

## S04: mcp — MCP 协议集成

### 核心文件

```
service/mcp/
├── cache.py           ← 连接缓存（核心）
├── adapter.py         ← MCP 工具适配为 FunctionTool
├── _progressive.py    ← 渐进式披露（按需加载工具）
├── _connection.py     ← 连接管理
├── _stdio_transport.py ← stdio 传输（自有实现）
├── _http_transport.py  ← HTTP 传输（自有实现）
├── owned.py           ← 自有 MCP 引擎
├── validation.py      ← 工具名称验证
└── result_limits.py   ← 结果截断
```

### 核心设计 1: 连接缓存

```python
class MCPConnectionCache:
    """Shared live MCP connection cache."""
    
    async def acquire(self, config: MCPServerConfig) -> MCPConnectionLease:
        """Acquire a connection, sharing if config matches."""
        
        key = compute_cache_key(config)
        
        async with self._lock:
            if key not in self._entries:
                # 首次连接
                self._entries[key] = MCPConnectionEntry(...)
            
            entry = self._entries[key]
            entry.refcount += 1
            
            if entry.mcp_tool is None:
                # 等待连接完成
                entry.mcp_tool = await entry.connect_task
        
        return MCPConnectionLease(key=key, functions=...)
```

**为什么**：多个 Agent 配置相同 MCP server 时，共享连接避免重复 connect/list。

### 核心设计 2: 渐进式披露

大型 MCP server（50+ 工具）会撑爆上下文窗口。解决方案：按需加载。

```python
class ProgressiveMCPExposure:
    """Lazy loading of MCP tools."""
    
    def __init__(self, mcp_tool, always_load: list[str]):
        self.loaded_tools = set(always_load)
        
        # 控制工具
        self.control_tools = [
            FunctionTool("list_mcp_tools", self.list_tools),
            FunctionTool("load_mcp_tool", self.load_tool),
            FunctionTool("unload_mcp_tool", self.unload_tool),
        ]
    
    async def list_tools(self) -> str:
        """List all available tools."""
        ...
    
    async def load_tool(self, name: str) -> str:
        """Load a specific tool."""
        self.loaded_tools.add(name)
        return f"Loaded tool {name}"
```

### 📚 详细解析

→ **service_implementation.md** 第 2 节：MCP Connection Cache（含完整源码注释）

---

## S05: skills — 技能系统

### 核心文件

```
service/skills/
├── loader.py    ← 技能发现（扫描目录、解析 SKILL.md）
└── runner.py    ← 脚本执行（按扩展名选择解释器）
```

### 什么是 Skill？

一个目录包含 `SKILL.md`（带 YAML frontmatter）：

```markdown
---
name: aws_deploy
description: How to deploy to AWS using CDK
---

# AWS Deployment

Instructions for deploying to AWS...
```

### 脚本执行

```python
def resolve_interpreter(script_path: Path) -> str:
    """Resolve the interpreter for a script."""
    
    ext = script_path.suffix
    
    if ext in (".js", ".mjs"):
        return "node"
    elif ext == ".ts":
        return "npx tsx"
    elif ext == ".py":
        # Python 优先用 uv run（支持 PEP 723 内联元数据）
        if shutil.which("uv"):
            return "uv run"
        return shutil.which("python3") or shutil.which("python") or sys.executable
    elif ext == ".sh":
        return "bash"
    ...
```

**关键设计**：
- **隔离**：脚本在子进程中执行，不影响主进程
- **截断**：结果默认 8k tokens，完整文本溢出到 `tool_results/`
- **安全**：路径遍历和符号链接逃逸守卫

---

## S06: hooks — 钩子系统

### 核心文件

```
service/hooks/
├── manager.py    ← HookManager（单一入口）
├── runner.py     ← 子进程执行
└── outbox.py     ← 持久化任务队列
```

### 支持的事件

| 事件 | 可阻塞？ | 作用 |
|------|---------|------|
| `user_prompt_submit` | ✅ | 用户输入提交前 |
| `before_tool_call` | ✅ | 工具调用前（可拒绝/改写参数） |
| `after_tool_call` | ❌ | 工具调用后（添加上下文） |
| `tool_error` | ❌ | 工具出错时 |

### 配置方式

```yaml
# ~/.chrys/hooks/hooks.yaml
user_prompt_submit:
  - command: "python3 ~/.chrys/hooks/check_secrets.py"
    timeout: 5s

before_tool_call:
  - match:
      tool_kind: shell
    command: "python3 ~/.chrys/hooks/scan_command.py"
```

**关键设计**：
- **外部子进程**：钩子是在独立进程中执行的命令
- **项目优先**：项目 `.chrys/hooks/` 先运行，其块结束链
- **永不绕过审批**：钩子可以拒绝，但不可以绕过审批

---

## S07: approval — 审批系统

### 核心文件

```
service/approval/
└── policy.py    ← ApprovalPolicy
```

### 三种模式

```python
class ApprovalMode(Enum):
    MANUAL = "manual"   # 每次都要问用户
    AUTO = "auto"       # LLM 判断是否需要问
    BYPASS = "bypass"   # 不问，直接执行（headless 模式）
```

### 解析链（哪个规则生效）

```
1. 会话级覆盖（如果 user_can_override）
2. kind.tool_name 限定覆盖（如 "shell.rm"）
3. 裸精确工具名覆盖（如 "read_file"）
4. Kind-only 覆盖（如 "shell"）
5. run_skill_script → require
6. Web kinds → require
7. Profile 默认
```

```python
def _resolve_rule(tool_name: str, tool_kind: str, overrides: dict) -> ApprovalRule:
    """First match wins."""
    
    # 1. 会话覆盖
    if session_override:
        return session_override
    
    # 2. 最长的已知 kind 前缀
    for kind in sorted(kinds, key=len, reverse=True):
        if tool_name.startswith(kind + "."):
            if f"{kind}.{tool_name.split('.', 1)[1]}" in overrides:
                return overrides[...]
    
    # 3. 裸工具名
    if tool_name in overrides:
        return overrides[tool_name]
    
    # 4. Kind
    if tool_kind in overrides:
        return overrides[tool_kind]
    
    # 5-6. 默认规则
    if tool_kind in ("run_skill_script", "web"):
        return ApprovalRule.REQUIRE
    
    # 7. Profile 默认
    return profile_default
```

---

## S08: context — 上下文压缩

### 核心文件

```
service/context/
├── compaction/strategy.py    ← 四阶段压缩（核心）
├── compaction/budgets.py     ← 预算计算
├── middleware/               ← 压缩中间件
└── providers/                ← 上下文提供者
```

### 四阶段压缩

```python
class FourPhaseCompactionStrategy:
    """Four-phase progressive compaction strategy."""
    
    def compact(self, messages, budget) -> bool:
        # Phase 1: 旧轮次工具结果摘要化（替换为一行摘要）
        if self._phase1_summarize_tool_results(messages, budget):
            return True
        
        # Phase 2: 旧轮次工具调用组完全移除
        if self._phase2_remove_tool_groups(messages, budget):
            return True
        
        # Phase 3: 文本密集轮次用 LLM 摘要
        if self._phase3_compress_text_turns(messages, budget):
            return True
        
        # Phase 4: 丢弃当前轮次 + LLM 生成累积 LAST_WORDS note
        if self._phase4_drop_current_turn(messages, budget):
            return True
        
        return False
```

### 触发阈值

```python
def calculate_trigger_threshold(window, max_output, reserve) -> int:
    """When to trigger compaction."""
    return window - max_output - max(5000, int(window * 0.05))
```

### 📚 详细解析

→ **service_implementation.md** 第 3 节：四阶段压缩（含完整源码注释）

---

## 下一步

- **上一篇**: S01-S03 核心模块
- **下一篇**: S09-S13 合并文档（Agent 中间件/变更追踪/会话持久化/状态序列化/工作流）
