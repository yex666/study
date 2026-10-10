# 从零手搓 Coding Agent：进阶组件

> 上一份文档讲了 8 个核心组件（LLM Client、Tool System、Message Loop 等）。本文补充 **10 个进阶组件**，让你的 Agent 从 Demo 升级到生产级。

---

## 1. Streaming（流式输出）⭐⭐⭐⭐⭐

### 为什么需要？

没有 Streaming，用户要等 LLM 生成完所有内容才能看到结果。有了它，用户可以**实时看到 LLM 在"想什么"**，体验大幅提升。

### 最小实现（30 行代码）

```python
async def chat_stream(self, messages: list[dict]) -> AsyncIterator[str]:
    """流式聊天"""
    stream = self.client.chat.completions.create(
        model=self.model,
        messages=messages,
        stream=True,
    )
    
    full_response = ""
    for chunk in stream:
        if chunk.choices[0].delta.content:
            content = chunk.choices[0].delta.content
            full_response += content
            yield content  # 实时产出内容
    
    return full_response

# 使用
async for token in llm.chat_stream(messages):
    print(token, end="", flush=True)
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **OpenAI SDK** | 原生支持 `stream=True` | ✅ 简单 ❌ 只有文本流 |
| **Claude Code** | 流式文本 + 工具调用 | ✅ 完整 ❌ 闭源 |
| **Chrys** | 流式文本 + 工具调用 + 中间文本 | ✅ 最完整 ❌ 复杂 |

### Chrys 的创新：**中间文本捕获**

```python
# 问题：LLM 在调用工具前会说"让我读一下这个文件"
# 这段文本应该实时显示给用户，而不是等工具返回后才显示

class InstrumentedClient:
    async def get_response(self, messages, tools):
        stream = await self.inner_client.stream(messages, tools)
        
        async for update in stream:
            # 检查是否有"中间文本"（工具调用前的说明）
            if update.type == "text" and not update.is_final:
                # 实时显示给用户
                await self.on_intermediate_text(update.text)
            
            # 检查是否有工具调用
            if update.type == "tool_call":
                # 先显示已积累的中间文本
                await self.flush_intermediate_text()
                # 然后执行工具
                result = await self.execute_tool(update.tool_call)
```

**好处**：用户能看到 Agent 的思考过程，而不是干等。

---

## 2. Trajectory Tracking（轨迹追踪）⭐⭐⭐⭐

### 为什么需要？

当 Agent 表现不好时，你需要知道：
- 哪一步出了问题？
- LLM 花了多少时间？
- 工具调用是否合理？

没有追踪，调试就是盲人摸象。

### 最小实现（40 行代码）

```python
import time
from dataclasses import dataclass

@dataclass
class TraceEvent:
    type: str  # "llm_call", "tool_call", "tool_result"
    timestamp: float
    duration: float
    details: dict

class SimpleTrajectoryTracker:
    def __init__(self):
        self.events = []
    
    def start_llm_call(self, messages: list[dict]):
        self.current_event = {
            "type": "llm_call",
            "start_time": time.time(),
            "messages": messages,
        }
    
    def end_llm_call(self, response: dict):
        event = self.current_event
        event["end_time"] = time.time()
        event["duration"] = event["end_time"] - event["start_time"]
        event["response"] = response
        self.events.append(event)
    
    def log_tool_call(self, tool_name: str, args: dict, result: dict, duration: float):
        self.events.append({
            "type": "tool_call",
            "timestamp": time.time(),
            "tool_name": tool_name,
            "args": args,
            "result": result,
            "duration": duration,
        })
    
    def export(self) -> list[dict]:
        return self.events
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **LangChain** | Callbacks + LangSmith | ✅ 可视化 ❌ 依赖外部服务 |
| **OpenAI SDK** | 无内建追踪 | ❌ 需要自己实现 |
| **Chrys** | 结构化事件流 + JSONL 导出 | ✅ 完整 ❌ 复杂 |

### Chrys 的做法

**结构化事件流**：

