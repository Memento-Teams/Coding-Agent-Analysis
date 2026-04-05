# 0.4 UI 渲染管线详图（Ink Rendering Pipeline）

> Claude Code 的终端界面不是简单的 `console.log`——它是一个完整的 React 应用，用自定义渲染器将 React 组件树渲染为 ANSI 终端输出。本节揭示这个令人惊叹的渲染管线。

## 先感受一下：普通 CLI vs Claude Code

```
普通 CLI（比如 git status）：
  输出一段文字，完事。不会变，不会动。

Claude Code 的终端界面：
  ├── AI 的回复在实时一个字一个字地蹦出来（流式输出）
  ├── 同时下面显示"正在读取文件..."的进度条在转
  ├── 突然弹出一个权限确认框"要执行 rm 命令吗？[y/n]"
  ├── 侧边还有 Agent 任务的状态在实时更新
  ├── 用户可以滚动查看历史对话
  └── 支持 Vim 快捷键编辑输入
```

这已经不是"打印文字"了，这是一个**动态交互界面**。用 `console.log` 写这些会疯掉——所以 Claude Code 用了跟**网页**一样的思路来渲染终端。

## 为什么用 React 渲染终端？

| 网页 | Claude Code 终端 | 干的事一样 |
|------|-----------------|-----------|
| `<div>` | `<Box>` | 容器，控制布局 |
| `<span>` | `<Text>` | 显示文字 |
| CSS Flexbox | Yoga（同一个引擎） | 计算每个元素放在哪里 |
| 浏览器渲染像素 | 输出 ANSI 转义序列 | 把内容画到屏幕上 |
| DOM diff（只更新变化的部分） | 双缓冲 diff（只输出变化的字符） | 避免全量重绘闪烁 |

React 的声明式模型让团队可以**描述 UI 应该是什么样**，而不是**手动管理每一帧怎么更新**。

## Claude Code 为什么看起来这么流畅？

流畅感来自**三层优化叠加**，不只是 API 的流式返回：

### 第一层：API 流式返回（Stream）

```
没有 Stream：
  等 3 秒... 等 3 秒... 等 3 秒... 突然一整段文字全部出现

有 Stream：
  我 → 来 → 看 → 看 → 这 → 个 → 文 → 件
  一个字一个字蹦出来，像在打字
```

Claude API 本身支持流式返回——一个 token 生成一个就发一个。这是最基础的流畅感来源。

### 第二层：只更新变化的部分（Diff 渲染）

如果每收到一个字就**清屏重画整个界面**：

```
收到"我"  → 清屏，重画 50 行 → 闪一下
收到"来"  → 清屏，重画 50 行 → 闪一下
收到"看"  → 清屏，重画 50 行 → 闪一下
每秒闪 20 次，眼睛要瞎了 💥
```

Claude Code 的做法——双缓冲 Diff：

```
收到"我"  → 跟上一帧对比，只有 1 个字变了 → 只输出 1 个字
收到"来"  → 跟上一帧对比，只有 1 个字变了 → 只输出 1 个字
收到"看"  → 跟上一帧对比，只有 1 个字变了 → 只输出 1 个字
完全不闪，丝滑 ✅
```

### 第三层：硬件滚动（DECSTBM）

滚动历史对话时，如果用软件重绘：

```
向上滚 3 行 → 重新绘制可见区域全部 50 行 → 发送 2000+ 个字符 → 慢
```

用硬件滚动：

```
向上滚 3 行 → 告诉终端"你自己滚" → 发送 2 个指令 → 瞬间完成
```

### 三层叠加 = 原生应用般的流畅

```
Stream      → 内容实时到达，不用等
Diff 渲染   → 只画变化的部分，不闪烁
硬件滚动    → 滚动时终端自己处理，不重绘

三层一起   → 看起来像一个原生应用，不像 CLI
```

## 渲染管线概览

渲染过程分为 5 个阶段：

```
React 组件 → Reconciler → DOM 树 → Yoga 布局 → Output Buffer → ANSI 输出
```

