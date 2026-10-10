# F11: net — 网络操作辅助工具

> **模块路径**: `src/chrys/foundation/net/`
> **核心文件**: `address_class.py`, `route_probe.py`, `url.py`
> **一句话**: 提供网络地址分类、路由探测、URL 标准化等网络安全相关的基础功能。

---

## 这是什么？

一个**网络安全工具箱**。包含多个用于网络地址验证和分类的工具函数：

- **address_class.py** — IP 地址分类（公网/内网/受限）
- **route_probe.py** — 默认路由可用性探测
- **url.py** — URL 和主机名严格规范化

---

## 为什么需要？

### 如果不做会怎样？

**问题 1: SSRF（服务器端请求伪造）漏洞**

```python
# ❌ 直接访问用户提供的 URL
async def fetch_url(url: str):
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.text

# 攻击者可以这样利用：
await fetch_url("http://127.0.0.1/admin")  # 访问本地服务
await fetch_url("http://169.254.169.254/latest/meta-data/")  # AWS 元数据
await fetch_url("http://192.168.1.1/router-config")  # 内网设备
```

**问题 2: 错误的 URL 格式导致崩溃**

```python
# ❌ 没有验证 URL 格式
url = "http://example.com:99999/path"  # 端口号超出范围
# → socket.gaierror 或其他未预期的异常

url = "file:///etc/passwd"  # 非 HTTP 协议
# → 可能读取本地文件

url = "http://user:pass@example.com"  # 包含凭据
# → 凭据泄露到日志
```

**问题 3: 无法判断是否离线**

```python
# ❌ 网络错误时不知道是 DNS 问题还是真的离线
try:
    response = await client.get(url)
except httpx.ConnectError as e:
    # 是 DNS 解析失败？还是真的没有网络连接？
    # 无法区分，只能盲目重试
```

### 用了 net 模块之后

```python
# ✅ 防止 SSRF
from chrys.foundation.net.address_class import is_restricted_address, literal_address
from chrys.foundation.net.route_probe import is_local_target

async def safe_fetch_url(url: str):
    parsed = urlsplit(url)
    host = parsed.hostname
    
    # 检查是否是受限地址
    if literal_address(host):
        if is_restricted_address(literal_address(host)):
            raise ValueError(f"Access to restricted address {host} is forbidden")
    
    # 检查是否是本地目标
    if is_local_target(host):
        raise ValueError(f"Access to local target {host} is forbidden")
    
    # 安全访问外部 URL
    async with httpx.AsyncClient() as client:
        return await client.get(url)

# ✅ 严格的 URL 验证
from chrys.foundation.net.url import normalize_url

safe_url = normalize_url(user_input)
# → 自动拒绝非 HTTP(S)、无效端口、包含凭据的 URL
# → 规范化主机名（IDNA 编码、小写化）
# → 移除 fragment，编码不可见字符

# ✅ 路由探测
from chrys.foundation.net.route_probe import default_route_available

if default_route_available() is False:
    logger.warning("No network route available; marking as offline")
    # 可以在 UI 中显示"离线"状态
```

---

## 核心设计：三个关键决策

### 决策 1: 保守的公网地址分类

```python
def is_restricted_address(value: str) -> bool:
    """Reject non-global, multicast and IPv4-mapped restricted destinations."""
    
    address = ip_address(value)
    
    # IPv4 特殊地址段
    if isinstance(address, IPv4Address):
        # IETF 协议分配（192.0.0.0/24）
        if address in _PROTOCOL_ASSIGNMENTS:
            return True
    
    # IPv6 特殊情况
    if isinstance(address, IPv6Address):
        # IPv4 映射地址（::ffff:192.168.1.1）
        if address.ipv4_mapped is not None:
            return is_restricted_address(str(address.ipv4_mapped))
        
        # NAT64 well-known prefix（64:ff9b::/96）
        # 嵌入的 IPv4 地址决定是否为公网
        if address in _NAT64_WELL_KNOWN:
            ipv4_part = IPv4Address(int(address) & 0xFFFFFFFF)
            return is_restricted_address(str(ipv4_part))
        
        # 过渡地址（6to4, Teredo）必须拒绝
        # 因为它们可能隐藏私有 IPv4 地址
        if address.sixtofour is not None or address.teredo is not None:
            return True
        
        # 已废弃的 site-local（fec0::/10）
        if address.is_site_local:
            return True
    
    # 标准分类：非全局、多播、保留地址都是受限的
    return not address.is_global or address.is_multicast or address.is_reserved
```