```python
class TrajectoryWriter:
    async def emit(self, event_type: str, **kwargs):
        """发射一个轨迹事件"""
        event = {
            "type": event_type,
            "timestamp": datetime.now().isoformat(),
            "operation_id": self.current_operation_id,
            **kwargs,
        }
        
        # 写入 JSONL 文件
        self.file.write(json.dumps(event) + "\n")
    
    async def track_llm_call(self, messages, response, duration):
        await self.emit("model.cycle", 
            input_tokens=self.count_tokens(messages),
            output_tokens=self.count_tokens(response),
            duration_ms=duration * 1000,
        )
    
    async def track_tool_call(self, tool_name, args, result, duration):
        await self.emit("tool.operation",
            tool_name=tool_name,
            args=args,
            result_preview=result[:200],  # 只存前 200 字符
            duration_ms=duration * 1000,
        )
```

**好处**：
- 可以回放整个执行过程
- 可以分析性能瓶颈（哪个工具最慢？）
- 可以用于审计和合规

---

## 3. Sub-agents（子代理）⭐⭐⭐⭐

### 为什么需要？

复杂任务需要分工协作。比如：
- 主 Agent 负责理解需求
- 子 Agent A 负责搜索代码
- 子 Agent B 负责写测试
- 子 Agent C 负责运行测试

### 最小实现（50 行代码）

```python
class SubAgent:
    def __init__(self, name: str, instructions: str, tools: list[ToolDefinition]):
        self.name = name
        self.instructions = instructions
        self.tools = tools
        self.llm = SimpleLLMClient(api_key="...")
    
    async def run(self, task: str) -> str:
        """执行子任务"""
        messages = [
            {"role": "system", "content": self.instructions},
            {"role": "user", "content": task},
        ]
        
        # 简单的工具循环
        for _ in range(5):
            response = await self.llm.chat(messages, tools=...)
            messages.append(response)
            
            if not response["tool_calls"]:
                return response["content"]
            
            for tool_call in response["tool_calls"]:
                result = await self.execute_tool(tool_call)
                messages.append({"role": "tool", "content": result})
        
        return "Sub-agent failed"

class MainAgent:
    def __init__(self):
        self.sub_agents = {
            "search": SubAgent("search", "You search code", [...]),
            "test": SubAgent("test", "You write tests", [...]),
        }
    
    async def run(self, user_input: str) -> str:
        # 主 Agent 可以调用子 Agent
        search_result = await self.sub_agents["search"].run("Find the login function")
        test_code = await self.sub_agents["test"].run(f"Write tests for: {search_result}")
        return test_code
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **OpenAI Swarm** | 简单的 Agent 切换 | ✅ 易用 ❌ 功能有限 |
| **AutoGen** | 多 Agent 对话 | ✅ 灵活 ❌ 复杂 |
| **Chrys** | 父子关系 + 并发控制 | ✅ 精细 ❌ 复杂 |

### Chrys 的创新：**并发控制**

```python
class SubAgentManager:
    def __init__(self, max_concurrent: int = 3):
        self.max_concurrent = max_concurrent
        self.active_count = 0
        self.semaphore = asyncio.Semaphore(max_concurrent)
    
    async def spawn(self, sub_agent: SubAgent, task: str) -> str:
        """启动子 Agent，受并发限制"""
        async with self.semaphore:
            self.active_count += 1
            try:
                return await sub_agent.run(task)
            finally:
                self.active_count -= 1
```

**好处**：防止同时启动太多子 Agent，导致资源耗尽。

---

## 4. Workflows（工作流）⭐⭐⭐⭐

### 为什么需要？

有些任务是**预定义的多步骤流程**，比如：
1. 读取需求文档
2. 分析代码结构
3. 生成设计方案
4. 实现代码
5. 写测试
6. 运行测试

用 Message Loop 硬写会很乱，用 Workflow 可以清晰定义步骤。

### 最小实现（60 行代码）

```python
from typing import Callable

class WorkflowStep:
    def __init__(self, name: str, fn: Callable):
        self.name = name
        self.fn = fn

