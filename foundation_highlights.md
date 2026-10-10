# Foundation 层：重点与创新点

> **TL;DR**: Foundation 层有 149 个文件，但真正值得你花时间的只有 **5 个核心创新**。本文只讲这些精华，不写流水账。

---

## 0. 一句话总结 Foundation 层

**Foundation = 整个系统的"地基"**：提供事件通信、配置管理、平台抽象、数据模型等基础能力。所有上层模块都依赖它，但它不依赖任何业务逻辑。

---

## 1. 五大核心创新

### 🏆 创新点 #1：Turn Boundary 判定 —— 不能用 `role == "user"` 判断轮次

#### 问题场景

假设你要实现"回滚到上一轮"功能，首先需要知道"哪些消息属于同一轮对话"。

**天真的做法**：

```python
def find_turns(messages):
    turns = []
    current_turn = []
    
    for msg in messages:
        if msg.role == "user":
            # 遇到 user 消息就开启新轮次
            if current_turn:
                turns.append(current_turn)
            current_turn = [msg]
        else:
            current_turn.append(msg)
    
    if current_turn:
        turns.append(current_turn)
    
    return turns
```

**这会出什么问题？**

考虑这个场景：

```
User: "帮我重构这个函数"           ← 真正的轮次开始
Assistant: [调用 read_file]
Tool Result: "函数代码..."
Assistant: "我看到了..."
User: "等等，先别改"               ← 这是中途注入的消息！
Assistant: "好的，我暂停了"
User: "继续"                       ← 这是系统自动发的 continue nudge！
Assistant: [继续执行工具]
Tool Result: "修改完成"
```

上面的天真算法会把这 3 条 user 消息都当成"轮次开始"，但实际上：
- 第 1 条是真正的轮次开始（opener）
- 第 2 条是中途注入（injected）——用户临时打断
- 第 3 条是系统续接（continuation）——不是用户发的

#### Chrys 的做法

```python
# foundation/models/turns.py

UserMessageKind = Literal["opener", "injected", "continuation"]

def opens_turn(msg: SupportsTurnMessage) -> bool:
    """只有 unflagged 的 user 消息才开启轮次"""
    return msg.role == "user" and not is_mid_turn_user_message(msg)

def is_mid_turn_user_message(msg: SupportsTurnMessage) -> bool:
    """检查是否是中途注入或续接消息"""
    return msg.role == "user" and any(
        msg.additional_properties.get(key, False) 
        for key in HistoryMarkerKind.MID_TURN_USER_KEYS
    )

def is_injected_message(msg: SupportsTurnMessage) -> bool:
    """用户中途注入的消息（live injection 或 resume guidance）"""
    return msg.role == "user" and bool(
        msg.additional_properties.get(HistoryMarkerKind.INJECTED_KEY, False)
    )

def is_continuation_message(msg: SupportsTurnMessage) -> bool:
    """系统自动发的 'continue' nudge，永远不是用户内容"""
    return msg.role == "user" and bool(
        msg.additional_properties.get(HistoryMarkerKind.CONTINUATION_KEY, False)
    )
```

#### 为什么这么设计？

**关键洞察**：在 Agent 运行时中，user-role 消息有三种来源：

| 类型 | 来源 | 是否开启轮次 | 例子 |
|------|------|--------------|------|
| **Opener** | 用户主动输入 | ✅ 是 | "帮我重构这个函数" |
| **Injected** | 用户中途打断 | ❌ 否 | "等等，先别改" |
| **Continuation** | 系统自动生成 | ❌ 否 | "continue"（用于恢复中断的运行） |

如果用 `role == "user"` 判断轮次边界，会把 injected 和 continuation 也当成轮次开始，导致：
- 回滚功能失效（找不到真正的轮次边界）
- 压缩算法出错（错误地切割对话）
- 审计重放混乱（轮次计数不对）

#### 源码位置

`src/chrys/foundation/models/turns.py`（80 行）

**学习价值**：⭐⭐⭐⭐⭐  
**实用性**：极高——任何需要处理多轮对话的系统都会遇到这个问题。

---

### 🏆 创新点 #2：Config Spec System —— 用 Dataclass Metadata 代替 DSL

#### 问题场景

你需要一个配置系统，支持：
- 从多个来源加载（默认值、配置文件、环境变量、CLI 参数）
- 类型验证和转换
- 实时生效（改了立刻起作用）
- 项目级配置白名单（某些 key 允许项目覆盖，某些不允许）

