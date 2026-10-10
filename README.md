# Chrys 源码完全指南：目录

> 本系列文档逐层、逐模块地讲解 Chrys 的每一个组件。
> 每个模块包含：是什么、为什么需要、源码在哪、核心代码解析、和其他模块的关系。

---

## Foundation 层（14 个模块）

| 序号 | 模块 | 文件 | 是什么 |
|------|------|------|--------|
| F01 | events | `foundation/events/` | 事件总线，模块间通信 |
| F02 | config | `foundation/config/` | 配置系统，分层优先级 |
| F03 | platform | `foundation/platform/` | 跨平台抽象 |
| F04 | models | `foundation/models/` | 核心数据模型（轮次、调用、工作区） |
| F05 | errors | `foundation/errors/` | 错误分类和处理 |
| F06 | i18n | `foundation/i18n/` | 国际化 |
| F07 | trajectory | `foundation/trajectory/` | 执行轨迹追踪 |
| F08 | text | `foundation/text/` | 文本处理 |
| F09 | util | `foundation/util/` | 通用工具函数 |
| F10 | io | `foundation/io/` | I/O 操作 |
| F11 | net | `foundation/net/` | 网络操作 |
| F12 | observability | `foundation/observability/` | 可观测性（OpenTelemetry） |
| F13 | patches | `foundation/patches/` | Textual 补丁 |
| F14 | vendor | `foundation/vendor/` | 第三方库（ripgrep） |

## Kernel 层（18 个文件）

| 序号 | 文件 | 是什么 |
|------|------|--------|
| K01 | `kernel/agent.py` | Agent 入口 |
| K02 | `kernel/loop.py` | 工具循环（核心中的核心） |
| K03 | `kernel/client.py` | Wire client 抽象 |
| K04 | `kernel/exchanges.py` | 交换语法 |
| K05 | `kernel/identity.py` | 弱身份注册 |
| K06 | `kernel/middleware.py` | 中间件管道 |
| K07 | `kernel/compaction.py` | 压缩原语 |
| K08 | `kernel/tools.py` | 工具定义 |
| K09 | `kernel/sessions.py` | Agent 会话 |
| K10 | `kernel/exceptions.py` | 异常定义 |
| K11 | `kernel/instrumentation.py` | 遥测层 |
| K12 | `kernel/_types.py` | Message/ChatResponse 定义 |
| K13 | `kernel/_content.py` | Content dataclass |
| K14 | `kernel/_serialization.py` | 序列化 |
| K15 | `kernel/_null_overlay.py` | null 恢复 |
| K16 | `kernel/_tool_arg_errors.py` | 参数错误诊断 |
| K17 | `kernel/_result_ceiling.py` | 结果截断 |
| K18 | `kernel/_tool_expansion.py` | 工具扩展 |

## Service 层（17 个模块）

| 序号 | 模块 | 文件 | 是什么 |
|------|------|------|--------|
| S01 | llm | `service/llm/` | LLM 提供者集成 |
| S02 | profiles | `service/profiles/` | Agent 和模型配置 |
| S03 | tools | `service/tools/builtins/` | 内置工具 |
| S04 | mcp | `service/mcp/` | MCP 协议集成 |
| S05 | skills | `service/skills/` | 技能系统 |
| S06 | hooks | `service/hooks/` | 钩子系统 |
| S07 | approval | `service/approval/` | 审批系统 |
| S08 | context | `service/context/` | 上下文管理（四阶段压缩） |
| S09 | agent_middleware | `service/agent_middleware/` | Agent 中间件 |
| S10 | mutations | `service/mutations/` | 变更追踪 |
| S11 | session | `service/session/` | 会话持久化 |
| S12 | state | `service/state/` | 状态序列化 |
| S13 | workflows | `service/workflows/` | 工作流 SDK |
| S14 | acp_client | `service/acp_client/` | ACP 客户端 |
| S15 | analytics | `service/analytics/` | 分析 |
| S16 | todos | `service/todos/` | TODO 追踪 |
| S17 | trajectory | `service/trajectory/` | 轨迹（Service 层部分） |

## Orchestration 层（4 个模块）

