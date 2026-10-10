# F03: platform — 跨平台抽象

> **模块路径**: `src/chrys/foundation/platform/`
> **核心文件**: `__init__.py`（~560 行）, `process.py`, `files.py`, `runtime_paths.py`
> **一句话**: 让同一套代码能在 Windows/Mac/Linux 上跑。

---

## 这是什么？

一个**跨平台检测层**。把操作系统、架构、Shell、文件系统等差异封装成统一接口。

```python
from chrys.foundation.platform import get_platform

platform = get_platform()

if platform.is_windows:
    # Windows 特定逻辑
    pass
elif platform.is_macos:
    # macOS 特定逻辑
    pass
else:
    # Linux 特定逻辑
    pass
```

---

## 为什么需要？

### 如果不做会怎样？

到处是 `if sys.platform == "win32": ...`：

```python
# ❌ 散落在各处的平台判断
import sys

def copy_to_clipboard(text):
    if sys.platform == "win32":
        os.system(f"echo {text} | clip")
    elif sys.platform == "darwin":
        os.system(f"echo '{text}' | pbcopy")
    else:
        os.system(f"echo '{text}' | xclip -selection clipboard")

def get_config_dir():
    if sys.platform == "win32":
        return Path(os.environ["APPDATA"]) / "chrys"
    else:
        return Path.home() / ".chrys"

def detect_shell():
    if sys.platform == "win32":
        # Windows 有 PowerShell 7、PowerShell 5、cmd、Git Bash...
        # 还要考虑安装路径、PATH 优先级...
        pass
    else:
        return os.environ.get("SHELL", "/bin/bash")
```

**问题**：
1. 平台判断逻辑重复出现在多个地方
2. 容易遗漏某些边界情况（比如 Windows 的 Git Bash）
3. 测试困难（要在三个平台上都测一遍）
4. 新增平台特性时要改很多地方

### 用了 platform 模块之后

```python
# ✅ 统一的抽象层
from chrys.foundation.platform import get_platform, clipboard_copy

platform = get_platform()

# 所有平台信息一次性获取
print(platform.os_name)         # "windows" / "macos" / "linux"
print(platform.arch)            # "arm64" / "amd64" / "x86"
print(platform.shell.name)      # "pwsh" / "bash" / "zsh"
print(platform.config_dir)      # 正确的配置目录

# 跨平台操作直接用
clipboard_copy("Hello")  # 内部自动处理平台差异
```

---

## 核心设计：三个关键决策

### 决策 1: 单例模式 + 缓存

```python
@functools.cache
def get_platform() -> PlatformInfo:
    """Cached platform singleton — safe to call from any module."""
    return detect_platform()
```

**为什么？**
- 平台信息在程序运行期间不会变
- 检测可能耗时（要启动子进程查 shell 版本）
- 多个模块都要用，不应该重复检测

### 决策 2: 结构化数据，不是零散函数

```python
@dataclass(frozen=True)
class PlatformInfo:
    os_name: str              # "windows" / "macos" / "linux"
    os_version: str           # "15.4.1" / "11" / "22.04"
    arch: str                 # "arm64" / "amd64" / "x86"
    shell: ShellInfo          # 默认 shell 信息
    config_dir: Path          # 配置目录
    data_dir: Path            # 数据目录
    extra_shells: tuple[ShellInfo, ...]  # 额外发现的 shell（如 Git Bash）
    
    @property
    def is_windows(self) -> bool:
        return self.os_name == "windows"
    
    @property
    def is_arm64(self) -> bool:
        return self.arch == "arm64"
```

**好处**：
- 一次调用获取所有信息
- IDE 自动补全友好
- 类型检查器能验证

### 决策 3: Shell 检测要多层 fallback

```python
def _detect_unix_shell() -> ShellInfo:
    """Detect the default shell on macOS/Linux."""
    
    # 1. 优先用 $SHELL 环境变量
    shell_env = os.environ.get("SHELL")
    if shell_env and _is_executable_file(shell_env):
        shell_path = shell_env
    
    # 2. 否则用 shutil.which 搜索 PATH
    else:
        for name in ("bash", "sh", "zsh", "dash", "ash"):
            resolved = shutil.which(name)
            if resolved and _is_executable_file(resolved):
                shell_path = resolved
                break
        
        # 3. 最后尝试常见绝对路径
        else:
            for directory in ("/bin", "/usr/bin", "/usr/local/bin"):
                for name in ("bash", "zsh", "sh"):
                    candidate = f"{directory}/{name}"
                    if _is_executable_file(candidate):
                        shell_path = candidate
                        break
    
    # 4. 获取版本号
    version = _get_shell_version(shell_path, name)
    
    return ShellInfo(name=name, path=shell_path, args=["-c"], version=version)
```

