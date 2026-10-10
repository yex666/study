# F12: observability — 可观测性集成

> **模块路径**: `src/chrys/foundation/observability/`
> **核心文件**: `gate.py`, `setup.py`, `sink.py`, `exporters.py`
> **一句话**: 集成 OpenTelemetry，支持 traces/logs/metrics 的收集和导出。

---

## 这是什么？

一个**OpenTelemetry 集成层**。提供：
- **TelemetryGate** — 全局开关，控制是否启用遥测
- **OtelSessionSink** — 会话级遥测数据接收器
- **Exporter 配置** — 从环境变量或配置读取 OTLP 端点
- **零配置回退** — 没有 OTLP 端点时自动写入本地 JSONL 文件

---

## 为什么需要？

### 如果不做会怎样？

**问题 1: 无法调试生产环境的问题**

```python
# ❌ 只有简单的日志
logger.info("LLM call started")
response = await llm.chat(messages)
logger.info(f"LLM call finished, tokens={response.usage.total_tokens}")

# 无法回答：
# - 这次调用花了多长时间？
# - 中间有没有重试？
# - 网络延迟是多少？
# - Token 消耗的成本是多少？
```

**问题 2: 无法分析性能瓶颈**

```python
# ❌ 不知道时间花在哪里
# - LLM 调用占了多少时间？
# - 工具执行占了多少时间？
# - 压缩历史占了多少时间？
# - 网络传输占了多少时间？
```

**问题 3: 无法追踪成本**

```python
# ❌ 不知道每次对话花了多少钱
# - GPT-4 的 token 成本是多少？
# - Claude 的 token 成本是多少？
# - 哪个 Agent 最费钱？
```

### 用了 observability 之后

```python
# ✅ 完整的 traces
from chrys.foundation.observability.gate import TELEMETRY_GATE

if TELEMETRY_GATE.enabled:
    tracer = trace.get_tracer("chrys.agent")
    
    with tracer.start_as_current_span("llm_call") as span:
        span.set_attribute("model", "gpt-4")
        span.set_attribute("input_tokens", count_tokens(messages))
        
        response = await llm.chat(messages)
        
        span.set_attribute("output_tokens", response.usage.output_tokens)
        span.set_attribute("duration_ms", elapsed_ms)

# → 可以在 Jaeger/Zipkin 中查看完整的调用链

# ✅ 结构化的 logs
import logging
from opentelemetry._logs import set_logger_provider

logger = logging.getLogger("chrys.agent")
logger.info("LLM call finished", extra={
    "model": "gpt-4",
    "input_tokens": 1500,
    "output_tokens": 200,
    "cost_usd": 0.012,
})

# → 可以在 ELK/Datadog 中查询和分析

# ✅ 实时的 metrics
from opentelemetry import metrics

meter = metrics.get_meter("chrys.agent")
llm_call_counter = meter.create_counter("llm_calls_total")
token_usage_histogram = meter.create_histogram("token_usage")

llm_call_counter.add(1, {"model": "gpt-4"})
token_usage_histogram.record(1700, {"model": "gpt-4"})

# → 可以在 Prometheus/Grafana 中监控
```

---

## 核心设计：三个关键决策

### 决策 1: 全局 TelemetryGate

```python
class TelemetryGate:
    """Process-wide switch shared by Chrys telemetry emitters."""
    
    def __init__(self) -> None:
        self.enabled: bool = False
        self.sensitive_data: bool = False
    
    @property
    def sensitive_enabled(self) -> bool:
        """Allow sensitive capture only when both flags are on."""
        return self.enabled and self.sensitive_data


TELEMETRY_GATE = TelemetryGate()


def configure_telemetry(*, enabled: bool, sensitive_data: bool = False) -> None:
    """Flip the telemetry gate."""
    TELEMETRY_GATE.enabled = enabled
    TELEMETRY_GATE.sensitive_data = sensitive_data
```

**为什么？**

- **统一开关**：所有遥测发射器都检查同一个 gate
- **敏感数据保护**：需要两个开关都打开才记录敏感信息（工具参数、结果）
- **运行时切换**：可以在不重启的情况下开启/关闭遥测