class SimpleWorkflow:
    def __init__(self, steps: list[WorkflowStep]):
        self.steps = steps
    
    async def run(self, context: dict) -> dict:
        """执行工作流"""
        for step in self.steps:
            print(f"Running step: {step.name}")
            context = await step.fn(context)
        return context

# 定义工作流
async def read_requirements(ctx):
    ctx["requirements"] = "..."
    return ctx

async def analyze_code(ctx):
    ctx["analysis"] = "..."
    return ctx

async def generate_design(ctx):
    ctx["design"] = "..."
    return ctx

workflow = SimpleWorkflow([
    WorkflowStep("read_requirements", read_requirements),
    WorkflowStep("analyze_code", analyze_code),
    WorkflowStep("generate_design", generate_design),
])

# 运行
result = await workflow.run({})
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **LangGraph** | 图状态机 | ✅ 强大 ❌ 学习曲线陡 |
| **Prefect** | DAG 编排 | ✅ 成熟 ❌ 面向数据工程 |
| **Chrys** | YAML 定义 + Python 执行 | ✅ 灵活 ❌ 自研协议 |

### Chrys 的创新：**双进程架构**

```python
# 主进程（Chrys）
class WorkflowCoordinator:
    async def run_workflow(self, workflow_file: Path):
        # 在用户的 Python 解释器中启动 worker
        worker = await self.spawn_worker(workflow_file)
        
        # 通过 JSON-RPC 通信
        result = await worker.execute_step("step1")
        ...

# Worker 进程（用户环境）
class WorkflowWorker:
    def __init__(self):
        # 加载用户的工作流定义
        self.workflow = load_workflow("my_workflow.py")
    
    async def execute_step(self, step_name: str):
        return await self.workflow.steps[step_name].fn()
```

**好处**：
- 工作流可以使用用户环境的依赖（如特定的库版本）
- 主进程崩溃不影响工作流执行
- 可以在不同机器上运行 worker

---

## 5. MCP Integration（MCP 协议集成）⭐⭐⭐⭐

### 为什么需要？

你可能想连接各种外部工具：
- GitHub API
- Jira
- Slack
- 数据库
- 自定义 API

MCP（Model Context Protocol）是一个标准协议，让 Agent 能统一接入这些工具。

### 最小实现（50 行代码）

```python
class MCPServer:
    """简化的 MCP server"""
    
    def __init__(self, name: str):
        self.name = name
        self.tools = {}
    
    def register_tool(self, name: str, fn: Callable, schema: dict):
        self.tools[name] = {"fn": fn, "schema": schema}
    
    async def list_tools(self) -> list[dict]:
        return [{"name": name, **tool["schema"]} for name, tool in self.tools.items()]
    
    async def call_tool(self, name: str, args: dict) -> dict:
        tool = self.tools[name]
        return await tool["fn"](**args)

class MCPClient:
    """简化的 MCP client"""
    
    def __init__(self, server_url: str):
        self.server_url = server_url
    
    async def list_tools(self) -> list[dict]:
        # 实际应该通过 HTTP/WebSocket 调用
        return await self._request("list_tools")
    
    async def call_tool(self, name: str, args: dict) -> dict:
        return await self._request("call_tool", {"name": name, "args": args})
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **Claude Desktop** | 原生支持 MCP | ✅ 官方 ❌ 绑定 Anthropic |
| **Cursor** | 部分支持 | ✅ 实用 ❌ 不完整 |
| **Chrys** | 完整实现 + 连接缓存 | ✅ 生产级 ❌ 复杂 |

### Chrys 的创新：**连接缓存**

```python
class MCPConnectionCache:
    def __init__(self):
        self.connections = {}  # server_url -> connection
    
    async def get_connection(self, server_url: str) -> MCPConnection:
        """复用已有连接"""
        if server_url not in self.connections:
            self.connections[server_url] = await MCPConnection.connect(server_url)
        return self.connections[server_url]
    
    async def close_all(self):
        for conn in self.connections.values():
            await conn.close()
