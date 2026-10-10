# 从零手搓 Coding Agent：你需要什么基础设施？

> **TL;DR**: 构建一个可用的 Coding Agent 需要 **8 个核心组件**。本文从第一性原理出发，说清楚每个组件为什么必需、最小实现长什么样、业界怎么做、Chrys 有什么不同。

---

## 0. 先搞清楚：Coding Agent 到底在干什么？

在你写任何代码之前，先想清楚一个问题：**Coding Agent 的本质是什么？**

答案：**一个能自主调用工具来完成编程任务的 AI 系统。**

用户输入自然语言指令 → Agent 理解意图 → 调用工具（读文件、写代码、执行命令）→ 观察结果 → 继续推理 → 重复直到完成。

这就是一个**工具调用循环（Tool Loop）**。所有其他组件都是为了支撑这个循环可靠运行。

---

## 1. 八大核心组件

```
┌─────────────────────────────────────────────────────────┐
│                     Coding Agent                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ① LLM Client          ← 与大模型对话                   │
│       ↓                                                 │
│  ② Tool System           ← 定义和执行工具               │
│       ↓                                                 │
│  ③ Message Loop          ← 工具调用循环（核心）         │
│       ↓                                                 │
│  ④ Context Management    ← 上下文窗口管理               │
│       ↓                                                 │
│  ⑤ Session Persistence   ← 会话保存和恢复               │
│       ↓                                                 │
│  ⑥ Error Handling        ← 错误处理和重试               │
│       ↓                                                 │
│  ⑦ Safety & Approval     ← 安全控制和审批               │
│       ↓                                                 │
│  ⑧ Platform Abstraction  ← 跨平台支持（可选）           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

下面逐个讲解。

---

## 2. 组件 #1：LLM Client（与大模型对话）

### 为什么必需？

没有 LLM，就没有"Agent"——只是一个普通的脚本。LLM Client 负责：
- 构建请求（system prompt + 用户消息 + 工具定义）
- 发送给 LLM API
- 解析响应（文本、工具调用、流式输出）

### 最小实现（30 行代码）

```python
import openai
from typing import Literal

class SimpleLLMClient:
    def __init__(self, api_key: str, model: str = "gpt-4"):
        self.client = openai.OpenAI(api_key=api_key)
        self.model = model
    
    async def chat(
        self, 
        messages: list[dict], 
        tools: list[dict] | None = None
    ) -> dict:
        """发送聊天请求，返回响应"""
        response = self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            tools=tools,
        )
        
        choice = response.choices[0]
        return {
            "role": "assistant",
            "content": choice.message.content,
            "tool_calls": choice.message.tool_calls or [],
        }
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **LangChain** | 封装多个 LLM Provider 的统一接口 | ✅ 简单 ❌ 抽象过厚，难以定制 |
| **OpenAI SDK** | 直接用官方 SDK | ✅ 原生支持 ❌ 绑定单一 Provider |
| **Claude Code** | 自研 client，支持多 Provider | ✅ 灵活 ❌ 维护成本高 |

### Chrys 的做法

**api_style 正交轴**：把"wire transport"和"request shape"解耦。

```python
# Provider 是传输协议（OpenAI / Anthropic）
# api_style 是请求形状（chat_completions / responses）

def create_client(profile: ModelProfile):
    if profile.provider == "openai":
        if profile.api_style == "chat_completions":
            return OpenAIChatCompletionClient()
        else:  # responses
            return OpenAIResponsesClient()
    elif profile.provider == "deepseek-openai":
        # DeepSeek 兼容 OpenAI，子类化即可
        return DeepSeekClient()  # 继承 OpenAIChatCompletionClient
```

**好处**：添加新 Provider（如 GLM）只需子类化，不需要重写整个 client。

### 如果不做会怎样？

**最简版本也能工作**，但你会遇到：
- 切换 Provider 时要改代码
- 无法处理流式输出
- 错误处理不完善

**建议**：先用最小实现跑通，后续再抽象。

---

## 3. 组件 #2：Tool System（定义和执行工具）

