# 3.13-3.17 模式切换、网络与元操作工具

> 这些工具覆盖了 Claude Code 的剩余能力：Plan/Worktree 模式切换、网络搜索与抓取、MCP/LSP 集成、以及 SkillTool 等元操作。

---

## 3.13 模式切换类

### 3.13.1 EnterPlanModeTool — 进入规划模式

将权限模式从 `default`/`auto` 切换到 `plan`。Plan 模式下只允许只读操作，Claude 只能"想"不能"做"。

- **Agent 中不可用**：子 Agent 不能进入 Plan 模式（通过 `context.agentId` 检测）
- **Channel 模式不可用**：无终端的 Channel 模式下禁用（Plan 审批对话框无法交互）

### 3.13.2 ExitPlanModeTool (V2) — 退出规划模式

```tsx
export const ExitPlanModeV2Tool = buildTool({
  name: 'ExitPlanMode',
  async call(input, context)
  isReadOnly: () => false,  // 写入计划文件到磁盘
})
```

- **模式恢复**：智能恢复到进入 Plan 前的模式。如果 auto 模式的 gate 在此期间被关闭了，回退到 `default` 而不是 `auto`
- **团队审批流**：队友的计划需要领导审批，审批通过后才能开始执行

### 3.13.3 EnterWorktreeTool — 创建 Git Worktree 隔离

```tsx
export const EnterWorktreeTool = buildTool({
  name: 'EnterWorktree',
  async call({ name? })
  isEnabled: () => isWorktreeModeEnabled(),
})
```

**流程**：

```
① 验证当前不在 Worktree 中
② 找到主仓库根目录（处理嵌套 Worktree 情况）
③ 创建 git worktree + 新分支
④ 切换 process.cwd() 到 Worktree 路径
⑤ 清除系统提示缓存、记忆文件缓存、计划目录缓存
```

- **禁止嵌套**：已经在 Worktree 中时不能再创建
- **缓存全清**：切换目录后所有基于路径的缓存都要重建

### 3.13.4 ExitWorktreeTool — 退出 Worktree

- `keep`：保留 Worktree 和分支，只切回原目录
- `remove`：杀 tmux session → 删 Worktree → 恢复原目录

> **fail-closed 安全策略**：如果 `git status` 失败，返回 null（假设有变更），拒绝删除。有未提交变更时必须 `discard_changes: true` 才能删除。

---

## 3.14 网络与搜索类

### 3.14.1 WebFetchTool — 抓取网页

```tsx
export const WebFetchTool = buildTool({
  name: 'WebFetch',
  async call({ url, prompt }, context)
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
  maxResultSizeChars: 100_000,
})
```

抓取 URL 内容 → HTML 转 Markdown → 对 Markdown 应用 prompt（通过 LLM 处理）→ 二进制内容存磁盘。

- **预批准域名白名单**：某些可信域名跳过权限检查
- **拒绝认证 URL**：Google Docs、Confluence、Jira 等需要登录的页面直接拒绝

### 3.14.2 WebSearchTool — 网络搜索

```tsx
export const WebSearchTool = buildTool({
  name: 'WebSearch',
  async call({ query, allowed_domains?, blocked_domains? }, context)
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
  maxResultSizeChars: 100_000,
})
```

委托给 Claude API 的 `web_search_20250305` beta 工具。

- **最多 8 次搜索**：单次调用内搜索次数上限
- **实时进度追踪**：从流式 JSON 中提取搜索关键词，实时更新 UI
- 在结果末尾追加"必须引用来源"提醒

### 3.14.3 ToolSearchTool — 延迟工具发现

```tsx
export const ToolSearchTool = buildTool({
  name: 'ToolSearch',
  async call({ query, max_results = 5 }, context)
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
})
```

搜索已注册但尚未加载的"延迟工具"。两种查询模式：
- 精确选择：`select:ToolName` 或 `select:A,B,C`
- 关键词搜索：按评分排序（名称匹配 12 分 > searchHint 4 分 > 描述 2 分）

> **存在的原因**：不是所有 42 个工具都需要每次都加载到 System Prompt 中。不常用的工具设为 `shouldDefer: true`，只在 Claude 需要时通过 ToolSearch 发现并加载。省 Token，也让 System Prompt 更聚焦。

---

## 3.15 MCP 与 LSP 集成类

