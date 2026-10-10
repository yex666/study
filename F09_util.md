# F09: util — 通用工具函数集

> **模块路径**: `src/chrys/foundation/util/`
> **核心文件**: `once_close.py`, `lock.py`, `session_ids.py`, `chrys_headers.py`, `env_templates.py`, `httpx_helpers.py`, `time.py`, `async_tasks.py`, `filenames.py`, `header_charset.py`, `shell_tokens.py`, `sub_agent_context.py`, `unicode_scalars.py`
> **一句话**: 提供跨平台锁、会话 ID 处理、HTTP 辅助函数、时间工具等通用功能。

---

## 这是什么？

一个**零依赖的工具箱**。包含多个独立的小工具函数和类，用于解决各种常见问题：

- **once_close.py** — 取消安全的单次关闭原语
- **lock.py** — 跨平台文件锁（进程间互斥）
- **session_ids.py** — 会话 ID 生成和格式化
- **chrys_headers.py** — HTTP 请求头常量
- **env_templates.py** — 环境变量模板解析
- **httpx_helpers.py** — httpx 客户端辅助函数
- **time.py** — 时间格式化和解析
- **async_tasks.py** — 异步任务管理
- **filenames.py** — 文件名安全处理
- **header_charset.py** — HTTP header 字符集验证
- **shell_tokens.py** — Shell token 解析
- **sub_agent_context.py** — 子代理上下文传递
- **unicode_scalars.py** — Unicode 标量值处理

---

## 为什么需要？

### 如果不做会怎样？

**问题 1: 资源清理的竞态条件**

```python
# ❌ 不安全的关闭
class Connection:
    async def close(self):
        await self._socket.close()
        await self._db.close()

# 如果多个地方同时调用 close()
await asyncio.gather(conn.close(), conn.close())
# → 可能导致 double-close 错误

# 如果一个等待者被取消
try:
    await conn.close()
except asyncio.CancelledError:
    pass
# → 关闭任务可能被中断，资源未释放
```

**问题 2: 多进程访问同一文件**

```python
# ❌ 没有锁保护
config_file = Path("~/.chrys/settings.yaml").expanduser()

data = config_file.read_text()
data = update_config(data)
config_file.write_text(data)

# 如果两个进程同时执行这段代码
# → 可能丢失更新或写入损坏的数据
```

**问题 3: 会话 ID 格式混乱**

```python
# ❌ 各处使用不同的 ID 格式
session_id = uuid4().hex  # 32 字符
folder_name = session_id[:8]  # 8 字符
display_id = f"session-{session_id[:6]}"  # 带前缀

# → 难以追踪，容易混淆
```

### 用了 util 模块之后

```python
# ✅ OnceClose：取消安全的单次关闭
from chrys.foundation.util.once_close import OnceClose

class Connection:
    def __init__(self):
        self._close = OnceClose(self._do_close)
    
    async def _do_close(self):
        await self._socket.close()
        await self._db.close()
    
    async def close(self):
        await self._close()  # 多次调用安全，取消也安全

# ✅ FileLock：跨平台文件锁
from chrys.foundation.util.lock import FileLock

with FileLock(config_file.with_suffix(".lock")):
    data = config_file.read_text()
    data = update_config(data)
    config_file.write_text(data)

# ✅ 统一的会话 ID 处理
from chrys.foundation.util.session_ids import session_short_id

session_id = "4201eebc-ca45-4328-8882-272f3d7c41cb"
short_id = session_short_id(session_id)  # → "4201eebcca45" (12 字符)
```

---

## 核心设计：四个关键决策

### 决策 1: OnceClose — 取消安全的单次关闭

```python
class OnceClose:
    """Run *close* at most once; concurrent and later callers share its task."""
    
    def __init__(self, close: Callable[[], Awaitable[None]]) -> None:
        self._close = close
        self._task: asyncio.Task[None] | None = None
    
    @property
    def started(self) -> bool:
        return self._task is not None
    
    async def _run(self) -> None:
        await self._close()
    
    async def __call__(self) -> None:
        if self._task is None:
            # 第一次调用：创建关闭任务
            self._task = asyncio.create_task(self._run())
        
        # 所有调用者等待同一个任务
        await finish_close(self._task)


async def finish_close(task: asyncio.Task[None]) -> None:
    """Drain an owned cleanup task even after repeated waiter cancellation."""
    cancelled: asyncio.CancelledError | None = None
    
    while not task.done():
        try:
            # shield 防止取消传播到关闭任务
            await asyncio.shield(task)
        except asyncio.CancelledError as exc:
            # 记录取消，但不中断关闭任务
            cancelled = exc
    
    # 确保关闭任务的异常被传播
    task.result()
    
    # 如果当前调用者被取消，重新抛出
    if cancelled is not None:
        raise cancelled
```

