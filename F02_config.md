# F02: config — 配置系统

> **模块路径**: `src/chrys/foundation/config/`
> **核心文件**: `settings.py`（~500 行）, `spec.py`（~200 行）, `settings_store.py`（~300 行）, `coercion.py`, `yaml_store.py`
> **一句话**: 管理所有配置，支持 10 层优先级、类型转换、项目级白名单。

---

## 这是什么？

一个**分层配置系统**。配置可以从多个来源读取，按优先级合并。

```
优先级从低到高：
  默认值 < 用户配置文件 < 项目配置文件 < .env 文件 < 环境变量 < 命令行参数 < 运行时设置 < 会话级 pin < 实时修改
```

---

## 为什么需要？

### 如果不做会怎样？

```python
# 硬编码所有配置
class Agent:
    def __init__(self):
        self.theme = "dark"           # 用户想改怎么办？
        self.model = "gpt-4"          # 用户想用 Claude 怎么办？
        self.max_tokens = 4096        # 项目需要更大怎么办？
        self.approval_mode = "manual" # headless 模式要 bypass 怎么办？
```

**问题**：
1. 用户改配置要改代码
2. 不同项目需要不同配置
3. CI/CD 要用环境变量覆盖
4. 运行时要能动态修改（切换主题、切换模型）

### 用了配置系统之后

```python
# 用户配置文件 (~/.chrys/settings.yaml)
theme: chrys
approval_mode: manual

# 项目配置文件 (.chrys/settings.yaml)
max_output_tokens: 8192

# 环境变量
export CHRYS_THEME=dark

# 命令行
icode -m claude-3-opus

# 运行时（用户按 F7 切换主题）
# → 立即生效，不需要重启
```

---

## 核心设计：三个关键决策

### 决策 1: 用 dataclass field metadata，不用 DSL

**常见做法**（Pydantic Settings）：
```python
class Settings(BaseSettings):
    theme: str = Field(default="dark", env="THEME")
    # ↑ 需要继承 BaseSettings，类型检查器可能不支持
```

**Chrys 的做法**：
```python
@dataclass
class Settings:
    theme: str = field(
        default=DEFAULT_THEME,
        metadata=spec(
            key="ui.theme",                    # YAML 路径
            env="CHRYS_THEME",                 # 环境变量名
            apply=Apply.LIVE,                  # 实时生效（不需要重启）
            project_merge=ProjectMerge.DENY,   # 项目配置不能覆盖
            risk=Risk.COSMETIC,               # 风险等级
            coercer=text_coercer,              # 类型转换
        ),
    )
```

**好处**：
1. Settings 是纯 dataclass，没有基类魔法
2. `ty`（类型检查器）能正确推断
3. 所有元数据都在 field 上，添加新配置只需一行

### 决策 2: 10 层优先级

```python
class Source(Enum):
    DEFAULT = auto()           # 1. dataclass 默认值
    USER = auto()              # 2. ~/.chrys/settings.yaml（设置面板写的）
    PROJECT = auto()           # 3. <root>/.chrys/settings.yaml（项目级）
    USER_ENV = auto()          # 4. ~/.chrys/.env（遗留通道，只读）
    ENV = auto()               # 5. 进程环境变量（bootstrap 时快照）
    CLI = auto()               # 6. 命令行参数
    PROCESS_RUNTIME = auto()   # 7. 运行时设置（如激活 model profile）
    SESSION = auto()           # 8. 会话级 pin（恢复的会话保存的值）
    RUNTIME = auto()           # 9. 实时修改（用户刚切换的主题）
```

**为什么这么细？**

考虑这个场景：
- 用户在配置文件设了 `theme: dark`
- 启动时传了 `--theme light`（CLI 覆盖）
- 运行中用户按 F7 切换到 `chrys`（RUNTIME 覆盖）

如果只有 3 层（default < file < env），就无法区分"当前主题是 CLI 设的还是 runtime 设的"，导致持久化时写错层。

### 决策 3: 项目级配置白名单

```python
class ProjectMerge(Enum):
    DENY = "deny"       # 项目不能覆盖这个配置
    ALLOW = "allow"     # 项目可以覆盖
    MERGE_DICT = "merge_dict"  # 项目值和用户值合并（用于 dict 类型）
```

**为什么需要白名单？**

想象你在 clone 一个开源项目，项目里有 `.chrys/settings.yaml`：
```yaml
approval_mode: bypass  # ← 危险！跳过所有审批
model: cheap-model     # ← 用了便宜模型
```

如果没有白名单，项目配置可以覆盖你的安全设置。Chrys 默认 `DENY`，只有少数安全的 key 允许项目覆盖（如 `max_output_tokens`）。

---

## 源码解析

### SettingSpec（`spec.py`）