1. **React 组件** — 开发者用 JSX 编写 UI 组件（如 `<MessageList/>`、`<Dialog/>`）
2. **Reconciler** — 自定义的 React Reconciler 将组件树转换为 Ink DOM 节点
3. **Yoga 布局** — 使用 Facebook 的 Yoga 引擎计算 Flexbox 布局（和 React Native 同款）
4. **Output Buffer** — 将布局结果渲染为二维字符矩阵
5. **ANSI 输出** — 通过增量 Diff 渲染，只更新终端上变化的部分

### 用一个具体例子走一遍

当 Claude 说"我来读取这个文件"并调用 FileRead 时：

```
第 1 步：React 组件更新
  <MessageList>
    <Message text="我来读取这个文件" />
    <ToolUseLoader tool="FileRead" status="执行中..." />   ← 新增
  </MessageList>

第 2 步：Ink DOM 更新
  ink-root
    └── ink-box (MessageList)
          ├── ink-text "我来读取这个文件"
          └── ink-box (ToolUseLoader)              ← 新增节点
                └── ink-text "⏳ 正在读取文件..."

第 3 步：Yoga 布局计算
  "我来读取这个文件"     → 第 1 行，x=0, y=0, 宽度=80
  "⏳ 正在读取文件..."   → 第 2 行，x=0, y=1, 宽度=80   ← 新算出来的位置

第 4 步：画到字符矩阵
  第 0 行: [我][来][读][取][这][个][文][件][ ][ ]...
  第 1 行: [⏳][ ][正][在][读][取][文][件][.][.][.]...   ← 新增这一行

第 5 步：跟上一帧对比，只输出变化的部分
  上一帧只有 1 行，这一帧有 2 行
  → 只需要输出第 2 行
  → 发送：ESC[2;1H⏳ 正在读取文件...    （移动光标到第2行第1列，写入内容）
```

这就是"渲染管线"——数据从 React 组件开始，经过一层层处理，最终变成终端上你看到的文字。

## 渲染管线架构图