**为什么？**

- **单次执行**：无论调用多少次，关闭逻辑只执行一次
- **取消安全**：等待者被取消不会中断关闭任务
- **资源共享**：所有调用者等待同一个任务，避免重复工作

**使用场景**：
- HTTP 客户端关闭
- 数据库连接池关闭
- MCP server 连接关闭

---

### 决策 2: FileLock — 跨平台文件锁

```python
class FileLock:
    """Context-managed exclusive lock on a sidecar ``.lock`` file."""
    
    _RETRY_INTERVAL = 0.05  # 50ms 轮询间隔
    
    def __init__(self, path: Path | str, timeout: float | None = None) -> None:
        self._path = Path(path)
        self._timeout = timeout
        self._fp: IO[bytes] | None = None
        self._proc_lock: threading.Lock | None = None
    
    def __enter__(self) -> FileLock:
        self.acquire()
        return self
    
    def __exit__(self, exc_type, exc, tb) -> None:
        self.release()
    
    def acquire(self) -> None:
        """Acquire the lock, blocking until success or timeout."""
        
        # 进程内锁（避免 Windows 上的线程饥饿）
        self._proc_lock = _get_path_lock(self._path)
        self._proc_lock.acquire()
        
        try:
            # 打开或创建锁文件
            self._fp = open(self._path, "a+b")
            
            deadline = None if self._timeout is None else time.monotonic() + self._timeout
            
            while True:
                try:
                    if _IS_WINDOWS:
                        # Windows: msvcrt.locking (mandatory lock)
                        msvcrt.locking(self._fp.fileno(), msvcrt.LK_NBLCK, 1)
                    else:
                        # POSIX: fcntl.flock (advisory lock)
                        fcntl.flock(self._fp.fileno(), fcntl.LOCK_EX | fcntl.LOCK_NB)
                    
                    # 成功获取锁
                    return
                
                except (IOError, OSError):
                    # 锁被占用，检查超时
                    if deadline is not None and time.monotonic() >= deadline:
                        self.release()
                        raise TimeoutError(f"Failed to acquire lock on {self._path}")
                    
                    # 等待后重试
                    time.sleep(self._RETRY_INTERVAL)
        
        except Exception:
            self._proc_lock.release()
            self._proc_lock = None
            raise
    
    def release(self) -> None:
        """Release the lock."""
        if self._fp is not None:
            try:
                if _IS_WINDOWS:
                    try:
                        msvcrt.locking(self._fp.fileno(), msvcrt.LK_UNLCK, 1)
                    except IOError:
                        pass
                else:
                    fcntl.flock(self._fp.fileno(), fcntl.LOCK_UN)
            finally:
                self._fp.close()
                self._fp = None
        
        if self._proc_lock is not None:
            self._proc_lock.release()
            self._proc_lock = None
```

**为什么？**

- **跨平台**：POSIX 用 `fcntl.flock`，Windows 用 `msvcrt.locking`
- **阻塞式**：默认阻塞等待，支持可选超时
- **进程内公平**：先获取线程锁，减少 Windows 上的竞争
- **自动清理**：进程崩溃时 OS 自动释放锁

**使用场景**：
- 配置文件读写
- 会话元数据更新
- Mutation snapshot 保存

---

### 决策 3: 统一的会话 ID 处理

```python
SESSION_SHORT_ID_LEN = 12

def session_short_id(session_id: str) -> str:
    """Return the short projection of *session_id* for display / folder.
    
    The canonical session id is the RFC 4122 UUID string with dashes
    (36 chars, e.g. "4201eebc-ca45-4328-8882-272f3d7c41cb").
    The short form strips dashes then truncates to SESSION_SHORT_ID_LEN
    hex chars — yielding a clean prefix of the dashless form
    (e.g. "4201eebcca45").
    """
    # 移除危险字符和 dashes
    safe_id = session_id.replace("/", "_").replace("\\", "_").replace("-", "")
    
    # 截取前 12 个字符
    return safe_id[:SESSION_SHORT_ID_LEN]
```

**为什么？**

