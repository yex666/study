# F05: errors — 错误分类和处理

> **模块路径**: `src/chrys/foundation/errors/`
> **核心文件**: `classify.py`, `kinds.py`, `network.py`, `provider.py`
> **一句话**: 把各种异常分类成统一的 ErrorKind，决定是否可以重试。

---

## 这是什么？

一个**错误分类系统**。把各种异常（网络超时、API 限流、认证失败等）分类成统一的 `ErrorKind`，并决定是否可以重试。

```python
from chrys.foundation.errors import classify_error, is_retryable

try:
    response = await llm.chat(messages)
except Exception as e:
    classification = classify_error(e)
    
    print(classification.kind)       # ErrorKind.RATE_LIMITED
    print(classification.retryable)  # True
    
    if is_retryable(e):
        await retry_with_backoff()
```

---

## 为什么需要？

### 如果不做会怎样？

到处是 if-else 判断异常类型：

```python
# ❌ 散落的异常处理
try:
    response = await llm.chat(messages)
except httpx.ConnectTimeout:
    logger.error("Connection timeout")
    await retry()  # 所有异常都重试？
except openai.RateLimitError:
    logger.error("Rate limited")
    await sleep(60)
    await retry()
except anthropic.AuthenticationError:
    logger.error("Auth failed")
    raise  # 这个不该重试
except Exception as e:
    # 不知道是什么错误，盲目重试？
    await retry()
```

**问题**：
1. 每种异常都要单独处理（代码重复）
2. 不知道哪些错误该重试（DNS 失败重试有用吗？）
3. 无法统一展示错误信息给用户
4. 新增 Provider 要改很多地方

### 用了错误分类之后

```python
# ✅ 统一的分类和决策
try:
    response = await llm.chat(messages)
except Exception as e:
    classification = classify_error(e)
    
    # 根据 kind 统一处理
    if classification.kind == ErrorKind.AUTH_FAILED:
        ui.show_error("Authentication failed. Check your API key.")
        raise
    
    if not classification.retryable:
        ui.show_error(f"Non-retryable error: {classification.kind}")
        raise
    
    # 可重试的错误，按策略重试
    await retry_with_backoff(classification)
```

---

## 核心设计：三个关键决策

### 决策 1: 封闭的 ErrorKind 集合

```python
class ErrorKind(StrEnum):
    """What went wrong, as far as explicit evidence shows."""
    
    # Network: the request never got a response.
    NO_ROUTE = "no_route"
    HOST_UNREACHABLE = "host_unreachable"
    DNS_FAILED = "dns_failed"
    CONNECTION_REFUSED = "connection_refused"
    CONNECT_TIMEOUT = "connect_timeout"
    CONNECTION_FAILED = "connection_failed"
    CONNECTION_LOST = "connection_lost"
    READ_TIMEOUT = "read_timeout"
    WRITE_TIMEOUT = "write_timeout"
    PROXY_REJECTED = "proxy_rejected"
    PROXY_AUTH_FAILED = "proxy_auth_failed"
    TLS_FAILED = "tls_failed"
    INVALID_ENDPOINT = "invalid_endpoint"
    
    # Provider: the service answered with an error.
    RATE_LIMITED = "rate_limited"
    QUOTA_EXHAUSTED = "quota_exhausted"
    OVERLOADED = "overloaded"
    SERVER_ERROR = "server_error"
    CONTEXT_OVERFLOW = "context_overflow"
    PAYLOAD_TOO_LARGE = "payload_too_large"
    AUTH_FAILED = "auth_failed"
    REQUEST_REJECTED = "request_rejected"
    CONTENT_FILTERED = "content_filtered"
    STREAM_TRUNCATED = "stream_truncated"
    
    # Runtime: Chrys's own policies concluded the attempt.
    STREAM_STALLED = "stream_stalled"
    INVALID_RESPONSE = "invalid_response"
    UNKNOWN = "unknown"
```

**为什么是 StrEnum？**
- 可以序列化成 JSON（用于日志和 UI）
- 值是稳定的契约（display text、logs 都依赖它们）

