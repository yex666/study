# Chrys Kernel 层深度解析：为什么全部自研？

> **TL;DR**: Chrys 没有用 LangChain、OpenAI Agents SDK 或其他框架，而是从零实现了一个"转录正确性引擎"。这不是重复造轮子，而是因为现有方案在**对象身份追踪、工具调用配对、回声消除、存储模式解析**等关键问题上存在根本缺陷。

---

## 目录

1. [先搞清楚：Kernel 到底是什么](#1-先搞清楚kernel-到底是什么)
2. [为什么不直接用 LangChain / OpenAI SDK](#2-为什么不直接用-langchain--openai-sdk)
3. [第一个问题：如何知道哪些工具调用已回答](#3-第一个问题如何知道哪些工具调用已回答)
4. [第二个问题：如何追踪对象身份](#4-第二个问题如何追踪对象身份)
5. [第三个问题：如何处理 LLM 回显的历史内容](#5-第三个问题如何处理-llm-回显的历史内容)
6. [第四个问题：如何保证调用顺序不被破坏](#6-第四个问题如何保证调用顺序不被破坏)
7. [第五个问题：如何处理中断恢复](#7-第五个问题如何处理中断恢复)
8. [总结：自研的代价与收益](#8-总结自研的代价与收益)

---

## 1. 先搞清楚：Kernel 到底是什么

### 打个比方：餐厅的点餐系统

想象你开了一家餐厅，顾客（用户）点菜（发送消息），服务员（Agent）需要：

1. **听懂顾客说什么**（解析用户输入）
2. **把订单传给厨房**（调用 LLM）
3. **如果厨房需要做菜**（工具调用），就通知对应厨师
4. **等菜做好后端给顾客**（返回结果）
5. **记住这桌点了什么**（保存到历史）

**Kernel 就是这个"服务员的工作流程"**——它定义了：
- 如何与厨房（LLM）沟通
- 如何管理多个订单（并发工具调用）
- 如何处理意外情况（工具失败、网络中断）
- 如何记住已完成的工作（持久化状态）

### Kernel 的位置

```
┌─────────────────────────────────────┐
│  App 层 (TUI/CLI/ACP)              │  ← 用户界面
├─────────────────────────────────────┤
│  Orchestration 层                   │  ← 编排多个 Agent
├─────────────────────────────────────┤
│  Service 层                         │  ← 连接外部服务（LLM/MCP）
├─────────────────────────────────────┤
│  Kernel 层 ⭐                       │  ← 单个 Agent 的核心循环
├─────────────────────────────────────┤
│  Foundation 层                      │  ← 基础设施（事件/配置/平台）
└─────────────────────────────────────┘
```

**Kernel 的职责**：
- 接收用户消息
- 构建 LLM 请求
- 执行工具调用循环
- 组装最终响应
- 管理中间状态

---

## 2. 为什么不直接用 LangChain / OpenAI SDK

### 业界主流方案

#### 方案 A：LangChain / LlamaIndex

```python
from langchain.agents import initialize_agent, Tool
from langchain.llms import OpenAI

# 定义工具
tools = [
    Tool(name="read_file", func=read_file_fn, description="Read a file"),
    Tool(name="write_file", func=write_file_fn, description="Write a file"),
]

# 初始化 Agent
agent = initialize_agent(
    tools,
    OpenAI(temperature=0),
    agent="zero-shot-react-description",
)

# 运行
result = agent.run("Read main.py and summarize it")
```

**看起来很简单，但有几个致命问题**：

##### 问题 1：不知道哪些工具调用已回答

假设对话历史是这样的：

```
User: "读一下 main.py"
Assistant: [call: read_file("main.py")]  ← 调用 C1
Tool: "def main(): ..."                   ← 结果 R1
Assistant: "文件内容是..."                 ← 文本 T1
```

**LangChain 怎么判断 R1 是回答 C1 的？**

```python
# LangChain 内部可能的做法
for msg in messages:
    if msg.type == "ai":
        calls = [c for c in msg.content if c.type == "tool_use"]
    elif msg.type == "tool":
        results = [r for r in msg.content if r.type == "tool_result"]
        
    # 按位置匹配：第 N 个 call 对应第 N 个 result
    for call, result in zip(calls, results):
        assert call.id == result.tool_use_id
```

**问题场景**：如果压缩（compaction）后消息 ID 变了怎么办？

```python
# 压缩前
messages = [
    {"id": "msg_0", "role": "user", "content": "..."},
    {"id": "msg_1", "role": "assistant", "content": [{"type": "tool_use", "id": "C1"}]},
    {"id": "msg_2", "role": "tool", "content": [{"type": "tool_result", "tool_use_id": "C1"}]},
]

# 压缩后（ID 重新编号）
messages = [
    {"id": "msg_0", "role": "user", "content": "..."},  # ID 没变
    {"id": "msg_1", "role": "assistant", "content": [{"type": "tool_use", "id": "C1"}]},  # ID 没变
    # 但某些框架可能会重新生成 ID！
]
```

**Chrys 的做法**：不依赖 ID，而是用**交换语法（Exchange Grammar）**。

##### 问题 2：无法处理 LLM 回显的历史

某些 LLM 提供商（特别是自部署的模型）会在响应中包含历史内容：

```python
# 发送给 LLM
messages = [
    {"role": "user", "content": "hi"},
    {"role": "assistant", "content": "hello"},  # 历史
    {"role": "user", "content": "how are you?"},  # 新输入
]

# LLM 返回（错误地包含了历史）
response = {
    "role": "assistant",
    "content": [
        {"type": "text", "text": "hello"},  # ← 这是历史的回声！
        {"type": "text", "text": "I'm fine, thanks!"},  # ← 这才是新的
    ]
}
```

**LangChain 的处理**：直接接受所有 content，导致 UI 显示重复内容。

**Chrys 的处理**：通过对象身份追踪，识别并移除回声。

##### 问题 3：工具调用顺序可能错乱

```python
# LLM 返回
response = {
    "content": [
        {"type": "tool_use", "id": "C1", "name": "read_file"},
        {"type": "tool_use", "id": "C2", "name": "grep"},
        {"type": "tool_use", "id": "C1", "name": "read_file"},  # ← 重复！
    ]
}
```

**LangChain**：按出现顺序编号 0, 1, 2，但重复的 C1 怎么处理？

**Chrys**：在响应到达时立即打上顺序戳，包括被过滤掉的调用。

#### 方案 B：OpenAI Agents SDK

```python
from agents import Agent, Runner

agent = Agent(
    name="Code Assistant",
    instructions="You are a helpful coding assistant.",
    tools=[read_file_tool, write_file_tool],
)

result = Runner.run(agent, "Read main.py")
```

**问题**：
- 绑定到 OpenAI 生态，无法切换到 Anthropic/DeepSeek
- 工具循环是黑盒，无法插入自定义逻辑
- 没有"交换语法"概念

#### 方案 C：Claude Code（Anthropic）

闭源，无法审查内部实现。但从外部行为推测：
- 可能用了简单的回调机制
- 不一定有严格的背压保证
- 绑定 Anthropic 模型

### Chrys 的选择：自研一切

**核心理念**：LLM 提供者的 SDK 是 wire 传输层，不是 Agent 运行时。Agent 需要拥有自己的：
- 身份系统（谁是谁）
- 排序系统（先后顺序）
- 配对系统（哪个结果回答哪个调用）
- 回声消除系统（移除重复内容）
- 压缩系统（上下文窗口管理）

这些是"转录正确性"的核心，不能委托给外部。

---

## 3. 第一个问题：如何知道哪些工具调用已回答

### 场景重现

```
User: "帮我重构这个函数"
Assistant: 
  [call C1: read_file("main.py")]
  [call C2: grep("def refactor_me")]
Tool Result R1: "def refactor_me(): ..."
Assistant: "我看到了这个函数..."
  [call C3: write_file("main.py", new_content)]
Tool Result R2: "写入成功"  ← 等等，R2 是回答 C3 的，但 C1 和 C2 呢？
```

**问题**：
- R1 回答了 C2（不是 C1）
- C1 没有被回答（可能是 LLM 忘了）
- R2 回答了 C3

**LangChain 怎么做？**

```python
# 简单粗暴：按位置匹配
calls = [C1, C2, C3]
results = [R1, R2]

# 假设 R1.tool_use_id == C2.id
# 假设 R2.tool_use_id == C3.id

# 那么 C1 就是"未回答"的
unanswered = [c for c in calls if c.id not in [r.tool_use_id for r in results]]
# unanswered = [C1]
```

**问题**：
- 如果 `tool_use_id` 丢失了怎么办？
- 如果有多个相同 ID 的调用怎么办？
- 如果压缩后 ID 变了怎么办？

### Chrys 的做法：交换语法（Exchange Grammar）

**核心思想**：不依赖 ID，而是用**结构化的语法规则**来判断。

#### 语法定义（简化版）

```python
"""
一个 EXCHANGE（交换）是：
1. 一组连续的 assistant 消息（包含至少一个工具调用）
2. 后接回答它们的 tool result 消息

规则：
- 携带 history marker 的消息是硬边界（不属于任何交换）
- result-only 的 assistant 消息属于输出块，不属于响应运行
- 工具输出后的 call-carrying assistant 开启新交换
- 配对是策略驱动的（不同消费者可以声明不同规则）
"""
```

#### 源码实现

```python
# kernel/exchanges.py

@dataclass(frozen=True)
class Exchange:
    """一个交换包含响应索引和输出索引"""
    response_indices: tuple[int, ...]   # assistant 消息索引
    output_indices: tuple[int, ...]     # tool result 消息索引

def iter_exchanges(messages: Sequence[Message]) -> Iterator[Exchange]:
    """遍历所有交换"""
    i = 0
    while i < len(messages):
        msg = messages[i]
        
        # 跳过硬边界（history marker）
        if _has_history_marker(msg):
            i += 1
            continue
        
        # 如果是 assistant 消息且包含工具调用
        if msg.role == "assistant" and _has_tool_calls(msg):
            # 收集连续的 assistant 兄弟消息
            response_start = i
            while i < len(messages) and _is_response_sibling(messages[i]):
                i += 1
            
            # 收集后续的 tool result
            output_start = i
            while i < len(messages) and _is_output_sibling(messages[i]):
                i += 1
            
            yield Exchange(
                response_indices=tuple(range(response_start, output_start)),
                output_indices=tuple(range(output_start, i))
            )
        else:
            i += 1
```

#### 配对策略

```python
@dataclass(frozen=True)
class PairingPolicy:
    """配对策略：决定如何处理各种边界情况"""
    none_id_policy: Literal["ignore", "unpairable", "positional"]
    empty_id_policy: Literal["ignore", "coerced_single_key", "unpairable"]
    # ... 更多策略

def pair_results(exchange: Exchange, policy: PairingPolicy) -> ExchangePairing:
    """根据策略配对调用和结果"""
    pairing = ExchangePairing()
    
    # 注册所有调用
    for idx in exchange.response_indices:
        msg = messages[idx]
        for content in msg.contents:
            if content.type in TOOL_CALL_CONTENT_TYPES:
                key = _make_pairing_key(content)  # ("call_id", "C1") 或 ("image_id", "img_123")
                pairing.register_call(key, Occurrence(idx, content))
    
    # 消费所有结果
    for idx in exchange.output_indices:
        msg = messages[idx]
        for content in msg.contents:
            if content.type in TOOL_RESULT_CONTENT_TYPES:
                key = _make_pairing_key(content)
                pairing.consume_result(key, Occurrence(idx, content))
    
    return pairing
```

**关键设计**：
- **命名空间隔离**：`PairingKey = tuple[Literal["call_id", "image_id"], str]`
  - 图像生成的 `image_id` 不会和 `call_id` 误配
- **策略驱动**：不同消费者（压缩、回声消除、完整性检查）可以声明不同规则
- **answered_keys vs result_keys**：前者仅含已消费的，后者含孤儿

### 生活类比：餐厅的点餐单

想象餐厅用点餐单管理订单：

```
桌号 3: 
  - 宫保鸡丁 (C1) → [等待]
  - 酸梅汤 (C2) → [已上菜 R1]
  
桌号 5:
  - 鱼香肉丝 (C3) → [等待]
```

**LangChain 的做法**：
- 只看菜名，不看桌号
- 如果两个桌都点了"宫保鸡丁"，就搞混了

**Chrys 的做法**：
- 看桌号 + 菜名（命名空间隔离）
- 每道菜有唯一标识（call_id 或 image_id）
- 用结构化规则判断哪道菜对应哪个结果

---

## 4. 第二个问题：如何追踪对象身份

### 问题背景

Python 的 `id()` 函数返回对象的内存地址：

```python
obj = Content(type="text", text="hello")
print(id(obj))  # 例如：140234567890
```

**问题**：
- GC 回收对象后，新对象可能占用同一地址
- `id()` 不保持对象存活

```python
obj1 = Content(type="text", text="hello")
id1 = id(obj1)  # 140234567890

del obj1  # GC 回收

obj2 = Content(type="text", text="world")
id2 = id(obj2)  # 可能又是 140234567890！

assert id1 == id2  # True，但它们是不同的对象！
```

### 典型做法及其问题

```python
# 用 set 记录见过的 id
seen_ids = set()

for content in message.contents:
    if id(content) in seen_ids:
        print("重复！")
    seen_ids.add(id(content))
```

**问题**：
- 如果 `content` 被 GC 回收，新对象占用同一地址，会误判为"重复"
- 无法区分"同一对象"和"地址相同的不同对象"

### Chrys 的做法：WeakIdentityRegistry

**核心思想**：用弱引用追踪对象，对象死亡时自动移除。

#### 源码实现

```python
# kernel/identity.py

import weakref

class WeakIdentityRegistry:
    """对象身份成员资格，注意到成员死亡"""
    
    __slots__ = ("_ignore_usage", "_refs")
    
    def __init__(self, *, ignore_usage: bool = False) -> None:
        self._refs: dict[int, weakref.ref[Any]] = {}
        self._ignore_usage = ignore_usage
    
    def register(self, obj: Any) -> None:
        """添加 obj 为成员，只要它还活着"""
        if self._ignore_usage and getattr(obj, "type", None) == "usage":
            return
        
        key = id(obj)
        refs = self._refs
        
        # 检查是否已注册且仍是同一对象
        existing = refs.get(key)
        if existing is not None and existing() is obj:
            return  # 已经注册过了
        
        # 创建弱引用，带清理回调
        def _evict(dead: weakref.ref[Any], *, key: int = key, refs: dict = refs) -> None:
            # 仅在未重注册时移除（防止地址重用导致的误删）
            if refs.get(key) is dead:
                del refs[key]
        
        refs[key] = weakref.ref(obj, _evict)
    
    def __contains__(self, obj: object) -> bool:
        """检查 obj 是否是成员"""
        ref = self._refs.get(id(obj))
        # 必须同时满足：
        # 1. ref 不为 None（曾经注册过）
        # 2. ref() is obj（仍然是同一对象，不是复制品）
        return ref is not None and ref() is obj
    
    def __len__(self) -> int:
        return len(self._refs)
```

**关键点**：
- `ref() is obj`：确保是**同一对象**，不是复制品
- `_evict` 回调：对象死亡时自动清理
- 地址重用安全：`if refs.get(key) is dead` 防止误删新注册的对象

#### 为什么需要 ContentList？

```python
# kernel/identity.py

class ContentList(list["Content"]):
    """可弱引用的 list"""
    
    __slots__ = ("__weakref__",)  # 添加弱引用支持，但不增加 __dict__
```

**问题**：内置 `list` 不支持弱引用。

```python
import weakref

lst = [1, 2, 3]
try:
    ref = weakref.ref(lst)  # TypeError: cannot create weak reference to 'list' object
except TypeError as e:
    print(e)
```

**但在 Chrys 中**：

```python
# kernel/_types.py

class Message:
    @property
    def contents(self) -> ContentList:
        return self.__dict__["contents"]
    
    @contents.setter
    def contents(self, value: Sequence[Content]) -> None:
        # 强制转换为 ContentList
        self.__dict__["contents"] = ContentList(value)
```

**为什么这样做？**

因为压缩锚点和摘要缓存需要捕获 `id(msg.contents)`：

```python
# 压缩模块
anchor_id = id(message.contents)  # 列表的身份

# 稍后检查是否还是同一个列表
if id(message.contents) == anchor_id:
    # 列表没被替换过
    pass
else:
    # 列表被替换了（可能是压缩导致的）
    pass
```

如果用普通 `list`，就无法用 `weakref.ref()` 追踪它的生死。

### 生活类比：快递柜 vs 储物柜

**普通 `set` 像储物柜**：
- 你租了一个柜子（`id(obj)`）
- 退租后，别人可以租同一个柜子
- 你无法区分"我的东西"和"别人的东西"

**WeakIdentityRegistry 像智能快递柜**：
- 每个包裹有唯一 RFID 标签（弱引用）
- 包裹被取走后，系统自动标记为"空"
- 即使新包裹放在同一格，系统也能区分

---

## 5. 第三个问题：如何处理 LLM 回显的历史内容

### 问题场景

```python
# 第 1 轮对话
messages = [
    Message(role="user", contents=[Content(type="text", text="hi")]),
    Message(role="assistant", contents=[Content(type="text", text="hello")]),
]

# 第 2 轮对话，发送给 LLM
provider_messages = [
    {"role": "user", "content": "hi"},
    {"role": "assistant", "content": "hello"},  # 历史
    {"role": "user", "content": "how are you?"},  # 新输入
]

# LLM 返回（某些模型会错误地包含历史）
response = ChatResponse(
    role="assistant",
    contents=[
        Content(type="text", text="hello"),  # ← 这是历史的回声！
        Content(type="text", text="I'm fine, thanks!"),  # ← 这才是新的
    ]
)
```

**如果不处理**：
- UI 会显示两次 "hello"
- 历史记录会包含重复内容
- 后续压缩会浪费 token

### Chrys 的做法：基于对象身份的回声消除

**核心思想**：在流式更新的最内层，按对象身份丢弃对话已持有的 Content。

#### 源码实现（简化版）

```python
# kernel/loop.py

def _strip_echoed_update(
    update: ChatResponseUpdate,
    echo_memo: WeakIdentityRegistry
) -> None:
    """从流式更新中剥离回声内容"""
    
    for content in update.contents[:]:  # 浅拷贝，避免迭代时修改
        # 检查这个 content 是否已经在对话历史中
        if content in echo_memo:
            # 这是回声，移除它
            update.contents.remove(content)
            
            # 标记此消息被剥离（用于溯源）
            update._chrys_echo_content_stripped = True
    
    # 豁免 usage content（永远不移除）
    # （实际代码中有特殊处理）
```

#### 为什么要在"最内层"做？

```python
# 流式更新的组装过程

# 1. LLM 返回片段
chunk1 = Content(type="text", text="hel")
chunk2 = Content(type="text", text="lo")

# 2. 组装器合并连续同类型片段
merged = Content(type="text", text="hello")  # ← 新对象！

# 3. 如果在组装后才过滤，洗白的副本已经存在
```

**问题**：如果历史中的 `Content(text="hello")` 参与了合并，合并后的新对象身份就不同了。

**解决**：在组装前过滤，确保洗白的副本永不存在。

#### 完整的回声消除流程

```python
# kernel/loop.py（简化版）

async def _process_streaming_response(self, stream: AsyncIterable[ChatResponseUpdate]):
    # 1. 初始化回声记忆（包含当前对话的所有 content）
    echo_memo = WeakIdentityRegistry(ignore_usage=True)
    for msg in self.history:
        for content in msg.contents:
            echo_memo.register(content)
    
    # 2. 处理每个流式更新
    async for update in stream:
        # 在最内层过滤回声
        _strip_echoed_update(update, echo_memo)
        
        # 如果整个更新都是回声，跳过
        if not update.contents:
            continue
        
        # 累积到最终响应
        self._accumulate_update(update)
        
        # 注册新的 content（防止后续回声）
        for content in update.contents:
            echo_memo.register(content)
```

### 生活类比：复印机的鬼影

**问题**：复印件上有上一页的淡淡痕迹（鬼影）。

**解决方案**：
- 在扫描时识别"旧墨迹"
- 只保留"新墨迹"

**Chrys 的做法**：
- 用弱引用记住"旧墨迹"（历史 content）
- 扫描时检测"新墨迹"是否与"旧墨迹"相同
- 如果相同，认为是鬼影，移除

---

## 6. 第四个问题：如何保证调用顺序不被破坏

### 问题场景

```python
# LLM 返回
response = ChatResponse(
    contents=[
        Content(type="function_call", call_id="C1", name="read_file"),
        Content(type="function_call", call_id="C2", name="grep"),
        Content(type="function_call", call_id="C1", name="read_file"),  # ← 重复！
    ]
)
```

**问题**：
- 第三个 content 的 `call_id` 是 "C1"，和第一个重复
- 如果按出现顺序编号 0, 1, 2，那第二个 C1 应该是 2
- 但如果分发过滤器丢弃了它，编号就乱了

### Chrys 的做法：在响应到达时打戳

**核心思想**：Kernel 循环是 `_chrys_tool_invocation_order` 的**唯一生产者**。

#### 源码实现（简化版）

```python
# kernel/loop.py

def _land_response_contents(
    response: ChatResponse,
    loop_recorder: LoopRecorder
) -> None:
    """在响应落地时打上调用顺序戳"""
    
    ordinal = 0
    for content in response.contents:
        # 只对非信息性的 function_call 打戳
        if content.type == "function_call" and not _is_informational(content):
            # 无论分发过滤器是否丢弃，都消费序数
            content.additional_properties[TOOL_INVOCATION_ORDER_KEY] = ordinal
            ordinal += 1
        
        # 其他类型的 content 不打戳
```

**关键点**：
- **在响应到达时**打戳（模型发射顺序）
- 包括分发过滤器丢弃的调用
- 持久化编号永不偏离管道编号

#### 为什么这么重要？

假设后续模块需要按顺序处理工具调用：

```python
# 压缩模块
for content in message.contents:
    order = read_tool_invocation_order(content)
    # 如果编号乱了，压缩就会出错
```

**如果编号不可靠**：
- 压缩可能丢掉错误的 content
- 审计重放会混乱
- 轨迹分析不准确

### 生活类比：流水线的工位编号

**问题**：流水线上有多个工位，每个工位处理一道工序。

**错误做法**：
- 按"完成的工件"编号
- 如果有工件被丢弃，编号就断了

**Chrys 的做法**：
- 按"进入流水线的工件"编号
- 即使工件被丢弃，编号也继续
- 保证编号连续性

---

## 7. 第五个问题：如何处理中断恢复

### 问题场景

```python
# 工具执行过程中，用户按下 Ctrl+C
try:
    result = await execute_tool("delete_file", {"path": "important.txt"})
except KeyboardInterrupt:
    # 用户取消了
    # 但工具可能已经执行了一半
    raise
```

**问题**：
- 工具执行了一半，状态不一致
- 下次重试时，不知道从哪里开始

### Chrys 的做法：LoopRecorder

**核心思想**：在每次 wire 调用前记录快照，在响应后密封。

#### 源码实现（简化版）

```python
# kernel/loop.py

@dataclass
class _PendingExchange:
    """待处理的交换"""
    call: FunctionCall
    result: FunctionResult | None = None
    sealed: bool = False
    
    def seal(self, result: FunctionResult) -> None:
        self.result = result
        self.sealed = True

class LoopRecorder:
    """中断恢复的状态机"""
    
    def __init__(self):
        self._pending: dict[str, _PendingExchange] = {}
    
    def record_pre_call(self, call: FunctionCall, ordinal: int) -> None:
        """在 wire 调用前记录"""
        self._pending[call.call_id] = _PendingExchange(call=call)
    
    def record_response(self, response: ChatResponse) -> None:
        """在响应后密封"""
        for content in response.contents:
            if content.type == "function_result":
                pending = self._pending.get(content.call_id)
                if pending:
                    pending.seal(content)
    
    def snapshot(self) -> LoopRecorderSnapshot:
        """创建快照（用于 whole-run 重试）"""
        return LoopRecorderSnapshot(
            pending={k: deepcopy(v) for k, v in self._pending.items()}
        )
    
    def restore(self, snapshot: LoopRecorderSnapshot) -> None:
        """从快照恢复"""
        self._pending = {k: deepcopy(v) for k, v in snapshot.pending.items()}
```

**关键设计**：
- 每个已回答槽是不可撤销的提交
- Whole-run 重试永不重放已提交的调用
- Hosted 调用绕过循环记录器

### 生活类比：游戏的存档点

**问题**：打 Boss 时断电了，怎么办？

**解决方案**：
- 在每个关键节点自动存档
- 断电后从最近的存档点继续

**Chrys 的做法**：
- 每次 wire 调用前存档（`record_pre_call`）
- 每次响应后密封（`record_response`）
- 中断后从存档恢复（`restore`）

---

## 8. 总结：自研的代价与收益

### 代价

| 方面 | 具体问题 |
|------|----------|
| **复杂度** | 四层 DAG、三条构建路径、交换语法等增加理解门槛 |
| **维护成本** | 需要持续跟进 Textual、MCP SDK、提供者 SDK 的变化 |
| **上手门槛** | 需要深入理解 Agent 运行时的概念（转录、身份、配对等） |
| **测试负担** | 每个不变量都需要守卫测试（`tests/architecture/test_hygiene_*.py`） |

### 收益

| 方面 | 具体优势 |
|------|----------|
| **转录正确性** | 拥有身份、排序、配对、回声消除、存储模式解析等不变量 |
| **跨提供者一致性** | api_style 正交轴，新变体是 api_style 而非 provider id |
| **可扩展性** | Agent = Profile，Skills/Hooks/MCP/子 Agent 全可组合 |
| **企业级持久化** | 崩溃恢复边车、事务性恢复、会话锁、fork、回滚 |
| **隐私优先** | 无遥测、无分析、无崩溃上报 |

### 适用场景

**Chrys 最适合**：
- 需要**多模型支持**的企业（不想被单一提供商锁定）
- 需要**可扩展 Agent 配置**的平台（用户自定义 Agent）
- 需要**工作流编排**的复杂任务（多 Agent 协作）
- 需要**企业级持久化**的生产环境（崩溃恢复、会话管理）
- 需要**跨平台支持**的团队（Windows/macOS/Linux）

**不适合**：
- 仅需"快速与一个提供者对话"的简单用例
- 不需要严格控制 Agent 行为的实验性项目
- 团队规模小，维护能力有限

### 一句话总结

Chrys 的 Kernel 不是一个 LLM client 包装器——它是一个**转录正确性引擎**，分层在模型/工具循环之上。它拥有身份、排序、配对、回声消除、存储模式解析和压缩作为一等不变量，每个都由守卫测试支持。

这类似于数据库领域：你不会用 ORM 替代 SQL 引擎，因为 ORM 不知道事务隔离级别；同样，你不会用 LangChain 替代 Agent kernel，因为 LangChain 不知道对象身份在 GC 后是否还有效。

---

## 附录：关键源码文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `kernel/agent.py` | ~500 | Agent 入口，惰性求值 |
| `kernel/loop.py` | ~3700 | 工具循环，回声消除，续接令牌 |
| `kernel/client.py` | ~1500 | Wire client 抽象，存储模式解析 |
| `kernel/exchanges.py` | ~800 | 交换语法，策略驱动配对 |
| `kernel/middleware.py` | ~600 | Chat/Function 中间件管道 |
| `kernel/compaction.py` | ~1200 | 分组注解，LastWordsCompleter |
| `kernel/identity.py` | 96 | WeakIdentityRegistry |
| `kernel/_types.py` | ~2500 | Message/Content 定义，ContentList 强制 |
| `kernel/_content.py` | ~1800 | Content dataclass 及变体 |
| `kernel/_null_overlay.py` | ~1200 | Pydantic null 恢复策略 |
| `kernel/_tool_arg_errors.py` | ~2000 | 工具参数错误诊断 |

---

**问**：这篇文档清楚了吗？有没有哪里还不明白的？我可以针对某个具体点再深入讲解。
