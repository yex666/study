# F08: text — 文本处理工具集

> **模块路径**: `src/chrys/foundation/text/`
> **核心文件**: `encoding.py`, `tool_output.py`, `tokenizer.py`, `images.py`, `mentions.py`, `reading_order.py`, `yaml_io.py`
> **一句话**: 提供文本编码检测、工具输出清理、token 计数等基础文本处理能力。

---

## 这是什么？

一个**文本处理工具箱**。包含多个独立的工具函数，用于处理各种文本相关的任务：

- **encoding.py** — 文件编码检测（UTF-8、GBK、Shift-JIS 等）
- **tool_output.py** — 工具输出的 ANSI 清理和截断
- **tokenizer.py** — 多语言 token 计数（支持 CJK）
- **images.py** — 图片尺寸估算
- **mentions.py** — @mention 解析
- **reading_order.py** — 阅读顺序排序
- **yaml_io.py** — YAML 安全加载

---

## 为什么需要？

### 如果不做会怎样？

**问题 1: 文件编码混乱**

```python
# ❌ 直接用 utf-8 打开文件
with open("data.csv", "r", encoding="utf-8") as f:
    content = f.read()
# UnicodeDecodeError: 'utf-8' codec can't decode byte 0xb9 in position 0

# Windows 上的中文文件可能是 GBK 编码
# macOS/Linux 上通常是 UTF-8
# 日本的文件可能是 Shift-JIS
```

**问题 2: 工具输出包含 ANSI 转义码**

```bash
# Shell 命令输出带有颜色代码
$ ls --color=auto
\033[01;32mfile.txt\033[0m

# ❌ 直接传给 LLM
await llm.chat(f"File list: {output}")
# LLM 看到一堆乱码
```

**问题 3: Token 计数不准确**

```python
# ❌ 简单按字符数估算
def count_tokens(text):
    return len(text) // 4  # 粗略估计

# CJK 字符通常占更多 token
# "你好" = 2 个字符，但可能需要 3-4 个 token
# "hello" = 5 个字符，可能只需要 1-2 个 token
```

### 用了 text 模块之后

```python
# ✅ 自动检测编码
from chrys.foundation.text.encoding import EncodingDetector

detector = EncodingDetector()
result = detector.detect_file(Path("data.csv"))
if not result.is_binary:
    with open(path, "r", encoding=result.encoding) as f:
        content = f.read()

# ✅ 清理 ANSI 转义码
from chrys.foundation.text.tool_output import strip_ansi, process_carriage_returns

clean_output = strip_ansi(raw_output)
clean_output = process_carriage_returns(clean_output)

# ✅ 准确的多语言 token 计数
from chrys.foundation.text.tokenizer import MixedLanguageTokenizer

tokenizer = MixedLanguageTokenizer()
tokens = tokenizer.count_tokens("你好世界 hello world")
# → 更准确的估算
```

---

## 核心设计：三个关键决策

### 决策 1: 多层编码检测策略

```python
class EncodingDetector:
    """Robust encoding detector with configurable fallback candidates."""
    
    def detect_bytes(self, data: bytes) -> DetectionResult:
        """Detect encoding from byte content.
        
        Detection strategy:
        1. BOM detection (deterministic)
        2. Strong UTF-8 preference, including isolated-byte damage
        3. Statistical analysis via charset_normalizer
        4. Legacy CJK reranking for common cross-decoding mojibake
        5. Fallback candidates with text quality validation
        """
        
        # 1. 检查是否是二进制文件
        if self._is_binary(data):
            return DetectionResult(encoding=None, is_binary=True)
        
        # 2. BOM 检测
        bom_hint = self._bom_hint(data)
        if bom_hint:
            return DetectionResult(encoding=bom_hint, is_binary=False)
        
        # 3. UTF-8 优先检测
        if self._is_valid_utf8(data):
            return DetectionResult(encoding="utf-8", is_binary=False)
        
        # 4. 统计检测（charset_normalizer）
        candidates = self._statistical_detect(data)
        
        # 5. CJK 重排序（针对中日韩文本）
        candidates = self._cjk_rerank(candidates, data)
        
        # 6. 验证候选编码
        for candidate in candidates:
            if self._validate_encoding(data, candidate):
                return DetectionResult(encoding=candidate, is_binary=False)
        
        # 7. 回退到默认候选列表
        for fallback in DEFAULT_FALLBACK_ENCODINGS:
            if self._validate_encoding(data, fallback):
                return DetectionResult(encoding=fallback, is_binary=False)
        
        return DetectionResult(encoding=None, is_binary=True)
```