### 为什么必需？

Agent 的能力来自工具。没有工具系统，LLM 只能说不能做。

工具系统负责：
- 定义工具的 schema（名称、描述、参数）
- 验证 LLM 的工具调用参数
- 执行工具并返回结果
- 处理工具执行中的错误

### 最小实现（50 行代码）

```python
from pydantic import BaseModel
from typing import Callable, Any

class ToolDefinition:
    """工具定义"""
    def __init__(self, name: str, description: str, parameters: type[BaseModel], fn: Callable):
        self.name = name
        self.description = description
        self.parameters = parameters
        self.fn = fn
    
    def to_openai_schema(self) -> dict:
        """转换为 OpenAI tool schema"""
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters.model_json_schema(),
            }
        }
    
    async def execute(self, args: dict) -> Any:
        """执行工具"""
        try:
            # 参数验证
            validated_args = self.parameters(**args)
            # 执行函数
            result = await self.fn(**validated_args.model_dump())
            return {"success": True, "result": result}
        except Exception as e:
            return {"success": False, "error": str(e)}

# 定义工具
class ReadFileArgs(BaseModel):
    path: str

async def read_file(path: str) -> str:
    with open(path, 'r', encoding='utf-8') as f:
        return f.read()

read_file_tool = ToolDefinition(
    name="read_file",
    description="Read a file",
    parameters=ReadFileArgs,
    fn=read_file,
)
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **LangChain** | `@tool` 装饰器，自动推断 schema | ✅ 简洁 ❌ 不够灵活 |
| **OpenAI Agents SDK** | `FunctionTool` 类 | ✅ 标准化 ❌ 绑定 OpenAI |
| **Claude Code** | 内置工具集（bash、file editor 等） | ✅ 开箱即用 ❌ 难以扩展 |

### Chrys 的做法

**渐进式工具暴露**：大型 MCP server 可能有 50+ 工具，全部加载会撑爆上下文。

```python
class ProgressiveMCPExposure:
    """按需加载工具"""
    
    def __init__(self, mcp_tools: list[Tool]):
        self.all_tools = mcp_tools
        self.loaded_tools = []  # 当前加载的工具
        self.control_tools = [
            Tool("list_tools", self.list_tools),
            Tool("load_tool", self.load_tool),
            Tool("unload_tool", self.unload_tool),
        ]
    
    async def list_tools(self) -> str:
        """列出所有可用工具"""
        return "\n".join(t.name for t in self.all_tools)
    
    async def load_tool(self, name: str) -> str:
        """加载指定工具"""
        tool = next(t for t in self.all_tools if t.name == name)
        self.loaded_tools.append(tool)
        return f"Loaded {name}"
```

**好处**：Agent 可以先 `list_tools` 查看有哪些工具，再按需 `load_tool`，避免一次性加载太多。

### 如果不做会怎样？

**最简版本也能工作**，但你需要注意：
- 参数验证必须严格（LLM 可能传错参数）
- 错误信息要对 LLM 友好（不要暴露内部细节）
- 工具执行要有超时（防止卡死）

**建议**：先用 Pydantic 做参数验证，后续再加高级特性。

---

## 4. 组件 #3：Message Loop（工具调用循环）⭐ 核心中的核心

### 为什么必需？

这是 Agent 的"心脏"。没有它，Agent 只能单轮对话，不能连续调用工具。

Message Loop 负责：
- 接收 LLM 响应
- 如果有工具调用，执行它们
- 把工具结果追加到历史
- 再次调用 LLM
- 重复直到 LLM 返回纯文本（无工具调用）

### 最小实现（40 行代码）

```python
class SimpleAgentLoop:
    def __init__(self, llm: SimpleLLMClient, tools: list[ToolDefinition]):
        self.llm = llm
        self.tools = {t.name: t for t in tools}
        self.history = []
    
    async def run(self, user_input: str) -> str:
        """运行 Agent 循环"""
        # 添加用户消息
        self.history.append({"role": "user", "content": user_input})
        
        max_iterations = 10  # 防止无限循环
        
        for _ in range(max_iterations):
            # 调用 LLM
            response = await self.llm.chat(
                self.history, 
                tools=[t.to_openai_schema() for t in self.tools.values()]
            )
            
            # 添加到历史
            self.history.append(response)
            
            # 如果没有工具调用，返回文本
            if not response["tool_calls"]:
                return response["content"] or ""
            
            # 执行工具调用
            for tool_call in response["tool_calls"]:
                tool_name = tool_call.function.name
                tool_args = json.loads(tool_call.function.arguments)
                
                # 执行工具
                result = await self.tools[tool_name].execute(tool_args)
                
                # 添加工具结果到历史
                self.history.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": json.dumps(result),
                })
        
        return "Max iterations reached"
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **LangChain** | `AgentExecutor.run()` | ✅ 开箱即用 ❌ 黑盒，难以调试 |
| **OpenAI Agents SDK** | `Runner.run()` | ✅ 标准化 ❌ 不可定制 |
| **手写** | 如上所示 | ✅ 完全可控 ❌ 需要自己处理边界情况 |