- **统一格式**：所有地方都用相同的短 ID 规则
- **向后兼容**：旧的 12 字符 ID 切片后不变
- **安全字符**：替换 `/` 和 `\`，防止路径遍历
- **简洁显示**：12 字符足够唯一且易于阅读

**使用场景**：
- 会话文件夹名
- UI 显示的会话标识
- 日志中的会话标记

---

### 决策 4: 环境变量模板解析

```python
def resolve_env_template(value: str, *, location: str) -> str:
    """Resolve ``{{ENV_VAR}}`` templates in a string value.
    
    Args:
        value: String possibly containing ``{{VAR}}`` placeholders.
        location: Human-readable location for error messages.
    
    Returns:
        String with all ``{{VAR}}`` replaced by environment values.
    
    Raises:
        ValueError: If a referenced variable is not set.
    """
    
    def replacer(match: re.Match[str]) -> str:
        var_name = match.group(1)
        env_value = os.environ.get(var_name)
        
        if env_value is None:
            raise ValueError(
                f"{location}: environment variable ${{{var_name}}} is not set"
            )
        
        return env_value
    
    return re.sub(r"\{\{(\w+)\}\}", replacer, value)
```

**为什么？**

- **灵活配置**：用户可以在 YAML 中引用环境变量
- **明确错误**：缺失的环境变量会立即报错，附带位置信息
- **简单语法**：`{{VAR}}` 直观易懂

**使用场景**：
- MCP server 配置中的 API key
- Hook 命令中的路径
- Profile 中的自定义参数

---

## 源码解析

### HTTP Header 常量

```python
# chrys_headers.py

X_SESSION_ID_HEADER = "X-Session-Id"
X_PARENT_SESSION_ID_HEADER = "X-Parent-Session-Id"
CHRYS_SESSION_ID_HEADER = "Chrys-Session-Id"
CHRYS_PARENT_SESSION_ID_HEADER = "Chrys-Parent-Session-Id"
CHRYS_MODEL_ID_HEADER = "Chrys-Model-Id"
CHRYS_AGENT_ID_HEADER = "Chrys-Agent-Id"

def build_session_headers(
    session_id: str,
    *,
    parent_session_id: str | None = None,
    model_id: str | None = None,
    agent_id: str | None = None,
) -> dict[str, str]:
    """Build Chrys-specific HTTP headers for tracing."""
    
    headers = {
        X_SESSION_ID_HEADER: session_id,
        CHRYS_SESSION_ID_HEADER: session_id,
    }
    
    if parent_session_id:
        headers[X_PARENT_SESSION_ID_HEADER] = parent_session_id
        headers[CHRYS_PARENT_SESSION_ID_HEADER] = parent_session_id
    
    if model_id:
        headers[CHRYS_MODEL_ID_HEADER] = model_id
    
    if agent_id:
        headers[CHRYS_AGENT_ID_HEADER] = agent_id
    
    return headers
```

### 时间工具

```python
# time.py

def utc_now_rfc3339() -> str:
    """Return current UTC time in RFC 3339 format."""
    return datetime.now(timezone.utc).isoformat(timespec="milliseconds").replace("+00:00", "Z")

def parse_rfc3339(value: str) -> datetime:
    """Parse an RFC 3339 timestamp."""
    # 处理 Z 后缀
    if value.endswith("Z"):
        value = value[:-1] + "+00:00"
    
    return datetime.fromisoformat(value)

def monotonic_now_ns() -> int:
    """Return monotonic clock in nanoseconds (for duration measurement)."""
    return int(time.monotonic() * 1_000_000_000)
```

### 异步任务管理

```python
# async_tasks.py

async def wait_for_all(tasks: list[asyncio.Task], timeout: float | None = None) -> None:
    """Wait for all tasks, optionally with a timeout."""
    
    if timeout is None:
        await asyncio.gather(*tasks, return_exceptions=True)
    else:
        try:
            await asyncio.wait_for(
                asyncio.gather(*tasks, return_exceptions=True),
                timeout=timeout,
            )
        except asyncio.TimeoutError:
            # 取消未完成的任务
            for task in tasks:
                if not task.done():
                    task.cancel()
            
            # 等待取消完成
            await asyncio.gather(*tasks, return_exceptions=True)
            
            raise

def create_daemon_task(coro: Coroutine) -> asyncio.Task:
    """Create a daemon task that won't block interpreter shutdown."""
    
    task = asyncio.create_task(coro)
    
    # 设置为守护任务（Python 3.12+）
    if hasattr(task, "set_daemon"):
        task.set_daemon(True)
    
    return task
```

### 文件名安全处理

```python
# filenames.py

import re

_UNSAFE_CHARS_RE = re.compile(r'[<>:"/\\|?*\x00-\x1f]')