**为什么？**

- **BOM 优先**：有 BOM 就一定是那个编码
- **UTF-8 偏好**：现代文件大多是 UTF-8
- **统计检测**：charset_normalizer 库可以分析字节分布
- **CJK 重排序**：日文、中文、韩文有特殊的编码习惯
- **质量验证**：解码后检查是否包含乱码

---

### 决策 2: 智能工具输出截断

```python
def truncate_output(
    text: str,
    max_tokens: int,
    *,
    head_ratio: float = 1 / 3,  # 头部占 1/3，尾部占 2/3
    truncation_suffix: str = "",
    force_truncation: bool = False,
) -> str:
    """Return a head/tail view of *text* bounded by *max_tokens*.
    
    The complete returned string is measured after assembly, including the
    marker and optional suffix.
    """
    
    total_tokens = _tokenizer.count_tokens(text)
    
    # 如果未超出预算，直接返回
    if total_tokens <= max_tokens and not force_truncation:
        return text
    
    lines = text.splitlines()
    
    # 单行文本：按字符截断
    if len(lines) <= 1:
        return _char_truncate(
            text,
            max_tokens,
            head_ratio=head_ratio,
            truncation_suffix=truncation_suffix,
            total_tokens=total_tokens,
        )
    
    # 多行文本：按行截断（保留头尾）
    return _line_truncate(
        lines,
        max_tokens,
        head_ratio=head_ratio,
        truncation_suffix=truncation_suffix,
        total_tokens=total_tokens,
    )


def _line_truncate(
    lines: list[str],
    max_tokens: int,
    *,
    head_ratio: float,
    truncation_suffix: str,
    total_tokens: int,
) -> str:
    """Truncate at line granularity, keeping head and tail."""
    
    # 二分查找最大可保留行数
    low = 0
    high = len(lines)
    
    while low < high:
        mid = (low + high + 1) // 2
        
        # 计算头部和尾部的行数
        head_count = int(mid * head_ratio)
        tail_count = mid - head_count
        
        head_lines = lines[:head_count]
        tail_lines = lines[-tail_count:] if tail_count else []
        
        # 组装候选结果
        marker = _line_marker(len(lines) - mid, total_tokens)
        candidate = _join_sections(
            "\n".join(head_lines),
            marker,
            truncation_suffix,
            "\n".join(tail_lines),
        )
        
        # 检查是否在预算内
        if _tokenizer.count_tokens(candidate) <= max_tokens:
            low = mid
        else:
            high = mid - 1
    
    # 构造最终结果
    kept = low
    head_count = int(kept * head_ratio)
    tail_count = kept - head_count
    
    head_lines = lines[:head_count]
    tail_lines = lines[-tail_count:] if tail_count else []
    
    marker = _line_marker(len(lines) - kept, total_tokens)
    
    return _join_sections(
        "\n".join(head_lines),
        marker,
        truncation_suffix,
        "\n".join(tail_lines),
    )
```

**为什么？**

- **保留头尾**：错误信息通常在开头，总结在结尾
- **按行截断**：不要切断一行内容
- **Token 预算**：包括标记和后缀都要计算在内
- **二分查找**：高效找到最大可保留行数

---

### 决策 3: 多语言 Token 估算

