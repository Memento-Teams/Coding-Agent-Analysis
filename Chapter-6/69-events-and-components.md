# 6.9-6.10 键盘事件系统与组件库

> 终端的键盘输入是字节流，一个"按键"可能对应 1-10 个字节。事件系统实现了和浏览器 DOM 完全一致的三阶段模型。144 个组件构成了完整的终端 UI 库。

## 6.9 键盘事件系统

### parse-keypress.ts — 字节流到按键

```
"a"          → [0x61]                   普通字符
"Enter"      → [0x0d]                   回车
"Ctrl+C"     → [0x03]                   控制字符
"↑"          → [0x1b, 0x5b, 0x41]      ESC [ A
"Shift+↑"    → [0x1b, 0x5b, 0x31, 0x3b, 0x32, 0x41]  ESC [ 1 ; 2 A
"F12"        → [0x1b, 0x5b, 0x32, 0x34, 0x7e]         ESC [ 2 4 ~
"你"          → [0xe4, 0xbd, 0xa0]      UTF-8 三字节序列
"😀"         → [0xf0, 0x9f, 0x98, 0x80] UTF-8 四字节序列
```

`parse-keypress.ts` 处理所有这些情况，还包括：
- **Kitty 键盘协议**（更精确的修饰键）
- **鼠标事件**（X10/SGR/URXVT 三种格式）
- **焦点事件**（终端窗口获得/失去焦点）
- **剪贴板粘贴检测**（bracketed paste）

### events/ — 事件冒泡系统

```
用户按 Ctrl+C
  │
  ▼ dispatch(KeyboardEvent)
  │
  ├── 捕获阶段（从根节点向下）
  │     root.onKeyDownCapture → 全局快捷键处理
  │
  ├── 目标阶段（当前焦点节点）
  │     BaseTextInput.onKeyDown → 处理输入
  │
  └── 冒泡阶段（从焦点节点向上）
        parent.onKeyDown → 更高层的处理
```

和浏览器 DOM 事件系统完全一致的三阶段模型。这让 Claude Code 能用熟悉的 React 事件处理模式（`onKeyDown`、`onKeyDownCapture`、`stopPropagation()`）来处理键盘输入。

---

## 6.10 组件库（components/ — 144 个组件）

### 组件分类

| 类别 | 主要组件 | 职责 |
|------|---------|------|
| **消息渲染** | `Message.tsx`、`Messages.tsx`、`MessageRow.tsx` | 对话历史展示 |
| **工具展示** | `AssistantToolUseMessage.tsx`、`ToolUseLoader.tsx` | 工具调用 UI |
| **输入交互** | `BaseTextInput.tsx`、`PromptInput.tsx`、`AutoCompleteMenu.tsx` | 用户输入 |
| **进度状态** | `AgentProgressLine.tsx`、`ContextVisualization.tsx` | 任务进度 |
| **权限对话** | `PermissionRequest.tsx`、`PermissionStatus.tsx` | 权限审批 |
| **弹窗系统** | `Dialog/`、各种 `*Dialog.tsx` | 模态对话框 |
| **布局** | `Box` (via Ink)、`ScrollBox` | 容器和滚动 |

### Message.tsx — 消息路由器

```tsx
function Message({ message, ...props }) {
  switch (message.type) {
    case 'user':       return <UserMessage {...props} message={message} />
    case 'assistant':  return <AssistantMessage {...props} message={message} />
    case 'tool_use':   return <AssistantToolUseMessage {...props} message={message} />
    case 'tool_result':return <UserToolResultMessage {...props} message={message} />
    case 'system':     return <SystemTextMessage {...props} message={message} />
    case 'attachment': return <AttachmentMessage {...props} message={message} />
  }
}
```

每种消息类型都有独立的组件，负责自己的渲染逻辑。Message.tsx 只负责分发。

### 多目的地输出 — cli/transports/

```tsx
// 渲染器不直接写 stdout，而是写到 Transport 抽象层
type Transport = {
  write(data: string): void
  moveCursorUp(lines: number): void
  clear(): void
}

// 可以同时挂载多个 Transport
const transports = [
  new StdoutTransport(),         // 终端显示
  new FileTransport(outputPath), // 同时写文件
]
```

> **模块化**：同样的渲染结果可以同时输出到终端（供用户看）、JSON 文件（供脚本解析）、远程 Session（供多人协作）。渲染层和传输层完全解耦。

---

> 返回 [Chapter 6 目录](./README.md) | 返回 [Chapter 0 总览](../Chapter-0/README.md)