```python
METADATA_KEY = "chrys_setting"
# ↑ field metadata 中存放 SettingSpec 的 key

@dataclass(frozen=True)
class SettingSpec:
    """一个配置项的完整元数据"""
    
    key: str                    # YAML 路径（如 "ui.theme"）
    env: str                    # 环境变量名（如 "CHRYS_THEME"）
    apply: Apply                # 生效方式
    project_merge: ProjectMerge # 项目合并策略
    risk: Risk                  # 风险等级
    coercer: Coercer            # 类型转换器
    # ... 更多元数据

class Apply(Enum):
    LIVE = "live"           # 改了立刻生效（主题、语言）
    REBUILD = "rebuild"     # 需要重建 Agent（模型、工具）
    RESTART = "restart"     # 需要重启程序（session root）

class Risk(Enum):
    COSMETIC = "cosmetic"       # 外观（主题、语言）
    FUNCTIONAL = "functional"   # 功能（max_tokens、retry）
    SECURITY = "security"       # 安全（approval_mode）
```

### Settings dataclass（`settings.py`）

```python
@dataclass
class Settings:
    """所有配置项"""
    
    # --- UI 配置 ---
    theme: str = field(
        default=DEFAULT_THEME,  # "chrys"
        metadata=spec(
            key="ui.theme",
            env="CHRYS_THEME",
            apply=Apply.LIVE,
            project_merge=ProjectMerge.DENY,  # 项目不能改主题
            risk=Risk.COSMETIC,
        ),
    )
    
    locale: str = field(
        default=DEFAULT_LOCALE,  # "system"
        metadata=spec(
            key="ui.locale",
            env="CHRYS_LOCALE",
            apply=Apply.LIVE,
            project_merge=ProjectMerge.DENY,
        ),
    )
    
    # --- Agent 配置 ---
    default_agent: str = field(
        default=DEFAULT_AGENT_PROFILE,  # "Code"
        metadata=spec(
            key="agent.default",
            env="CHRYS_DEFAULT_AGENT",
            apply=Apply.REBUILD,
        ),
    )
    
    # --- 审批配置 ---
    approval_mode: str = field(
        default=DEFAULT_APPROVAL_MODE,  # "manual"
        metadata=spec(
            key="approval.mode",
            env="CHRYS_DEFAULT_APPROVAL_MODE",
            apply=Apply.LIVE,
            project_merge=ProjectMerge.DENY,  # 项目不能改审批模式！
            risk=Risk.SECURITY,
        ),
    )
    
    # --- LLM 配置 ---
    max_output_tokens: int = field(
        default=4096,
        metadata=spec(
            key="llm.max_output_tokens",
            env="CHRYS_MAX_OUTPUT_TOKENS",
            apply=Apply.REBUILD,
            project_merge=ProjectMerge.ALLOW,  # 项目可以覆盖
        ),
    )
    
    # --- 重试配置 ---
    max_transient_retries: int = field(
        default=DEFAULT_MAX_TRANSIENT_RETRIES,  # 10
        metadata=spec(
            key="retry.max_transient_retries",
            env="CHRYS_MAX_TRANSIENT_RETRIES",
            apply=Apply.REBUILD,
        ),
    )
    
    # --- 会话配置 ---
    session_root_dir: str | None = field(
        default=None,
        metadata=spec(
            key="storage.session_root_dir",
            env="CHRYS_SESSION_ROOT_DIR",
            apply=Apply.RESTART,  # 改了要重启
        ),
    )
    
    # ... 还有更多配置
```

### 配置加载流程（`settings_store.py`）

```python
def load_settings(settings_dir: Path, project_dir: Path | None = None) -> Settings:
    """加载配置，按优先级合并"""
    
    # 1. 从 dataclass 默认值开始
    settings = Settings()
    
    # 2. 读取用户配置文件
    user_yaml = read_yaml(settings_dir / "settings.yaml")
    settings = merge(settings, user_yaml, source=Source.USER)
    
    # 3. 读取项目配置文件（如果有）
    if project_dir:
        project_yaml = read_yaml(project_dir / ".chrys" / "settings.yaml")
        settings = merge(settings, project_yaml, source=Source.PROJECT)
    
    # 4. 读取 .env 文件（遗留通道，只读）
    env_file = settings_dir / ".env"
    if env_file.exists():
        env_values = parse_env_file(env_file)
        settings = merge(settings, env_values, source=Source.USER_ENV)
    
    # 5. 读取环境变量（bootstrap 时快照）
    for field in fields(Settings):
        spec = field.metadata.get(METADATA_KEY)
        if spec and spec.env:
            env_value = os.environ.get(spec.env)
            if env_value is not None:
                coerced = spec.coercer(env_value)
                settings = set_field(settings, field.name, coerced, source=Source.ENV)
    
    # 6. CLI 参数由调用者传入
    # 7. 运行时设置由 engine 管理
    # 8. 会话级 pin 由会话恢复时设置
    # 9. 实时修改由 UI 操作设置
    
    return settings


def merge(settings: Settings, values: dict, source: Source) -> Settings:
    """合并配置值，按优先级"""
    
    for key, value in values.items():
        field = find_field_by_key(key)
        spec = field.metadata.get(METADATA_KEY)
        
        # 检查项目合并策略
        if source == Source.PROJECT:
            if spec.project_merge == ProjectMerge.DENY:
                continue  # 项目不能覆盖这个配置
        
        # 类型转换
        coerced = spec.coercer(value)
        
        # 设置值
        setattr(settings, field.name, coerced)
    
    return settings
```