**三大类错误**：
1. **Network**：请求没有收到响应（DNS 失败、连接超时等）
2. **Provider**：服务返回了错误（限流、认证失败、上下文溢出等）
3. **Runtime**：Chrys 自己的策略判定（流停滞、无效响应等）

---

### 决策 2: 显式证据优先于隐式链

```python
def classify_error(exc: BaseException) -> ErrorClassification:
    """Classify *exc* for retry policy and display."""
    
    # 只遍历显式的异常图（__cause__、args 中的异常、group members）
    explicit = tuple(iter_explicit_graph(exc))
    
    # 不遍历 __context__（可能是过时的旧异常）
    # full = tuple(iter_full_chain(exc))  # ← 只给 legacy layers 用
    
    # 从显式证据中提取信号
    signal = provider_signal(explicit)
    network = classify_connection_failure(exc, route)
    
    # 优先级：overflow > signal > network > status code > type
    kind, evidence = _explicit_kind(explicit, signal, network, overflow)
```

**为什么？**

考虑这个场景：
```python
try:
    # 第一次请求失败（413 Payload Too Large）
    await request_1()
except Exception as first_error:
    try:
        # 第二次请求也失败了（500 Server Error）
        await request_2()
    except Exception as second_error:
        # second_error.__context__ = first_error
        raise second_error
```

如果遍历完整链（包括 `__context__`），会看到 413 错误，可能误判为 `PAYLOAD_TOO_LARGE`。但实际上当前失败的是 500，应该分类为 `SERVER_ERROR`。

**解决方案**：只遍历显式图（`__cause__`、args 中的异常），忽略 `__context__`。

---

### 决策 3: 不可重试的种类硬编码

```python
# Kinds no retry can fix, whatever the transient layers say.
_NON_RETRYABLE_KINDS = frozenset({
    ErrorKind.QUOTA_EXHAUSTED,     # 配额用完了
    ErrorKind.CONTEXT_OVERFLOW,     # 上下文溢出
    ErrorKind.PAYLOAD_TOO_LARGE,    # 载荷太大
})

def classify_error(exc: BaseException) -> ErrorClassification:
    # ...
    
    # 这些错误无论 transient layers 怎么说，都不重试
    if kind in _NON_RETRYABLE_KINDS:
        retryable = False
    
    # DNS 失败是确定性的，重试也没用
    elif network is not None and network.deterministic:
        retryable = False
```

**为什么？**

有些错误重试多少次都没用：
- **QUOTA_EXHAUSTED**：配额用完了，等一个小时再来
- **CONTEXT_OVERFLOW**：上下文太长，需要压缩而不是重试
- **DNS_FAILED**：域名解析失败，网络不通，重试也没用

---

## 源码解析

### ErrorClassification 数据类

```python
@dataclass(frozen=True, slots=True)
class ErrorClassification:
    """What an exception means for retry policy and for the user."""
    
    kind: ErrorKind                           # 错误类型
    retryable: bool                           # 是否可重试
    signal: ProviderSignal | None = None      # Provider 返回的信号
    timeout_phase: TimeoutPhase | None = None # 超时阶段（connect/write/read/pool）
    route: RouteFacts | None = None           # 路由快照（代理、目标地址）
    failed_at_first_hop: bool = False         # 第一跳就失败了（代理或目标）
    from_model_service: bool = False          # 来自模型服务（不是 MCP server）
    invalidates_continuation_token: bool = False  # 使续接令牌失效
    evidence: str = ""                        # 决策证据（如 "http 429"）
```

### classify_error() 主函数