### Chrys 的做法

**转录正确性引擎**：Chrys 的 Message Loop 不是简单的 while 循环，而是一个复杂的"转录正确性引擎"。

关键创新：

#### 1. Exchange Grammar（交换语法）

```python
# 问题：如何判断哪些工具调用已回答？

# 天真做法：按位置匹配
calls = [C1, C2, C3]
results = [R1, R2]
# 假设 R1.tool_use_id == C2.id，R2.tool_use_id == C3.id
# 那么 C1 就是"未回答"的

# Chrys 做法：用结构化语法规则
@dataclass(frozen=True)
class Exchange:
    response_indices: tuple[int, ...]   # assistant 消息索引
    output_indices: tuple[int, ...]     # tool result 消息索引

def iter_exchanges(messages):
    """遍历所有交换（call + result 对）"""
    # 识别"最大连续 assistant 兄弟 + tool output"
    ...
```

**为什么需要？**
- 某些 LLM 会回显历史内容（bug 或特性）
- 压缩后 message ID 可能变化
- 需要可靠的配对机制

#### 2. Invocation Order Stamps（调用顺序戳）

```python
# 问题：如何保证工具调用顺序不被破坏？

# LLM 返回
response = [
    Content(type="function_call", call_id="C1"),
    Content(type="function_call", call_id="C2"),
    Content(type="function_call", call_id="C1"),  # 重复！
]

# Chrys：在响应到达时立即打序数戳
ordinal = 0
for content in response.contents:
    if content.type == "function_call":
        content._chrys_tool_invocation_order = ordinal
        ordinal += 1  # 即使被过滤也消费序数
```

**为什么需要？**
- 后续模块（压缩、审计）需要可靠的顺序
- 分发过滤器丢弃的调用也要消费序数（否则编号会乱）

#### 3. Echo Removal（回声消除）

```python
# 问题：LLM 回显了历史内容怎么办？

# 发送给 LLM
messages = [
    {"role": "user", "content": "hi"},
    {"role": "assistant", "content": "hello"},  # 历史
    {"role": "user", "content": "how are you?"},
]

# LLM 返回（错误地包含了历史）
response = {
    "content": [
        {"type": "text", "text": "hello"},  # ← 回声！
        {"type": "text", "text": "I'm fine"},  # ← 新的
    ]
}

# Chrys：用对象身份追踪，移除回声
echo_memo = WeakIdentityRegistry()
for msg in history:
    for content in msg.contents:
        echo_memo.register(content)

for content in response.contents[:]:
    if content in echo_memo:  # 按对象身份检查
        response.contents.remove(content)  # 移除回声
```

**为什么需要？**
- 某些模型（特别是自部署的）会回显历史
- 不去除会导致 UI 显示重复、历史记录混乱

### 如果不做会怎样？

**最简版本能工作**，但你会遇到：
- LLM 回显历史 → UI 显示重复
- 工具调用配对错误 → 审计重放混乱
- 压缩后状态不一致 → 会话恢复失败