def sanitize_filename(name: str) -> str:
    """Sanitize a string for use as a filename.
    
    Replaces unsafe characters with underscores and strips trailing dots/spaces.
    """
    
    # 替换不安全字符
    safe = _UNSAFE_CHARS_RE.sub("_", name)
    
    # 移除尾部点和空格（Windows 不允许）
    safe = safe.rstrip(". ")
    
    # 限制长度
    if len(safe) > 255:
        safe = safe[:255]
    
    # 空名称回退
    if not safe:
        safe = "_"
    
    return safe
```

---

## 实际使用场景

### 场景 1: 安全的资源关闭

```python
from chrys.foundation.util.once_close import OnceClose

class LLMClient:
    def __init__(self):
        self._http_client = httpx.AsyncClient()
        self._close_once = OnceClose(self._do_close)
    
    async def _do_close(self):
        await self._http_client.aclose()
    
    async def close(self):
        """Safe to call multiple times, even concurrently."""
        await self._close_once()

# 使用
client = LLMClient()
try:
    response = await client.chat(messages)
finally:
    await client.close()  # 即使多次调用也安全
    await client.close()  # 第二次调用是 no-op
```

### 场景 2: 配置文件原子更新

```python
from pathlib import Path
from chrys.foundation.util.lock import FileLock

def update_config_atomic(config_path: Path, updates: dict) -> None:
    """Update config file with exclusive lock."""
    
    lock_path = config_path.with_suffix(".lock")
    
    with FileLock(lock_path, timeout=5.0):
        # 读取现有配置
        if config_path.exists():
            config = yaml.safe_load(config_path.read_text())
        else:
            config = {}
        
        # 应用更新
        config.update(updates)
        
        # 写入临时文件
        temp_path = config_path.with_suffix(".tmp")
        temp_path.write_text(yaml.dump(config))
        
        # 原子重命名
        temp_path.replace(config_path)
```

### 场景 3: MCP server 配置中的环境变量

```yaml
# ~/.chrys/mcp_servers.yaml
github:
  command: npx
  args: ["-y", "@modelcontextprotocol/server-github"]
  env:
    GITHUB_TOKEN: "{{GITHUB_PERSONAL_ACCESS_TOKEN}}"
    # ↑ 运行时解析为 os.environ["GITHUB_PERSONAL_ACCESS_TOKEN"]
```

```python
from chrys.foundation.util.env_templates import resolve_env_template_mapping

def load_mcp_config(raw_config: dict) -> dict:
    """Resolve environment templates in MCP config."""
    
    resolved = {}
    for key, value in raw_config.items():
        if isinstance(value, str):
            resolved[key] = resolve_env_template(value, location=f"MCP config {key}")
        elif isinstance(value, dict):
            resolved[key] = resolve_env_template_mapping(value, location=f"MCP config {key}")
        else:
            resolved[key] = value
    
    return resolved
```

---

## 和其他模块的关系

```
util 被以下模块使用：
├── service/llm/clients.py（OnceClose, chrys_headers）
├── service/session/persistence.py（FileLock）
├── orchestration/engine/state/（session_ids）
├── service/mcp/cache.py（env_templates）
├── foundation/trajectory/envelope.py（time）
└── app/tui/screens/sessions/（session_ids）
```

---

## 如果你要自己实现

### 最小版本（只实现最常用的）

```python
import asyncio
import sys
from pathlib import Path

if sys.platform == "win32":
    import msvcrt
else:
    import fcntl

# OnceClose
class SimpleOnceClose:
    def __init__(self, close_fn):
        self._close = close_fn
        self._task = None
    
    async def __call__(self):
        if self._task is None:
            self._task = asyncio.create_task(self._close())
        await asyncio.shield(self._task)

# FileLock (简化版)
class SimpleFileLock:
    def __init__(self, path: Path):
        self._path = path
        self._fp = None
    
    def __enter__(self):
        self._fp = open(self._path, "a+b")
        if sys.platform == "win32":
            msvcrt.locking(self._fp.fileno(), msvcrt.LK_LOCK, 1)
        else:
            fcntl.flock(self._fp.fileno(), fcntl.LOCK_EX)
        return self
    
    def __exit__(self, *exc):
        if sys.platform == "win32":
            msvcrt.locking(self._fp.fileno(), msvcrt.LK_UNLCK, 1)
        else:
            fcntl.flock(self._fp.fileno(), fcntl.LOCK_UN)
        self._fp.close()

# Session ID
def simple_session_short_id(session_id: str) -> str:
    return session_id.replace("-", "")[:12]
```

### 进阶版本

加完整的超时处理、错误恢复、平台特定的优化。参考 Chrys 的实现。

---

## 下一步

- **上一篇**: F08 text — 文本处理工具集
- **下一篇**: F10 io — I/O 操作辅助
