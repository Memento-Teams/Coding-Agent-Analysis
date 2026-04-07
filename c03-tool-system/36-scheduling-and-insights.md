# 3.18-3.22 调度远程工具与工程启示

> 本节覆盖 Cron/Remote 调度工具、未开源工具的全景，以及从工具系统中提炼出的工程设计原则。

---

## 3.18 调度与远程类

### 3.18.1 CronCreateTool — 创建定时任务

```tsx
export const CronCreateTool = buildTool({
  name: 'CronCreate',
  async call({ cron, prompt, recurring = true, durable = false })
  isEnabled: () => isKairosCronEnabled(),
})
```

- **最多 50 个任务**
- **自动过期**：重复任务 7 天后自动删除
- **一次性任务**：`recurring: false` 执行一次后自动删除
- **队友限制**：队友不能创建持久化定时任务（只在会话内有效）

### 3.18.2 CronDeleteTool / CronListTool

- **CronDeleteTool**：根据 ID 删除定时任务，队友只能删除自己的
- **CronListTool**：列出所有定时任务（队友只能看到自己的）

### 3.18.3 RemoteTriggerTool — 远程触发器管理

```tsx
export const RemoteTriggerTool = buildTool({
  name: 'RemoteTrigger',
  async call({ action, trigger_id?, body? }, context)
  isConcurrencySafe: () => true,
  isReadOnly: (input) => input.action === 'list' || input.action === 'get',
})
```

管理 claude.ai 上的远程触发器，支持 `list`、`get`、`create`、`update`、`run` 操作。通过 OAuth 认证调用 Triggers API，20 秒超时。

### 3.18.4 SyntheticOutputTool — 结构化输出

由工厂函数 `createSyntheticOutputTool(jsonSchema)` 动态创建。用 Ajv 校验输入是否符合动态提供的 JSON Schema。

- **WeakMap 缓存**：80 次调用的 Ajv 开销从 ~110ms 降到 ~4ms
- 仅 SDK/CLI 模式：非交互式会话专用

---

## 3.19 未开源工具

`tools.ts` 注册表中引用了约 **15 个工具**，但它们的源代码不在开源仓库中。这些工具在外部构建时会被 **Dead Code Elimination** 完全移除。

### Anthropic 内部工具（ant-only）

| 工具 | 触发条件 | 推测用途 |
|------|---------|---------|
| **TungstenTool** | `USER_TYPE === 'ant'` | Anthropic 内部调试/运维工具 |
| **SuggestBackgroundPRTool** | `USER_TYPE === 'ant'` | 自动建议后台 PR |
| **REPLTool** | `USER_TYPE === 'ant'` | REPL 交互模式 |

### 实验性工具（Feature-gated）

| 工具 | Feature Flag | 推测用途 |
|------|-------------|---------|
| **WebBrowserTool** | `WEB_BROWSER_TOOL` | 网页浏览器交互 |
| **TerminalCaptureTool** | `TERMINAL_PANEL` | 终端截图/捕获 |
| **OverflowTestTool** | `OVERFLOW_TEST_TOOL` | 上下文溢出测试 |
| **CtxInspectTool** | `CONTEXT_COLLAPSE` | 上下文检查 |
| **MonitorTool** | `MONITOR_TOOL` | 系统监控 |
| **WorkflowTool** | `WORKFLOW_SCRIPTS` | 工作流脚本执行 |
| **SnipTool** | `HISTORY_SNIP` | 历史消息裁剪 |
| **ListPeersTool** | `UDS_INBOX` | UDS 对等节点发现 |
| **VerifyPlanExecutionTool** | `CLAUDE_CODE_VERIFY_PLAN` | 计划执行验证 |

### KAIROS 生态工具

| 工具 | Feature Flag | 推测用途 |
|------|-------------|---------|
| **SendUserFileTool** | `KAIROS` | 向用户发送文件 |
| **PushNotificationTool** | `KAIROS` / `KAIROS_PUSH_NOTIFICATION` | 推送通知到用户设备 |
| **SubscribePRTool** | `KAIROS_GITHUB_WEBHOOKS` | 订阅 GitHub PR 事件 |

### 为什么引用不存在的代码不会报错？

因为这些引用全部被 `feature()` 或环境变量包裹：

```tsx
const WebBrowserTool = feature('WEB_BROWSER_TOOL')
  ? require('./tools/WebBrowserTool/WebBrowserTool.js').WebBrowserTool
  : null
```

Bun 构建时 `feature('WEB_BROWSER_TOOL')` 被替换为 `false`，三元表达式变成 `false ? require(...) : null`，DCE 直接消除 `require` 分支。最终 bundle 中连工具名字的字符串都不存在。

> `feature()` vs `process.env.FLAG`：前者**构建时**消除（代码不进 bundle），后者**运行时**检查（代码在 bundle 中但不执行）。

---

## 3.20 工程启示

### 1. 复杂度封装在工具内部

