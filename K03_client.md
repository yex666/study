# K03: client — Wire Client 抽象基类

> **文件路径**: `src/chrys/kernel/client.py` (~1500 行)
> **一句话**: 定义 LLM 客户端的抽象接口，处理存储模式解析、续接令牌、请求观察等跨提供者逻辑。

---

## 这是什么？

**BaseChatClient** 是所有 LLM 客户端的基类。它不直接调用任何 LLM API，而是提供：

1. **存储模式解析**：判断使用服务端存储还是本地存储
2. **续接令牌管理**：处理 OpenAI Responses API 等有状态 API 的 continuation token
3. **压缩准入控制**：决定何时触发上下文压缩
4. **请求观察者**：在每次 wire 调用前记录消息身份（用于回声消除）

---

## 核心设计

### 1. 存储模式解析

```python
def resolve_storage_mode_and_handles(
    options: dict[str, Any],
    *,
    stores_by_default: bool = False,
    client_kwargs: Mapping[str, Any] | None = None,
    force_stateless: bool = False,
) -> StorageModeResolution:
    """Resolve whether to use service-side or client-side storage.
    
    Rules:
    - force_stateless=True → always client-side (e.g., DeepSeek Responses)
    - store=False in options → client-side
    - store=True or stores_by_default → service-side
    - client_kwargs can escalate to service-side but never downgrade
    """
    
    # 检查强制无状态标志
    if force_stateless:
        return StorageModeResolution(service_side=False, handles_present=False)
    
    # 从 options 中读取 store 参数
    store_value = options.get("store")
    
    # 从 client_kwargs 中读取（可以升级但不能降级）
    kwargs_store = (client_kwargs or {}).get("store")
    
    # 确定最终模式
    if store_value is False:
        service_side = False
    elif kwargs_store is True and store_value is None:
        service_side = True
    else:
        service_side = store_value if store_value is not None else stores_by_default
    
    # 收集会话句柄
    handles = collect_conversation_handles(options, client_kwargs)
    
    return StorageModeResolution(
        service_side=service_side,
        handles_present=bool(handles),
    )
```

### 2. BaseChatClient 基类

```python
class BaseChatClient(SerializationMixin, ABC):
    """Abstract base for all LLM wire clients."""
    
    # 子类可以覆盖这些类变量
    STORES_BY_DEFAULT: ClassVar[bool] = False
    FORCES_STATELESS: ClassVar[bool] = False
    MIN_OUTPUT_CAP_TOKENS: ClassVar[int] = 0
    OTEL_PROVIDER_NAME: ClassVar[str] = "chrys"
    
    @abstractmethod
    async def _inner_get_response(
        self,
        messages: Sequence[Message],
        *,
        stream: bool = False,
        **kwargs: Any,
    ) -> ChatResponse | ResponseStream:
        """Subclasses implement the actual wire call here."""
        ...
    
    async def get_response(
        self,
        messages: Sequence[Message],
        *,
        stream: bool = False,
        **kwargs: Any,
    ) -> ChatResponse | ResponseStream:
        """Public entry point with compaction admission and observation."""
        
        # 1. 解析存储模式
        storage = resolve_storage_mode_and_handles(
            kwargs,
            stores_by_default=self.STORES_BY_DEFAULT,
            force_stateless=self.FORCES_STATELESS,
        )
        
        # 2. 检查是否需要压缩
        if self._should_compact(messages, kwargs, storage):
            messages = await self._run_compaction(messages, kwargs)
        
        # 3. 创建请求观察者（用于回声消除）
        echo_registry = WeakIdentityRegistry()
        observer = partial(_record_wire_request_content_identities, echo_registry=echo_registry)
        
        # 4. 调用实际实现
        if stream:
            return await self._inner_get_response(messages, stream=True, **kwargs)
        else:
            return await self._inner_get_response(messages, stream=False, **kwargs)
```

### 3. 会话句柄管理

```python
CONVERSATION_HANDLE_KEYS = ("conversation_id", "previous_response_id", "conversation")

def collect_conversation_handles(
    options: dict[str, Any],
    client_kwargs: dict[str, Any] | None = None,
) -> list[str]:
    """Collect all conversation handle keys from options and kwargs."""
    
    handles = []
    
    for key in CONVERSATION_HANDLE_KEYS:
        if key in options and options[key] is not None:
            handles.append(options[key])
        
        if client_kwargs and key in client_kwargs and client_kwargs[key] is not None:
            handles.append(client_kwargs[key])
        
        # 也检查 extra_body 中
        extra_body = options.get("extra_body") or {}
        if key in extra_body and extra_body[key] is not None:
            handles.append(extra_body[key])
    
    return handles


def strip_invalidated_conversation_handles(
    options: dict[str, Any],
    client_kwargs: dict[str, Any] | None = None,
) -> None:
    """Remove all conversation handles (called when state is invalidated)."""
    
    for key in CONVERSATION_HANDLE_KEYS:
        options.pop(key, None)
        
        if client_kwargs:
            client_kwargs.pop(key, None)
        
        extra_body = options.get("extra_body")
        if isinstance(extra_body, dict):
            extra_body.pop(key, None)
```

---

## 与其他模块的关系

```
client.py 依赖：
├── _types.py (Message, ChatResponse, etc.)
├── identity.py (WeakIdentityRegistry)
└── foundation/ (各种辅助函数)

被以下模块使用：
├── loop.py (ToolLoopLayer 调用 client.get_response)
├── service/llm/ (具体 LLM 客户端实现)
└── agent.py (Agent 组合 client)
```

---

## 📚 延伸阅读

- **kernel_implementation.md** 第 1 节：LLM Client Factory 的详细解析
- **service_implementation.md** 第 1 节：api_style 正交轴的设计

---

## 下一步

- **上一篇**: K02 loop — 工具循环层
- **下一篇**: K04+K05 合并概述（交换语法 + 弱身份注册）