**常见做法**：用 DSL 定义 schema

```python
# Pydantic Settings 的做法
class Settings(BaseSettings):
    theme: str = Field(default="dark", env="THEME")
    max_tokens: int = Field(default=1000, env="MAX_TOKENS")
    
    class Config:
        env_file = ".env"
```

**Chrys 的做法**：用 dataclass field metadata

```python
# foundation/config/settings.py + spec.py

@dataclass
class Settings:
    theme: str = field(
        default=DEFAULT_THEME,
        metadata=spec(
            key="ui.theme",              # YAML 路径
            env="CHRYS_THEME",           # 环境变量名
            apply=Apply.LIVE,            # 实时生效
            project_merge=ProjectMerge.DENY,  # 项目不能覆盖
            risk=Risk.COSMETIC,          # 风险等级
        ),
    )
    
    max_output_tokens: int = field(
        default=4096,
        metadata=spec(
            key="llm.max_output_tokens",
            env="CHRYS_MAX_OUTPUT_TOKENS",
            apply=Apply.REBUILD,         # 需要重建 Agent
            project_merge=ProjectMerge.ALLOW,  # 项目可以覆盖
            risk=Risk.FUNCTIONAL,
        ),
    )
```

#### 关键设计

**`spec()` 函数返回一个 `SettingSpec` 对象**，包含：

```python
@dataclass(frozen=True)
class SettingSpec:
    key: str                    # YAML 路径（如 "ui.theme"）
    env: str                    # 环境变量名（如 "CHRYS_THEME"）
    apply: Apply                # 生效方式：LIVE / REBUILD / RESTART
    project_merge: ProjectMerge # 项目合并策略：DENY / ALLOW / MERGE_DICT
    risk: Risk                  # 风险等级：COSMETIC / FUNCTIONAL / SECURITY
    coercer: Coercer            # 类型转换器
    # ... 更多元数据
```

**好处**：
1. **Settings 是纯 dataclass**：没有基类魔法，类型检查器能正确推断
2. **所有元数据都在 field 上**：添加新配置只需一个 field 声明
3. **可组合**：不同维度的策略（apply、project_merge、risk）独立正交

#### Source Priority System

**10 层优先级**（从低到高）：

```python
class Source(Enum):
    DEFAULT = auto()           # 1. dataclass 默认值
    USER = auto()              # 2. ~/.chrys/settings.yaml
    PROJECT = auto()           # 3. <root>/.chrys/settings.yaml
    USER_ENV = auto()          # 4. ~/.chrys/.env（遗留）
    ENV = auto()               # 5. 进程环境变量
    CLI = auto()               # 6. 命令行参数
    PROCESS_RUNTIME = auto()   # 7. 运行时设置（如激活 model profile）
    SESSION = auto()           # 8. 会话级 pin
    RUNTIME = auto()           # 9. 实时修改（如切换主题）
```

**为什么这么细？**

考虑这个场景：
- 用户在 `~/.chrys/settings.yaml` 设置了 `theme: dark`
- 启动时传了 `--theme light`（CLI 覆盖）
- 运行中用户按 F7 切换到 `chrys` 主题（RUNTIME 覆盖）

如果只有 3 层（default < file < env），就无法区分"当前主题是 CLI 设的还是 runtime 设的"，导致持久化时写错层。

#### 源码位置

- `src/chrys/foundation/config/spec.py`（定义 SettingSpec）
- `src/chrys/foundation/config/settings.py`（Settings dataclass）
- `src/chrys/foundation/config/settings_store.py`（加载和合并逻辑）

**学习价值**：⭐⭐⭐⭐⭐  
**实用性**：极高——任何需要复杂配置管理的系统都能借鉴。

---

### 🏆 创新点 #3：WeakIdentityRegistry —— 解决 GC 后地址重用误判

#### 问题场景

你需要追踪"哪些 Content 对象已经在对话历史中"，用于回声消除。

**天真的做法**：

```python
seen_ids = set()

for content in message.contents:
    if id(content) in seen_ids:
        print("重复！这是回声！")
    seen_ids.add(id(content))
```

**这会出什么问题？**

```python
# 第 1 轮
obj1 = Content(type="text", text="hello")
id1 = id(obj1)  # 例如：140234567890
seen_ids.add(id1)

del obj1  # GC 回收

# 第 2 轮
obj2 = Content(type="text", text="world")
id2 = id(obj2)  # 可能又是 140234567890！

assert id2 in seen_ids  # True！但 obj2 不是回声！
```

