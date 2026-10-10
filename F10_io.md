# F10: io — I/O 结果标准化

> **模块路径**: `src/chrys/foundation/io/`
> **核心文件**: `result_content.py`
> **一句话**: 将各种格式的工具结果统一转换为可显示的文本和图片内容。

---

## 这是什么？

一个**工具结果标准化层**。把不同来源、不同格式的工具返回结果（字符串、Content 对象列表、MCP 结果等）统一转换为：
- **显示文本**：给 TUI 和 LLM 看的纯文本
- **图片内容**：给多模态模型看的图像数据

```python
from chrys.foundation.io.result_content import extract_result_text, extract_result_images

# 工具返回的结果可能是各种格式
result = [
    Content(type="text", text="File content..."),
    Content(type="data", media_type="image/png", data="base64..."),
]

# 统一提取文本
text = extract_result_text(result)
# → "File content..."

# 统一提取图片
images = extract_result_images(result)
# → [Content(type="data", media_type="image/png", ...)]
```

---

## 为什么需要？

### 如果不做会怎样？

**问题 1: 工具结果格式不统一**

```python
# ❌ 每个地方都要处理不同的结果格式
if isinstance(result, str):
    display_text = result
elif isinstance(result, list):
    # 可能是 Content 对象列表
    texts = []
    for item in result:
        if hasattr(item, "text"):
            texts.append(item.text)
        elif hasattr(item, "result"):
            texts.append(item.result)
        elif hasattr(item, "output"):
            texts.append(item.output)
    display_text = "\n".join(texts)
elif isinstance(result, dict):
    display_text = json.dumps(result)
else:
    display_text = str(result)
```

**问题 2: 未识别的内容格式导致崩溃**

```python
# ❌ 假设所有结果都有 .text 属性
for item in result:
    print(item.text)  # AttributeError: 'FunctionCall' object has no attribute 'text'
```

**问题 3: 大对象占满日志**

```python
# ❌ 直接把整个结果打印到日志
logger.debug(f"Tool result: {result}")
# → 几百 KB 的 base64 图片数据占满日志文件
```

### 用了 io 模块之后

```python
# ✅ 统一的提取接口
text = extract_result_text(result)
images = extract_result_images(result)

# ✅ 自动处理未知格式
# 如果 Content 没有 text/result/output/message 属性
# 会记录 WARNING 并返回 "Error: unrecognized tool result content..."

# ✅ 日志友好
# 大对象会被摘要：<str len=5000 preview='first 80 chars...'>
```

---

## 核心设计：三个关键决策

### 决策 1: 多属性优先级查找

```python
def _content_to_text(item: Any) -> str:
    """Render a single Content-like item to a string."""
    
    if isinstance(item, str):
        return item
    
    if isinstance(item, Mapping):
        return _render_attr_value(item)
    
    # 按优先级尝试多个属性
    # 第一个非 None 的属性胜出
    for attr in ("text", "result", "output", "message"):
        val = getattr(item, attr, None)
        if val is None:
            continue
        return _render_attr_value(val)
    
    # 如果是数据 URI（图片、音频等）
    uri = getattr(item, "uri", None)
    if uri:
        media_type = getattr(item, "media_type", None) or "binary"
        if _is_image_data_uri(media_type, uri):
            return f"[{media_type} image]"
        snippet = uri if len(uri) <= 120 else uri[:117] + "..."
        return f"[{media_type}: {snippet}]"
    
    # 未识别的格式
    logger.warning("Unrecognized tool result Content")
    return f"{_ERROR_PREFIX}unrecognized tool result content"
```

**为什么？**

不同的 Content 类型用不同的属性名：
- `type="text"` → `item.text`
- `type="function_result"` → `item.result`
- `type="mcp_server_tool_result"` → `item.output`
- `type="error"` → `item.message`

优先级查找确保无论哪种类型都能正确提取文本。

---

### 决策 2: 智能属性值渲染