**建议**：
- 先用简单 while 循环跑通
- 如果遇到回声问题，加一个简单的去重（按 `call_id`）
- 如果需要复杂场景（压缩、审计），再考虑 Exchange Grammar

---

## 5. 组件 #4：Context Management（上下文窗口管理）

### 为什么必需？

LLM 有上下文窗口限制（如 GPT-4 是 128k tokens）。随着对话变长，你需要：
- 截断或压缩历史
- 保留关键信息
- 避免超出限制导致 API 报错

### 最小实现（20 行代码）

```python
class SimpleContextManager:
    def __init__(self, max_tokens: int = 100_000):
        self.max_tokens = max_tokens
    
    def truncate(self, messages: list[dict]) -> list[dict]:
        """简单截断：保留最近的 N 条消息"""
        total_tokens = sum(self._count_tokens(m) for m in messages)
        
        while total_tokens > self.max_tokens and len(messages) > 2:
            # 移除最早的非 system 消息
            messages.pop(1)  # 保留 system 和最新的
            total_tokens = sum(self._count_tokens(m) for m in messages)
        
        return messages
    
    def _count_tokens(self, message: dict) -> int:
        """粗略估计 token 数（实际应该用 tiktoken）"""
        return len(json.dumps(message)) // 4
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **LangChain** | `ConversationBufferWindowMemory` | ✅ 简单 ❌ 丢失重要信息 |
| **Claude Code** | 简单截断 + 摘要 | ✅ 平衡 ❌ 实现复杂 |
| **Chrys** | 四阶段压缩 | ✅ 精细 ❌ 非常复杂 |

### Chrys 的做法

**四阶段压缩**：从细粒度到粗粒度逐步压缩。

```python
class FourPhaseCompaction:
    def compact(self, messages: list[dict], budget: int) -> list[dict]:
        # Phase 1: 旧轮次工具结果摘要化
        if self._phase1_summarize_tool_results(messages, budget):
            return messages
        
        # Phase 2: 旧轮次工具调用组完全移除
        if self._phase2_remove_tool_groups(messages, budget):
            return messages
        
        # Phase 3: 文本密集的已完成轮次压缩
        if self._phase3_compress_text_turns(messages, budget):
            return messages
        
        # Phase 4: 当前轮次丢弃 + LLM 生成累积 LAST_WORDS
        if self._phase4_drop_current_turn_with_accumulating_note(messages, budget):
            return messages
        
        return messages
```

**Phase 4 的累积机制**：
- 第一次触发：丢弃当前轮次工具调用，生成 LAST_WORDS note
- 第二次触发：把上次的 note + 新丢弃的工作回馈给 LLM，生成更丰富的 note
- 这样 note 会越来越丰富，而不会从头重摘要

### 如果不做会怎样？

**最简版本（截断）能工作**，但你会遇到：
- 丢失重要上下文 → Agent 表现下降
- 突然超出限制 → API 报错，用户体验差

**建议**：
- 先用简单截断跑通
- 如果发现 Agent 在长对话中表现下降，加摘要功能
- 如果需要精细控制，再考虑多阶段压缩

---

## 6. 组件 #5：Session Persistence（会话持久化）

### 为什么必需？

用户可能：
- 关闭程序后想继续之前的对话
- 程序崩溃后不想丢失进度
- 在不同设备间同步会话

Session Persistence 负责：
- 保存会话状态（历史、元数据）
- 恢复会话
- 处理并发访问（多进程/线程）

### 最小实现（30 行代码）

```python
import json
from pathlib import Path