```mermaid
graph TB
    subgraph REACT_LAYER["React 组件层"]
        APP["<App/><br/>根组件"]
        MSG_LIST["<MessageList/><br/>对话消息流"]
        TOOL_MSG["<ToolUseMessage/><br/>工具调用展示"]
        DIFF_VIEW["<FileEditToolDiff/><br/>Diff 渲染"]
        INPUT["<BaseTextInput/><br/>用户输入框"]
        DIALOG["<Dialog/><br/>弹窗 · 确认"]
        SELECT["<CustomSelect/><br/>多选组件"]
        AGENT_STATUS["<AgentProgressLine/><br/>Agent 进度"]
        CTX_VIZ["<ContextVisualization/><br/>上下文可视化 (76KB)"]
        FEEDBACK["<Feedback/><br/>反馈收集 (87KB)"]
    end

    subgraph RECONCILER["React Reconciler"]
        REC["reconciler.ts<br/>React ↔ Ink DOM 桥接"]
        DOM["dom.ts<br/>Ink DOM 节点树<br/>InkElement · InkText"]
    end

    subgraph LAYOUT["布局引擎"]
        YOGA["layout/yoga.ts<br/>Yoga (Flexbox) 绑定"]
        ENGINE["layout/engine.ts<br/>布局计算"]
        NODE["layout/node.ts<br/>布局节点"]
    end

    subgraph RENDER["渲染管线"]
        R2O["render-node-to-output.ts (63KB)<br/>DOM → Output Buffer<br/>处理 wrap · clip · overflow"]
        OUTPUT["output.ts<br/>Output Buffer<br/>二维字符矩阵"]
        R2S["render-to-screen.ts<br/>Buffer → ANSI 字符串<br/>增量 Diff 渲染"]
        SCREEN["screen.ts (49KB)<br/>终端 Screen Buffer<br/>双缓冲 · 脏区域检测"]
    end

    subgraph TERMINAL_IO["终端 I/O"]
        ANSI["Ansi.tsx (33KB)<br/>ANSI 颜色/样式<br/>SGR 序列生成"]
        TERMIO["termio/<br/>tokenize · parser<br/>CSI · DEC · OSC · SGR<br/>ANSI 序列解析"]
        KEYPRESS["parse-keypress.ts (23KB)<br/>键盘事件解析<br/>修饰键 · 特殊键"]
        TERMINAL["terminal.ts<br/>终端接口抽象<br/>stdin/stdout 管理"]
    end

    subgraph EVENTS["事件系统"]
        DISPATCHER["events/dispatcher.ts<br/>事件分发"]
        KB_EVENT["events/keyboard.ts<br/>键盘事件"]
        CLICK["events/click.ts<br/>鼠标点击"]
        FOCUS["events/focus.ts<br/>焦点管理"]
        HIT_TEST["hit-test.ts<br/>点击命中检测"]
    end

    subgraph SELECTION_SYS["选择系统"]
        SEL["selection.ts (34KB)<br/>文本选择<br/>复制/粘贴"]
    end

    subgraph TRANSPORTS["输出传输层 (cli/)"]
        STDOUT["transports/stdout<br/>标准输出"]
        FILE_T["transports/file<br/>文件输出"]
        REMOTE_T["transports/remote<br/>远程输出"]
        JSON_T["structuredIO.ts<br/>NDJSON 结构化输出"]
    end

    %% 渲染管线流程
    APP --> REC
    REC -->|"创建/更新"| DOM
    DOM -->|"附加布局属性"| YOGA
    YOGA --> ENGINE
    ENGINE --> NODE
    NODE -->|"布局完成"| R2O
    R2O --> OUTPUT
    OUTPUT --> R2S
    R2S -->|"ANSI Diff"| SCREEN
    SCREEN --> TERMINAL

    %% 样式
    ANSI -.-> R2O
    TERMIO --> KEYPRESS

    %% 事件流（反向）
    TERMINAL -->|"stdin"| KEYPRESS
    KEYPRESS --> DISPATCHER
    DISPATCHER --> KB_EVENT
    DISPATCHER --> CLICK
    CLICK --> HIT_TEST
    KB_EVENT -->|"React 事件"| APP

    %% 选择
    SEL --> SCREEN

    %% 输出分发
    SCREEN --> STDOUT
    SCREEN --> FILE_T
    SCREEN --> REMOTE_T
    SCREEN --> JSON_T

    classDef react fill:#e3f2fd,stroke:#1565c0
    classDef reconciler fill:#e8eaf6,stroke:#283593
    classDef layout fill:#f3e5f5,stroke:#6a1b9a
    classDef render fill:#fff3e0,stroke:#ef6c00
    classDef term fill:#e0f2f1,stroke:#00695c
    classDef event fill:#fce4ec,stroke:#c62828
    classDef transport fill:#f1f8e9,stroke:#33691e

    class APP,MSG_LIST,TOOL_MSG,DIFF_VIEW,INPUT,DIALOG,SELECT,AGENT_STATUS,CTX_VIZ,FEEDBACK react
    class REC,DOM reconciler
    class YOGA,ENGINE,NODE layout
    class R2O,OUTPUT,R2S,SCREEN render
    class ANSI,TERMIO,KEYPRESS,TERMINAL term
    class DISPATCHER,KB_EVENT,CLICK,FOCUS,HIT_TEST event
    class STDOUT,FILE_T,REMOTE_T,JSON_T transport
```

## 关键技术细节

### 双缓冲与增量渲染

`screen.ts` 实现了**双缓冲**机制：维护两个 Buffer（当前帧和上一帧），每次渲染时只计算差异部分，生成最少的 ANSI 转义序列发送到终端。这大幅减少了终端闪烁和 I/O 开销。

### 事件系统的双向流动

渲染管线是**单向**的（组件 → 终端），但事件系统是**反向**的（终端 → 组件）：

1. 终端接收到 stdin 数据
2. `parse-keypress.ts` 解析为结构化的键盘事件
3. 事件分发器将事件路由到正确的组件
4. React 组件处理事件，触发状态更新
5. 状态更新触发重新渲染（回到渲染管线）

### 多输出传输层

终端输出不一定只发往 `stdout`。Claude Code 支持多种输出传输：
- **stdout** — 标准终端输出
- **file** — 输出到文件（用于日志和调试）
- **remote** — 通过 WebSocket 发送到远程客户端
- **NDJSON** — 结构化 JSON 输出（用于程序集成）

> **下一节**：[0.5 服务层与扩展机制](./05-services-and-extensions.md) — 了解 API 调用、MCP、Skills 等服务如何运作。
