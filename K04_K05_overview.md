# K04+K05: exchanges + identity — 交换语法与弱身份注册

> **文件路径**: 
> - `src/chrys/kernel/exchanges.py` (~800 行)
> - `src/chrys/kernel/identity.py` (96 行)
> 
> **一句话**: Exchange Grammar 定义工具调用与结果的配对规则；WeakIdentityRegistry 追踪对象身份，防止 GC 后地址重用导致的误判。

---

## 📚 重要提示

这两个模块是 Chrys 最核心的创新，已在 **KERNEL_DEEP_DIVE.md** 中有详细的概念解释和源码解析。本文档只提供简要概述和链接。

---

## K04: exchanges.py — 交换语法

### 核心思想

不依赖 `call_id` 来配对工具调用和结果，而是用**结构化的语法规则**。

### 关键概念

- **Exchange（交换）**：一组连续的 assistant 消息（包含工具调用）+ 回答它们的 tool result 消息
- **PairingPolicy（配对策略）**：不同消费者可以声明不同的配对规则
- **命名空间隔离**：`call_id` 和 `image_id` 在不同的命名空间中配对，不会误配

### 为什么需要？

1. **LLM 可能回显历史**：某些模型会在响应中包含之前的对话内容
2. **压缩后 ID 可能变化**：如果依赖 ID，压缩后会找不到对应的结果
3. **重复 ID**：LLM 可能返回重复的 call_id

### 核心代码

```python
@dataclass(frozen=True)
class Exchange:
    response_indices: tuple[int, ...]   # assistant 消息索引
    output_indices: tuple[int, ...]     # tool result 消息索引
    embedded_results: tuple[Occurrence, ...]

def iter_exchanges(accessor, messages) -> Iterator[Exchange]:
    """遍历所有交换"""
    ...

def pair_results(exchange, policy, accessor, messages) -> ExchangePairing:
    """根据策略配对调用和结果"""
    ...
```

### 📖 详细解析

→ **KERNEL_DEEP_DIVE.md** 第 3 节："第一个问题：如何知道哪些工具调用已回答"

---

## K05: identity.py — 弱身份注册

### 核心思想

用弱引用追踪对象身份，对象被 GC 回收时自动移除。

### 为什么需要？

Python 的 `id()` 返回内存地址。GC 回收对象后，新对象可能占用同一地址：

```python
obj1 = Content(text="hello")
id1 = id(obj1)  # 例如：140234567890

del obj1  # GC 回收

obj2 = Content(text="world")
id2 = id(obj2)  # 可能又是 140234567890！

assert id1 == id2  # True，但它们是不同的对象！
```

### 核心代码

```python
class WeakIdentityRegistry:
    """Object-identity membership that notices member death."""
    
    def __init__(self):
        self._refs: dict[int, weakref.ref[Any]] = {}
    
    def register(self, obj: Any) -> None:
        key = id(obj)
        
        # 检查是否已注册且仍是同一对象
        existing = self._refs.get(key)
        if existing is not None and existing() is obj:
            return
        
        # 创建弱引用，带清理回调
        def _evict(dead_ref, *, key=key, refs=self._refs):
            # 仅在未重注册时移除（防止地址重用导致的误删）
            if refs.get(key) is dead_ref:
                del refs[key]
        
        self._refs[key] = weakref.ref(obj, _evict)
    
    def __contains__(self, obj: object) -> bool:
        ref = self._refs.get(id(obj))
        # 必须同时满足：
        # 1. ref 不为 None（曾经注册过）
        # 2. ref() is obj（仍然是同一对象，不是复制品）
        return ref is not None and ref() is obj


class ContentList(list["Content"]):
    """A weakref-able list of message contents."""
    __slots__ = ("__weakref__",)  # 添加弱引用支持
```

### 为什么需要 ContentList？

内置 `list` 不支持弱引用。但压缩锚点和摘要缓存需要捕获 `id(msg.contents)`：

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

### 📖 详细解析

→ **KERNEL_DEEP_DIVE.md** 第 4 节："第二个问题：如何追踪对象身份"

---

## 两者如何协作？

```
LLM 返回响应
    ↓
_land_response_contents()
    ├─ 给每个 function_call 打上序数戳
    └─ 注册到 echo_registry（WeakIdentityRegistry）
    ↓
iter_exchanges() + pair_results()
    ├─ 识别 exchange 边界
    └─ 按位置配对调用和结果（不依赖 ID）
    ↓
_echo_removal()
    ├─ 在流式更新组装前过滤
    └─ 用 echo_registry 检查对象身份
    ↓
执行工具调用
    ↓
循环直到完成
```

---

## 📊 总结

| 模块 | 解决的问题 | 核心价值 |
|------|------------|----------|
| **exchanges.py** | 如何判断哪些调用已被回答 | 不依赖 ID，按位置配对 |
| **identity.py** | GC 后地址重用导致身份误判 | 弱引用追踪对象生死 |

---

## 下一步

- **上一篇**: K03 client — Wire Client 抽象基类
- **下一篇**: K07 compaction — 上下文压缩策略（简短版）