class SimpleSessionStore:
    def __init__(self, session_dir: Path):
        self.session_dir = session_dir
        self.session_dir.mkdir(exist_ok=True)
    
    def save(self, session_id: str, history: list[dict], metadata: dict):
        """保存会话"""
        path = self.session_dir / f"{session_id}.json"
        data = {
            "history": history,
            "metadata": metadata,
        }
        path.write_text(json.dumps(data, indent=2))
    
    def load(self, session_id: str) -> tuple[list[dict], dict]:
        """加载会话"""
        path = self.session_dir / f"{session_id}.json"
        if not path.exists():
            return [], {}
        
        data = json.loads(path.read_text())
        return data["history"], data["metadata"]
    
    def delete(self, session_id: str):
        """删除会话"""
        path = self.session_dir / f"{session_id}.json"
        if path.exists():
            path.unlink()
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **LangChain** | 内存存储或 SQLite | ✅ 简单 ❌ 不适合生产 |
| **Claude Code** | JSON 文件 | ✅ 轻量 ❌ 无并发控制 |
| **Chrys** | JSON + 锁文件 + 恢复边车 | ✅ 鲁棒 ❌ 复杂 |

### Chrys 的做法

**崩溃恢复边车**：除了主文件，还有一个 `.recovery.json`。

```python
class RobustSessionStore:
    def save(self, session_id: str, state: dict):
        main_path = self.sessions_dir / session_id / "session.json"
        recovery_path = self.sessions_dir / session_id / "session.recovery.json"
        lock_path = self.sessions_dir / ".locks" / f"{session_id}.active.lock"
        
        # 1. 获取锁
        with FileLock(lock_path):
            # 2. 写入主文件
            main_path.write_text(json.dumps(state))
            
            # 3. 删除恢复边车（表示保存成功）
            if recovery_path.exists():
                recovery_path.unlink()
    
    def save_checkpoint(self, session_id: str, state: dict):
        """中途保存（用于崩溃恢复）"""
        recovery_path = self.sessions_dir / session_id / "session.recovery.json"
        recovery_path.write_text(json.dumps(state))
    
    def load(self, session_id: str) -> dict:
        main_path = self.sessions_dir / session_id / "session.json"
        recovery_path = self.sessions_dir / session_id / "session.recovery.json"
        
        # 优先使用恢复边车（如果存在且比主文件新）
        if recovery_path.exists():
            if not main_path.exists() or recovery_path.stat().st_mtime > main_path.stat().st_mtime:
                return json.loads(recovery_path.read_text())
        
        return json.loads(main_path.read_text())
```

**好处**：即使程序在保存过程中崩溃，也能从 recovery checkpoint 恢复。

### 如果不做会怎样？

**最简版本能工作**，但你会遇到：
- 程序崩溃 → 会话丢失
- 多进程访问 → 数据损坏
- 大文件写入慢 → 用户体验差

**建议**：
- 先用 JSON 文件跑通
- 如果要支持崩溃恢复，加 checkpoint 机制
- 如果要支持多进程，加文件锁

---

## 7. 组件 #6：Error Handling & Retry（错误处理和重试）

### 为什么必需？

网络请求会失败，LLM API 会限流，工具执行会出错。没有错误处理，Agent 一遇到错误就崩溃。

### 最小实现（20 行代码）

```python
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

class SimpleRetryHandler:
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=2, max=10),
    )
    async def call_llm(self, messages: list[dict]) -> dict:
        """带重试的 LLM 调用"""
        try:
            return await self.llm.chat(messages)
        except RateLimitError:
            logger.warning("Rate limit hit, retrying...")
            raise  # tenacity 会自动重试
        except Exception as e:
            logger.error(f"LLM call failed: {e}")
            raise
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **LangChain** | 内置重试逻辑 | ✅ 开箱即用 ❌ 不够灵活 |
| **OpenAI SDK** | 自动重试（指数退避） | ✅ 简单 ❌ 无法定制 |
| **Chrys** | 分类错误 + 确定性重试守卫 | ✅ 精细 ❌ 复杂 |

### Chrys 的做法

**错误分类**：不是所有错误都应该重试。

```python
class ErrorClassification(Enum):
    RETRIABLE = "retriable"              # 可重试（如限流）
    RETRIABLE_WITH_BACKOFF = "backoff"   # 需要等待后重试
    TERMINAL = "terminal"                # 不可重试（如认证失败）