**为什么？**

- **IPv4 映射**：`::ffff:192.168.1.1` 实际上是 IPv4 地址，要按 IPv4 规则检查
- **NAT64**：`64:ff9b::192.0.2.1` 嵌入的 IPv4 地址决定是否公网
- **过渡地址**：6to4、Teredo 可能被滥用隐藏私有地址
- **保守策略**：宁可误杀，不可放过（安全第一）

---

### 决策 2: 无侵入的路由探测

```python
def default_route_available() -> bool | None:
    """Return whether either address family has a route to the internet.
    
    Returns:
        True: At least one family has a route
        False: All present families have no route
        None: Cannot determine (no stack, probe failed, etc.)
    """
    
    codes = codes_for(sys.platform)
    if codes is None:
        return None
    
    verdicts: list[_Verdict] = []
    
    for family, address in _PROBES:
        verdict = _family_has_route(family, address, codes)
        
        # 任何一个有路由就返回 True
        if verdict is True:
            return True
        
        verdicts.append(verdict)
    
    # 过滤掉不存在的协议栈
    present = [v for v in verdicts if v != "absent"]
    
    # 所有存在的协议栈都没有路由才返回 False
    return False if present and all(v is False for v in present) else None


def _family_has_route(
    family: socket.AddressFamily,
    address: tuple[str, int],
    codes: NetworkCodes,
) -> _Verdict:
    """Check if a specific address family has a route."""
    
    try:
        # UDP connect 不会发送数据，只查询路由表
        with socket.socket(family, socket.SOCK_DGRAM) as probe:
            probe.connect(address)
    except OSError as exc:
        answered = {exc.errno, getattr(exc, "winerror", None)}
        
        # 协议栈不存在
        if codes.eafnosupport in answered:
            return "absent"
        
        # 无路由的错误码
        no_route_codes = _no_route_codes(codes)
        if answered & no_route_codes:
            return False
        
        # 其他错误（防火墙、权限等）不确定
        return None
    
    # 连接成功说明有路由
    return True
```

**为什么？**

- **UDP connect**：不发送数据包，只查询内核路由表，开销极小
- **双协议栈**：同时检查 IPv4 和 IPv6，只要有一个有路由就算在线
- **三态返回值**：True/False/None 区分"确定在线"/"确定离线"/"无法判断"
- **协议栈缺失**：某些系统没有 IPv6 栈，不算离线

---

### 决策 3: 严格的 URL 规范化