| 序号 | 模块 | 文件 | 是什么 |
|------|------|------|--------|
| O01 | startup | `orchestration/startup.py` | 启动引导 |
| O02 | engine | `orchestration/engine/` | 引擎（装配、构建、运行、状态） |
| O03 | invoker | `orchestration/invoker/` | 调用器（Kernel/ACP 后端） |
| O04 | sub_agents | `orchestration/sub_agents/` | 子代理 |
| O05 | session_host | `orchestration/session_host.py` | 会话宿主 |
| O06 | workflows | `orchestration/workflows/` | 工作流编排 |

## App 层（4 个模块）

| 序号 | 模块 | 文件 | 是什么 |
|------|------|------|--------|
| A01 | cli | `app/cli/` | 命令行路由 |
| A02 | tui | `app/tui/` | 终端 UI |
| A03 | acp | `app/acp/` | ACP 服务器 |
| A04 | features | `app/features/` | 附加功能（buddy、session_title） |

---

## 阅读顺序

### 第一阶段：基础设施（Foundation）
```
F01 events    → 先看这个，后面所有模块都用它
F04 models    → 核心数据结构
F02 config    → 配置系统
F03 platform  → 跨平台
F05 errors    → 错误处理
F07 trajectory → 轨迹追踪
F06 i18n      → 国际化
F08-F14       → 按需阅读
```

### 第二阶段：核心引擎（Kernel）
```
K12 _types    → 先看 Message 和 Content 的定义
K13 _content  → Content 的各种类型
K08 tools     → 工具定义
K02 loop      → 核心循环（最重要！）
K04 exchanges → 交换语法
K05 identity  → 弱身份注册
K06 middleware → 中间件
K03 client    → Wire client
K07 compaction → 压缩原语
K01 agent     → Agent 入口
K09 sessions  → 会话
K10-K18       → 按需阅读
```

### 第三阶段：服务集成（Service）
```
S01 llm       → LLM 提供者
S02 profiles  → 配置
S03 tools     → 内置工具
S04 mcp       → MCP 协议
S07 approval  → 审批
S08 context   → 上下文压缩
S09 agent_middleware → Agent 中间件
S06 hooks     → 钩子
S05 skills    → 技能
S10 mutations → 变更追踪
S11 session   → 会话持久化
S12 state     → 状态序列化
S13 workflows → 工作流 SDK
S14-S17       → 按需阅读
```

### 第四阶段：编排协调（Orchestration）
```
O01 startup   → 启动引导
O02 engine    → 引擎
O03 invoker   → 调用器
O04 sub_agents → 子代理
O05 session_host → 会话宿主
O06 workflows → 工作流编排
```

### 第五阶段：用户界面（App）
```
A01 cli       → 命令行
A02 tui       → 终端 UI
A03 acp       → ACP 服务器
A04 features  → 附加功能
```

---

## 文档状态

| 状态 | 含义 |
|------|------|
| ✅ | 已完成 |
| ⏳ | 进行中 |
| ⬜ | 未开始 |

### Foundation 层
- [x] F01 events — ✅ 已完成（`F01_events.md`）
- [x] F02 config — ✅ 已完成（`F02_config.md`）
- [x] F03 platform — ✅ 已完成（`F03_platform.md`）
- [x] F04 models — ✅ 已完成（`F04_models.md`）
- [x] F05 errors — ✅ 已完成（`F05_errors.md`）
- [x] F06 i18n — ✅ 已完成（`F06_i18n.md`）
- [x] F07 trajectory — ✅ 已完成（`F07_trajectory.md`）
- [x] F08 text — ✅ 已完成（`F08_text.md`）
- [x] F09 util — ✅ 已完成（`F09_util.md`）
- [x] F10 io — ✅ 已完成（`F10_io.md`）
- [x] F11 net — ✅ 已完成（`F11_net.md`）
- [x] F12 observability — ✅ 已完成（`F12_observability.md`）
- [x] F13 patches — ✅ 已完成（`F13_patches.md`）
- [x] F14 vendor — ✅ 已完成（`F14_vendor.md`）