def classify_error(exc: Exception) -> ErrorClassification:
    """分类错误"""
    
    # 限流错误 → 可重试
    if isinstance(exc, RateLimitError):
        return ErrorClassification.RETRIABLE
    
    # DNS 失败 → 不可重试（重试也不会解决）
    if isinstance(exc, socket.gaierror):
        return ErrorClassification.TERMINAL
    
    # 连接超时 → 可重试
    if isinstance(exc, TimeoutError):
        return ErrorClassification.RETRIABLE
    
    return ErrorClassification.TERMINAL

class DeterministicRetryGuard:
    """确定性重试守卫"""
    
    async def call_with_retry(self, func, *args, **kwargs):
        for attempt in range(3):
            try:
                return await func(*args, **kwargs)
            except Exception as e:
                classification = classify_error(e)
                
                if classification == ErrorClassification.TERMINAL:
                    raise  # 不重试
                
                if attempt == 2:
                    raise  # 最后一次尝试
                
                # 等待后重试
                await asyncio.sleep(2 ** attempt)
```

### 如果不做会怎样？

**最简版本（直接抛异常）能工作**，但用户体验很差：
- 网络抖动 → Agent 崩溃
- API 限流 → 任务中断

**建议**：至少加重试逻辑，用 `tenacity` 库很方便。

---

## 8. 组件 #7：Safety & Approval（安全控制和审批）

### 为什么必需？

Agent 可以执行 shell 命令、修改文件。如果不加控制，可能会：
- 删除重要文件
- 执行危险命令（`rm -rf /`）
- 泄露敏感信息

### 最小实现（30 行代码）

```python
class SimpleApprovalSystem:
    def __init__(self):
        self.dangerous_patterns = [
            r"rm\s+-rf\s+/",
            r"mkfs\.",
            r"dd\s+if=",
        ]
    
    async def check_command(self, command: str) -> bool:
        """检查命令是否安全"""
        import re
        
        for pattern in self.dangerous_patterns:
            if re.search(pattern, command):
                return False  # 危险
        
        return True  # 安全
    
    async def ask_user(self, tool_name: str, args: dict) -> bool:
        """询问用户是否批准"""
        print(f"\n[APPROVAL] Tool: {tool_name}")
        print(f"  Args: {args}")
        response = input("  Approve? (y/n): ")
        return response.lower() == 'y'
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **Claude Code** | 简单的 allow/deny 列表 | ✅ 易用 ❌ 不够灵活 |
| **Cursor** | 沙箱隔离 | ✅ 安全 ❌ 限制功能 |
| **Chrys** | 三模式审批（MANUAL/AUTO/BYPASS） | ✅ 灵活 ❌ 复杂 |

### Chrys 的做法

**三模式审批**：

```python
class ApprovalMode(Enum):
    MANUAL = "manual"   # 每次都要问用户
    AUTO = "auto"       # LLM 判断是否需要问
    BYPASS = "bypass"   # 不问，直接执行（headless 模式）

class ApprovalPolicy:
    def __init__(self, mode: ApprovalMode):
        self.mode = mode
        self.rules = {
            "shell": "require",           # shell 命令需要审批
            "filesystem.write": "require", # 写文件需要审批
            "read_file": "skip",          # 读文件不需要
        }
    
    async def should_approve(self, tool_name: str, args: dict) -> bool:
        """决定是否批准"""
        
        # BYPASS 模式：直接批准
        if self.mode == ApprovalMode.BYPASS:
            return True
        
        # 检查规则
        rule = self.rules.get(tool_name, "auto")
        
        if rule == "skip":
            return True
        elif rule == "require":
            # MANUAL 模式：问用户
            if self.mode == ApprovalMode.MANUAL:
                return await self.ask_user(tool_name, args)
            # AUTO 模式：让 LLM 判断
            elif self.mode == ApprovalMode.AUTO:
                return await self.llm_judge(tool_name, args)
        
        return False
```

### 如果不做会怎样？

**最简版本（全信任）能工作**，但非常危险：
- Agent 可能执行 `rm -rf /`
- 用户数据可能被误删

**建议**：至少加一个简单的黑名单检查，或者默认要求用户确认。