```python
def _render_attr_value(val: Any) -> str:
    """Render a Content text-bearing attribute value to a string."""
    
    if isinstance(val, str):
        return val
    
    if isinstance(val, (list, tuple)):
        # 递归处理嵌套的 Content 列表
        return extract_result_text(list(val))
    
    if isinstance(val, Mapping):
        # 字典转为 JSON（保留结构化信息）
        try:
            return json.dumps(val, default=str, ensure_ascii=False)
        except TypeError, ValueError:
            return str(val)
    
    # 其他类型直接转字符串
    return str(val)
```

**为什么？**

- **列表递归**：MCP server 可能返回 `list[Content]`
- **字典 JSON 化**：结构化错误信息保持可读
- **兜底 str()**：数字、布尔值等简单类型

---

### 决策 3: 日志友好的摘要

```python
_DEBUG_STR_PREVIEW_CHARS = 80

def _summarize_value(v: Any) -> Any:
    """Render a single value as a debug-log-safe summary."""
    
    if isinstance(v, str):
        if len(v) <= _DEBUG_STR_PREVIEW_CHARS:
            return v
        # 长字符串只显示前 80 个字符
        return f"<str len={len(v)} preview={v[:_DEBUG_STR_PREVIEW_CHARS]!r}>"
    
    if isinstance(v, (bytes, bytearray, list, tuple, dict, set)):
        # 容器只显示类型和长度
        return f"<{type(v).__name__} len={len(v)}>"
    
    if isinstance(v, (int, float, bool)):
        return v
    
    return f"<{type(v).__name__}>"


def _summarize_item_shape(item: Any) -> Any:
    """Render a result-list item as a debug-log-safe shape descriptor."""
    
    if not hasattr(item, "__dict__"):
        return _summarize_value(item)
    
    try:
        # 提取所有非空字段
        fields = {
            k: v 
            for k, v in vars(item).items() 
            if v not in (None, "") and not k.startswith("_")
        }
    except TypeError:
        return _summarize_value(item)
    
    # 递归摘要每个字段
    return {k: _summarize_value(v) for k, v in fields.items()}
```

**为什么？**

- **防止日志爆炸**：base64 图片可能有几 MB
- **保留关键信息**：长度、类型、前 80 字符预览
- **调试友好**：可以看到 Content 的结构，但不会被大量数据淹没

---

## 源码解析

### extract_result_text 主函数

```python
def extract_result_text(result: Any) -> str:
    """Extract displayable text from a tool invocation result.
    
    FunctionTool.invoke() returns list[Content]. This walks the list
    and extracts text from each Content item.
    """
    
    if result is None:
        return ""
    
    # 防护：取消的工具可能留下未完成的协程
    import inspect
    if inspect.isawaitable(result):
        close = getattr(result, "close", None)
        if close is not None:
            close()
        return ""
    
    if isinstance(result, str):
        return result
    
    if isinstance(result, list):
        # DEBUG 日志：记录列表结构
        if logger.isEnabledFor(logging.DEBUG):
            logger.debug(
                "Tool result list: %d item(s); types=%s; shapes=%s",
                len(result),
                [getattr(item, "type", type(item).__name__) for item in result],
                [_summarize_item_shape(item) for item in result],
            )
        
        # 转换每个元素
        rendered = [_content_to_text(item) for item in result]
        parts = [p for p in rendered if p]  # 过滤空字符串
        
        if not parts:
            return ""
        
        # 检查是否有错误标记
        any_error = any(p.startswith(_ERROR_PREFIX) for p in parts)
        joined = "\n".join(parts)
        
        # 如果有错误但整体没有错误前缀，加上前缀
        if any_error and not joined.startswith(_ERROR_PREFIX):
            joined = _ERROR_PREFIX + joined
        
        return joined
    
    # 单个 Content 对象
    return _content_to_text(result)
```

### extract_result_images

