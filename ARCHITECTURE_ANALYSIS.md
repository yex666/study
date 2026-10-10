# Chrys/iCode 架构深度分析

> Chrys（对外品牌名 **iCode** / `icode`）是一个基于 Python 3.14+ 的通用可扩展 Agent 平台，支持 Textual TUI、headless CLI 与 ACP server 三种运行形态。本文基于源码深度分析其整体架构、各项实现，并对比业界 Codex（OpenAI）与 Claude Code（Anthropic）。

---

## 目录

1. [整体架构](#1-整体架构)
2. [分层 DAG 与源码映射](#2-分层-dag-与源码映射)
3. [Kernel 层：转录正确性引擎](#3-kernel-层转录正确性引擎)
4. [Service 层：集成与策略层](#4-service-层集成与策略层)
5. [Orchestration 层：编排与生命周期](#5-orchestration-层编排与生命周期)
6. [App 层：TUI/CLI/ACP 三态](#6-app-层tuicliacp-三态)
7. [横切关注点](#7-横切关注点)
8. [与 Codex / Claude Code 的对比](#8-与-codex--claude-code-的对比)
9. [创新点与优势总结](#9-创新点与优势总结)

---

## 1. 整体架构

### 1.1 设计哲学

Chrys 的核心理念是 **"Agent 是配置，而非代码"**（README 原文）。这体现在：

- **Agent = Profile**：指令、工具、子 Agent、Skills、MCP server、Memory、Hooks 全部通过 YAML 配置组合，无需写代码即可定义新 Agent。
- **平台中立于模型**：不内置任何 AI 模型，所有模型由用户接入（OpenAI/Anthropic/DeepSeek/GLM/Mock）。
- **隐私优先**：无遥测、无分析、无崩溃上报；Web 工具默认关闭；Shell/Skill/Workflow 是用户程序，网络行为由用户决定。
- **转录正确性优先**：不信任 LLM 提供商会正确回传历史，而是自己拥有身份、排序、配对、回声消除、存储模式解析等不变量。

### 1.2 三种运行形态

| 形态 | 入口 | 审批模式 | 用户交互 |
|------|------|----------|----------|
| **TUI** | `uv run icode` | manual / auto / bypass | 全交互（ask_user、对话框） |
| **Headless CLI** | `icode run "<prompt>"` | BYPASS | 无（ask_user 从工具列表移除） |
| **ACP Server** | `icode acp` | 继承客户端 | stdio JSON-RPC |

### 1.3 启动引导

所有运行时入口调用 `orchestration/startup.py::bootstrap_runtime()`（`serve` 和 `install` 豁免）。该函数：

1. 加载设置（`load_settings()`，`Settings.from_env()` 是兼容 shim）
2. 初始化 EventBus、i18n、平台探测、vendored ripgrep
3. 应用 foundation/patches（开发环境改写 Textual site-packages，CI 仅运行时补丁）
4. 注册工作流内置模板、locale catalog
5. 返回一个未构建的引擎句柄，由调用者通过 `assemble_agent_engine` 完成

---

## 2. 分层 DAG 与源码映射

### 2.1 四层有向无环图

```
app        → {orchestration, service, kernel, foundation}
orchestration → {service, kernel, foundation}
service    → {kernel, foundation}          # 包含 src/chrys/workflows.py 门面
kernel     → {foundation}                   # kernel→kernel 用相对导入
foundation → {}                              # 无依赖
```

- 同层允许互相导入；`if TYPE_CHECKING:` 体豁免；字符串字面量 `import_module`/`__import__` 计入导入
- `import chrys` 会失败（`from chrys import __version__` 合法）
- 禁止导入（即使 TYPE_CHECKING）：`system_reminder.py` → orchestration；`invoker/sub_agents/` → `engine/`；`invoker/` → `sub_agents/`

### 2.2 源码地图

| 层 | 路径 | 职责 |
|----|------|------|
| **foundation/** | `src/chrys/foundation/` | 配置、事件、模型（session_env/workspace/turns/history_markers）、i18n、trajectory、平台、补丁、工具种类 |
| **kernel/** | `src/chrys/kernel/` | 扁平结构，无私有子包。agent、loop、client、tools、compaction、exchanges、identity、middleware。`_*.py` 为私有模块 |
| **service/** | `src/chrys/service/` | llm、profiles、tools（builtins 含 web）、mcp、skills、hooks、approval、session、state、context（compaction）、mutations、agent_middleware、acp_client、workflows（SDK/graph/scheduler/protocol/worker_host） |
| **orchestration/** | `src/chrys/orchestration/` | startup、engine（assembly/build/run/state）、invoker、sub_agents、session_host、workflows（worker client/agent nodes） |
| **app/** | `src/chrys/app/` | cli、tui（screens/widgets/behaviors/terminal）、acp、features |

---

## 3. Kernel 层：转录正确性引擎

Kernel（tier 1）是 Chrys 自有的 Agent 内核——**不是 OpenAI Agents SDK 的薄包装**，而是从零实现的 Agent 循环、工具分发、转录语法和上下文压缩原语。

### 3.1 运行时分层（外到内）

```
AgentTelemetryLayer → _AgentCore → ToolLoopLayer → ChatMiddlewareLayer → BaseChatClient → (service/llm wire client)
```

- **`Agent`**（`agent.py`）：`Agent = AgentTelemetryLayer over _AgentCore`。`Agent.run()` 是**惰性**的：`stream=False` 返回未 await 的协程；`stream=True` 同步返回 `ResponseStream.from_awaitable(...)`。在调用者驱动结果前不执行任何操作。
- **`ToolLoopLayer`**（`loop.py`，180KB / ~3700 行）：组合式工具调用循环，位于 `ChatMiddlewareLayer` 之上。拥有每次运行的函数中间件管道、分发、结果组装、续接令牌跟踪、中断恢复记录、回声消除不变量。
- **`ChatMiddlewareLayer`**（`middleware.py`）：组合式 chat 中间件管道，包裹每次模型调用。
- **`BaseChatClient`**（`client.py`）：wire client 抽象基类。`get_response` 重载是单一分发点，决定压缩准入、构建 `CompactionCallContext`、准备 wire message、调用子类实现的 `_inner_get_response`。

### 3.2 交换语法（Exchange Grammar）—— 核心创新

`exchanges.py` 定义了所有转录消费者共享的**统一转录语法**，基于访问器协议，使实时 `Message` 对象和其 `to_dict` 序列化运行**同一代码**。

**语法定义**：
- 一个 **EXCHANGE** 是包含至少一个工具调用的连续 assistant 兄弟消息的最大运行（纯文本和纯推理兄弟消息归属同一运行），后接回答它的工具输出。
- 携带 history marker 的消息是两条遍历的**硬边界**：既不是响应兄弟，也不属于输出运行。
- **result-only** assistant 消息（hosted MCP 以 assistant 角色报告结果）属于输出块，永不属于响应运行。
- 工具输出后的 call-carrying assistant 开启新交换。
- call-carrying assistant 自身的 result-type contents 是**嵌入式结果**：配对遵循逐消息规则——消息 N 的调用在消息 N 的嵌入式结果消费前注册。

**配对是策略驱动的**：`PairingPolicy` 让每个消费者声明自己的规则行：
- `NoneIdPolicy`：`IGNORE`/`UNPAIRABLE_OCCURRENCE`/`POSITIONAL_FAILURES`/`POSITIONAL`
- `EmptyIdPolicy`：`IGNORE`/`COERCED_SINGLE_KEY`/`POSITIONAL`/`UNPAIRABLE_OCCURRENCE`
- `PairingKey = tuple[Literal["call_id","image_id"], str]`：图像生成内容在自己的命名空间配对，相等字符串不会跨命名空间误配

**关键区分**：`answered_keys`（仅已消费视图）vs `result_keys`（含孤儿的库存视图）——完整性消费者读 `answered_keys`。

### 3.3 调用顺序戳（Invocation Order Stamps）

Kernel 循环是 `_chrys_tool_invocation_order`（序数）的**唯一生产者**，在响应到达时戳记到每个非信息性 `function_call`（模型发射顺序），包括分发过滤器丢弃的调用（已回答、重复 id、无工具批次）——**所以持久化编号永远不会偏离管道编号**。

`_finalize` 是最终 `ChatResponse` 的单一重建权威：调用 `ChatResponse.from_updates` 处理响应级字段，然后**用循环组装的 `Message` 对象覆写 `response.messages`**。重新合并原始片段会用新 `Content` 对象重建多片段函数调用，丢弃 kernel 戳记的序数、工具种类/上下文和折叠的结果元数据。

### 3.4 弱身份注册（WeakIdentityRegistry）

`identity.py`（96 行）解决了基于 `id()` 的簿记的根本问题：持久化的裸 `id()` 整数既不保持对象存活也不注意其死亡——一旦被回收，回收的地址可以冒充它。

- `WeakIdentityRegistry` 将 `id(obj)` 映射到 `weakref.ref`
- 成员资格（`obj in registry`）仅在存储的引用仍指向那个确切对象时成立
- 复制永不是成员；死成员永不匹配；地址重用按构造无害
- `ContentList(list[Content])` 存在是因为内置 `list` 不支持弱引用，而多个关键身份是 LIST 身份（`id(msg.contents)` 被压缩锚点和摘要缓存捕获）

### 3.5 内容身份共享对象

- **出站消息** 是 `_wire_message_view`（新鲜包装器和 contents 列表；共享 `Content` 对象和 `additional_properties`）
- **落地消息** 是 `_message_snapshot` 副本
- 永不循环↔client 共享 Message 包装器，也不在匹配回状态的路径上重建 Contents
- 每个wire→state匹配器需要内容身份层（注入/排除锚点、压缩的 `_state_correspondence`）

### 3.6 回声消除（Echo Removal）

按对象身份（而非 call_id）丢弃对话已持有的 Content 对象：
- 在流式更新最终处理边界（`_strip_echoed_update`）
- 在响应落地时（`_land_response_contents`）
- 永不变异传入消息（浅拷贝它）
- 按每消息 `_chrys_echo_content_stripped` 溯源丢弃回声清空的流式壳（非调用级标志）
- 豁免 usage content

更新组装将连续同调用片段合并为**新对象**，所以参与合并的回声历史对象会洗白身份。在组装前过滤意味着洗白的合并副本永不存在。

### 3.7 存储模式解析

`resolve_storage_mode_and_handles` 读取选项中的 `store`（wire 真相：`extra_body` 优先于命名参数；`store: null` 表示提供者默认），client kwargs 只能升级到 service-side，`force_stateless=client.FORCES_STATELESS` 否决 service-side。

`CONVERSATION_HANDLE_KEYS = ("conversation_id", "previous_response_id", "conversation")`——每种拼写都算，顶层和 `extra_body` 内都算。`strip_invalidated_conversation_handles` 是单一 wire 节流点。

### 3.8 压缩原语（Compaction Primitives）

Kernel 拥有原语；四阶段算法在 `service/context/compaction`。Kernel 模块提供：
- 分组注解键（`GROUP_ANNOTATION_KEY`、`GROUP_ID_KEY` 等）
- `group_messages`——计算组跨度。交换边界来自共享交换语法；跨度是每个交换的次级、压缩拥有的分区
- `_exchange_spans`——一旦交换有结果，调用和结果必须共享一个组
- `CompactionStrategy` Protocol：`__call__(self, messages, context) -> bool`
- `CompactionCallContext`——故意最小化：`last_words_completer`、`tool_definition_tokens`、`request_overhead_tokens`。无 client 对象，无按引用的 options
- `LastWordsCompleter` Protocol——作用域自然补全侧调用。实现强制 `tool_choice="none"`，剥离续接句柄，通过 `_inner_get_response`（非 `get_response`）运行以永不重入压缩

### 3.9 LoopRecorder 日志

一个 pending/sealed 交换状态机，每个已回答槽是不可撤销的提交。Whole-run 重试永不重放已提交的调用。Hosted 调用绕过循环记录器，所以 `ResponseValidationMiddleware` 记录它们（`hosted_commits`），且没有 lane 在阻塞 hosted 工作后重发请求——除非活跃续接令牌使其成为轮询。

### 3.10 私有模块

| 模块 | 职责 |
|------|------|
| `_types.py` (107KB) | `Message`/`ChatResponse`/`ChatResponseUpdate`/`AgentResponse`/`ResponseStream`/`ChatOptions` 定义。`Message.contents` 是 setter 强制转换为 `ContentList` 的属性 |
| `_content.py` (71KB) | `Content` dataclass 及所有类型变体（function_call、function_result、text_reasoning、hosted_tool_* 等），`UsageDetails`、`Annotation` |
| `_serialization.py` (19KB) | `SerializationMixin`——`to_dict`/`from_dict` 契约 |
| `_null_overlay.py` (46KB) | **静态分类引擎**，恢复 Pydantic `model_dump(exclude_none=True)` 剥离的调用者显式 `None` 工具参数。构建时将每个输入模型的注解图分类为 `FULL`/`TOP_LEVEL`/`OFF` 层 |
| `_tool_arg_errors.py` (74KB) | `ValidationError` 的紧凑、模型可操作诊断。有界工作预算防止敌意或指数参数图 |
| `_result_ceiling.py` (8KB) | 幂等兜底，截断过大的本地工具结果到 token 上限 |
| `_tool_expansion.py` (0.8KB) | 进程全局缝隙，kernel 不导入 service 层 MCP 类型 |

---

## 4. Service 层：集成与策略层

Service（tier 2）定义 *如何* 与提供者对话、*哪些* 工具存在、*何时* 压缩、*是否* 需要审批、*如何* 执行用户工作流。

### 4.1 LLM 提供者集成

**核心设计——api_style 正交轴**：提供者是两个正交选择——wire 传输（openai vs anthropic SDK）和请求形状（`chat_completions` vs `responses`）。新变体是 `api_style`，永不是 provider id。

- `openai`：支持 `api_style: chat_completions | responses`
- `deepseek-openai`：子类化 Chrys 拥有的 OpenAI wire client，仅覆写策略
- `glm-openai`：同上
- `anthropic`：独立 wire

**Instrumented shims**（`instrumented.py`）：每个提供者通过 `create_instrumented_*` 工厂组合最终栈：

```
ToolLoopLayer(ChatMiddlewareLayer(_Instrumented*(ChatTelemetryLayer + Raw* wire client)))
```

捕获"中间文本"（模型在工具调用旁发出的文本，如调用 `read_file` 前的"让我读那个文件"），在工具执行前发射到回调，使 TUI 实时渲染。

**确定性连接重试守卫**：`_DeterministicConnectionRetryGuard` 子类化两个 SDK，覆写 `_sleep_for_retry` 检查 `sys.exception()`，当错误无法自愈时停止重试（如 DNS 失败不会因重试而解决）。

**HTTP client 所有权**：Chrys 而非提供者 SDK 拥有每个 httpx 池。使用 SDK 的 `DefaultAsyncHttpxClient`（保留池限制、重定向、Anthropic 的 TCP keepalive），但叠加 route-fact 钩子和可选的 raw-HTTP 日志。

### 4.2 Profiles 系统

**双注册表**：
- `AgentProfileRegistry`：Agent 人格（指令、工具、子 Agent、审批配置、Memory 文件）
- `ModelProfile` 注册表：连接配置（provider、api_style、model_id、base_url、api_key）

**四个内置 Agent**：`Code`（main-eligible）、`QA`、`Explore`（`sub_agent_only`）、`General`。用户的 `~/.chrys/agents/<Name>.yaml` 按名称遮蔽内置并继承稳定 id。退役的 `Plan` profile 被丢弃，永不重映射。

**文件系统感知规范化**：处理大小写不敏感的 macOS/Windows 文件系统、Unicode 规范化差异（NFD vs NFC）。将每个加载文件移动到 `<name>.yaml`，将真正冲突隔离为 `<file>.conflict`。

### 4.3 工具内置（Tools Builtins）

| 工具 | 实现 |
|------|------|
| **Shell** | 实例化工具持有 `SessionEnvironment`。Unix 上命令在 PTY（`TERM=dumb`）上运行，stdout/stderr 附加。stdin 是 `/dev/null`，PTY 不是控制终端。输出按 1:2 头:尾比例截断；完整结果溢出到会话目录 |
| **Shell 过滤** | `ShellCommandFilter`、`READONLY`/`UNRESTRICTED` 预设、`is_safe_readonly_command`、`parse_commands`。`Explore` profile 用只读预设限制为 `ls, git log, git diff, find, wc` |
| **Web** | `assemble_web_tools` 是每个构建路径调用的唯一装配。Web 配置错误仅丢弃 web 工具（`web_tools_unavailable` 警告），永不影响 Agent。所有出口经过 `http.py`（IP 固定） |
| **文件系统** | read/write/edit，路径模糊校正仅用于只读工具 |
| **ask_user** | 交互式提问；headless 模式从工具列表移除 |
| **doc_converter** | PDF/DOCX/PPTX/XLSX 转换 |

### 4.4 MCP 集成

**连接缓存**（`cache.py`）：`MCPConnectionCache` 存储成功连接的 `MCPTool` 实例，为每个消费者提供新鲜的 `FunctionTool` 包装器。共享实时传输避免重复 connect/list 延迟，同时保持包装器级可变状态隔离。

**渐进式披露**（`_progressive.py`）：`_ProgressiveMCPExposure` 提供每适配器懒加载。缓存 `MCPTool` 保持完整目录的真实来源；此包装器仅拥有稳定控制工具（`list`、`load`、`unload`）和初始 `always_load` 克隆。加载状态从调用的实时工具列表派生，所以一个适配器/运行不会泄漏暴露状态到共享同一缓存会话的另一个消费者。

**自有传输**：Chrys 拥有自己的 stdio（`_stdio_transport.py`）和 HTTP（`_http_transport.py`）传输，而非使用 MCP SDK 默认——容忍 banner 捕获、硬超时限制不合作的 server、每请求结果上限。

### 4.5 Skills 系统

遵循 [Agent Skills 规范](https://agentskills.io/specification)：
- **发现**：扫描 skill 目录最多 2 层深。skill 是包含 `SKILL.md`（YAML frontmatter）的目录。路径遍历和符号链接逃逸守卫
- **脚本运行器**：子进程、异步。解释器按扩展名解析：`.js`/`.mjs` → `node`，`.ts` → `npx tsx`，Python 脚本优先 `uv run`（支持 PEP 723 内联脚本元数据）
- **截断**：脚本结果默认 8k tokens；截断将完整文本溢出到会话 `tool_results/`

### 4.6 Hooks 系统

外部子进程，来自 `<config_dir>/hooks/hooks.{yaml,yml,json}` 和项目 `.chrys/hooks/`（项目先运行；其块结束链）。

- 只有 `before_tool_call` 和 `user_prompt_submit` 可阻塞
- `before_tool_call` 可拒绝或重写参数
- `user_prompt_submit` 可注入 system-reminder 上下文
- `after_tool_call` 和 `tool_error` 添加额外上下文
- **Hooks 永不授予或绕过审批**
- `match.tool_kind` 是 kind-only；`match.args` 键是字面参数名

在途 hook 任务分为"本轮"/"本会话"/"后台"，所以 drain 可以只针对等待的工作。

### 4.7 审批系统

**三种模式**：`MANUAL`（每个 require 规则工具显示对话框）、`AUTO`（LLM 判断；仅标记的调用显示对话框）、`BYPASS`（所有工具静默自动批准）。

**解析链**（`_resolve_rule`，首次匹配胜出）：
1. 会话级覆盖（如果 `user_can_override`）
2. `kind.tool_name` 限定覆盖（在最长已知 kind 前缀处分割，非 `rsplit`）
3. 裸精确工具名覆盖
4. Kind-only 覆盖
5. `run_skill_script` 空白/skill kind → require
6. Web kinds → require
7. Profile 默认

无效规则值 fail-closed 到 `require`。模式仅存于 `ActiveSession`——永不在 session 文件/workflow checkpoint/draft 中——跨 `/new` 和 restore 存活，由 chat 和 workflows 共享。

### 4.8 四阶段上下文压缩

触发阈值：context window − max output tokens − max(5k, 5% of window)，下限钳制。

| 阶段 | 操作 |
|------|------|
| **Phase 1** | 旧轮次工具结果摘要化（保留调用参数，用一行摘要替换结果） |
| **Phase 2** | 旧轮次工具调用组完全移除（参数+结果）。每轮仅用户/助手文本存活 |
| **Phase 3** | 当 Phase 1/2 无法达预算时，压缩文本密集的已完成轮次 |
| **Phase 4** | 当前轮次所有工具调用和内联助手文本消息一次性丢弃，替换为 `<system-reminder>[LAST_WORDS]…</system-reminder>` 块。LLM 生成（无程序化回退）；同一轮次内多次 Phase 4 触发时，前次 note + 新丢弃的工作回馈给生成器，使 note **累积**洞察而无需从头重摘要 |

`ExclusionLedger` 跟踪内容身份锚点，防止压缩丢弃对话仍引用的内容。

### 4.9 Agent 中间件

| 中间件 | 职责 |
|--------|------|
| `SystemReminderMiddleware` | 每次调用注入 `<system-reminder>` 标签。轮次提醒在 `prepare_turn()` 快照。仅 user-role content 转义；tool-result content 原样传递（不可信）。拥有 Phase-4 LAST_WORDS 注入 |
| `ResponseValidationMiddleware` | 对畸形响应重试（空 content、空白文本、vLLM/量化模型泄漏的工具调用标记）。位于最内层。服务存储模式 raise `RetryableResponseValidationError` 到 whole-run 边界（永不原地重放） |
| `InjectionMiddleware` | 交付工具循环中发送的 mid-loop 用户消息。一个 drain = 一个不可变批次，在首次 `await` 前注册用于重试重放 |

### 4.10 变更追踪（Mutations）

**GitDiffCalibrator**（`git_calibrator.py`）：每 shell 命令隐式变更检测，使用有界 Git 状态。`ImplicitChange` 记录 `before_data`、`before_skip`（过大的旧 blob 暂扣）、`before_symlink`（Git mode 120000）。Git 作用域是 `命令 cwd ∪ workspace.primary_cwd`，故意不钳制到 workspace——monorepo 命令从根运行。

### 4.11 工作流 SDK——架构上最独特的特性

**双进程架构**：
- **主进程**：`scheduler.py`（纯、确定性——到达事件进、决策出、无 I/O）、`graph.py`、`admission.py` 等
- **Worker host**（`worker_host.py`）：独立进程，在任意 Python ≥ 3.9 上启动为 `<interpreter> worker_host.py <sdk_dir>`。故意自包含：仅标准库，不导入 chrys，3.9 语法和运行时 API

**引导顺序**：
1. `dup(1)` 创建私有协议写入 fd
2. `dup2` fd 1 和 2 到捕获管道，使 `os.write(1, ...)`、C 扩展和孙进程永不触及协议通道
3. 替换 `sys.stdout`/`sys.stderr` 为上下文本地捕获
4. 将注入的 SDK 工件前置到 `sys.path` 并导入 `chrys.workflows`
5. 用一个读取线程、一个写入线程、一个 asyncio 循环、一个同步体卸载池、一个独立求值池服务请求

**SDK**（`sdk/`）：11 个公共名称（`Answer`、`BuilderScope`、`NodeContext`、`NodeHandle`、`Option`、`Question`、`Retry`、`SourceValue`、`Workflow`、`WorkflowBuilder`、`WorkflowValue`）。纯标准库，Python 3.9 语法。

**意义**：用户编写的工作流文件在用户自己的解释器中运行，而非 Chrys 的。这让工作流可以使用用户实际的项目依赖。

### 4.12 会话管理

- **配置/数据目录**：`~/.chrys`（macOS/Linux）、`%APPDATA%/chrys`（Windows）
- **每会话**：`session.json`、`session.json.bak`、崩溃边车 `session.recovery.json`；`<session_root>/sessions/.locks/<short-id>.active.lock` 标记拥有进程
- **恢复边车**：轮次中写入，每次主保存删除，永不属于 `.bak`/快照愈合链。仅在 `prefer_recovery=True`、拥有活跃锁或发现未持有时考虑；当比主文件新或主文件缺失/不可读时胜出
- **启动恢复**（`-s`）：事务性——仅在观察到 `SessionRestored` 后成功。失败时抑制持久化，拆除部分绑定，重置到新鲜基线
- **Fork**：复制整个会话目录；嵌入会话 id 或绝对路径的每会话文件需在 `state/_fork.py` 处理

### 4.13 状态序列化器

三类提供者状态键：
- `_COPY_AS_DICT_KEYS`：如 `LAST_USAGE_KEY`——按 dict 复制
- `_PASS_THROUGH_OPTIONAL_KEYS`：仅在真值时保存——`chrys_mutations`、`last_words` 等
- `_PASS_THROUGH_NULLABLE_KEYS`：`None` 和 `0` 语义不同——如 `CATALOG_POINTER_RECORD_COUNT_STATE_KEY`

新提供者状态键必须在键表中，否则保存静默丢弃。

---

## 5. Orchestration 层：编排与生命周期

Orchestration（tier 3）协调引擎、调用器、子 Agent、会话宿主和工作流。

### 5.1 引擎装配（assemble_agent_engine）

`orchestration/engine/assembly.py::assemble_agent_engine` 是**唯一**引擎工厂（`agent_engine` fixture 是唯一其他允许的调用者）。它连接约 12 个长生命周期组件：

1. `ActiveSession`——会话身份、锁所有权、审批模式
2. `CurrentAgent`——拥有一个已安装构建（`LoadedAgent`）；关闭后为 None
3. `TurnRuntimeState`——拥有 `ExecutionLease`
4. `LifecyclePermits`——`session_generation`/`build_generation`/`load_generation` 时钟
5. `SessionWriter`——会话保存和恢复检查点
6. `AgentLoader`——候选构建、安装、构建资源释放
7. `SessionLifecycle`——start/new/restore/fork/clear/delete/shutdown
8. `RollbackController`——轮次快照管理和回滚
9. `TurnCoordinator`——面向事件的门面
10. `RuntimeControls`——profile 切换、model 切换
11. `WorkflowCoordinator`——工作流准入和路由
12. `EngineStateMachine`——FSM（`UNINITIALIZED → IDLE → RUNNING → PENDING_RETRY / AWAITING_SUB_AGENTS → INTERRUPTED / FAILED`）

**关键接线规则**：
- 组件永不被给予引擎作为宿主
- 构建代码返回值而非写入引擎状态
- Handler 在首次构建前订阅，使设置/profile 事件可恢复失败的启动
- 每次重建替换 `engine.current.loaded`；调用者每次使用 `current.require_loaded()`，永不在 `await` 间持有

### 5.2 三条构建路径

三条路径各自构建 `Agent` + 绑定 + 支持基础设施。工具/MCP/skill/memory/web/start-hook 接线必须落在所有三条。

| 路径 | 位置 | 所有者 | 策略 |
|------|------|--------|------|
| **主构建** | `engine/build/builder.py::build_agent` | `AgentLoader` | 返回 `AgentBuildResult`（`PreparedAgent`、`Conversation`、`Agent`、`TurnBindings`） |
| **子 Agent 构建** | `sub_agents/tools.py::SubAgentTools` | 父工具循环 | 全局 `max_total_concurrency` + 每 Agent `max_concurrency` |
| **工作流节点构建** | `workflows/agent_node_build.py::build_kernel_node` | 调度器 | 镜像 `build_agent`，headless 移除 `ask_user` |

**为何三条路径**：三种调用者有不同所有权模型。主轮次拥有 `ExecutionLease` 和用户 FSM。子 Agent 是父工具循环拥有的工具。工作流节点由调度器拥有，从 `FailureReport` 跨 pass 决策，从修复的历史（kernel）或新会话（ACP）恢复失败的 pass。

### 5.3 ExecutionLease——单一互斥原语

`ExecutionLease` 是轮次运行任务从准入到最终保存的单一所有者，以及工作流运行的排他所有者。

**两种顶层工作**：
- **Turn**：用户驱动的 agent pass。跟踪 `_run_task`、`current_run_scope`、`pending_retry`、每运行注入窗口
- **Workflow run**：工作流驱动的运行。全局最多一个，仅在无轮次活跃时准入

**双向互斥**：工作流运行持有租约时，每个轮次准入和重建被拒绝（`refuse_while_workflow_active`）；每个会话族操作（new/clear/fork/restore/delete/rollback）被拒绝。反之会话操作进行中时工作流运行被拒绝。

**准入令牌**：`PromptAdmissionScope`（冻结 dataclass，含 `admission_id`、`kind`、`session_generation`、`build_generation`、`preparation_trace`）。在每个检查点验证。

**代际围栏**：
- `session_generation`——会话转换时递增
- `build_generation`——成功构建时递增
- `load_generation`——agent-load 尝试时递增
- `conversation_revision`——单调新鲜/重试生命周期修订
- `run_generation`——每运行递增

确保在 `await` 之前捕获的准入在 `await` 期间发生的重建/会话切换后无效。

### 5.4 轮次 Pass 机制

`TurnCoordinator → TurnRunner → TurnBindings → KernelConversation.run(RunRequest)`

**准入流程**（`coordinator.py::on_user_message`）：
1. `refuse_while_workflow_active`——工作流拥有租约则拒绝
2. `begin_inflight_injection`——跟踪注入标识的提交
3. 开启 `PreparationTrace`
4. 等待 `agent_load_idle`
5. 检查 `current.loaded`
6. 如 FSM `is_running()`，通过 `ActiveTurnInjector.inject` 注入
7. 等待前次运行任务清理
8. 预留 `PromptAdmissionScope(kind="fresh")`
9. 评估 `user_prompt_submit` hooks
10. 准备用户 contents
11. 最终取消检查
12. `promote_fresh_admission_to_run`——创建 `CurrentRunScope`
13. `fsm.try_transition(Trigger.USER_MESSAGE)`

**KernelConversation pass**（`invoker/kernel.py::run`）：
1. `validate(request)`——准入无变异、hooks、历史访问、进程创建
2. `invalidate_continuation()`——提升 `_generation`、清除令牌
3. 分配 `PassHandle`
4. 创建 `_ActivePass`、绑定到 `Conversation`
5. 设置 `current_invocation_origin` 和 `invocation_routing_key` 上下文变量
6. 重置 `LoopRecorder`
7. 运行 `AttemptRunner.run`
8. 成功/取消/异常分别处理
9. 构建 `PassEvidence`
10. 失败时铸造 `ContinuationTicket` 并返回 `Failed`
11. 成功时返回 `Ok`

### 5.5 续接令牌（Continuation Tickets）

`ContinuationTicket` 是冻结的、单次使用的、身份承载的令牌。`continuation_is_live(ticket, origin)` 检查：
- `not owner.closing`
- `ticket is self._ticket`（身份，非相等）
- `ticket.state_generation == self._generation`
- `origin.same_invocation(self._ticket_origin)`

`TurnResumePolicy`（`run/resume.py`）为 fresh/continue/retry 意图构建 `RunRequest`。重试时如果令牌不再活跃，回退到 `RunIntent.CONTINUE`——从历史继续而非重放已提交调用。

**重试 lane**：
- 主轮次本地存储：kernel `WireRetryPolicy` 原地重试每次 wire 调用；流式停顿耗尽后同一 `agent.run()` 回退到非流式请求，新鲜 N——每逻辑调用最多 3N+2 个提供者调用
- 主轮次服务端存储：无 wire 策略；`AttemptRunner` 运行 whole-run `StreamRetryLoop`；仅在停顿耗尽时一次非流式 `agent.run()` 尝试，无新预算
- 子 Agent：停顿耗尽时 raise（无回退）到 shell 的 pause/abort 路径
- 工作流节点：取主轮次的 lane

### 5.6 InvocationOrigin 与事件追踪

`InvocationOrigin` 是不可变冻结 dataclass，标识一个逻辑调用：
- `kind`：`turn | sub_agent | workflow_node`
- `session_id`
- `invocation_id`：新鲜 `new_analytics_id()`
- 父链（root origin）

**BoundEmitter**（`invoker/origin.py`）：在创建时绑定，永不在发布时推断。验证 `event.session_id` 匹配 `origin.session_id`；`InvocationEvent` 的 `event.origin == self.origin`。

**不变量**：永不从环境上下文派生 origin。在首次 `await` 前绑定 `BoundEmitter`。共享 client 回调仅通过 `invocation_routing_key` 查找已注册的发布者。

### 5.7 子 Agent Shell

`SubAgentToolShell` 是子 Agent 的唯一 `OperationBinding` 和唯一 pause/Retry/Abort 循环：
- 无 ACP 协议代码依赖
- 在附加后端策略前开启配额、写入器和 start-hook 作用域
- `human_failure_decisions=False`（headless chat、每个工作流节点）时失败即结束，向调用者模型返回 `Error: sub-agent 'X' failed — …`，无待处理记录
- 仅 TUI/ACP chat 暂停等待 Retry/Abort 卡
- Kernel 子 Agent 仅返回最后一个工具/hosted 输出之后的文本（`FinalSegment.of`），而非 `response.text`

### 5.8 ChrysSessionHost

为非 TUI 调用者拥有一个 event bus + engine + session。是 `icode run`、`icode workflow run` 和 ACP 会话的宿主。

- **Headless 默认**：`ApprovalMode.BYPASS`、`allow_user_interaction=False`（从工具列表移除 `ask_user`）
- **运行锁**：`asyncio.Lock` 防止并发轮次
- **事件流**：`iter_run_events` 自动启动，产出后端事件直到引擎运行完成
- **恢复**：事务性——仅在观察到 `SessionRestored` 后成功

### 5.9 工作流协调器

`WorkflowCoordinator` 准入流程：
1. 预留执行租约（一个全局工作流运行，永不 beside 活跃轮次）
2. 解析工作流文件，检查确认字节
3. 检查准备的环境，在新鲜 worker 上加载
4. 检查调用者期望摘要
5. 准入清单到注册表
6. 开启运行记录
7. 请求被接受，runner 启动

**幂等回复**：`_replies` OrderedDict（上限 256）记忆每个 `request_id` 的回复。重复的 `request_id` 获得相同答案且不启动任何东西——使工作流运行请求在不可靠传输上可安全重试。

---

## 6. App 层：TUI/CLI/ACP 三态

### 6.1 CLI

`app/cli/app.py::main` 是手写的 if-chain（无 argparse subparsers）；未匹配的 argv → TUI。子命令：`run`、`acp`、`serve`、`agents`、`models`、`workflow`、`trajectory`、`install`。

### 6.2 TUI（Textual 8.2.7）

**一屏架构**：`MainScreen` + 对话框。小部件永不在运行时导入 Screens；聊天 replay/toc 模型不导入 Textual；终端模拟器仅自引用。

**关键性能优化**：
- 任何 App 级样式表更新/restyle/recompose/layout 是 O(已挂载转录) 的
- 隐藏用 `widget.display = False/True`，非 `-hidden` 类
- 大历史回放计划分离的、预组合小部件森林，有界批次挂载
- `gc.freeze()` 稳定对象在空闲点，使集合跳过它们
- `GcFreezeParticipant` 注册；转录小部件加入 `_gc_freeze_render_cache_surfaces`

**Textual 陷阱规避**：
- `_on_x` 优先于 `on_x`；`prevent_default()` 仅在类间检查
- 可取消任务通过 `remove_shielded` 移除小部件（取消的移除留下僵尸小部件）
- `widget.prevent(...)` 共享一个 App 级栈——跨 `await` 持有会丢弃该消息类型
- `_running` 永不定义在小部件上（是 `MessagePump` 的）
- 实时 `TextArea` 拥有草稿——永不写回 `await` 前的快照

### 6.3 ACP Server

`app/acp/server.py`；非标准 caps 通过 `ext_method`。
- stdout 仅 JSON-RPC，日志到 stderr
- ACP 不应用 ask-user 超时（除非 `--ask-user-timeout`）
- 客户端提供的 MCP server：仅 HTTP，headers 字面（永不解析 `{{ENV_VAR}}`）
- headless/ACP 流仅转发 `_HEADLESS_RUN_EVENT_TYPES` 携带 `session_id` 的事件

---

## 7. 横切关注点

### 7.1 身份与去重

- `call_id` 跨交换递归；`message_id` `msg_N` 是位置性的；出现身份是 `_chrys_analytics_item_id`（铸造一次，由 save/restore/compaction/fork/rollback 原样复制——**永不复制时重新铸造**）
- 在任何 id 上去重前，读其生成器的唯一性范围
- `InjectionMiddleware` 用 `_content_signature`（排序键 JSON 载荷的 SHA-256）去重
- kernel 回声消除按对象身份丢弃 Content，永不按 call_id

### 7.2 轮次边界

仅通过 `foundation/models/turns.py`（`opens_turn`、`turn_slices`、`current_turn_start`）解析轮次，永不裸 `role == "user"` 或 `"continue"` 文本。轮次中添加的用户消息携带 `_injected`（创建时设置，永不在线发送）。文本去重用 `user_text_matches(kind=...)`，使注入的 note 永不匹配同词的 opener。

### 7.3 供应链与版本锁定

- **Actions**：每个第三方 `uses:` 是 40 字符 SHA + `# <version>` 注释
- **Python**：`[project]` 依赖和 extras 用精确 `==`；`pyjwt[crypto]` 和四个 `opentelemetry-*` 是注释的例外
- `uv.lock` 锁定并哈希整个闭包：在同一提交中提交
- Ruff 版本在 `pyproject.toml` 和 `.pre-commit-config.yaml`

### 7.4 测试哲学

- **守卫**：新 AST 卫生规则在 `tests/architecture/test_hygiene_*.py`；"无法变红的守卫比没有更糟——它被相信"
- **等待**：`wait_for(...)` 用于正等待；`wait_until` 仅在 bool 被断言或分支时
- **预算**：引擎轮次和冷启动进程的等待通过 `ENGINE_TURN_TIMEOUT`（15s）；pytest-timeout 在 60s 杀死
- **隔离**：每个测试在空 cwd 中开始；会话根总是每测试临时
- **证据**：Flake 修复 = 因果机制 + 确定性复现器

### 7.5 i18n

- 仅本地化用户可见显示散文；模型/工具/协议/MCP 文本、持久化标记字面量、日志/追踪保持英文
- 模块级 `NAME = msg("area.dotted_key", fallback="Exact English text")`
- 添加/更改/移动 `msg()` 后：`extract → update → translate → compile → check`

---

## 8. 与 Codex / Claude Code 的对比

### 8.1 定位对比

| 维度 | Chrys/iCode | OpenAI Codex | Claude Code |
|------|-------------|--------------|-------------|
| **开源** | Apache 2.0，完全开源 | 部分开源（CLI） | 闭源 |
| **语言** | Python 3.14+ | Rust + TypeScript | TypeScript |
| **模型** | 用户接入（多提供者） | 仅 OpenAI | 仅 Anthropic |
| **形态** | TUI + Headless CLI + ACP | CLI + Web + IDE | CLI + IDE |
| **工作流** | 用户 YAML 工作流（双进程） | 无 | 无 |
| **MCP** | 内置 + 渐进式披露 | 部分 | 内置 |
| **Skills** | Agent Skills 规范 | 无 | 无 |
| **Hooks** | YAML 配置子进程 | 无 | 无 |
| **审批** | 三模式 + LLM 判断 | 简单批准/拒绝 | 简单批准/拒绝 |
| **压缩** | 四阶段 + 累积 LAST_WORDS | 提供者管理 | 简单截断 |
| **子 Agent** | 内置（ACP + Kernel） | 无 | 无 |
| **ACP** | 原生 server | 无 | 无 |

### 8.2 架构对比

**Chrys/iCode 的独特架构特征**：

1. **自有的转录语法**：Codex 和 Claude Code 信任提供者回传历史。Chrys 定义正式的交换语法、策略驱动的配对、命名空间隔离的键、硬边界标记系统。每个转录消费者（压缩、回声消除、完整性检查、审计重放）共享一个实现。

2. **内容身份追踪**：Codex 和 Claude Code 在每次响应时创建新 `Content` 对象让 GC 回收。Chrys 通过 `WeakIdentityRegistry` 跟踪内容身份，因为回声消除、压缩锚点和摘要缓存依赖对象身份跨调用存活。`ContentList` 子类存在仅为使列表身份可弱引用。

3. **调用顺序权威**：Codex 和 Claude Code 按位置编号工具调用。Chrys 在响应到达时戳记 `_chrys_tool_invocation_order`，包括分发过滤器丢弃的调用。

4. **回声消除**：Codex 和 Claude Code 信任提供者返回的是新工作。Chrys 在记录器、分发过滤器和最终转录组装观察到它们之前，按对象身份丢弃对话持有的内容。

5. **存储模式解析**：Codex 和 Claude Code 透传 `store`。Chrys 从选项（wire 真相）、client kwargs 和 `FORCES_STATELESS` 解析 service-side vs 本地存储，然后在单一 wire 节流点剥离失效句柄。

6. **双进程工作流 SDK**：Codex 和 Claude Code 在自己进程中运行工作流逻辑。Chrys 将纯标准库 SDK 注入到用户拥有的任意 Python ≥ 3.9 解释器，带硬协议边界（JSON-RPC over 管道 fd，stdout/stderr 分离捕获）。让工作流使用用户实际的项目依赖。

7. **ExecutionLease 单一互斥**：Chrys 用一个原语仲裁轮次、工作流运行和会话族操作之间的互斥，带代际围栏确保 `await` 前捕获的准入在 `await` 期间重建/会话切换后无效。

8. **续接令牌**：Chrys 的 `ContinuationTicket` 是冻结的、单次使用的、身份承载的令牌。`continuation_is_live` 检查身份（`ticket is self._ticket`）、代际和调用 origin。重试仅在后端状态自上次 pass 以来完全未变时可重放已提交调用。

### 8.3 实现深度对比

| 特性 | Chrys/iCode | Codex | Claude Code |
|------|-------------|-------|-------------|
| **工具循环** | 自有（ToolLoopLayer + LoopRecorder） | SDK 内置 | SDK 内置 |
| **中间件** | 组合式 ChatMiddleware + FunctionMiddleware | 固定管线 | 固定管线 |
| **工具诊断** | `_tool_arg_errors.py` 有界预算 | SDK 默认 | SDK 默认 |
| **null 恢复** | `_null_overlay.py` 静态分类 | 不处理 | 不处理 |
| **结果截断** | `_result_ceiling.py` 幂等兜底 | 简单截断 | 简单截断 |
| **错误可见性** | `ModelVisibleToolError` vs 固定文本 | 全部可见 | 全部可见 |
| **HTTP 所有权** | Chrys 拥有 httpx 池 | SDK 默认 | SDK 默认 |
| **重试守卫** | 确定性连接错误停止 | 盲指数退避 | 盲指数退避 |
| **Profile 规范化** | 文件系统感知（NFD/NFC/大小写） | 不处理 | 不处理 |
| **Shell** | PTY + TERM=dumb + 1:2 头尾比 | 直接执行 | 直接执行 |
| **Git 变更** | GitDiffCalibrator 有界状态 | 简单 diff | 简单 diff |

---

## 9. 创新点与优势总结

### 9.1 核心创新

1. **转录正确性引擎**：Chrys kernel 不是一个 LLM client 包装器——它是一个转录正确性引擎，分层在模型/工具循环之上。它拥有身份、排序、配对、回声消除、存储模式解析和压缩作为一等不变量，每个都由 `tests/architecture/` 和 `tests/kernel/` 中的守卫测试支持。

2. **交换语法**：一个正式的、访问器协议驱动的语法，策略驱动的配对（None/empty id 策略、畸形 id 规范化、图像生成的命名空间隔离键）。`answered_keys` vs `result_keys` 区分已消费和库存含孤儿。

3. **弱身份注册**：对象身份成员资格注意到成员死亡。成员资格要求注册对象本身仍存活（`ref() is obj`），所以复制永不是成员、死成员永不匹配、地址重用按构造无害。

4. **内容身份共享对象**：共享 `Content` 对象循环↔client 但永不共享 `Message` 包装器。回声消除永不变异传入消息；有状态的 client 后来变异返回的消息对象永不能重写落地历史。

5. **双进程工作流 SDK**：纯标准库 SDK 注入到用户拥有的 Python ≥ 3.9 解释器，带硬协议边界。让工作流使用用户实际的项目依赖——这是业界独有的。

6. **四阶段累积压缩**：而非单次摘要传递，从工具结果摘要 → 工具组移除 → 文本轮次压缩 → 当前轮次丢弃带 LLM 生成的累积 LAST_WORDS note。

7. **渐进式 MCP 披露**：大型 MCP server 不撑爆上下文窗口——工具按需通过 `list`/`load`/`unload` 加载，每适配器状态从实时工具列表派生。

8. **api_style 正交轴**：`deepseek` 和 `glm` 不是独立提供者，而是 `deepseek-openai` 和 `glm-openai`——OpenAI 兼容传输子类化 Chrys 拥有的 OpenAI wire client，仅覆写策略。

9. **确定性连接重试守卫**：子类化两个提供者 SDK，在重试睡眠期间检查 `sys.exception()`，当错误无法自愈时停止重试——将盲指数退避转为分类重试。

10. **ExecutionLease + 代际围栏**：一个原语仲裁轮次、工作流运行和会话族操作的互斥。代际围栏确保 `await` 前捕获的准入在期间重建/会话切换后无效。

11. **续接令牌身份检查**：`continuation_is_live` 检查身份（`ticket is self._ticket`）、代际和调用 origin。任何状态恢复/重建通过 `invalidate_continuation()` 使所有未完成令牌无效。

12. **绑定发射器**：在创建时绑定 origin，永不在发布时推断。过期或未注册回调独立失败而非借用后来调用的身份。

### 9.2 相对优势

1. **模型无关**：不锁定任何提供者，支持 OpenAI/Anthropic/DeepSeek/GLM，用户可自由组合。
2. **完全开源**：Apache 2.0，社区可审查和扩展。
3. **可扩展性**：Agent = Profile（配置而非代码），Skills/Hooks/MCP/子 Agent 全可组合。
4. **隐私优先**：无遥测、无分析、无崩溃上报；Web 工具默认关闭。
5. **跨平台**：Windows/macOS/Linux 三平台支持，带平台特定补丁（IOCP、ConPTY、FSEvents）。
6. **企业级持久化**：崩溃恢复边车、事务性恢复、会话锁、fork、回滚。
7. **工作流编排**：用户可定义 YAML 工作流，在用户自己的解释器中运行，带调度器、重试、超时。
8. **ACP 原生**：作为 ACP server 运行，可被其他 ACP 客户端调用。

### 9.3 潜在劣势

1. **Python 3.14+ 要求**：需要较新 Python，可能限制采用。
2. **复杂度**：转录正确性引擎、四层 DAG、三条构建路径等增加了理解和贡献门槛。
3. **无内置模型**：需要用户自行配置模型 profile，首次使用门槛较高。
4. **TUI 依赖 Textual 8.2.7**：锁定在特定版本，带大量补丁。
5. **工作流 SDK 限于 Python 3.9 语法**：为兼容性牺牲了现代 Python 特性。

### 9.4 总结

Chrys/iCode 是一个**转录正确性优先**的 Agent 平台，与 Codex 和 Claude Code 的"信任提供者"哲学形成鲜明对比。它在以下方面展现了工程深度：

- **不变量驱动**：每个关键状态（身份、排序、配对、回声、存储模式、压缩）都有明确的所有者、守卫测试和文档化的不变量。
- **故障闭合**：无效审批 fail-closed 到 require；MCP 连接失败不阻塞构建；响应验证放弃是终端的。
- **并发正确性**：ExecutionLease + 代际围栏 + 续接令牌确保并发安全。
- **可观测性**：trajectory 记录每个模型调用、hook 和工具调用在时间线上。

它最适合需要**多模型支持、可扩展 Agent 配置、工作流编排、企业级持久化和跨平台支持**的场景。对于仅需"快速与一个提供者对话"的简单用例，其复杂度可能是过度工程。

---

*本文基于 iCode v0.27.1 源码分析，参考 AGENTS.md、pyproject.toml 和 src/chrys/ 实现代码。*