**使用模式**：

```python
# 在 kernel/loop.py 中
if TELEMETRY_GATE.enabled:
    # 记录非敏感的 span 属性
    span.set_attribute("model", model_id)
    span.set_attribute("tool_count", len(tool_calls))
    
    if TELEMETRY_GATE.sensitive_enabled:
        # 记录敏感信息（工具参数、结果）
        span.set_attribute("tool_args", json.dumps(args))
        span.set_attribute("tool_result", result[:200])
```

---

### 决策 2: 零配置的本地回退

```python
def setup_otel(settings: Settings, session_dir: Path | None = None) -> None:
    """Setup OpenTelemetry providers and exporters."""
    
    # 先撤销之前的设置
    _revoke_prior_setup()
    
    # 检查是否有 OTLP 端点配置
    endpoint = _base_endpoint(settings.otel_endpoint)
    has_otlp_config = _otlp_endpoint_configured(settings.otel_endpoint)
    
    if has_otlp_config:
        # 有 OTLP 配置：使用 OTLP exporter
        exporters = _create_otlp_exporters(
            protocol=settings.otel_protocol,
            traces_endpoint=f"{endpoint}/v1/traces",
            logs_endpoint=f"{endpoint}/v1/logs",
            metrics_endpoint=f"{endpoint}/v1/metrics",
        )
    else:
        # 没有 OTLP 配置：使用本地 JSONL 文件
        if session_dir:
            exporters = [
                JsonlSpanExporter(session_dir / "otel" / "traces.jsonl"),
                JsonlLogExporter(session_dir / "otel" / "logs.jsonl"),
            ]
        else:
            exporters = []
    
    if not exporters:
        # 没有 exporter，禁用遥测
        configure_telemetry(enabled=False)
        return
    
    # 配置 provider
    _setup_providers(exporters, settings)
    
    # 开启遥测 gate
    configure_telemetry(
        enabled=settings.otel_enabled,
        sensitive_data=settings.otel_sensitive_data,
    )
```

**为什么？**

- **零配置即可用**：不需要配置 OTLP collector 也能收集遥测数据
- **本地文件安全**：JSONL 文件只在会话目录，不会泄露到外部
- **可选升级**：用户可以随时切换到 OTLP exporter

**本地文件格式**：

```jsonl
// session_dir/otel/traces.jsonl
{"trace_id":"abc123","span_id":"def456","name":"llm_call","start_time":1696123456.789,"end_time":1696123461.234,"attributes":{"model":"gpt-4","input_tokens":1500}}
{"trace_id":"abc123","span_id":"ghi789","name":"tool_execution","start_time":1696123461.235,"end_time":1696123461.456,"attributes":{"tool_name":"read_file"}}

// session_dir/otel/logs.jsonl
{"timestamp":1696123456.789,"level":"INFO","body":"LLM call started","attributes":{"model":"gpt-4"}}
{"timestamp":1696123461.234,"level":"INFO","body":"LLM call finished","attributes":{"output_tokens":200,"duration_ms":4445}}
```

---

### 决策 3: 可撤销的生命周期

```python
# 跟踪已安装的状态
_INSTALLED_ROOT_HANDLER: logging.Handler | None = None
_INSTALLED_LOGGER_PROVIDER: Any = None
_INSTALLED_TRACER_PROVIDER: Any = None
_PRIOR_CHRYS_LOGGER_LEVEL: int | None = None


def _revoke_prior_setup() -> None:
    """Revoke whatever a previous setup_otel attached."""
    
    global _INSTALLED_ROOT_HANDLER, _INSTALLED_LOGGER_PROVIDER
    global _INSTALLED_TRACER_PROVIDER, _PRIOR_CHRYS_LOGGER_LEVEL
    
    # 移除 root logger 的 handler
    if _INSTALLED_ROOT_HANDLER is not None:
        logging.getLogger().removeHandler(_INSTALLED_ROOT_HANDLER)
        _INSTALLED_ROOT_HANDLER = None
    
    # 恢复 chrys logger 的级别
    if _PRIOR_CHRYS_LOGGER_LEVEL is not None:
        logging.getLogger("chrys").setLevel(_PRIOR_CHRYS_LOGGER_LEVEL)
        _PRIOR_CHRYS_LOGGER_LEVEL = None
    
    # 关闭遥测 gate
    configure_telemetry(enabled=False, sensitive_data=False)
    
    # 注意：OTel global provider 是进程级别的，无法撤销
    # set_*_provider 只能调用一次，第二次会警告并保持第一个
```