**根本原因**：`id()` 返回内存地址，GC 回收后新对象可能占用同一地址。裸 `id()` 整数既不保持对象存活，也不注意其死亡。

#### Chrys 的做法

```python
# kernel/identity.py

import weakref

class WeakIdentityRegistry:
    """对象身份成员资格，注意到成员死亡"""
    
    def __init__(self, *, ignore_usage: bool = False):
        self._refs: dict[int, weakref.ref[Any]] = {}
        self._ignore_usage = ignore_usage
    
    def register(self, obj: Any) -> None:
        """添加 obj 为成员，只要它还活着"""
        key = id(obj)
        
        # 检查是否已注册且仍是同一对象
        existing = self._refs.get(key)
        if existing is not None and existing() is obj:
            return  # 已经注册过了
        
        # 创建弱引用，带清理回调
        def _evict(dead: weakref.ref, *, key=key, refs=self._refs):
            # 仅在未重注册时移除（防止地址重用导致的误删）
            if refs.get(key) is dead:
                del refs[key]
        
        self._refs[key] = weakref.ref(obj, _evict)
    
    def __contains__(self, obj: object) -> bool:
        """检查 obj 是否是成员"""
        ref = self._refs.get(id(obj))
        # 必须同时满足：
        # 1. ref 不为 None（曾经注册过）
        # 2. ref() is obj（仍然是同一对象，不是复制品）
        return ref is not None and ref() is obj
```

#### 为什么需要 ContentList？

```python
class ContentList(list["Content"]):
    """可弱引用的 list"""
    __slots__ = ("__weakref__",)  # 添加弱引用支持
```

**问题**：内置 `list` 不支持弱引用。

```python
import weakref
lst = [1, 2, 3]
weakref.ref(lst)  # TypeError: cannot create weak reference to 'list' object
```

**但在 Chrys 中**，`Message.contents` 被强制转换为 `ContentList`：

```python
class Message:
    @property
    def contents(self) -> ContentList:
        return self.__dict__["contents"]
    
    @contents.setter
    def contents(self, value: Sequence[Content]) -> None:
        self.__dict__["contents"] = ContentList(value)
```

**为什么这样做？**

压缩锚点和摘要缓存需要捕获 `id(msg.contents)`：

```python
# 压缩模块
anchor_id = id(message.contents)  # 列表的身份

# 稍后用 WeakIdentityRegistry 检查
if message.contents in echo_memo:
    # 列表没被替换过，还是原来那个
    pass
else:
    # 列表被替换了（可能是压缩导致的）
    pass
```

#### 源码位置

`src/chrys/kernel/identity.py`（96 行）

**学习价值**：⭐⭐⭐⭐⭐  
**实用性**：中高——任何需要追踪对象身份的系统都会遇到这个问题。

---

### 🏆 创新点 #4：EventBus with Backpressure —— 永不 fire-and-forget

#### 问题场景

Agent 内部需要通知多个模块（UI、日志、历史、变更追踪）。

**Fire-and-forget 的做法**：

```python
async def publish(self, event):
    for handler in self.handlers.get(type(event), []):
        asyncio.create_task(handler(event))  # 发了就走
```

**这会出什么问题？**

1. **程序退出时任务没完成**：`create_task` 的任务可能在 `main()` 返回前还没执行完
2. **错误被吞掉**：后台任务的异常不会被捕获
3. **顺序无法保证**：B 可能比 A 先完成

#### Chrys 的做法

```python
# foundation/events/bus.py

async def publish(self, event: Event, *, raise_handler_errors: bool = False) -> None:
    """Publish an event to all subscribers."""
    failures: list[Exception] = []
    
    for handler in list(self._handlers.get(type(event), [])):
        try:
            await handler(event)  # ← 关键：await，同步等待
        except Exception as exc:
            failures.append(exc)
            logger.exception("Handler failed")
    
    if raise_handler_errors and failures:
        raise ExceptionGroup("Handler failures", failures)
```

**关键设计**：
- **await handler**：背压（backpressure）——如果 handler 慢，publisher 也被阻塞
- **错误隔离**：一个 handler 失败不影响其他
- **可选抛出**：`raise_handler_errors=True` 给严格模式用

#### Stream：无界队列 + 生产者侧过滤

对于流式消费者（如实时显示 LLM 输出），Chrys 提供了 `_EventStream`：