### 3.15.1 MCPTool — MCP 工具代理（模板工具）

```tsx
export const MCPTool = buildTool({
  name: 'MCP',  // 运行时被覆盖为 "mcp__serverName__toolName"
  isMcp: true,
  async call()  // 占位符，运行时被 mcpClient.ts 覆盖
  isConcurrencySafe: () => true,
  maxResultSizeChars: 100_000,
})
```

这是一个**模板工具**。它本身的 `call()` 是占位符，运行时由 `mcpClient.ts` 动态覆盖 name、description、call 等属性。

```
MCPTool 实例化流程：
  MCP 服务器连接 → 获取工具列表 → 为每个工具创建 MCPTool 实例
  → 覆盖 name、description、inputSchema、call()
  → 注册到工具池中
```

### 3.15.2 McpAuthTool — MCP OAuth 认证

> **伪工具模式**：未认证的 MCP 服务器会生成一个 McpAuthTool 作为占位。用户触发认证后，这个伪工具被替换为真正的 MCP 工具。OAuth 回调在后台运行，不阻塞主流程。

### 3.15.3 ListMcpResourcesTool / ReadMcpResourceTool

- **ListMcpResourcesTool**：从连接的 MCP 服务器获取资源列表，有 LRU 缓存，启动时预取。一个服务器失败不影响其他
- **ReadMcpResourceTool**：通过 MCP 协议读取指定资源，二进制内容自动存磁盘（防止 Base64 撞爆上下文）

### 3.15.4 LSPTool — 语言服务器协议

```tsx
export const LSPTool = buildTool({
  name: 'LSP',
  async call({ operation, filePath, line, character })
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
  isEnabled: () => isLspConnected(),
})
```

**9 种操作**：`goToDefinition`、`findReferences`、`hover`、`documentSymbol`、`workspaceSymbol`、`goToImplementation`、`prepareCallHierarchy`、`incomingCalls`、`outgoingCalls`

- 使用编辑器惯例的 1-based 行号/列号
- 10MB 以上的文件不处理
- `shouldDefer: true`，只在 LSP 连接时启用

---

## 3.16 元操作与交互类

### 3.16.1 SkillTool — 执行 Slash Commands

```tsx
export const SkillTool = buildTool({
  name: 'Skill',
  async call({ skill, args }, context)
})
```

```
SkillTool.call({ skill: "simplify" })
  │
  ① 从注册表中查找 "simplify" 的 Skill 定义
  ② 获取 Skill 的 Prompt 文本
  ③ 在隔离的子 Agent 中执行（独立 Token 预算）
  ④ 收集结果并返回
```

> **Skill ≠ Tool**：Skill 本质是一段预写好的 Prompt，通过 SkillTool 这个 Tool 来执行。每个 Skill 在独立的子 Agent 中运行，有自己的 Token 预算，不会消耗主会话的上下文。

**17 个内建 Skill**：

| Skill | 用途 |
|-------|------|
| `commit` | Git 提交 |
| `review` | 代码审查 |
| `simplify` | 代码简化（启动 3 个并行 Agent 审查） |
| `claude-api` | 构建 Claude API 应用 |
| `loop` | 循环执行命令 |
| `remember` | 保存到记忆 |
| `schedule` | 调度远程 Agent |
| `verify` | 验证执行结果 |
| `stuck` | 解决卡住的问题 |
| `debug` | 调试辅助 |
| `batch` | 批量操作 |
| `skillify` | 将操作转为 Skill |

三种来源：内建 Skill（`skills/bundled/`）、用户自定义（`~/.claude/skills/`）、MCP Skill。

### 3.16.2 AskUserQuestionTool — 向用户提问

向用户展示选择题（1-4 个问题，每个 2-4 个选项），支持 markdown/代码预览、多选、唯一性约束。

### 3.16.3 ConfigTool — 读写配置

支持嵌套路径（如 `permissions.defaultMode`），某些设置需要异步校验（如调 API 确认模型可用）。

### 3.16.4 BriefTool / SleepTool

- **BriefTool**：需要 `KAIROS` feature flag，向用户发送带附件的消息
- **SleepTool**：代码在 feature-gated 仓库中（未开源），用户可随时中断，定期收到 `<tick>` 提示

> **下一节**：[3.18-3.22 调度、远程与工程启示](./36-scheduling-and-insights.md)