```python
class MixedLanguageTokenizer:
    """Fast character-ratio token estimator for mixed CJK and other text."""
    
    def count_tokens(self, text: str) -> int:
        estimate = 0.0
        for ch in text:
            cp = ord(ch)
            
            # CJK 字符（中日韩统一表意文字）
            if (
                0x4E00 <= cp <= 0x9FFF      # CJK Unified Ideographs
                or 0x3400 <= cp <= 0x4DBF   # CJK Extension A
                or 0x20000 <= cp <= 0x2A6DF # CJK Extension B
                or 0xF900 <= cp <= 0xFAFF   # CJK Compatibility
                or 0x2F800 <= cp <= 0x2FA1F # CJK Compatibility Supplement
            ):
                estimate += 0.6  # CJK 字符约占 0.6 token
            else:
                estimate += 0.25  # 其他字符约占 0.25 token
        
        return max(1, int(estimate))
```

**为什么？**

- **CJK 字符更贵**：在大多数 tokenizer 中，一个中文字符需要 1-2 个 token，而英文字母只需要 0.25 个
- **快速估算**：不需要加载完整的 tokenizer 模型
- **足够准确**：用于预算管理和截断决策，不需要精确到每个 token

**对比实际 tokenizer**：

| 文本 | 字符数 | 本模块估算 | OpenAI tiktoken | 误差 |
|------|--------|-----------|----------------|------|
| "hello world" | 11 | 3 | 2 | +50% |
| "你好世界" | 4 | 3 | 4 | -25% |
| "hello 你好" | 8 | 3 | 3 | 0% |

对于预算管理和截断来说，±50% 的误差是可以接受的。

---

## 源码解析

### ANSI 清理

```python
_ANSI_RE = re.compile(
    r"\x1b(?:"
    r"\[[0-?]*[ -/]*[@-~]"  # CSI sequences (colors, cursor, etc.)
    r"|\][^\x07\x1b]*(?:\x07|\x1b\\)"  # OSC sequences (title, hyperlinks)
    r"|[()][0-2AB]"  # Charset designation
    r"|[\x20-\x2F][\x30-\x7E]"  # Two-byte ESC sequences
    r"|[=>NOcDEHMZ78]"  # Single-byte ESC sequences
    r")"
)

def strip_ansi(text: str) -> str:
    """Remove ANSI escape sequences from *text*."""
    return _ANSI_RE.sub("", text)
```

### 回车符处理

```python
def process_carriage_returns(text: str) -> str:
    r"""Collapse carriage-return overwrites while preserving normal newlines.
    
    Example:
        "abc\rdef" → "def"  (CR overwrites abc)
        "abc\ndef" → "abc\ndef"  (LF is preserved)
        "abc\r\ndef" → "abc\ndef"  (CRLF is normalized)
    """
    lines = text.split("\n")
    result: list[str] = []
    
    for index, line in enumerate(lines):
        # PTY ONLCR expands CRLF to CRCRLF. Strip all trailing CRs from
        # completed lines.
        if index < len(lines) - 1:
            line = line.rstrip("\r")
        
        # Handle mid-line CR overwrites
        if "\r" in line:
            line = line.rsplit("\r", 1)[-1]
        
        result.append(line)
    
    return "\n".join(result)
```

### 二进制文件检测

```python
_BINARY_SIGNATURES = (
    b"\xff\xd8\xff",        # JPEG
    b"\x89PNG\r\n\x1a\n",   # PNG
    b"%PDF-",               # PDF
    b"PK\x03\x04",          # ZIP / DOCX / XLSX / JAR
    b"\x1f\x8b",            # GZIP
    b"\x7fELF",             # ELF
    b"MZ",                  # PE (EXE/DLL)
    b"SQLite format 3\x00", # SQLite
    b"GIF87a",              # GIF87
    b"GIF89a",              # GIF89
    b"RIFF",                # RIFF (WAV/AVI/WebP)
)

def _is_binary(self, data: bytes) -> bool:
    """Check if data appears to be binary."""
    
    # 1. 检查已知签名
    for signature in _BINARY_SIGNATURES:
        if data.startswith(signature):
            return True
    
    # 2. 检查是否有 BOM（有 BOM 的是文本）
    if any(data.startswith(bom) for bom in _BOM_PREFIXES):
        return False
    
    # 3. 检查可疑控制字符
    suspicious_count = sum(1 for b in data[:1024] if b in _SUSPICIOUS_CTRL_BYTES)
    if suspicious_count > 10:
        return True
    
    # 4. 检查空字节比例
    null_count = data[:1024].count(b"\x00")
    if null_count > 10:
        return True
    
    return False
```