**为什么要这么多层？**
- `$SHELL` 可能是过时的（用户换了 shell 但没更新环境变量）
- `shutil.which` 在某些发行版上找不到（NixOS、Alpine）
- 硬编码路径在某些系统上不存在

---

## 源码解析

### PlatformInfo 数据结构

```python
@dataclass(frozen=True)
class ShellInfo:
    """Detected shell information."""
    
    name: str       # "bash" / "zsh" / "pwsh" / "powershell" / "cmd"
    path: str       # "/bin/bash" / "C:\\Windows\\System32\\cmd.exe"
    args: list[str] # ["-c"] for bash, ["-NoProfile", "-Command"] for pwsh
    version: str    # "5.2.37" / "7.5.0"

@dataclass(frozen=True)
class PlatformInfo:
    """Detected platform information."""
    
    os_name: str        # "windows" / "macos" / "linux"
    os_version: str     # "15.4.1" (macOS), "11" (Win), "22.04" (Ubuntu)
    arch: str           # "arm64" / "amd64" / "x86"
    shell: ShellInfo
    config_dir: Path    # ~/.chrys or %APPDATA%/chrys
    data_dir: Path      # same as config_dir for now
    extra_shells: tuple[ShellInfo, ...]  # additional shells (e.g. Git Bash on Windows)
    
    # --- OS convenience properties ---
    
    @property
    def is_windows(self) -> bool:
        return self.os_name == "windows"
    
    @property
    def is_macos(self) -> bool:
        return self.os_name == "macos"
    
    @property
    def is_linux(self) -> bool:
        return self.os_name == "linux"
    
    # --- Architecture convenience properties ---
    
    @property
    def is_arm64(self) -> bool:
        return self.arch == "arm64"
    
    @property
    def is_amd64(self) -> bool:
        return self.arch == "amd64"
```

### detect_platform() 主函数

```python
def detect_platform() -> PlatformInfo:
    """Detect the current platform and return unified info."""
    
    system = platform.system().lower()
    arch = _detect_arch()
    os_version = _detect_os_version(system)
    
    if system == "windows":
        os_name = "windows"
        config_dir = Path(os.environ.get("APPDATA", Path.home() / "AppData" / "Roaming")) / "chrys"
        
        # Windows 要检测默认 shell 和 Git Bash
        with ThreadPoolExecutor(max_workers=2) as pool:
            shell_future = pool.submit(_detect_windows_shell)
            git_bash_future = pool.submit(_detect_git_bash)
            shell = shell_future.result()
            git_bash = git_bash_future.result()
        
        extra_shells = (git_bash,) if git_bash else ()
    
    elif system == "darwin":
        os_name = "macos"
        shell = _detect_unix_shell()
        config_dir = Path.home() / ".chrys"
        extra_shells = ()
    
    else:  # linux
        os_name = "linux"
        shell = _detect_unix_shell()
        config_dir = Path.home() / ".chrys"
        extra_shells = ()
    
    return PlatformInfo(
        os_name=os_name,
        arch=arch,
        shell=shell,
        config_dir=config_dir,
        data_dir=config_dir,
        extra_shells=extra_shells,
        os_version=os_version,
    )
```

### Windows Shell 检测

```python
def _detect_windows_shell() -> ShellInfo:
    """Detect the default shell on Windows.
    
    Preference order:
    1. pwsh (PowerShell 7+) — newest, best UX
    2. powershell (Windows PowerShell 5.1) — built-in
    3. cmd — fallback
    """
    
    # 1. PowerShell 7+ (cross-platform, installed separately)
    pwsh = shutil.which("pwsh")
    if pwsh:
        version = _get_shell_version(pwsh, "pwsh")
        return ShellInfo(name="pwsh", path=pwsh, args=["-NoProfile", "-Command"], version=version)
    
    # 2. Windows PowerShell 5.1 (built-in)
    ps5 = shutil.which("powershell")
    if ps5:
        version = _get_shell_version(ps5, "powershell")
        return ShellInfo(name="powershell", path=ps5, args=["-NoProfile", "-Command"], version=version)
    
    # 3. Fallback to cmd
    cmd = os.environ.get("COMSPEC")
    if not cmd:
        system_root = os.environ.get("SYSTEMROOT", r"C:\Windows")
        cmd = os.path.join(system_root, "System32", "cmd.exe")
    
    return ShellInfo(name="cmd", path=cmd, args=["/c"])


def _detect_git_bash() -> ShellInfo | None:
    """Detect Git Bash on Windows."""
    
    candidates = []
    
    # Known Git for Windows install paths
    for base in windows_program_files_dirs():
        path = os.path.join(base, "Git", "bin", "bash.exe")
        if os.path.isfile(path):
            candidates.append(path)
    
    # Check PATH — only accept if path looks like Git Bash (not WSL)
    which_bash = shutil.which("bash")
    if which_bash and "git" in which_bash.lower():
        candidates.append(which_bash)
    
    if not candidates:
        return None
    
    bash_path = candidates[0]
    version = _get_shell_version(bash_path, "bash")
    
    return ShellInfo(name="git_bash", path=bash_path, args=["-c"], version=version)
```