```python
def normalize_url(value: str, *, limit: int = 2048, endpoint: bool = False) -> str:
    """Keep path/query spelling, normalize the authority, and remove fragments."""
    
    # 基本验证
    if not isinstance(value, str) or not value or len(value) > limit:
        raise ValueError("Invalid URL length")
    
    # 拒绝控制字符和反斜杠
    if any(ord(c) <= 32 or 127 <= ord(c) <= 159 for c in value) or "\\" in value:
        raise ValueError("Invalid URL characters")
    
    parts = urlsplit(value)
    
    # 只接受 HTTP(S)
    if parts.scheme.lower() not in {"http", "https"}:
        raise ValueError("An absolute HTTP(S) URL without credentials is required")
    
    # 必须有主机名
    if not parts.hostname:
        raise ValueError("An absolute HTTP(S) URL without credentials is required")
    
    # 拒绝包含凭据的 URL
    if parts.username is not None:
        raise ValueError("An absolute HTTP(S) URL without credentials is required")
    
    # 规范化主机名
    host = hostname(parts.hostname)
    
    # 验证端口
    port = parts.port
    if port is not None and not 1 <= port <= 65535:
        raise ValueError("Invalid port")
    
    # 端点不允许 query 和 fragment
    if endpoint and (parts.query or parts.fragment):
        raise ValueError("Endpoints cannot contain query strings or fragments")
    
    # 构造规范化的 authority
    authority = f"[{host}]" if ":" in host else host
    if port is not None and port != (443 if parts.scheme.lower() == "https" else 80):
        authority += f":{port}"
    
    # 编码不可见字符（防止 URL 欺骗）
    path, query = _encode_invisible(parts.path), _encode_invisible(parts.query)
    
    # 重建 URL（移除 fragment）
    return urlunsplit((parts.scheme.lower(), authority, path or "/", query, ""))


def hostname(value: str) -> str:
    """Normalize an exact DNS hostname or IP literal."""
    
    if not isinstance(value, str) or not value or any(c.isspace() for c in value):
        raise ValueError("Invalid hostname")
    
    # 拒绝 URL 语法字符
    if any(c in value for c in "/@?#\\%*"):
        raise ValueError("Invalid hostname")
    
    value = value.rstrip(".").lower()
    
    # 尝试作为 IP 地址解析
    try:
        return str(ipaddress.ip_address(value))
    except ValueError:
        pass
    
    # IDNA 编码（国际化域名）
    if "_" not in value:
        return idna.encode(value, uts46=True, std3_rules=True).decode("ascii").rstrip(".")
    
    # 允许下划线（DNS SRV 记录、Docker Compose）
    if not value.isascii():
        raise ValueError("Invalid hostname")
    
    labels = []
    for label in value.split("."):
        if "_" in label:
            # 带下划线的标签用宽松规则验证
            if _UNDERSCORE_LABEL.fullmatch(label) is None:
                raise ValueError("Invalid hostname")
            labels.append(label)
        else:
            # 普通标签用严格 IDNA 规则
            labels.append(idna.encode(label, uts46=True, std3_rules=True).decode("ascii"))
    
    result = ".".join(labels)
    if len(result) > 253:
        raise ValueError("Invalid hostname")
    
    return result
```

**为什么？**

- **拒绝非 HTTP**：防止 `file://`、`ftp://` 等协议
- **拒绝凭据**：防止密码泄露到日志
- **IDNA 编码**：支持国际化域名（如 `例え.jp`）
- **下划线例外**：DNS SRV 记录和 Docker Compose 允许下划线
- **不可见字符编码**：防止 bidi 攻击和零宽字符欺骗
- **移除 fragment**：fragment 不发送到服务器，应该移除

---

## 源码解析

### 地址分类常量

```python
# IETF 协议分配地址段
_PROTOCOL_ASSIGNMENTS = ip_network("192.0.0.0/24")

# NAT64 well-known prefix
_NAT64_WELL_KNOWN = ip_network("64:ff9b::/96")

# 路由探测用的文档地址（RFC 5737, RFC 3849）
_PROBES: tuple[tuple[socket.AddressFamily, tuple[str, int]], ...] = (
    (socket.AF_INET, ("192.0.2.1", 9)),    # TEST-NET-1
    (socket.AF_INET6, ("2001:db8::1", 9)),  # Documentation prefix
)
```

### 不可见字符检测

```python
# Unicode 类别：格式字符、空格分隔符、行分隔符、段分隔符
_INVISIBLE_CATEGORIES = frozenset({"Cf", "Zs", "Zl", "Zp"})

def _encode_invisible(text: str) -> str:
    """Percent-encode invisible Unicode characters."""
    if text.isascii():
        return text
    
    return "".join(
        quote(c, safe="") if unicodedata.category(c) in _INVISIBLE_CATEGORIES else c
        for c in text
    )
```

### 同源检测