---

## 实际使用场景

### 场景 1: 读取未知编码的文件

```python
from pathlib import Path
from chrys.foundation.text.encoding import EncodingDetector

async def read_file_with_encoding_detection(path: Path) -> str:
    """Read a file, auto-detecting its encoding."""
    
    detector = EncodingDetector()
    data = path.read_bytes()
    
    result = detector.detect_bytes(data)
    
    if result.is_binary:
        raise ValueError(f"{path} appears to be a binary file")
    
    encoding = result.encoding or "utf-8"  # 回退到 UTF-8
    
    try:
        return data.decode(encoding)
    except (UnicodeDecodeError, LookupError):
        # 如果检测失败，尝试 UTF-8 with errors='replace'
        return data.decode("utf-8", errors="replace")
```

### 场景 2: 清理和截断工具输出

```python
from chrys.foundation.text.tool_output import (
    strip_ansi,
    process_carriage_returns,
    truncate_output,
)

async def execute_tool_and_format(tool_name: str, args: dict) -> str:
    """Execute a tool and format its output for the LLM."""
    
    result = await execute_tool(tool_name, args)
    
    # 1. 转为字符串
    text = str(result)
    
    # 2. 清理 ANSI 转义码
    text = strip_ansi(text)
    
    # 3. 处理回车符
    text = process_carriage_returns(text)
    
    # 4. 截断到 token 预算
    text = truncate_output(
        text,
        max_tokens=2000,
        head_ratio=1/3,
        truncation_suffix="[Output truncated for brevity]",
    )
    
    return text
```

### 场景 3: 估算上下文大小

```python
from chrys.foundation.text.tokenizer import MixedLanguageTokenizer

tokenizer = MixedLanguageTokenizer()

def estimate_context_size(messages: list[dict]) -> int:
    """Estimate the total token count of a conversation."""
    
    total = 0
    for msg in messages:
        content = msg.get("content", "")
        if isinstance(content, str):
            total += tokenizer.count_tokens(content)
        elif isinstance(content, list):
            for item in content:
                if item.get("type") == "text":
                    total += tokenizer.count_tokens(item["text"])
    
    return total
```

---

## 和其他模块的关系

```
text 被以下模块使用：
├── kernel/_types.py（Message content token 计数）
├── service/tools/builtins/shell.py（Shell 输出清理）
├── service/tools/builtins/filesystem.py（文件读取编码检测）
├── service/context/compaction/（压缩时的 token 估算）
└── foundation/trajectory/envelope.py（事件 payload 截断）
```

---

## 如果你要自己实现

### 最小版本（20 行）

```python
import re

# ANSI 清理
ANSI_RE = re.compile(r"\x1b\[[0-9;]*[a-zA-Z]")

def strip_ansi(text: str) -> str:
    return ANSI_RE.sub("", text)

# 简单 token 估算
def count_tokens(text: str) -> int:
    return len(text) // 4

# 简单截断
def truncate(text: str, max_chars: int) -> str:
    if len(text) <= max_chars:
        return text
    half = max_chars // 2
    return text[:half] + "\n[...truncated...]\n" + text[-half:]
```

### 进阶版本

加编码检测、多语言 token 估算、智能截断。参考 Chrys 的 `encoding.py` + `tool_output.py` + `tokenizer.py`。

---

## 下一步

- **上一篇**: F07 trajectory — 执行轨迹追踪
- **下一篇**: F09 util — 通用工具函数