### 跨平台剪贴板操作

```python
def clipboard_copy(text: str) -> None:
    """Copy text to system clipboard (cross-platform)."""
    
    info = get_platform()
    
    if info.is_macos:
        subprocess.run(["pbcopy"], input=text.encode(), check=True, timeout=2)
    
    elif info.is_windows:
        # Direct Win32 SetClipboardData(CF_UNICODETEXT, ...)
        # 不用 clip.exe，因为它会在开头加 UTF-16 BOM
        _clipboard_copy_windows(text)
    
    else:  # linux
        for cmd in (["xclip", "-selection", "clipboard"], ["xsel", "--clipboard", "--input"]):
            try:
                subprocess.run(cmd, input=text.encode(), check=True, timeout=2)
                return
            except FileNotFoundError:
                continue


def _clipboard_copy_windows(text: str) -> None:
    """Direct Win32 clipboard API — no subprocess, no codepage ambiguity."""
    
    api = _win32_clipboard_api()
    user32, kernel32 = api.user32, api.kernel32
    
    # CF_UNICODETEXT requires NUL-terminated UTF-16-LE
    data = text.encode("utf-16-le") + b"\x00\x00"
    
    h = kernel32.GlobalAlloc(GMEM_MOVEABLE, len(data))
    if not h:
        raise _win32_error()
    
    try:
        ptr = kernel32.GlobalLock(h)
        if not ptr:
            raise _win32_error()
        
        ctypes.memmove(ptr, data, len(data))
        kernel32.GlobalUnlock(h)
        
        _open_clipboard_with_retry(api)
        try:
            user32.EmptyClipboard()
            if not user32.SetClipboardData(CF_UNICODETEXT, h):
                raise _win32_error()
            h = None  # ownership transferred to the clipboard
        finally:
            user32.CloseClipboard()
    finally:
        if h:
            kernel32.GlobalFree(h)
```

**为什么不直接用 `clip.exe`？**
- `clip.exe` 会在开头加 UTF-16 BOM（U+FEFF）
- 粘贴时会多出一个不可见字符
- 直接调 Win32 API 更可控

---

## 实际使用场景

### 场景 1: 确定配置目录

```python
platform = get_platform()

# Windows: C:\Users\Alice\AppData\Roaming\chrys
# macOS:   /Users/alice/.chrys
# Linux:   /home/alice/.chrys

config_dir = platform.config_dir
```

### 场景 2: 执行 Shell 命令

```python
platform = get_platform()

# 根据 shell 类型选择正确的参数
if platform.shell.name in ("pwsh", "powershell"):
    cmd_args = platform.shell.args + ["Get-Process"]
else:
    cmd_args = platform.shell.args + ["ps aux"]

subprocess.run([platform.shell.path] + cmd_args)
```

### 场景 3: 显示平台信息

```python
platform = get_platform()

# 用于 UI 头部或日志
print(platform.display_label)  # "macOS-arm64" / "Windows-amd64" / "Linux-x86_64"
print(f"Shell: {platform.shell.name} {platform.shell.version}")
```

---

## 和其他模块的关系

```
platform 被以下模块使用：
├── config（确定 config_dir 的位置）
├── process（跨平台进程管理）
├── files（跨平台文件系统操作）
├── TUI（显示平台信息、处理快捷键差异）
└── Tool System（执行 shell 命令时选择正确的 shell）
```

---

## 如果你要自己实现

### 最小版本（20 行）

```python
import sys
from pathlib import Path
from dataclasses import dataclass

@dataclass
class SimplePlatform:
    os_name: str
    config_dir: Path
    
    @property
    def is_windows(self):
        return self.os_name == "windows"

def detect_simple_platform() -> SimplePlatform:
    if sys.platform == "win32":
        os_name = "windows"
        config_dir = Path(os.environ["APPDATA"]) / "myapp"
    else:
        os_name = "unix"
        config_dir = Path.home() / ".myapp"
    
    return SimplePlatform(os_name=os_name, config_dir=config_dir)
```

### 进阶版本

加架构检测、shell 检测、版本号获取。参考 Chrys 的 `detect_platform()`。

---

## 下一步

- **上一篇**: F02 config — 配置系统
- **下一篇**: F04 models — 核心数据模型