```python
def extract_result_images(result: Any) -> list[Any]:
    """Extract image Content-like items from a tool invocation result."""
    
    if result is None or isinstance(result, str):
        return []
    
    if isinstance(result, list):
        images: list[Any] = []
        for item in result:
            images.extend(extract_result_images(item))
        return images
    
    item_type = getattr(result, "type", None)
    
    # function_result 可能包含嵌套的 items
    if item_type == "function_result":
        items = getattr(result, "items", None)
        return extract_result_images(items) if isinstance(items, list) else []
    
    # data/uri 类型可能是图片
    if item_type in ("data", "uri"):
        media_type = getattr(result, "media_type", None)
        properties = getattr(result, "additional_properties", None)
        property_media_type = properties.get("media_type") if isinstance(properties, Mapping) else None
        
        if is_image_media_type(media_type) or is_image_media_type(property_media_type):
            return [result]
    
    return []
```

### 辅助函数

```python
_ERROR_PREFIX = "Error: "

def _is_image_data_uri(media_type: Any, uri: Any) -> bool:
    """Check if this is an inline image (data URI with image/* media type)."""
    return (
        isinstance(media_type, str)
        and media_type.lower().startswith("image/")
        and isinstance(uri, str)
        and uri.startswith("data:")
    )
```

---

## 实际使用场景

### 场景 1: 工具结果展示给 LLM

```python
async def execute_and_format(tool_name: str, args: dict) -> str:
    """Execute a tool and format its result for the LLM."""
    
    result = await tool.invoke(args)
    
    # 提取文本（自动处理各种格式）
    text = extract_result_text(result)
    
    # 如果没有文本但有图片，告知 LLM
    if not text:
        images = extract_result_images(result)
        if images:
            text = f"[Tool returned {len(images)} image(s), see attached]"
        else:
            text = "[Tool returned empty result]"
    
    return text
```

### 场景 2: TUI 显示工具结果

```python
class ToolResultWidget:
    def __init__(self, result: Any):
        self.result = result
        
        # 提取文本
        text = extract_result_text(result)
        
        # 检查是否是错误
        if text.startswith("Error: "):
            self.style = "error"
            self.text = text[7:]  # 移除 "Error: " 前缀
        else:
            self.style = "normal"
            self.text = text
        
        # 提取图片（如果有）
        self.images = extract_result_images(result)
```

### 场景 3: 调试日志

```python
async def debug_tool_execution(tool_name: str, result: Any):
    """Log tool execution details for debugging."""
    
    if logger.isEnabledFor(logging.DEBUG):
        logger.debug(
            "Tool %s returned: %d chars, types=%s",
            tool_name,
            len(extract_result_text(result)),
            [type(item).__name__ for item in (result if isinstance(result, list) else [result])],
        )
        
        # 详细形状（不会占满日志）
        if isinstance(result, list):
            for i, item in enumerate(result):
                logger.debug(
                    "  [%d] %s",
                    i,
                    _summarize_item_shape(item),
                )
```

---

## 和其他模块的关系

```
io 被以下模块使用：
├── kernel/loop.py（工具循环中的结果处理）
├── service/tools/builtins/（内置工具的结果格式化）
├── app/tui/widgets/chat/（TUI 中的结果展示）
└── foundation/trajectory/envelope.py（轨迹事件的 payload 处理）
```

---

## 如果你要自己实现

### 最小版本（20 行）

```python
import json

def simple_extract_result_text(result):
    """Simple version of extract_result_text."""
    
    if result is None:
        return ""
    
    if isinstance(result, str):
        return result
    
    if isinstance(result, list):
        texts = []
        for item in result:
            if hasattr(item, "text") and item.text:
                texts.append(item.text)
            elif hasattr(item, "result") and item.result:
                texts.append(str(item.result))
        return "\n".join(texts)
    
    if hasattr(result, "text"):
        return result.text or ""
    
    return str(result)
```

### 进阶版本

加多属性优先级查找、智能渲染、日志摘要。参考 Chrys 的 `result_content.py`。

---

## 下一步

- **上一篇**: F09 util — 通用工具函数集
- **下一篇**: F11 net — 网络操作辅助