```

**好处**：避免重复建立连接，提升性能。

---

## 6. Memory System（记忆系统）⭐⭐⭐

### 为什么需要？

Agent 可能需要在多次会话间保持知识：
- 用户偏好（"我喜欢用 pytest"）
- 项目约定（"这个项目用 TypeScript"）
- 学到的经验（"上次这个 bug 是因为..."）

### 最小实现（40 行代码）

```python
class SimpleMemory:
    def __init__(self, memory_file: Path):
        self.memory_file = memory_file
        self.memories = self._load()
    
    def _load(self) -> dict:
        if self.memory_file.exists():
            return json.loads(self.memory_file.read_text())
        return {}
    
    def save(self):
        self.memory_file.write_text(json.dumps(self.memories, indent=2))
    
    def add(self, key: str, value: str):
        self.memories[key] = value
        self.save()
    
    def get(self, key: str) -> str | None:
        return self.memories.get(key)
    
    def to_prompt(self) -> str:
        """转换为 system prompt 的一部分"""
        if not self.memories:
            return ""
        
        lines = ["# User Memories"]
        for key, value in self.memories.items():
            lines.append(f"- {key}: {value}")
        return "\n".join(lines)
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **MemGPT** | 向量数据库 + 检索 | ✅ 强大 ❌ 复杂 |
| **LangChain** | ConversationSummaryMemory | ✅ 简单 ❌ 不够智能 |
| **Chrys** | 文件存储 + 按需加载 | ✅ 实用 ❌ 基础 |

---

## 7. Hooks System（钩子系统）⭐⭐⭐

### 为什么需要？

你想在特定时机执行自定义逻辑：
- 工具调用前：检查参数
- 工具调用后：记录日志
- 用户输入提交前：过滤敏感信息

### 最小实现（40 行代码）

```python
class HookSystem:
    def __init__(self):
        self.hooks = {
            "before_tool_call": [],
            "after_tool_call": [],
        }
    
    def register(self, event: str, fn: Callable):
        self.hooks[event].append(fn)
    
    async def fire(self, event: str, **kwargs):
        for hook in self.hooks[event]:
            await hook(**kwargs)

# 使用
hooks = HookSystem()

async def log_tool_call(tool_name: str, args: dict):
    print(f"Calling {tool_name} with {args}")

hooks.register("before_tool_call", log_tool_call)

# 在工具调用前触发
await hooks.fire("before_tool_call", tool_name="read_file", args={"path": "main.py"})
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **LangChain** | Callbacks | ✅ 标准 ❌ 不够灵活 |
| **Chrys** | YAML 配置 + 阻塞/非阻塞 | ✅ 灵活 ❌ 复杂 |

### Chrys 的创新：**阻塞钩子**

```python
class HookDecision(Enum):
    ALLOW = "allow"       # 允许继续
    DENY = "deny"         # 拒绝执行
    MODIFY = "modify"     # 修改参数

async def check_sensitive_data(tool_name: str, args: dict) -> HookDecision:
    """检查是否包含敏感数据"""
    if "password" in str(args).lower():
        return HookDecision.DENY
    return HookDecision.ALLOW

# 钩子可以阻止工具执行
hooks.register("before_tool_call", check_sensitive_data, blocking=True)
```

---

## 8. Skills System（技能系统）⭐⭐⭐

### 为什么需要？

你想让 Agent 掌握一些**可复用的能力**：
- "如何部署到 AWS"
- "如何写一个好的 README"
- "如何处理用户反馈"

### 最小实现（50 行代码）

```python
class Skill:
    def __init__(self, name: str, instructions: str, examples: list[str]):
        self.name = name
        self.instructions = instructions
        self.examples = examples
    
    def to_prompt(self) -> str:
        return f"# {self.name}\n\n{self.instructions}\n\nExamples:\n" + \
               "\n".join(f"- {ex}" for ex in self.examples)

class SkillRegistry:
    def __init__(self):
        self.skills = {}
    
    def register(self, skill: Skill):
        self.skills[skill.name] = skill
    
    def get_skill_instructions(self, name: str) -> str:
        return self.skills[name].to_prompt()