BashTool 18 个文件处理安全、语义、路径校验。引擎对此一无所知——只调 `call()` 和 `checkPermissions()`。

### 2. 默认值偏向安全

`buildTool()` 的默认值：不并行、假设写入、不跳过权限。新手写的工具即使遗漏属性也不会出安全问题。

### 3. Schema 是安全边界

BashTool 的 `_simulatedSedEdit` 对 Claude 不可见。Schema 不只定义数据格式，也定义了模型**能看到什么**。

### 4. 工具自治

每个工具自己声明 `isConcurrencySafe()`、`isReadOnly()`、`isDestructive()`。编排器不需要硬编码规则——复杂度 O(n) 而非 O(n²)。

### 5. 延迟加载

不常用的工具设为 `shouldDefer: true`，不加入 System Prompt。只在 Claude 通过 ToolSearchTool 发现它们时才加载。省 Token，也让 System Prompt 更聚焦。

### 6. 大输出落盘

每个工具有 `maxResultSizeChars` 限制（BashTool 30K、GrepTool 20K、WebFetch 100K）。超过就存磁盘，只发预览。Claude 需要完整内容时可以用 FileReadTool 去读。

---

## 工具全景总图

```
┌─────────────────────────────────────────────────────────────────────┐
│                   开源工具（40 个，代码可读）                        │
│                                                                     │
│  ┌── 文件操作 (6) ──────┐  ┌── 命令执行 (2) ────┐                 │
│  │ FileReadTool        │  │ BashTool          │                 │
│  │ FileWriteTool       │  │ PowerShellTool    │                 │
│  │ FileEditTool        │  └──────────────────┘                 │
│  │ NotebookEditTool    │                                        │
│  │ GlobTool            │  ┌── 网络搜索 (3) ────┐                 │
│  │ GrepTool            │  │ WebFetchTool      │                 │
│  └─────────────────────┘  │ WebSearchTool     │                 │
│                            │ ToolSearchTool    │                 │
│  ┌── Agent 协作 (4) ───┐  └──────────────────┘                 │
│  │ AgentTool           │                                        │
│  │ SendMessageTool     │  ┌── MCP/LSP (5) ─────┐               │
│  │ TeamCreateTool      │  │ MCPTool            │               │
│  │ TeamDeleteTool      │  │ McpAuthTool        │               │
│  └─────────────────────┘  │ ListMcpResourcesTool│               │
│                            │ ReadMcpResourceTool│               │
│  ┌── 任务管理 (7) ─────┐  │ LSPTool            │               │
│  │ TaskCreateTool      │  └────────────────────┘               │
│  │ TaskGetTool         │                                        │
│  │ TaskListTool        │  ┌── 模式切换 (4) ────┐                │
│  │ TaskUpdateTool      │  │ EnterPlanModeTool  │                │
│  │ TaskStopTool        │  │ ExitPlanModeTool   │                │
│  │ TaskOutputTool      │  │ EnterWorktreeTool  │                │
│  │ TodoWriteTool       │  │ ExitWorktreeTool   │                │
│  └─────────────────────┘  └────────────────────┘                │
│                                                                   │
│  ┌── 元操作 (5) ───────┐  ┌── 调度远程 (5) ────┐                │
│  │ SkillTool           │  │ CronCreateTool     │                │
│  │ AskUserQuestionTool │  │ CronDeleteTool     │                │
│  │ ConfigTool          │  │ CronListTool       │                │
│  │ BriefTool           │  │ RemoteTriggerTool  │                │
│  │ SleepTool           │  │ SyntheticOutputTool│                │
│  └─────────────────────┘  └────────────────────┘                │
│                                                                   │
├─────────────────────────────────────────────────────────────────────┤
│          未开源工具（~15 个，构建时 DCE 消除）                      │
│                                                                     │
│  ┌── ant-only (3) ─────┐  ┌── 实验性 (9) ────────────┐           │
│  │ TungstenTool        │  │ WebBrowserTool             │           │
│  │ SuggestBackgroundPR │  │ TerminalCaptureTool        │           │
│  │ REPLTool            │  │ OverflowTestTool           │           │
│  └─────────────────────┘  │ CtxInspectTool             │           │
│                            │ MonitorTool                │           │
│  ┌── KAIROS (3) ───────┐  │ WorkflowTool               │           │
│  │ SendUserFileTool    │  │ SnipTool                   │           │
│  │ PushNotificationTool│  │ ListPeersTool              │           │
│  │ SubscribePRTool     │  │ VerifyPlanExecutionTool    │           │
│  └─────────────────────┘  └────────────────────────────┘           │
│                                                                     │
│  ┌── 测试 (1) ─────────┐                                           │
│  │ TestingPermissionTool│  (NODE_ENV=test 时才存在)                  │
│  └──────────────────────┘                                           │
└─────────────────────────────────────────────────────────────────────┘
```

---

> 返回 [Chapter 3 目录](./README.md) | 返回 [c00 总览](../c00-project-overview/README.md)
