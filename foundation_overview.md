# Foundation 层全景地图

> **学习目标**：在深入源码之前，先搞清楚 Foundation 层到底有哪些东西、各自解决什么问题、之间怎么协作。
>
> **文件数量**：149 个 Python 文件
>
> **预计阅读时间**：30 分钟（本概览）+ 后续每个子模块的深入解析

---

## 0. Foundation 层在整个系统中的位置

```
┌─────────────────────────────────────┐
│  App 层 (TUI/CLI/ACP)              │  ← 用户界面
├─────────────────────────────────────┤
│  Orchestration 层                   │  ← 编排协调
├─────────────────────────────────────┤
│  Service 层                         │  ← 外部服务集成
├─────────────────────────────────────┤
│  Kernel 层                          │  ← Agent 核心循环
├─────────────────────────────────────┤
│  Foundation 层 ⭐                    │  ← 基础设施（本文重点）
└─────────────────────────────────────┘
```

**Foundation 层的职责**：提供所有上层模块共享的基础能力。它不依赖任何业务逻辑，是最底层的一层。

---

## 1. 子模块分类

Foundation 层包含 **12 个子模块**，按功能可以分为 4 大类：

### 📦 第一类：通信与事件（2 个文件）

| 子模块 | 文件数 | 核心职责 | 关键文件 |
|--------|--------|----------|----------|
| **events/** | 2 | 模块间异步通信 | `bus.py`（EventBus）、`types.py`（事件定义） |
| **observability/** | ~5 | 遥测和监控 | OpenTelemetry 集成 |

**你已经学过的**：✅ EventBus（`docs/build-guide/01_event_bus.md`）

---

### ⚙️ 第二类：配置与环境（~15 个文件）

| 子模块 | 文件数 | 核心职责 | 关键文件 |
|--------|--------|----------|----------|
| **config/** | ~15 | 配置加载、优先级合并、存储 | `settings.py`（Settings 类）、`spec.py`（配置规范）、`yaml_store.py`（YAML 读写） |
| **platform/** | ~10 | 跨平台抽象 | `get_platform()`、进程管理、文件系统操作 |

**解决的问题**：
- 配置从哪里来？（默认值 < 配置文件 < 环境变量 < CLI 参数）
- 如何在 Windows/macOS/Linux 三平台统一行为？
- 如何处理路径、进程、终端的跨平台差异？

---

### 🌍 第三类：国际化与文本（~10 个文件）

| 子模块 | 文件数 | 核心职责 | 关键文件 |
|--------|--------|----------|----------|
| **i18n/** | ~5 | 多语言支持 | `msg()` 函数、catalog 管理 |
| **text/** | ~5 | 文本处理 | 格式化、清理、转义 |

**解决的问题**：
- 如何让 UI 显示中文/英文/其他语言？
- 如何避免 markup 注入（`list[int]` 被渲染成 `list`）？
- 如何处理 Unicode、emoji、特殊字符？

---

### 🧩 第四类：领域模型与工具（~120 个文件）

这是最大的一类，包含各种领域特定的模型和工具函数：

| 子模块 | 文件数 | 核心职责 | 关键文件 |
|--------|--------|----------|----------|
| **models/** | ~20 | 核心数据结构 | `turns.py`（轮次边界）、`invocations.py`（调用身份）、`workspace.py`（工作区） |
| **trajectory/** | ~30 | 执行轨迹追踪 | 记录每次 LLM 调用、工具调用的时间和元数据 |
| **tool_*/** | ~15 | 工具相关常量 | `tool_kinds.py`、`tool_invocation_order.py` 等 |
| **errors/** | ~10 | 错误分类和处理 | `classify.py`（错误分类）、`network.py`（网络错误） |
| **patches/** | ~15 | Textual 补丁 | monkey-patch Textual 的性能问题 |
| **util/** | ~10 | 通用工具函数 | 日期、哈希、锁等 |
| **vendor/** | ~5 | 第三方库 | vendored ripgrep（搜索工具） |
| **io/** | ~5 | I/O 操作 | 文件读写、流处理 |
| **net/** | ~5 | 网络操作 | HTTP client、代理设置 |

---

## 2. 学习路线图

基于依赖关系和重要性，我建议按这个顺序深入学习：

### 🟢 阶段 1：核心基础（必读）

这些是整个系统的基石，必须彻底理解。

#### 1.1 Platform 抽象 —— 跨平台的根基
- **文件**：`foundation/platform/__init__.py` + `process.py` + `files.py`
- **为什么重要**：所有上层模块都要用它来处理平台差异
- **你会学到**：
  - 如何检测当前操作系统（Windows/macOS/Linux）
  - 如何统一进程管理（Windows IOCP vs POSIX fork/exec）
  - 如何处理路径差异（反斜杠 vs 正斜杠）
  - 为什么 stdin=DEVNULL 这么重要

#### 1.2 Config 系统 —— 配置的来源和优先级
- **文件**：`foundation/config/settings.py` + `spec.py` + `yaml_store.py`
- **为什么重要**：Agent 的所有行为都由配置控制
- **你会学到**：
  - 配置的分层优先级（defaults < user config < project config < env vars < CLI）
  - 如何实现 live changes（运行时修改立即生效）
  - YAML 配置的 schema 验证
  - 项目级配置的白名单机制（哪些 key 允许项目覆盖）

#### 1.3 Models 核心 —— 数据结构定义
- **文件**：`foundation/models/turns.py` + `invocations.py` + `workspace.py`
- **为什么重要**：定义了"轮次""调用""工作区"等核心概念
- **你会学到**：
  - 什么是 turn boundary（轮次边界）
  - 为什么不能用 `role == "user"` 判断轮次
  - InvocationOrigin 的设计（不可变的调用身份）
  - Workspace 如何管理多个工作目录

---

### 🟡 阶段 2：支撑系统（推荐）

这些是重要的支撑功能，理解后能看清全貌。

#### 2.1 Errors 分类 —— 错误的识别和处理
- **文件**：`foundation/errors/classify.py` + `network.py` + `provider.py`
- **为什么重要**：决定哪些错误可以重试、哪些必须失败
- **你会学到**：
  - 如何区分"可重试"和"不可重试"的错误
  - DNS 失败为什么不应该无限重试
  - 错误链的遍历（`__cause__` vs `__context__`）
  - 网络错误的 OS 特定处理（winerror vs errno）

#### 2.2 I18n —— 多语言支持
- **文件**：`foundation/i18n/__init__.py` + `formatting.py`
- **为什么重要**：UI 需要显示多种语言
- **你会学到**：
  - `msg()` 函数的设计（类型安全的翻译）
  - catalog 的加载和查找
  - 如何处理 plural forms（单复数）
  - 为什么 model/tool/protocol 文本保持英文

#### 2.3 Trajectory —— 执行轨迹追踪
- **文件**：`foundation/trajectory/writer.py` + `events.py` + `ids.py`
- **为什么重要**：记录每次 LLM 调用和工具调用的详细信息
- **你会学到**：
  - 什么是 analytics_item_id（出现身份）
  - 如何记录时间线（model cycle、tool call、hook）
  - trajectory context 的作用域管理
  - 为什么 trajectory 不能阻塞用户可见路径

---

### 🔴 阶段 3：高级特性（可选）

这些是锦上添花的功能，可以在掌握核心后再看。

#### 3.1 Patches —— Textual 补丁
- **文件**：`foundation/patches/textual_*.py`
- **为什么有趣**：monkey-patch 的技巧和注意事项
- **你会学到**：
  - 为什么要 patch 第三方库（性能优化、bug 修复）
  - 开发环境 vs CI 的不同处理方式
  - 如何安全地访问私有 API

#### 3.2 Tool Kinds & Metadata —— 工具元数据
- **文件**：`foundation/tool_kinds.py` + `tool_invocation_order.py` 等
- **为什么小众**：主要是常量和辅助函数
- **你会学到**：
  - 工具的 kind 分类（shell、filesystem、web 等）
  - 调用顺序戳的实现
  - 结果元数据的附加和读取

---

## 3. 各子模块之间的关系

```
Foundation 层内部依赖图：

events ←── 无依赖（最基础）
  ↑
config ←── platform, util
  ↑
platform ←── util
  ↑
models ←── config, platform
  ↑
errors ←── platform, util
  ↑
i18n ←── config
  ↑
trajectory ←── models, util
  ↑
tool_* ←── models
```

**关键观察**：
- `events` 和 `util` 是最底层的，几乎不依赖其他子模块
- `config` 和 `platform` 是中间层，被很多模块依赖
- `models` 和 `trajectory` 是高层，依赖较多

---

## 4. 文件大小分布

为了帮你评估工作量，这里列出各子模块的文件数和总行数（估算）：

| 子模块 | 文件数 | 总行数 | 复杂度 | 建议时间 |
|--------|--------|--------|--------|----------|
| events/ | 2 | ~300 | 🟢 低 | 1 小时 |
| config/ | ~15 | ~2000 | 🟡 中 | 3 小时 |
| platform/ | ~10 | ~1500 | 🟡 中 | 2 小时 |
| models/ | ~20 | ~2500 | 🟡 中 | 3 小时 |
| errors/ | ~10 | ~1500 | 🟡 中 | 2 小时 |
| i18n/ | ~5 | ~800 | 🟢 低 | 1 小时 |
| trajectory/ | ~30 | ~4000 | 🔴 高 | 5 小时 |
| patches/ | ~15 | ~2000 | 🔴 高 | 3 小时 |
| tool_* | ~15 | ~1000 | 🟢 低 | 1 小时 |
| util/ | ~10 | ~1000 | 🟢 低 | 1 小时 |
| vendor/ | ~5 | N/A | ⚪ 跳过 | - |
| io/, net/ | ~10 | ~1000 | 🟢 低 | 1 小时 |

**总计**：约 22 小时（如果你要彻底搞懂每一个文件）

---

## 5. 下一步选择

现在你有三个选项：

### 选项 A：按阶段顺序学习（推荐新手）

从 **Platform 抽象** 开始，因为它是最基础的，被很多模块依赖。

我会带你读：
- `foundation/platform/__init__.py`（平台检测）
- `foundation/platform/process.py`（进程管理）
- `foundation/platform/files.py`（文件系统操作）

### 选项 B：挑最重要的先学（推荐有经验者）

直接看 **Config 系统**，因为它是整个 Agent 行为的控制中心。

我会带你读：
- `foundation/config/settings.py`（Settings 类）
- `foundation/config/spec.py`（配置规范）
- `foundation/config/yaml_store.py`（YAML 读写）

### 选项 C：你有特别感兴趣的点

比如你想先看 Trajectory（执行轨迹）、Errors（错误处理）、I18n（国际化），告诉我就行。

---

## 6. 学习方法建议

对于每个子模块，我建议这样做：

1. **先看 __init__.py**：了解导出了什么、主要接口是什么
2. **找主入口函数**：通常是某个核心类或函数
3. **跟踪数据流**：输入是什么？输出了什么？经过了哪些步骤？
4. **看测试用例**：`tests/foundation/xxx/` 下的测试能帮你理解用法
5. **动手改一改**：试着修改某个行为，看会发生什么

---

**你想从哪个开始？** 告诉我，我马上开始写下一份源码解析文档。
