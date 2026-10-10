# F13: patches — 第三方库补丁系统

> **模块路径**: `src/chrys/foundation/patches/`
> **核心文件**: `patcher.py`, `textual_*.py` (20+ 个 Textual 补丁), `watchdog_*.py`
> **一句话**: 在启动时对第三方库（主要是 Textual TUI 框架）应用必要的补丁，修复 bug 和添加缺失功能。

---

## 这是什么？

一个**运行时补丁系统**。在程序启动时，自动修改已安装的第三方库源码文件，应用 Chrys 需要的修复和优化。

```python
# 启动时自动应用所有补丁
from chrys.foundation.patches import apply_all

apply_all()  # 修改 site-packages 中的 Textual 源码

# 之后导入 Textual 时，已经是修补后的版本
from textual.app import App
```

---

## 为什么需要？

### 如果不做会怎样？

**问题 1: Textual 的性能问题**

```python
# ❌ Textual 8.2.7 的原始代码有性能瓶颈
class Widget:
    def refresh(self):
        # 每次 refresh 都重新计算整个布局
        self._recalculate_layout()  # O(n) 复杂度
        self._redraw()              # 即使没有变化也重绘
```

**问题 2: Textual 的 bug**

```python
# ❌ Textual 在某些情况下会崩溃
class MessagePump:
    def dispatch(self, message):
        # 如果 handler 抛出异常，会导致消息队列卡死
        try:
            handler(message)
        except Exception:
            pass  # 吞掉异常，但消息状态不正确
```

**问题 3: 缺少需要的功能**

```python
# ❌ Textual 没有我们需要的 CJK 宽度计算
class Compositor:
    def calculate_width(self, text):
        # 只支持 ASCII 和拉丁字符
        return len(text)  # 中文字符宽度计算错误
```

### 用了 patches 之后

```python
# ✅ 启动时自动修补
from chrys.foundation.patches import apply_all

results = apply_all()

for result in results:
    if result.status == "applied":
        logger.info(f"Patched: {result.detail}")
    elif result.status == "skipped":
        logger.debug(f"Already patched: {result.patch.description}")
    elif result.status == "error":
        logger.error(f"Patch failed: {result.detail}")

# 现在 Textual 已经有了：
# - 优化的布局算法（O(1) 增量更新）
# - 修复的消息派发（正确处理异常）
# - CJK 宽度支持（中文、日文、韩文）
```

---

## 核心设计：三个关键决策

### 决策 1: 源码级补丁（而非运行时 monkey-patch）

```python
@dataclass(frozen=True)
class FilePatch:
    """A single find-and-replace patch for an installed package file."""
    
    package: str          # 包名，如 "textual"
    module_file: str      # 文件路径，如 "_tools.py"
    old_fragment: str     # 要替换的原始代码片段
    new_fragment: str     # 替换后的新代码片段
    description: str      # 补丁描述
    equivalent_fragments: tuple[str, ...] = ()  # 等价的已修补版本


def apply_patch(patch: FilePatch) -> PatchResult:
    """Apply a single patch, returning the result."""
    
    # 定位包目录
    target = _locate_package_dir(patch.package) / patch.module_file
    
    if not target.is_file():
        return PatchResult(patch, "error", f"File not found: {target}")
    
    source = target.read_text(encoding="utf-8")
    
    # 检查是否已经打过补丁
    if patch.new_fragment in source:
        return PatchResult(patch, "skipped", "Already patched.")
    
    # 检查是否有等价的补丁版本
    if any(fragment in source for fragment in patch.equivalent_fragments):
        return PatchResult(patch, "skipped", "Equivalent patch already applied.")
    
    # 检查原始片段是否存在
    if patch.old_fragment not in source:
        return PatchResult(
            patch, 
            "error", 
            "Source file does not contain the expected original fragment. "
            "The upstream package may have been updated."
        )
    
    # 应用补丁
    patched = source.replace(patch.old_fragment, patch.new_fragment, 1)
    _atomic_write_text(target, patched)
    
    return PatchResult(patch, "applied", f"Patched {target}")
```

**为什么？**