```python
def classify_error(exc: BaseException) -> ErrorClassification:
    """Classify *exc* for retry policy and display."""
    
    # 1. 遍历显式异常图
    explicit = tuple(iter_explicit_graph(exc))
    
    # 2. 提取路由信息
    route = route_of(explicit)
    
    # 3. 提取 Provider 信号（HTTP 状态码、错误消息）
    signal = provider_signal(explicit)
    
    # 4. 分类网络连接失败
    network = classify_connection_failure(exc, route)
    
    # 5. 检查是否是上下文溢出
    overflow = names_context_overflow(
        signal, 
        (_clean_exception_text(node) for node in explicit)
    )
    
    # 6. 确定 kind（优先级：overflow > signal > network > status > type）
    kind, evidence = _explicit_kind(explicit, signal, network, overflow)
    
    # 7. 判断是否可重试
    full = tuple(iter_full_chain(exc))  # 完整链只给 legacy layers
    
    if _legacy.is_owner_terminal(full):
        retryable = False
    elif signal is not None and signal.explicit_retryable is not None:
        retryable = signal.explicit_retryable
    elif (network is not None and network.deterministic) or kind in _NON_RETRYABLE_KINDS:
        retryable = False
    elif signal is not None and is_2xx(signal.status_code):
        retryable = stream_error_retryable(signal)
    else:
        retryable = _legacy.is_transient(full)
    
    return ErrorClassification(
        kind=kind,
        retryable=retryable,
        signal=signal,
        timeout_phase=(network.timeout_phase if network else None) or _first_timeout_phase(explicit),
        route=route,
        failed_at_first_hop=network is not None and network.first_hop_evidence,
        from_model_service=_from_model_service(explicit, route),
        invalidates_continuation_token=_invalidates_continuation_token(explicit),
        evidence=evidence,
    )
```

### Provider Signal 提取

```python
@dataclass(frozen=True)
class ProviderSignal:
    """A provider's HTTP response, parsed into structured fields."""
    
    status_code: int
    headers: dict[str, str]
    body: str
    source: BaseException
    explicit_retryable: bool | None = None  # Provider 明确说的（如 Retry-After header）

def provider_signal(explicit: tuple[BaseException, ...]) -> ProviderSignal | None:
    """Extract the most recent provider signal from explicit exceptions."""
    
    for node in reversed(explicit):
        if isinstance(node, ProviderResponseError):
            return ProviderSignal(
                status_code=node.status_code,
                headers=node.headers,
                body=node.body,
                source=node,
                explicit_retryable=node.retryable,
            )
    
    return None


def signal_kind(signal: ProviderSignal) -> tuple[ErrorKind, str]:
    """Map a provider signal to an ErrorKind."""
    
    status = signal.status_code
    
    if status == 401:
        return ErrorKind.AUTH_FAILED, "http 401"
    elif status == 403:
        return ErrorKind.AUTH_FAILED, "http 403"
    elif status == 429:
        return ErrorKind.RATE_LIMITED, "http 429"
    elif status == 413:
        return ErrorKind.PAYLOAD_TOO_LARGE, "http 413"
    elif status == 400:
        # 检查是否是上下文溢出
        if names_context_overflow_from_body(signal.body):
            return ErrorKind.CONTEXT_OVERFLOW, "http 400 context overflow"
        return ErrorKind.REQUEST_REJECTED, "http 400"
    elif 500 <= status < 600:
        return ErrorKind.SERVER_ERROR, f"http {status}"
    
    return ErrorKind.UNKNOWN, f"http {status}"
```

### Network Failure 分类

```python
@dataclass(frozen=True)
class NetworkFailure:
    """A classified network failure."""
    
    kind: ErrorKind
    evidence: str
    deterministic: bool  # True = 重试也没用（如 DNS 失败）
    timeout_phase: TimeoutPhase | None = None
    first_hop_evidence: bool = False  # 第一跳就失败了

def classify_connection_failure(
    exc: BaseException, 
    route: RouteFacts | None,
) -> NetworkFailure | None:
    """Classify a network-level failure."""
    
    # 遍历异常链，找网络相关的异常
    for node in iter_explicit_graph(exc):
        # httpx/httpcore 超时
        if (phase := _timeout_phase(node)) is not None:
            return NetworkFailure(
                kind=ErrorKind.CONNECT_TIMEOUT if phase == "connect" else ErrorKind.READ_TIMEOUT,
                evidence=f"timeout {phase}",
                deterministic=False,  # 超时可能是暂时的
                timeout_phase=phase,
            )
        
        # socket 错误
        if isinstance(node, OSError):
            errno = node.errno
            
            # DNS 失败（确定性，重试没用）
            if errno in (socket.EAI_NONAME, socket.EAI_NODATA):
                return NetworkFailure(
                    kind=ErrorKind.DNS_FAILED,
                    evidence=f"errno {errno}",
                    deterministic=True,
                    first_hop_evidence=True,
                )
            
            # 连接拒绝（确定性）
            elif errno == socket.ECONNREFUSED:
                return NetworkFailure(
                    kind=ErrorKind.CONNECTION_REFUSED,
                    evidence="errno ECONNREFUSED",
                    deterministic=True,
                    first_hop_evidence=True,
                )
            
            # 网络不可达（确定性）
            elif errno in (socket.ENETUNREACH, socket.EHOSTUNREACH):
                return NetworkFailure(
                    kind=ErrorKind.HOST_UNREACHABLE,
                    evidence=f"errno {errno}",
                    deterministic=True,
                )
    
    return None
```

