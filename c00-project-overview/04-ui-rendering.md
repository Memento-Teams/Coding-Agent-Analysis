# 0.4 UI 渲染管线详图（Ink Rendering Pipeline）

> Claude Code 的终端界面不是简单的 `console.log`——它是一个完整的 React 应用，用自定义渲染器将 React 组件树渲染为 ANSI 终端输出。本节揭示这个令人惊叹的渲染管线。

## 为什么用 React 渲染终端？

传统的 CLI 工具通常直接用 `process.stdout.write` 输出文本。但 Claude Code 的 UI 远比普通 CLI 复杂：

- 流式输出的实时更新
- Diff 预览和语法高亮
- 多任务并行时的进度展示
- 弹窗、选择器、输入框
- Vim 模式的键盘交互

用命令式代码管理这些状态会极其复杂。React 的声明式模型让团队可以**描述 UI 应该是什么样**，而不是**如何更新 UI**。

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
