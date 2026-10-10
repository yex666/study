# A01-A04: App 层全览

> **文件路径**: `src/chrys/app/`
> **一句话**: 用户界面层——CLI 路由、Textual TUI、ACP 服务器、附加功能。

---

## 模块总览

```
app/
├── cli/               ← A01 命令行
│   ├── app.py         ←   命令分发器（手写 if-chain）
│   ├── acp.py         ←   ACP server 入口
│   └── serve.py       ←   浏览器托管 TUI
├── tui/               ← A02 终端 UI
│   ├── app.py         ←   TUI 主应用
│   ├── screens/       ←   界面（main/sessions/models/...）
│   ├── widgets/       ←   小部件（chat/sidebar/diff_view/...）
│   ├── terminal/      ←   终端模拟器
│   ├── themes/        ←   主题系统
│   └── support/       ←   支持（gc_freeze 等）
├── acp/               ← A03 ACP 服务器
│   └── server.py      ←   JSON-RPC 服务器
└── features/          ← A04 附加功能
    ├── buddy/         ←   Buddy（AI 伴侣面板）
    └── session_title/ ←   会话标题生成
```

---

## A01: cli — 命令行路由

### 手写 if-chain

```python
def main() -> int:
    """Top-level chrys command dispatcher."""
    
    argv = sys.argv[1:]
    
    # 版本检查
    if "-v" in argv or "--version" in argv:
        print(f"{APP_DISPLAY_NAME} {__version__}")
        return 0
    
    # 帮助
    if not argv or "-h" in argv or "--help" in argv:
        print_help()
        return 0
    
    # 手写 if-chain（不用 argparse subparsers）
    command = argv[0]
    rest = argv[1:]
    
    if command == "run":
        return _run_headless(rest)     # headless 模式
    
    elif command == "acp":
        return _run_acp(rest)          # ACP server
    
    elif command == "serve":
        return _run_serve(rest)        # 浏览器托管
    
    elif command == "agents":
        return _list_agents(rest)      # 列出 agents
    
    elif command == "models":
        return _list_models(rest)      # 列出 models
    
    elif command == "workflow":
        return _run_workflow(rest)     # 工作流
    
    elif command == "trajectory":
        return _export_trajectory(rest)  # 轨迹导出
    
    elif command == "install":
        return _run_install(rest)      # 安装
    
    else:
        # 未匹配 → TUI
        return _run_tui(argv)
```

**为什么不用 argparse subparsers？**
- 默认行为复杂（无命令 → TUI）
- 灵活的参数传递（`icode -s session_id` 要传给 TUI）
- 避免 argparse 的限制

### 📚 详细解析

→ **app_implementation.md** 第 1 节（CLI 路由）

---

## A02: tui — 终端 UI

### 架构

```
ChrysApp (app.py)
  └── MainScreen
      ├── ChatPanel          ← 聊天面板
      │   ├── MessageWidget  ← 消息小部件
      │   ├── ToolCallWidget ← 工具调用卡片
      │   └── replay_mount   ← 大历史回放
      ├── Sidebar            ← 侧边栏（会话列表）
      ├── Footer             ← 底部状态栏
      └── InputBar           ← 输入栏
```

### 性能优化（关键）

1. **分批挂载**：大历史回放时，一次挂载一个 viewport（3 轮对话）
2. **GC Freeze**：回放完成后冻结稳定的 widget，减少 GC 压力
3. **可见性控制**：隐藏用 `display = False`，不用 `-hidden` 类
4. **预组合**：挂载前预组合 widget 森林

```python
# app/tui/widgets/chat/replay_mount.py
class ChatReplayMount:
    """Detached replay plan for large history mounting."""
    
    async def mount_in_batches(self, chat_panel):
        """Mount transcript entries in bounded batches."""
        
        for batch in self._split_into_batches(reverse=True):
            widgets = await self._precompose_batch(batch)
            await chat_panel.mount(*widgets)
            await asyncio.sleep(0)  # 让出控制权
```

### GC Freeze