```python
def same_site(left: str, right: str) -> bool:
    """Permit only same scheme/port and an optional www prefix change."""
    
    a, b = urlsplit(left), urlsplit(right)
    
    return (
        a.scheme,
        a.port,
        (a.hostname or "").removeprefix("www."),
    ) == (
        b.scheme,
        b.port,
        (b.hostname or "").removeprefix("www."),
    )
```

---

## 实际使用场景

### 场景 1: Web 工具的 SSRF 防护

```python
from chrys.foundation.net.address_class import is_restricted_address, literal_address
from chrys.foundation.net.route_probe import is_local_target

async def safe_web_fetch(url: str) -> str:
    """Fetch a URL with SSRF protection."""
    
    parsed = urlsplit(url)
    host = parsed.hostname
    
    if not host:
        raise ValueError("URL must have a hostname")
    
    # 检查是否是 IP 字面量
    if literal_address(host):
        addr = literal_address(host)
        if is_restricted_address(addr):
            raise ValueError(f"Access to restricted address {addr} is forbidden")
    
    # 检查是否是本地目标
    if is_local_target(host):
        raise ValueError(f"Access to local target {host} is forbidden")
    
    # 安全的 URL，可以访问
    async with httpx.AsyncClient() as client:
        response = await client.get(url, timeout=10.0)
        return response.text
```

### 场景 2: 配置中的 URL 验证

```python
from chrys.foundation.net.url import normalize_url

def validate_mcp_endpoint(user_input: str) -> str:
    """Validate and normalize an MCP server endpoint from user config."""
    
    try:
        normalized = normalize_url(user_input, endpoint=True)
        return normalized
    except ValueError as e:
        raise ConfigError(f"Invalid MCP endpoint: {e}")

# 用户配置
config = {
    "mcp_server": "http://Example.COM:8080/api/",
}

# 验证和规范化
config["mcp_server"] = validate_mcp_endpoint(config["mcp_server"])
# → "http://example.com:8080/api"
```

### 场景 3: 离线状态检测

```python
from chrys.foundation.net.route_probe import default_route_available

class NetworkStatusMonitor:
    def __init__(self):
        self._is_offline = False
    
    async def check_connectivity(self) -> bool:
        """Check if we have network connectivity."""
        
        route_available = default_route_available()
        
        if route_available is False:
            if not self._is_offline:
                logger.warning("Network appears to be offline")
                self._is_offline = True
                await event_bus.publish(NetworkOfflineEvent())
            return False
        
        if route_available is True:
            if self._is_offline:
                logger.info("Network connection restored")
                self._is_offline = False
                await event_bus.publish(NetworkOnlineEvent())
            return True
        
        # None: 无法判断，保持当前状态
        return not self._is_offline
```

---

## 和其他模块的关系

```
net 被以下模块使用：
├── service/tools/builtins/web/（Web 工具的 SSRF 防护）
├── foundation/config/web_values.py（配置中的 URL 验证）
├── foundation/errors/network.py（网络错误分类时参考路由状态）
└── service/mcp/_http_transport.py（MCP HTTP 传输的地址验证）
```

---

## 如果你要自己实现

### 最小版本（30 行）

```python
from ipaddress import ip_address
from urllib.parse import urlsplit

def simple_is_private(host: str) -> bool:
    """Simple check if a host is private/restricted."""
    try:
        addr = ip_address(host)
        return not addr.is_global
    except ValueError:
        # 不是 IP 地址，假设是域名
        return host in ("localhost", "127.0.0.1", "::1")

def simple_validate_url(url: str) -> str:
    """Simple URL validation."""
    parts = urlsplit(url)
    
    if parts.scheme not in ("http", "https"):
        raise ValueError("Only HTTP(S) URLs are allowed")
    
    if not parts.hostname:
        raise ValueError("URL must have a hostname")
    
    if parts.username:
        raise ValueError("URL must not contain credentials")
    
    return url
```

### 进阶版本

加完整的地址分类、路由探测、IDNA 编码。参考 Chrys 的实现。

---

## 下一步

- **上一篇**: F10 io — I/O 结果标准化
- **下一篇**: F12 observability — 可观测性集成