---

## 9. 组件 #8：Platform Abstraction（跨平台支持，可选）

### 为什么可能需要？

如果你的 Agent 要在 Windows/macOS/Linux 上运行，你需要处理：
- 路径差异（反斜杠 vs 正斜杠）
- 进程管理差异（Windows IOCP vs POSIX epoll）
- 终端差异（ConPTY vs PTY）

### 最小实现（10 行代码）

```python
import sys
from pathlib import Path

def get_platform() -> str:
    """获取当前平台"""
    if sys.platform == "win32":
        return "windows"
    elif sys.platform == "darwin":
        return "macos"
    else:
        return "linux"

def normalize_path(path: str) -> Path:
    """统一路径格式"""
    return Path(path).resolve()
```

### 业界怎么做？

| 框架 | 做法 | 优缺点 |
|------|------|--------|
| **LangChain** | 不处理（假设 POSIX） | ✅ 简单 ❌ Windows 不工作 |
| **Claude Code** | 部分支持 | ✅ macOS/Linux 好用 ❌ Windows 体验差 |
| **Chrys** | 完整抽象 | ✅ 三平台一致 ❌ 维护成本高 |

### Chrys 的做法

**统一的 PlatformInfo**：

```python
@dataclass(frozen=True)
class PlatformInfo:
    os_name: str  # "windows", "macos", "linux"
    config_dir: Path  # ~/.chrys or %APPDATA%/chrys
    
    @property
    def is_windows(self) -> bool:
        return self.os_name == "windows"

@functools.cache
def get_platform() -> PlatformInfo:
    """Cached platform detection singleton"""
    ...
```

### 如果不做会怎样？

**如果你只在一个平台用**：完全没问题。

**如果你要跨平台**：
- Windows 用户会遇到路径问题
- 进程管理可能不工作

**建议**：先用 `pathlib` 处理路径，其他按需加。

---

## 10. 总结：你应该从哪里开始？

### 最小可行版本（MVP）

如果你想快速跑通一个 Demo，只需要：

```python
class MinimalCodingAgent:
    def __init__(self):
        self.llm = SimpleLLMClient(api_key="...")
        self.tools = [read_file_tool, write_file_tool, shell_tool]
        self.history = []
    
    async def run(self, user_input: str) -> str:
        # 简单的工具调用循环
        self.history.append({"role": "user", "content": user_input})
        
        for _ in range(10):  # 最多 10 次迭代
            response = await self.llm.chat(self.history, tools=...)
            self.history.append(response)
            
            if not response["tool_calls"]:
                return response["content"]
            
            for tool_call in response["tool_calls"]:
                result = await self.execute_tool(tool_call)
                self.history.append({"role": "tool", "content": result})
        
        return "Max iterations reached"
```

**这大概 200 行代码就能搞定**，足够你理解核心概念。

### 生产级版本

如果你想做一个可以实际使用的产品，需要加：

1. **错误处理和重试**（1-2 天）
2. **基本的上下文管理**（截断或摘要）（1 天）
3. **会话持久化**（JSON 文件）（半天）
4. **基本的安全检查**（黑名单）（半天）

**总共大约 1-2 周**。

### 企业级版本（像 Chrys 这样）

如果你要做到 Chrys 这种程度：
- 转录正确性引擎（Exchange Grammar、Echo Removal）
- 四阶段压缩
- 崩溃恢复
- 三模式审批
- 跨平台支持

**这需要几个月甚至一年的工程投入**。

---

## 11. 最后的建议

1. **先跑通 MVP**：不要一开始就追求完美，先用最简单的实现跑通整个流程
2. **遇到问题再加功能**：比如发现 Agent 在长对话中表现下降，再加压缩；发现程序崩溃会丢数据，再加持久化
3. **参考业界但不照搬**：LangChain 的抽象可能太厚，OpenAI SDK 可能太绑定，找到适合自己的平衡点
4. **重视测试**：每加一个功能，都写测试。Agent 系统的边界情况很多，没有测试很难保证质量

祝你好运！🚀