```python
class _EventStream:
    def __init__(self, bus, event_types):
        self._queue = asyncio.Queue()  # 无界队列
        self._event_types = event_types
    
    def _deliver(self, event: Event) -> None:
        # 生产者侧过滤
        if self._event_types and not isinstance(event, self._event_types):
            return
        
        # 永不丢弃
        self._queue.put_nowait(event)
        
        # 高水位告警
        if self._queue.qsize() >= 10_000 and not self._warned:
            logger.warning("Stream backlog exceeded 10000 events")
```

**为什么用无界队列？**
- 有界队列满了会阻塞 publisher 或丢事件
- 无界队列保证永不丢事件，但可能内存暴涨
- Chrys 选择：**宁可内存暴涨，也不丢 LLM 输出**

#### 源码位置

`src/chrys/foundation/events/bus.py`（144 行）

**学习价值**：⭐⭐⭐⭐  
**实用性**：极高——任何异步系统都需要处理事件通信。

---

### 🏆 创新点 #5：Platform Abstraction —— 统一三平台行为

#### 问题场景

你的 Agent 要在 Windows/macOS/Linux 上运行，但：
- Windows 用 IOCP，POSIX 用 epoll/kqueue
- Windows 路径用反斜杠，POSIX 用正斜杠
- Windows 进程隐藏窗口的方式不同
- stdin/stdout 的处理在不同平台有差异

**如果不抽象**：到处是 `if sys.platform == "win32": ...`

#### Chrys 的做法

```python
# foundation/platform/__init__.py

@dataclass(frozen=True)
class PlatformInfo:
    os_name: str  # "windows", "macos", "linux"
    os_version: str
    arch: str
    shell: ShellInfo
    config_dir: Path  # ~/.chrys or %APPDATA%/chrys
    data_dir: Path

@functools.cache
def get_platform() -> PlatformInfo:
    """Cached platform detection singleton"""
    # 一次性检测，全局复用
    ...
```

**使用方式**：

```python
from chrys.foundation.platform import get_platform

platform = get_platform()

if platform.is_windows:
    # Windows 特定逻辑
    handle = loop._proactor.wait_for_handle(handle)
else:
    # POSIX 逻辑
    loop.add_reader(fd, callback)
```

#### 关键抽象

**进程管理**（`foundation/platform/process.py`）：

```python
async def spawn_managed_stdio_process(cmd, ...):
    """统一的子进程管理"""
    
    # Windows: 隐藏窗口、IOCP wait_for_handle
    if platform.is_windows:
        kwargs["creationflags"] = subprocess.CREATE_NO_WINDOW
        # 不能用 WaitFor* 在 executor 线程（会阻塞线程池）
        # 必须用 loop._proactor.wait_for_handle
    
    # POSIX: fork/exec、SIGKILL
    else:
        kwargs["start_new_session"] = True
    
    proc = await asyncio.create_subprocess_exec(*cmd, **kwargs)
    return proc
```

**stdin=DEVNULL 的重要性**：

```python
# 每个非交互式子进程都必须显式设置 stdin=DEVNULL
proc = await asyncio.create_subprocess_exec(
    *cmd,
    stdin=asyncio.subprocess.DEVNULL,  # ← 关键
    stdout=asyncio.subprocess.PIPE,
    stderr=asyncio.subprocess.PIPE,
)
```

**为什么？**
- ACP 的 stdin 是 JSON-RPC 管道
- 继承 stdin 可能导致子进程阻塞或吃掉协议字节
- Windows 上尤其重要（ConPTY 的行为差异）

#### 源码位置

- `src/chrys/foundation/platform/__init__.py`（平台检测）
- `src/chrys/foundation/platform/process.py`（进程管理）
- `src/chrys/foundation/platform/files.py`（文件系统操作）

**学习价值**：⭐⭐⭐⭐  
**实用性**：极高——任何跨平台 Python 应用都需要这个。

---

## 2. 次要亮点（值得一看）

### 2.1 Error Classification —— 错误的识别和处理

**文件**：`foundation/errors/classify.py` + `network.py`

**核心思想**：不是所有错误都应该重试。DNS 失败不会因重试而解决。