### Kernel 层
- [x] K01 agent — ✅ 已完成（`K01_agent.md`）
- [x] K02 loop — ✅ 已完成（`K02_loop.md`）
- [x] K03 client — ✅ 已完成（`K03_client.md`）
- [x] K04 exchanges — ✅ 已完成（`K04_K05_overview.md`）
- [x] K05 identity — ✅ 已完成（`K04_K05_overview.md`）
- [x] K06 middleware — ✅ 已完成（`K06_middleware.md`）
- [x] K07 compaction — ✅ 已完成（`K07_compaction.md`）
- [x] K08 tools — ✅ 已完成（`K08_K11_secondary.md`）
- [x] K09 sessions — ✅ 已完成（`K08_K11_secondary.md`）
- [x] K10 exceptions — ✅ 已完成（`K08_K11_secondary.md`）
- [x] K11 instrumentation — ✅ 已完成（`K08_K11_secondary.md`）
- [x] K12 _types — ✅ 已完成（`K12_K18_private.md`）
- [x] K13 _content — ✅ 已完成（`K12_K18_private.md`）
- [x] K14 _serialization — ✅ 已完成（`K12_K18_private.md`）
- [x] K15 _null_overlay — ✅ 已完成（`K12_K18_private.md`）
- [x] K16 _tool_arg_errors — ✅ 已完成（`K12_K18_private.md`）
- [x] K17 _result_ceiling — ✅ 已完成（`K12_K18_private.md`）
- [x] K18 _tool_expansion — ✅ 已完成（`K12_K18_private.md`）

### Service 层
- [x] S01 llm — ✅ 已完成（`S01_S03_core.md`）
- [x] S02 profiles — ✅ 已完成（`S01_S03_core.md`）
- [x] S03 tools — ✅ 已完成（`S01_S03_core.md`）
- [x] S04 mcp — ✅ 已完成（`S04_S08_integration.md`）
- [x] S05 skills — ✅ 已完成（`S04_S08_integration.md`）
- [x] S06 hooks — ✅ 已完成（`S04_S08_integration.md`）
- [x] S07 approval — ✅ 已完成（`S04_S08_integration.md`）
- [x] S08 context — ✅ 已完成（`S04_S08_integration.md`）
- [x] S09 agent_middleware — ✅ 已完成（`S09_S13_middleware.md`）
- [x] S10 mutations — ✅ 已完成（`S09_S13_middleware.md`）
- [x] S11 session — ✅ 已完成（`S09_S13_middleware.md`）
- [x] S12 state — ✅ 已完成（`S09_S13_middleware.md`）
- [x] S13 workflows — ✅ 已完成（`S09_S13_middleware.md`）
- [x] S14 acp_client — ✅ 已完成（`S14_S17_auxiliary.md`）
- [x] S15 analytics — ✅ 已完成（`S14_S17_auxiliary.md`）
- [x] S16 todos — ✅ 已完成（`S14_S17_auxiliary.md`）
- [x] S17 trajectory — ✅ 已完成（`S14_S17_auxiliary.md`）

### Orchestration 层
- [x] O01 startup — ✅ 已完成（`O01_O06_orchestration.md`）
- [x] O02 engine — ✅ 已完成（`O01_O06_orchestration.md`）
- [x] O03 invoker — ✅ 已完成（`O01_O06_orchestration.md`）
- [x] O04 sub_agents — ✅ 已完成（`O01_O06_orchestration.md`）
- [x] O05 session_host — ✅ 已完成（`O01_O06_orchestration.md`）
- [x] O06 workflows — ✅ 已完成（`O01_O06_orchestration.md`）

### App 层
- [x] A01 cli — ✅ 已完成（`A01_A04_app.md`）
- [x] A02 tui — ✅ 已完成（`A01_A04_app.md`）
- [x] A03 acp — ✅ 已完成（`A01_A04_app.md`）
- [x] A04 features — ✅ 已完成（`A01_A04_app.md`）

---

## 🎉 全部 59 个模块已完成！

| 层 | 模块数 | 文档 |
|----|--------|------|
| Foundation | 14 | F01-F14 |
| Kernel | 18 | K01-K18 |
| Service | 17 | S01-S17 |
| Orchestration | 6 | O01-O06 |
| App | 4 | A01-A04 |

---

**注意**：这个目录会持续更新。每写完一个模块，我会在这里打勾。

你想从哪个模块开始？我建议从 **F01 events** 开始，因为它是所有模块的基础。