**为什么？**

- **允许重新配置**：用户可能在运行中改变遥测设置
- **防止资源泄露**：旧的 handler 和 provider 必须清理
- **明确的限制**：OTel global provider 无法撤销，这是 SDK 的限制

---

## 源码解析

### OtelSessionSink

```python
class OtelSessionSink:
    """Process-wide singleton for session-scoped telemetry state."""
    
    def __init__(self):
        self._current_session_id: str | None = None
        self._session_attributes: dict[str, Any] = {}
    
    def set_session(self, session_id: str, **attributes) -> None:
        """Set the current session context."""
        self._current_session_id = session_id
        self._session_attributes = {
            "session_id": session_id,
            **attributes,
        }
    
    def clear_session(self) -> None:
        """Clear the current session context."""
        self._current_session_id = None
        self._session_attributes.clear()
    
    def get_session_attributes(self) -> dict[str, Any]:
        """Get attributes to attach to all spans in this session."""
        return self._session_attributes.copy()


_SINK: OtelSessionSink | None = None


def set_otel_sink(sink: OtelSessionSink) -> None:
    """Install the process-wide sink singleton."""
    global _SINK
    _SINK = sink


def get_otel_sink() -> OtelSessionSink:
    """Get the process-wide sink singleton."""
    if _SINK is None:
        raise RuntimeError("OTel sink not initialized")
    return _SINK
```

### Exporter 配置

```python
def _create_otlp_exporters(
    *,
    protocol: str = "grpc",
    traces_endpoint: str | None = None,
    traces_headers: dict[str, str] | None = None,
    logs_endpoint: str | None = None,
    logs_headers: dict[str, str] | None = None,
    metrics_endpoint: str | None = None,
    metrics_headers: dict[str, str] | None = None,
) -> list[Any]:
    """Build exporters for the configured signals and selected transport."""
    
    exporters: list[Any] = []
    
    if not any([traces_endpoint, logs_endpoint, metrics_endpoint]):
        return exporters
    
    if protocol == "grpc":
        # gRPC 协议
        try:
            from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
            from opentelemetry.exporter.otlp.proto.grpc._log_exporter import OTLPLogExporter
            from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
        except ImportError as e:
            raise ImportError(
                "OTLP gRPC exporter not installed. "
                "Install with: pip install opentelemetry-exporter-otlp-proto-grpc"
            ) from e
        
        if traces_endpoint:
            exporters.append(OTLPSpanExporter(
                endpoint=traces_endpoint,
                headers=traces_headers or {},
            ))
        
        if logs_endpoint:
            exporters.append(OTLPLogExporter(
                endpoint=logs_endpoint,
                headers=logs_headers or {},
            ))
        
        if metrics_endpoint:
            exporters.append(OTLPMetricExporter(
                endpoint=metrics_endpoint,
                headers=metrics_headers or {},
            ))
    
    elif protocol == "http/protobuf":
        # HTTP/Protobuf 协议
        try:
            from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
            from opentelemetry.exporter.otlp.proto.http._log_exporter import OTLPLogExporter
            from opentelemetry.exporter.otlp.proto.http.metric_exporter import OTLPMetricExporter
        except ImportError as e:
            raise ImportError(
                "OTLP HTTP exporter not installed. "
                "Install with: pip install opentelemetry-exporter-otlp-proto-http"
            ) from e
        
        # 类似的 exporter 创建逻辑...
    
    return exporters
```

---

## 实际使用场景

### 场景 1: 配置遥测

