# F14: vendor — 第三方二进制打包

> **模块路径**: `src/chrys/foundation/vendor/`
> **核心文件**: `__init__.py` (唯一文件)
> **一句话**: 在构建时将 ripgrep (rg) 二进制文件打包到 Chrys 中，确保用户无需单独安装。

---

## 这是什么？

一个**二进制打包系统**。在构建 wheel 时，将 ripgrep（一个高性能的代码搜索工具）的可执行文件嵌入到 Chrys 包中，按平台提供对应的二进制。

```python
from chrys.foundation.vendor import find_rg

# 自动找到对应平台的 rg 二进制
rg_path = find_rg()
# → "/path/to/chrys/foundation/vendor/ripgrep/rg" (Linux/macOS)
# → "C:\...\chrys\foundation\vendor\ripgrep\rg.exe" (Windows)

# 使用它进行代码搜索
result = subprocess.run([rg_path, "--json", "pattern", "."], capture_output=True)
```

---

## 为什么需要？

### 如果不做会怎样？

**问题 1: 用户需要手动安装 ripgrep**

```bash
# ❌ 要求用户自己安装 rg
# macOS
brew install ripgrep

# Ubuntu/Debian
sudo apt-get install ripgrep

# Windows
scoop install ripgrep

# 很多用户不知道 ripgrep 是什么，或者没有权限安装
```

**问题 2: 版本不一致导致行为差异**

```bash
# ❌ 不同用户的 rg 版本不同
user1$ rg --version
ripgrep 13.0.0

user2$ rg --version
ripgrep 14.1.0

# 某些功能在新版本才有，导致用户体验不一致
```

**问题 3: PATH 配置问题**

```bash
# ❌ 用户安装了 rg 但不在 PATH 中
which rg
# → 找不到

# Agent 无法使用搜索功能
```

### 用了 vendor 之后

```python
# ✅ 自动打包，开箱即用
rg_path = find_rg()

if rg_path is None:
    # 回退到系统 PATH
    rg_path = shutil.which("rg")
    
if rg_path is None:
    raise RuntimeError("ripgrep not found")

# 保证所有用户使用相同版本的 rg
# 不需要用户手动安装
# 不依赖 PATH 配置
```

---

## 核心设计：三个关键决策

### 决策 1: 平台特定的二进制命名

```python
# Platform triple → rg binary filename inside foundation/vendor/ripgrep/.
_TRIPLE_MAP: dict[tuple[str, str], tuple[str, str]] = {
    # (system, machine) → (triple-suffixed name, generic name)
    ("Darwin", "arm64"): ("rg-aarch64-apple-darwin", "rg"),
    ("Darwin", "x86_64"): ("rg-x86_64-apple-darwin", "rg"),
    ("Linux", "x86_64"): ("rg-x86_64-unknown-linux-musl", "rg"),
    ("Linux", "aarch64"): ("rg-aarch64-unknown-linux-gnu", "rg"),
    ("Windows", "AMD64"): ("rg-x86_64-pc-windows-msvc.exe", "rg.exe"),
}
```

**为什么？**

- **多平台 wheel**：CI/CD 可以构建包含所有平台二进制的通用 wheel
- **优先级查找**：先找平台特定的名称（如 `rg-aarch64-apple-darwin`），再找通用名称（如 `rg`）
- **回退到 PATH**：如果打包的二进制不存在，使用系统的 rg

---

### 决策 2: 三级回退策略

```python
def find_rg() -> str | None:
    """Locate the bundled ``rg`` binary, or return *None* if not found."""
    
    key = (_platform.system(), _platform.machine())
    names = _TRIPLE_MAP.get(key)
    
    if names:
        # 1. 尝试平台特定的名称（多平台 wheel）
        for name in names:
            candidate = VENDOR_RIPGREP / name
            if candidate.is_file():
                return str(candidate)
    
    # 2. 回退到系统 PATH
    which = shutil.which("rg")
    if which:
        return which
    
    # 3. 都没找到
    return None
```

**为什么？**

- **优先使用打包版本**：保证版本一致性和可预测性
- **允许系统覆盖**：用户可以通过 PATH 强制使用自己的 rg
- **优雅降级**：如果都没有，返回 None 让调用者决定如何处理

---

### 决策 3: 构建时打包，运行时只读

```python
VENDOR_RIPGREP = Path(__file__).resolve().parent / "ripgrep"

# 目录结构：
# src/chrys/foundation/vendor/
# ├── __init__.py
# └── ripgrep/
#     ├── rg-aarch64-apple-darwin
#     ├── rg-x86_64-apple-darwin
#     ├── rg-x86_64-unknown-linux-musl
#     ├── rg-aarch64-unknown-linux-gnu
#     └── rg-x86_64-pc-windows-msvc.exe
```

**为什么？**

- **简单可靠**：直接读取文件，不需要解压或动态加载
- **跨平台兼容**：二进制文件在各个平台上都是原生格式
- **易于调试**：可以直接运行打包的 rg 测试功能

---

## 源码解析

### find_rg() 主函数

```python
def find_rg() -> str | None:
    """Locate the bundled ``rg`` binary, or return *None* if not found.
    
    Checks the platform-specific name first (CD multi-platform wheel), then
    the generic name (local single-platform build), then ``PATH``.
    """
    
    key = (_platform.system(), _platform.machine())
    names = _TRIPLE_MAP.get(key)
    
    if names:
        for name in names:
            candidate = VENDOR_RIPGREP / name
            if candidate.is_file():
                return str(candidate)
    
    # Fallback: system PATH
    which = shutil.which("rg")
    if which:
        return which
    
    return None
```

### 使用示例