# 定义技能
deploy_skill = Skill(
    name="aws_deploy",
    instructions="How to deploy to AWS using CDK...",
    examples=["cdk deploy", "cdk diff"],
)

registry = SkillRegistry()
registry.register(deploy_skill)

# 在 system prompt 中注入
system_prompt = f"You are a helpful assistant.\n\n{registry.get_skill_instructions('aws_deploy')}"
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **Claude Code** | CLAUDE.md 文件 | ✅ 简单 ❌ 不灵活 |
| **Cursor** | .cursorrules | ✅ 实用 ❌ 绑定 IDE |
| **Chrys** | SKILL.md + YAML frontmatter | ✅ 标准化 ❌ 自研规范 |

---

## 9. Observability（可观测性）⭐⭐⭐

### 为什么需要？

生产环境需要：
- 监控 Agent 的运行状态
- 收集性能指标
- 追踪错误

### 最小实现（30 行代码）

```python
class Metrics:
    def __init__(self):
        self.counters = {}
        self.histograms = {}
    
    def increment(self, name: str, value: int = 1):
        self.counters[name] = self.counters.get(name, 0) + value
    
    def observe(self, name: str, value: float):
        if name not in self.histograms:
            self.histograms[name] = []
        self.histograms[name].append(value)
    
    def export(self) -> dict:
        return {
            "counters": self.counters,
            "histograms": {k: {"avg": sum(v)/len(v), "count": len(v)} 
                          for k, v in self.histograms.items()},
        }

metrics = Metrics()

# 使用
metrics.increment("llm_calls_total")
metrics.observe("llm_duration_seconds", 2.5)
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **LangSmith** | 全托管平台 | ✅ 强大 ❌ 贵 |
| **OpenTelemetry** | 开源标准 | ✅ 通用 ❌ 配置复杂 |
| **Chrys** | OpenTelemetry 集成 | ✅ 标准 ❌ 可选 |

---

## 10. Testing Framework（测试框架）⭐⭐

### 为什么需要？

Agent 系统的边界情况很多，没有测试很難保证质量：
- LLM 输出不确定
- 工具调用可能失败
- 并发场景复杂

### 最小实现（40 行代码）

```python
class MockLLM:
    """模拟 LLM，用于测试"""
    
    def __init__(self, responses: list[dict]):
        self.responses = responses
        self.call_count = 0
    
    async def chat(self, messages, tools=None):
        response = self.responses[self.call_count % len(self.responses)]
        self.call_count += 1
        return response

# 测试用例
async def test_agent_loop():
    mock_llm = MockLLM([
        {"role": "assistant", "content": None, "tool_calls": [...]},
        {"role": "assistant", "content": "Done!", "tool_calls": []},
    ])
    
    agent = SimpleAgent(llm=mock_llm, tools=[...])
    result = await agent.run("Do something")
    
    assert result == "Done!"
    assert mock_llm.call_count == 2
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **LangChain** | Unit tests + Integration tests | ✅ 全面 ❌ 慢 |
| **Chrys** | Architecture tests + Property-based tests | ✅ 严格 ❌ 复杂 |

---

## 总结：优先级建议

### 如果你只有 1 周

只做这 3 个：
1. **Streaming** - 用户体验提升最大
2. **Trajectory Tracking** - 调试必备
3. **基本测试框架** - 保证质量

### 如果你有 1 个月

加上：
4. **Sub-agents** - 处理复杂任务
5. **Hooks** - 扩展性强
6. **Memory** - 长期记忆

### 如果你想做产品

全部都要，但优先级：
1. Streaming > Trajectory > Testing（基础体验）
2. Sub-agents > Workflows（复杂任务）
3. MCP > Memory > Skills（生态集成）
4. Hooks > Observability（运维能力）

---

## 最后的建议

不要试图一次做完所有东西。**先跑通 MVP，再逐步迭代**。

每加一个组件，都问自己：
- 这个解决了什么实际问题？
- 没有它我会遇到什么困难？
- 有没有更简单的实现方式？

祝你好运！🚀