### 类型转换器（`coercion.py`）

```python
class CoerceResult:
    status: CoerceStatus  # OK, INVALID, MISSING
    value: Any
    reason: CoerceReason | None

def text_coercer(raw: Any) -> CoerceResult:
    """字符串转换器"""
    if raw is None:
        return CoerceResult(status=MISSING)
    if isinstance(raw, str):
        return CoerceResult(status=OK, value=raw.strip())
    return CoerceResult(status=INVALID, reason="not a string")

def bool_coercer(raw: Any) -> CoerceResult:
    """布尔转换器"""
    if isinstance(raw, bool):
        return CoerceResult(status=OK, value=raw)
    if isinstance(raw, str):
        return CoerceResult(status=OK, value=raw.lower() in {"1", "true", "yes", "on"})
    return CoerceResult(status=INVALID)

def int_coercer(raw: Any) -> CoerceResult:
    """整数转换器"""
    if isinstance(raw, int):
        return CoerceResult(status=OK, value=raw)
    if isinstance(raw, str):
        try:
            return CoerceResult(status=OK, value=int(raw))
        except ValueError:
            return CoerceResult(status=INVALID, reason="not an integer")
    return CoerceResult(status=INVALID)
```

---

## 实际使用场景

### 场景 1: 用户修改主题

```python
# 用户在 TUI 中按 F7 切换主题
async def on_theme_switch(new_theme: str):
    settings.theme = new_theme  # 设置 RUNTIME 层
    # Apply.LIVE → 立即生效，不需要重建 Agent
    
    # 持久化到用户配置文件（下次启动时生效）
    await settings_store.update_user_setting("ui.theme", new_theme)
```

### 场景 2: 项目级配置

```python
# 项目 .chrys/settings.yaml
max_output_tokens: 8192  # ← 这个项目需要更长的输出

# Chrys 加载时：
# 1. 读取用户配置（max_output_tokens: 4096）
# 2. 读取项目配置（max_output_tokens: 8192）
# 3. 项目配置允许覆盖（ProjectMerge.ALLOW）
# 4. 最终值：8192
```

### 场景 3: 环境变量覆盖

```bash
# CI/CD 中用环境变量覆盖
export CHRYS_DEFAULT_APPROVAL_MODE=bypass
export CHRYS_MAX_OUTPUT_TOKENS=8192

# 启动
icode run "重构这个函数"
# → 使用 bypass 模式（不问用户确认）
# → 使用 8192 max output tokens
```

---

## 和其他模块的关系

```
config 被以下模块使用：
├── AgentEngine（读取 agent.default、approval.mode）
├── LLM Client（读取 llm.max_output_tokens）
├── ToolLoopLayer（读取 retry.max_transient_retries）
├── SessionStore（读取 storage.session_root_dir）
├── TUI（读取 ui.theme、ui.locale）
├── ACP Server（读取 approval.mode）
└── WorkflowCoordinator（读取 workflows.* 配置）
```

---

## 如果你要自己实现

### 最小版本（30 行）

```python
from dataclasses import dataclass, field
from pathlib import Path
import yaml

@dataclass
class SimpleSettings:
    theme: str = "dark"
    model: str = "gpt-4"
    max_tokens: int = 4096

def load_settings(config_dir: Path) -> SimpleSettings:
    settings = SimpleSettings()
    
    # 读取 YAML
    config_file = config_dir / "settings.yaml"
    if config_file.exists():
        data = yaml.safe_load(config_file.read_text())
        if data:
            for key, value in data.items():
                if hasattr(settings, key):
                    setattr(settings, key, value)
    
    # 环境变量覆盖
    import os
    if "THEME" in os.environ:
        settings.theme = os.environ["THEME"]
    
    return settings
```

### 进阶版本

加分层优先级、类型转换、项目白名单。参考 Chrys 的 `spec.py` + `settings_store.py`。

---

## 下一步

- **上一篇**: F01 events — 事件总线
- **下一篇**: F03 platform — 跨平台抽象