---

## 实际使用场景

### 场景 1: LLM 调用的重试逻辑

```python
async def call_llm_with_retry(messages, max_retries=3):
    for attempt in range(max_retries + 1):
        try:
            return await llm.chat(messages)
        except Exception as e:
            classification = classify_error(e)
            
            # 不可重试的错误，直接抛
            if not classification.retryable:
                raise
            
            # 最后一次尝试也失败了
            if attempt == max_retries:
                raise
            
            # 计算退避时间
            backoff = calculate_backoff(classification, attempt)
            await asyncio.sleep(backoff)
```

### 场景 2: UI 显示友好的错误信息

```python
def show_error_to_user(exc: Exception):
    classification = classify_error(exc)
    
    error_messages = {
        ErrorKind.AUTH_FAILED: "Authentication failed. Please check your API key.",
        ErrorKind.RATE_LIMITED: "Rate limited. Please wait a moment and try again.",
        ErrorKind.CONTEXT_OVERFLOW: "Context too long. Please start a new session or compact history.",
        ErrorKind.DNS_FAILED: "DNS resolution failed. Please check your network connection.",
        ErrorKind.CONNECT_TIMEOUT: "Connection timeout. The server may be down.",
    }
    
    message = error_messages.get(classification.kind, f"Unknown error: {classification.kind}")
    ui.show_error(message)
```

### 场景 3: 上下文溢出时自动压缩

```python
async def chat_with_auto_compact(messages):
    try:
        return await llm.chat(messages)
    except Exception as e:
        if is_context_overflow(e):
            # 自动压缩历史
            compressed = await compact_messages(messages)
            return await llm.chat(compressed)
        raise
```

---

## 和其他模块的关系

```
errors 被以下模块使用：
├── kernel/loop.py（工具循环中的错误处理）
├── service/llm/clients.py（LLM 客户端的重试逻辑）
├── orchestration/engine/run/（turn 运行时的错误处理）
├── service/context/compaction/（上下文溢出检测）
└── app/tui/screens/（UI 错误提示）
```

---

## 如果你要自己实现

### 最小版本（20 行）

```python
from enum import Enum

class SimpleErrorKind(Enum):
    NETWORK = "network"
    RATE_LIMIT = "rate_limit"
    AUTH = "auth"
    UNKNOWN = "unknown"

def classify_simple(exc: Exception) -> SimpleErrorKind:
    msg = str(exc).lower()
    
    if "timeout" in msg or "connection" in msg:
        return SimpleErrorKind.NETWORK
    elif "rate limit" in msg or "429" in msg:
        return SimpleErrorKind.RATE_LIMIT
    elif "auth" in msg or "401" in msg or "403" in msg:
        return SimpleErrorKind.AUTH
    else:
        return SimpleErrorKind.UNKNOWN

def is_retryable(exc: Exception) -> bool:
    kind = classify_simple(exc)
    return kind in (SimpleErrorKind.NETWORK, SimpleErrorKind.RATE_LIMIT)
```

### 进阶版本

加 Provider Signal 提取、Network Failure 分类、显式证据优先。参考 Chrys 的 `classify.py`。

---

## 下一步

- **上一篇**: F04 models — 核心数据模型
- **下一篇**: F06 i18n — 国际化