```python
def classify_error(exc: Exception) -> ErrorClassification:
    """分类错误：可重试 vs 不可重试"""
    
    # 网络错误：检查 OS 特定的错误码
    if isinstance(exc, OSError):
        if platform.is_windows:
            code = exc.winerror
        else:
            code = exc.errno
        
        # 查表判断是否可重试
        if code in RETRIABLE_NETWORK_ERRORS:
            return ErrorClassification.RETRIABLE
        else:
            return ErrorClassification.TERMINAL
    
    # 提供者错误：看具体消息
    if "rate limit" in str(exc).lower():
        return ErrorClassification.RETRIABLE_WITH_BACKOFF
    
    return ErrorClassification.TERMINAL
```

**学习价值**：⭐⭐⭐  
**实用性**：高——任何需要网络重试的系统都需要。

---

### 2.2 I18n —— 多语言支持

**文件**：`foundation/i18n/__init__.py`

**核心思想**：类型安全的翻译函数。

```python
# 定义
NAME = msg("app.name", fallback="iCode")

# 使用
print(NAME)  # 根据 locale 自动选择翻译
```

**关键设计**：
- Model/tool/protocol 文本保持英文（避免混淆）
- 只有 UI 显示文本才翻译
- Catalog 在启动时加载，运行时查找

**学习价值**：⭐⭐⭐  
**实用性**：中——如果你的应用需要多语言。

---

## 3. 不值得花时间的地方

以下部分相对简单或重复，可以快速跳过：

| 子模块 | 为什么跳过 |
|--------|------------|
| `util/` | 通用工具函数（日期、哈希、锁），没什么特别的 |
| `vendor/` | Vendored ripgrep，就是下载二进制 |
| `patches/` | Textual monkey-patch，除非你在用 Textual，否则不重要 |
| `tool_kinds.py` 等 | 主要是常量定义，看看就行 |

---

## 4. 学习建议

### 如果你只有 2 小时

只看这 3 个：
1. **Turn Boundary**（`models/turns.py`）——理解轮次判定的复杂性
2. **Config Spec**（`config/spec.py` + `settings.py`）——学习如何设计灵活的配置系统
3. **WeakIdentityRegistry**（`kernel/identity.py`）——理解对象身份追踪的陷阱

### 如果你有半天

加上：
4. **EventBus**（`events/bus.py`）——异步事件通信的最佳实践
5. **Platform**（`platform/__init__.py` + `process.py`）——跨平台抽象

### 如果你想彻底搞懂

按这个顺序逐个文件读：
1. `events/bus.py` → `events/types.py`
2. `config/spec.py` → `config/settings.py` → `config/settings_store.py`
3. `models/turns.py` → `models/invocations.py` → `models/workspace.py`
4. `platform/__init__.py` → `platform/process.py` → `platform/files.py`
5. `errors/classify.py` → `errors/network.py`
6. `kernel/identity.py`（虽然属于 Kernel，但概念上是基础设施）

---

## 5. 总结：Foundation 层的核心价值

| 创新点 | 解决的问题 | 适用场景 |
|--------|------------|----------|
| **Turn Boundary** | 不能用 `role == "user"` 判断轮次 | 任何多轮对话系统 |
| **Config Spec** | 复杂配置的灵活管理 | 企业级应用的配置系统 |
| **WeakIdentityRegistry** | GC 后地址重用误判 | 需要追踪对象身份的系统 |
| **EventBus with Backpressure** | 异步事件通信的可靠性 | 任何事件驱动系统 |
| **Platform Abstraction** | 跨平台行为统一 | 跨平台 Python 应用 |

**一句话总结**：Foundation 层的价值不在于"做了什么新功能"，而在于"把基础问题想透了、做对了"。每一个设计决策都有明确的 trade-off 分析和守卫测试。

---

## 附录：源码索引

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| `events/bus.py` | 144 | EventBus 实现 |
| `events/types.py` | ~400 | 事件类型定义 |
| `config/spec.py` | ~200 | SettingSpec 定义 |
| `config/settings.py` | ~500 | Settings dataclass |
| `config/settings_store.py` | ~300 | 配置加载和合并 |
| `models/turns.py` | 80 | 轮次边界判定 |
| `models/invocations.py` | ~150 | 调用身份定义 |
| `platform/__init__.py` | ~100 | 平台检测 |
| `platform/process.py` | ~200 | 进程管理 |
| `errors/classify.py` | ~150 | 错误分类 |
| `kernel/identity.py` | 96 | WeakIdentityRegistry |

---

**下一步**：你想深入哪个创新点？我可以带你逐行读源码，或者你可以自己读完后来问我具体问题。