- **持久化**：补丁写入磁盘，不需要每次启动都重新应用
- **幂等性**：重复运行 `apply_all()` 是安全的
- **版本检测**：如果上游库更新了，补丁会失败并报错
- **性能**：运行时没有 monkey-patch 开销

**对比运行时 monkey-patch**：

```python
# ❌ 运行时 monkey-patch（不推荐）
import textual.widget

original_refresh = textual.widget.Widget.refresh

def patched_refresh(self):
    # 优化逻辑
    pass

textual.widget.Widget.refresh = patched_refresh
# → 每次启动都要执行，有运行时开销
```

---

### 决策 2: 原子写入保证一致性

```python
def _atomic_write_text(path: Path, payload: str) -> None:
    """Atomically write text to an installed patch target."""
    
    # 保存原始权限
    target_mode = stat.S_IMODE(path.stat().st_mode)
    
    # 创建临时文件（同目录，确保同一文件系统）
    fd, tmp_name = tempfile.mkstemp(
        prefix=f"{path.name}.",
        suffix=".tmp",
        dir=path.parent,
    )
    tmp_path = Path(tmp_name)
    
    try:
        # 二进制模式写入（避免 Windows 的 \r\n 转换）
        with os.fdopen(fd, "wb") as f:
            f.write(payload.encode("utf-8"))
            f.flush()
            os.fsync(f.fileno())  # 确保写入磁盘
        
        # 恢复权限
        os.chmod(tmp_path, target_mode)
        
        # 原子替换
        os.replace(tmp_path, path)
    
    except BaseException:
        # 失败时清理临时文件
        with contextlib.suppress(FileNotFoundError):
            tmp_path.unlink()
        raise
```

**为什么？**

- **防止损坏**：如果在写入过程中崩溃，原文件不会被破坏
- **跨平台**：`os.replace` 在 POSIX 和 Windows 上都是原子的
- **权限保留**：保持原文件的权限位

---

### 决策 3: 分组应用保证一致性

```python
def apply_patch_group(patches: list[FilePatch]) -> list[PatchResult]:
    """Apply same-file patches all-or-nothing.
    
    The patcher often registers several fragments against one upstream
    module. Staging them as a group prevents a package upgrade from
    leaving that module half-mutated if one later fragment no longer matches.
    """
    
    # 按文件分组
    by_file: dict[Path, list[FilePatch]] = defaultdict(list)
    for patch in patches:
        try:
            target = _patch_target(patch)
            by_file[target].append(patch)
        except ImportError:
            continue
    
    results: list[PatchResult] = []
    
    # 对每个文件，要么全部应用，要么全部跳过
    for target, file_patches in by_file.items():
        # 读取一次源文件
        source = target.read_text(encoding="utf-8")
        
        # 检查所有补丁是否可以应用
        can_apply_all = True
        for patch in file_patches:
            if patch.new_fragment in source:
                continue  # 已经打过
            if patch.old_fragment not in source:
                can_apply_all = False
                break
        
        if not can_apply_all:
            # 有一个补丁无法应用，整个文件跳过
            for patch in file_patches:
                results.append(PatchResult(
                    patch,
                    "error",
                    "Cannot apply partial patch group; upstream changed.",
                ))
            continue
        
        # 应用所有补丁
        for patch in file_patches:
            if patch.new_fragment in source:
                results.append(PatchResult(patch, "skipped", "Already patched."))
                continue
            
            source = source.replace(patch.old_fragment, patch.new_fragment, 1)
            results.append(PatchResult(patch, "applied", f"Patched {target}"))
        
        # 一次性写入
        _atomic_write_text(target, source)
    
    return results
```

**为什么？**

- **防止半修补**：如果一个文件有多个补丁，要么全部应用，要么全部不应用
- **减少 IO**：多个补丁合并为一次写入
- **一致性**：避免上游更新导致的碎片化状态

---

## 实际补丁示例

### 示例 1: Textual CJK 宽度修复