```python
# app/tui/support/gc_freeze.py
class GcFreezeManager:
    """Manage GC freeze for stable content."""
    
    async def on_replay_complete(self):
        import gc
        # 冻结稳定的 widget
        for widget in self.app.query("UserMessageWidget, AssistantMessageWidget"):
            gc.freeze(widget)
```

### 📚 详细解析

→ **app_implementation.md** 第 2 节（TUI 性能优化）

---

## A03: acp — ACP 服务器

### JSON-RPC 协议

```python
class AcpServer:
    """ACP stdio server."""
    
    async def handle_request(self, request: dict) -> dict:
        """Handle a JSON-RPC request."""
        
        method = request["method"]
        params = request.get("params", {})
        request_id = request.get("id")
        
        try:
            if method == "initialize":
                result = await self.handle_initialize(params)
            
            elif method == "chat/send":
                result = await self.handle_chat_send(params)
            
            elif method == "chat/cancel":
                result = await self.handle_chat_cancel(params)
            
            elif method == "shutdown":
                result = await self.handle_shutdown()
            
            else:
                raise ValueError(f"Unknown method: {method}")
            
            return {"jsonrpc": "2.0", "id": request_id, "result": result}
        
        except Exception as e:
            return {"jsonrpc": "2.0", "id": request_id, "error": {...}}
```

### 关键设计

- **stdout 仅 JSON-RPC**：日志写到 stderr
- **流式响应**：`chat/send` 立即返回，通过 notification 流式输出
- **非标准 caps**：通过 `ext_method` 扩展

### 📚 详细解析

→ **app_implementation.md** 第 4 节（ACP Server）

---

## A04: features — 附加功能

### Buddy（AI 伴侣）

一个常驻的 AI 辅助面板，提供实时建议：

```python
class BuddyPanel:
    """AI companion panel."""
    
    async def on_user_message(self, event):
        # 异步生成建议
        suggestion = await self.generate_suggestion(event.text)
        self.display_suggestion(suggestion)
```

### Session Title（会话标题生成）

自动生成会话的标题：

```python
class SessionTitleGenerator:
    """Generate session titles from conversation content."""
    
    async def generate_title(self, messages) -> str:
        """Generate a concise title for a session."""
        
        # 使用 LLM 生成简短标题
        response = await self.llm.generate(
            f"Generate a short title for this conversation:\n{messages}"
        )
        return response.text[:50]
```

---

## 📊 App 层完成总结

| 序号 | 模块 | 文档 |
|------|------|------|
| A01 | cli | `A01_A04_app.md` |
| A02 | tui | `A01_A04_app.md` |
| A03 | acp | `A01_A04_app.md` |
| A04 | features | `A01_A04_app.md` |

---

## 🎉 全部五层完成！

恭喜！你已经完整地学习了 Chrys 的所有五个层！

### 完成进度

```
✅ Foundation 层:     14/14 (100%)
✅ Kernel 层:         18/18 (100%)
✅ Service 层:        17/17 (100%)
✅ Orchestration 层:   6/6 (100%)
✅ App 层:             4/4 (100%)
─────────────────────────────
总计: 59/59 模块完成 (100%)
```

### 文档总览

```
docs/build-guide/
├── README.md                          ← 总目录
├── 00_overview.md                     ← 全局概念
├── F01-F14 *.md                       ← Foundation 层（14 篇）
├── K01-K18 *.md                       ← Kernel 层（8 篇）
├── S01-S17 *.md                       ← Service 层（5 篇）
├── O01-O06 *.md                       ← Orchestration 层（1 篇）
└── A01-A04 *.md                       ← App 层（1 篇）
```

### 下一步建议

1. **回顾**：重读 00_overview.md，检查是否有遗漏的概念
2. **实践**：打开源码，找到文档中提到的关键函数
3. **动手**：尝试自己实现一个最小版本的 Coding Agent
4. **深入**：选择你最感兴趣的部分深入阅读

祝你在 Chrys 的世界里探索愉快！🚀