```python
# service/tools/builtins/search.py

import subprocess
from chrys.foundation.vendor import find_rg

class SearchTool:
    async def execute(self, pattern: str, path: str) -> str:
        """Search for a pattern using ripgrep."""
        
        rg_path = find_rg()
        if rg_path is None:
            return "Error: ripgrep not found. Please install ripgrep."
        
        try:
            result = subprocess.run(
                [rg_path, "--json", pattern, path],
                capture_output=True,
                text=True,
                timeout=30,
            )
            
            if result.returncode == 0:
                return self._format_results(result.stdout)
            elif result.returncode == 1:
                return "No matches found."
            else:
                return f"Error: {result.stderr}"
        
        except subprocess.TimeoutExpired:
            return "Error: search timed out."
        except Exception as e:
            return f"Error: {e}"
```

---

## 构建流程

### 本地构建（单平台）

```bash
# 下载对应平台的 ripgrep
./scripts/fetch_rg.sh  # Linux x86_64
# 或
./scripts/fetch_rg.ps1  # Windows AMD64

# 二进制文件被放到 src/chrys/foundation/vendor/ripgrep/rg
# 或 src/chrys/foundation/vendor/ripgrep/rg.exe

# 构建 wheel
uv build
```

### CI/CD 构建（多平台）

```yaml
# .github/workflows/build.yml

jobs:
  build-multiplatform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # 下载所有平台的 ripgrep
      - name: Download all platform binaries
        run: ./scripts/fetch_rg.sh --all
      
      # 生成的目录结构：
      # src/chrys/foundation/vendor/ripgrep/
      # ├── rg-aarch64-apple-darwin
      # ├── rg-x86_64-apple-darwin
      # ├── rg-x86_64-unknown-linux-musl
      # ├── rg-aarch64-unknown-linux-gnu
      # └── rg-x86_64-pc-windows-msvc.exe
      
      - name: Build universal wheel
        run: uv build --wheel
```

---

## 实际使用场景

### 场景 1: 代码搜索工具

```python
from chrys.foundation.vendor import find_rg

async def search_code(pattern: str, directory: str) -> list[dict]:
    """Search for code patterns using ripgrep."""
    
    rg_path = find_rg()
    if not rg_path:
        raise RuntimeError("ripgrep not available")
    
    cmd = [
        rg_path,
        "--json",           # JSON 输出格式
        "--line-number",    # 显示行号
        "--context", "2",   # 上下文 2 行
        pattern,
        directory,
    ]
    
    result = subprocess.run(cmd, capture_output=True, text=True)
    
    return [json.loads(line) for line in result.stdout.splitlines()]
```

### 场景 2: 文件类型过滤

```python
async def search_python_files(function_name: str, project_dir: str) -> str:
    """Search for a function definition in Python files only."""
    
    rg_path = find_rg()
    
    cmd = [
        rg_path,
        "--type", "python",  # 只搜索 .py 文件
        "--word-regexp",     # 全词匹配
        f"def {function_name}",
        project_dir,
    ]
    
    result = subprocess.run(cmd, capture_output=True, text=True)
    return result.stdout
```

### 场景 3: 正则表达式搜索

```python
async def find_todo_comments(directory: str) -> list[str]:
    """Find all TODO comments in the codebase."""
    
    rg_path = find_rg()
    
    cmd = [
        rg_path,
        "--only-matching",  # 只输出匹配的部分
        r"TODO:?\s+.*",     # 正则表达式
        directory,
    ]
    
    result = subprocess.run(cmd, capture_output=True, text=True)
    return result.stdout.splitlines()
```

---

## 和其他模块的关系

```
vendor 被以下模块使用：
├── service/tools/builtins/search.py（代码搜索工具）
├── service/tools/builtins/grep.py（grep 工具）
└── orchestration/startup.py（启动时验证 rg 可用性）
```

---

## 如果你要自己实现

### 最小版本（硬编码路径）

```python
import platform
import shutil
from pathlib import Path

VENDOR_DIR = Path(__file__).parent / "binaries"

def find_tool(tool_name: str) -> str | None:
    """Find a vendored tool binary."""
    
    # 平台特定名称
    system = platform.system().lower()
    machine = platform.machine().lower()
    
    candidates = [
        f"{tool_name}-{machine}-{system}",
        f"{tool_name}.exe" if system == "windows" else tool_name,
    ]
    
    for name in candidates:
        path = VENDOR_DIR / name
        if path.is_file():
            return str(path)
    
    # 回退到 PATH
    return shutil.which(tool_name)
```

### 进阶版本

加多平台支持、构建脚本、版本检查。参考 Chrys 的实现和 `scripts/fetch_rg.sh`。

---

## 🎉 Foundation 层完成！

恭喜你！现在你已经了解了 Chrys Foundation 层的所有 14 个模块：

| 序号 | 模块 | 职责 | 状态 |
|------|------|------|------|
| F01 | events | 事件总线 | ✅ |
| F02 | config | 配置系统 | ✅ |
| F03 | platform | 跨平台抽象 | ✅ |
| F04 | models | 核心数据模型 | ✅ |
| F05 | errors | 错误分类 | ✅ |
| F06 | i18n | 国际化 | ✅ |
| F07 | trajectory | 执行轨迹 | ✅ |
| F08 | text | 文本处理 | ✅ |
| F09 | util | 通用工具 | ✅ |
| F10 | io | I/O 标准化 | ✅ |
| F11 | net | 网络辅助 | ✅ |
| F12 | observability | 可观测性 | ✅ |
| F13 | patches | 第三方补丁 | ✅ |
| F14 | vendor | 二进制打包 | ✅ |

**下一步**：进入 Kernel 层（K01-K18），这是 Chrys 最核心的创新所在！

---

## 下一步

- **上一篇**: F13 patches — 第三方库补丁系统
- **下一篇**: K01 agent — Agent 入口点（Kernel 层开始）