```python
# patches/textual_compositor_cjk.py

from chrys.foundation.patches.patcher import FilePatch, register

CJK_WIDTH_PATCH = FilePatch(
    package="textual",
    module_file="_compositor.py",
    old_fragment="""
def _calculate_cell_width(text: str) -> int:
    return len(text)
""",
    new_fragment="""
def _calculate_cell_width(text: str) -> int:
    from wcwidth import wcswidth
    width = wcswidth(text)
    return width if width >= 0 else len(text)
""",
    description="Fix CJK character width calculation in compositor",
)

register(CJK_WIDTH_PATCH)
```

### 示例 2: Textual 消息派发修复

```python
# patches/textual_message_pump.py

MESSAGE_PUMP_PATCH = FilePatch(
    package="textual",
    module_file="_message_pump.py",
    old_fragment="""
async def _dispatch_message(self, message: Message) -> None:
    try:
        await self._handle_message(message)
    except Exception:
        pass  # Swallow exceptions
""",
    new_fragment="""
async def _dispatch_message(self, message: Message) -> None:
    try:
        await self._handle_message(message)
    except Exception as exc:
        # Log the exception and mark message as processed
        logger.exception(f"Error handling message {message}")
        message._processed = True
        raise
""",
    description="Fix message pump exception handling",
)

register(MESSAGE_PUMP_PATCH)
```

### 示例 3: Textual Windows sleep 修复

```python
# patches/textual_win_sleep.py

WIN_SLEEP_PATCH = FilePatch(
    package="textual",
    module_file="_timer.py",
    old_fragment="""
async def _sleep(self, delay: float) -> None:
    await asyncio.sleep(delay)
""",
    new_fragment="""
async def _sleep(self, delay: float) -> None:
    # Windows 上的 asyncio.sleep 精度差，使用 time.sleep 替代
    import sys
    if sys.platform == "win32" and delay < 0.01:
        await asyncio.get_event_loop().run_in_executor(None, time.sleep, delay)
    else:
        await asyncio.sleep(delay)
""",
    description="Improve timer precision on Windows",
)

register(WIN_SLEEP_PATCH)
```

---

## 启动流程

```python
# orchestration/startup.py

def bootstrap_runtime() -> None:
    """Bootstrap the Chrys runtime environment."""
    
    # 1. 应用第三方库补丁（必须在导入这些库之前）
    from chrys.foundation.patches import apply_all
    
    patch_results = apply_all()
    
    # 记录补丁结果
    for result in patch_results:
        if result.status == "applied":
            logger.info(f"Applied patch: {result.patch.description}")
        elif result.status == "error":
            logger.warning(f"Patch failed: {result.detail}")
    
    # 2. 现在可以安全导入修补后的库
    from textual.app import App  # ← 已经是修补后的版本
    
    # 3. 继续其他初始化...
```

---

## 和其他模块的关系

```
patches 被以下模块使用：
├── orchestration/startup.py（启动时应用补丁）
├── app/tui/（依赖修补后的 Textual）
└── foundation/observability/（某些补丁影响遥测）
```

---

## 如果你要自己实现

### 最小版本（单个补丁）

```python
import importlib.util
from pathlib import Path

def apply_single_patch(package: str, file_path: str, old: str, new: str) -> bool:
    """Apply a single patch to an installed package."""
    
    # 定位包目录
    spec = importlib.util.find_spec(package)
    if spec is None or spec.origin is None:
        return False
    
    target = Path(spec.origin).parent / file_path
    
    # 读取源文件
    source = target.read_text(encoding="utf-8")
    
    # 检查是否已经打过补丁
    if new in source:
        return True
    
    # 检查原始片段是否存在
    if old not in source:
        return False
    
    # 应用补丁
    patched = source.replace(old, new, 1)
    target.write_text(patched, encoding="utf-8")
    
    return True

# 使用
apply_single_patch(
    package="textual",
    file_path="_compositor.py",
    old="def _calculate_cell_width(text: str) -> int:\n    return len(text)",
    new="def _calculate_cell_width(text: str) -> int:\n    from wcwidth import wcswidth\n    return wcswidth(text) or len(text)",
)
```

### 进阶版本

加分组应用、原子写入、等价片段检测。参考 Chrys 的 `patcher.py`。

---

## 下一步

- **上一篇**: F12 observability — 可观测性集成
- **下一篇**: F14 vendor — 第三方库打包