```yaml
# ~/.chrys/settings.yaml
otel_enabled: true
otel_sensitive_data: false  # 不记录工具参数和结果
otel_endpoint: "http://localhost:4318"  # OTLP HTTP 端点
otel_protocol: "http/protobuf"
```

```python
# 启动时
from chrys.foundation.observability.setup import setup_otel

settings = load_settings()
setup_otel(settings, session_dir=Path("/path/to/session"))

# 现在所有遥测数据都会发送到 http://localhost:4318
```

### 场景 2: 在代码中添加 span

```python
from opentelemetry import trace

tracer = trace.get_tracer("chrys.kernel.loop")

async def run_tool_loop(messages, tools):
    sink = get_otel_sink()
    
    with tracer.start_as_current_span(
        "tool_loop",
        attributes=sink.get_session_attributes(),
    ) as span:
        span.set_attribute("tool_count", len(tools))
        
        iteration = 0
        while True:
            iteration += 1
            
            with tracer.start_as_current_span(f"iteration_{iteration}"):
                # LLM 调用
                with tracer.start_as_current_span("llm_call") as llm_span:
                    llm_span.set_attribute("model", model_id)
                    response = await llm.chat(messages)
                    llm_span.set_attribute("output_tokens", response.usage.output_tokens)
                
                # 工具执行
                if response.tool_calls:
                    for tool_call in response.tool_calls:
                        with tracer.start_as_current_span(f"tool_{tool_call.name}") as tool_span:
                            result = await execute_tool(tool_call)
                            tool_span.set_attribute("result_length", len(str(result)))
                
                if not response.tool_calls:
                    break
```

### 场景 3: 分析遥测数据

```bash
# 使用 Jaeger 查看 traces
docker run -d --name jaeger \
  -p 16686:16686 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest

# 配置 Chrys 发送到 Jaeger
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
uv run icode

# 打开 http://localhost:16686 查看调用链

# 或者查看本地 JSONL 文件
jq . /path/to/session/otel/traces.jsonl | less
```

---

## 和其他模块的关系

```
observability 被以下模块使用：
├── kernel/instrumentation.py（Agent/tool loop 的 span 和 metric）
├── service/llm/instrumented.py（LLM 调用的 span）
├── service/mcp/owned.py（MCP 操作的 span）
├── orchestration/engine/run/（turn 级别的 span）
└── foundation/trajectory/envelope.py（与 trajectory 系统的集成）
```

---

## 如果你要自己实现

### 最小版本（只记录到本地文件）

```python
import json
import time
from pathlib import Path

class SimpleTracer:
    def __init__(self, log_file: Path):
        self.log_file = log_file
        self.log_file.parent.mkdir(parents=True, exist_ok=True)
    
    def start_span(self, name: str, **attributes):
        return SimpleSpan(name, attributes, self.log_file)

class SimpleSpan:
    def __init__(self, name: str, attributes: dict, log_file: Path):
        self.name = name
        self.attributes = attributes
        self.log_file = log_file
        self.start_time = time.time()
    
    def __enter__(self):
        return self
    
    def __exit__(self, *exc):
        end_time = time.time()
        record = {
            "name": self.name,
            "start_time": self.start_time,
            "end_time": end_time,
            "duration_ms": int((end_time - self.start_time) * 1000),
            "attributes": self.attributes,
        }
        with open(self.log_file, "a", encoding="utf-8") as f:
            f.write(json.dumps(record) + "\n")
    
    def set_attribute(self, key: str, value):
        self.attributes[key] = value

# 使用
tracer = SimpleTracer(Path("traces.jsonl"))

with tracer.start_span("llm_call", model="gpt-4") as span:
    response = await llm.chat(messages)
    span.set_attribute("output_tokens", response.usage.output_tokens)
```

### 进阶版本

加 OpenTelemetry SDK 集成、OTLP exporter、sensitive data gate。参考 Chrys 的实现。

---

## 下一步

- **上一篇**: F11 net — 网络操作辅助工具
- **下一篇**: F13 patches — Textual 补丁
