# S01-S03: llm/profiles/tools — LLM 集成、配置、内置工具

> **文件路径**: 
> - `src/chrys/service/llm/` (LLM 提供者集成)
> - `src/chrys/service/profiles/` (Agent/Model 配置)
> - `src/chrys/service/tools/builtins/` (内置工具)
> 
> **一句话**: Service 层的三大基础模块——怎么连接 LLM、怎么配置 Agent/Model、怎么使用内置工具。

---

## S01: llm — LLM 提供者集成

### 核心文件

```
service/llm/
├── clients.py            ← LLM Client 工厂（核心）
├── instrumented.py       ← 遥测包装
├── openai_chat_completion.py  ← OpenAI Chat Completions 实现
├── openai_responses.py   ← OpenAI Responses API 实现
├── deepseek.py           ← DeepSeek 特化
├── glm.py                ← GLM 特化
├── anthropic_chat.py     ← Anthropic 实现
├── raw_http_log.py       ← 原始 HTTP 日志
└── route_facts.py        ← 路由快照
```

### 核心设计：api_style 正交轴

**问题**：DeepSeek 和 GLM 都兼容 OpenAI API，但又不完全一样。怎么优雅支持多个 Provider？

**答案**：Provider（wire transport）和 api_style（request shape）是两个正交维度。

```python
def create_client(profile: ModelProfile) -> ChatClient:
    """Create a chat client for the given profile."""
    
    provider = profile.provider
    
    if provider == "anthropic":
        return _create_anthropic_client(profile)
    
    elif provider in {"openai", "deepseek-openai", "glm-openai"}:
        return _create_openai_flavored_client(profile)
    
    else:
        raise ValueError(f"Unknown provider: {provider}")
```

**Client Stack 组合**：

```
ToolLoopLayer (工具循环)
    ↓
ChatMiddlewareLayer (中间件)
    ↓
_Instrumented* (遥测 + 中间文本捕获)
    ↓
Raw* Wire Client (实际 LLM 调用)
```

### 关键设计点

1. **HTTP Client 所有权**：Chrys 拥有 httpx 池，不用 SDK 默认的
2. **Wire Charset 验证**：API key 和 headers 必须 ASCII 可编码
3. **确定性连接重试守卫**：DNS 失败不应该无限重试
4. **中间文本捕获**：LLM 在调用工具前说的"让我看看文件..."实时显示给用户

### 📚 详细解析

→ **service_implementation.md** 第 1 节：LLM Client Factory（含完整源码注释）

---

## S02: profiles — Agent 和模型配置

### 核心文件

```
service/profiles/
├── agents/registry.py        ← Agent Profile 注册表
├── agents/builtins/          ← 内置 Agent（Code/QA/Explore/General）
├── agents/serializer.py      ← Profile 序列化
├── models/schema.py          ← Model Profile schema
└── models/options.py         ← Chat 选项构建
```

### 什么是 Agent Profile？

一个 YAML 文件定义了一个 Agent 的完整配置：

```yaml
# service/profiles/agents/builtins/Code.yaml
name: Code
id: b011c0de0001        # 稳定 ID
eligibility: main       # main-eligible

instructions: |
  - Use read_file not cat
  - Prefer edit over write
  - ...

tools:
  - read_file
  - write_file
  - edit_file
  - shell
  # ...

sub_agents:
  - name: Explore
    profile: Explore

approval:
  default: auto
  overrides:
    shell: require
    filesystem.write: require
    todo: skip
```

### 什么是 Model Profile？

```yaml
# ~/.chrys/models/gpt-4.yaml
name: gpt-4
provider: openai
api_style: chat_completions
model_id: gpt-4
base_url: https://api.openai.com/v1
```

### 关键设计点

1. **稳定 ID**：内置 Agent 有固定的 ID（`b011c0de0001`），跨安装保持身份
2. **用户覆盖**：用户 `~/.chrys/agents/<Name>.yaml` 可以覆盖内置 Agent
3. **文件系统感知**：处理大小写不敏感的文件系统、Unicode 规范化
4. **CASELESS 比较**：macOS/Windows 上比较 profile 名用存储的 `name:` 而非路径

---

## S03: tools/builtins — 内置工具

### 核心文件

```
service/tools/builtins/
├── shell.py               ← Shell 命令执行
├── filesystem.py          ← 文件读写/编辑
├── search.py              ← grep/glob 搜索
├── ask_user.py            ← 询问用户
├── todo.py                ← TODO 列表管理
├── sleep.py               ← 等待
├── doc_converter/         ← PDF/DOCX/PPTX 转换
├── web/                   ← 网页搜索/抓取
└── shell_command_filtering/ ← Shell 命令过滤
```

### 内置工具列表

| 工具 | 用途 | 需要审批？ |
|------|------|-----------|
| `read_file` | 读取文件 | 否 |
| `write_file` | 写入文件 | 是 |
| `edit_file` | 编辑文件 | 是 |
| `shell` | 执行命令 | 是 |
| `grep` | 搜索文件内容 | 否 |
| `glob` | 查找文件 | 否 |
| `ask_user` | 询问用户 | - |
| `todo` | 管理 TODO 列表 | 否 |
| `search_web` | 网页搜索 | 是 |
| `fetch_web` | 抓取网页 | 是 |

### 关键设计点

1. **Shell PTY**：命令在 PTY 中执行，`TERM=dumb`，stdin 是 `/dev/null`
2. **Shell 过滤**：`READONLY` 预设限制为 `ls, git log, git diff, find, wc` 等
3. **结果截断**：工具输出按 1:2 头:尾比例截断，完整结果溢出到会话目录
4. **Web 出口统一**：所有 Web 请求走 `http.py`（IP 固定、SSRF 防护）

### Shell 命令过滤

```python
class ShellCommandFilter:
    """Filter shell commands for safety."""
    
    def is_safe(self, command: str) -> bool:
        """Check if a shell command is safe to execute."""
        ...

# 预设
READONLY_PRESET = ShellCommandFilter(allow_only=["ls", "git log", "git diff", "find", "wc", ...])
UNRESTRICTED_PRESET = ShellCommandFilter(allow_all=True)
```

### 📚 详细解析

→ **service_implementation.md** 第 1-2 节：LLM Client Factory 和 MCP Cache

---

## 下一步

- **上一篇**: K12-K18 私有类型与辅助模块（Kernel 层完成！）
- **下一篇**: S04-S08 合并文档（MCP/Skills/Hooks/Approval/Context）
