# 3.13-3.17 模式切换、网络与元操作工具

> 这些工具覆盖了 Claude Code 的剩余能力：Plan/Worktree 模式切换、网络搜索与抓取、MCP/LSP 集成、以及 SkillTool 等元操作。

---

## 3.13 模式切换类

### 3.13.1 EnterPlanModeTool — 进入规划模式

```tsx
export const EnterPlanModeTool = buildTool({
  name: 'EnterPlanMode',
  async call(input, context)
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
  shouldDefer: true,
})
```

将权限模式从 `default`/`auto` 切换到 `plan`。Plan 模式下只允许只读操作，Claude 只能"想"不能"做"。

**切换流程**：

```
① handlePlanModeTransition()
    → 清除之前残留的 exit attachments
    → 保存当前模式到 prePlanMode（供退出时恢复）
② prepareContextForPlanMode()
    → 如果当前是 auto 模式，激活分类器
③ applyPermissionUpdate({ type: 'setMode', mode: 'plan' })
    → 权限模式生效
④ 返回确认消息 + Plan 模式工作流说明
```

**禁用条件**：
- **Agent 中不可用**：子 Agent 不能进入 Plan 模式（通过 `context.agentId` 检测，直接抛错）
- **Channel 模式不可用**：无终端的 Channel 模式下禁用（Plan 审批对话框无法交互）

---

### 3.13.2 ExitPlanModeTool (V2) — 退出规划模式

```tsx
export const ExitPlanModeV2Tool = buildTool({
  name: 'ExitPlanMode',
  async call(input, context)
  isReadOnly: () => false,  // 写入计划文件到磁盘
})
```

**参数**：
- `allowedPrompts?: Array<{tool: 'Bash', prompt: string}>` — 语义化的权限请求（如 "run tests"）

**两条执行路径**：

```
ExitPlanMode 被调用
  │
  ├── 路径 A：队友（isPlanModeRequired）
  │   ① 将计划写入磁盘
  │   ② 生成 requestId
  │   ③ 写入领导的信箱（writeToMailbox）
  │   ④ 更新任务状态为 "awaiting approval"
  │   ⑤ 返回 { awaitingLeaderApproval: true }
  │   → 等待领导审批后才能开始执行
  │
  └── 路径 B：普通用户 / 自愿 Plan 的队友
      ① 从 prePlanMode 恢复之前的权限模式
      ② 断路器防御：如果 auto-mode gate 已关闭
      │   → 回退到 default，不是 auto
      ③ 恢复危险操作的权限
      ④ 返回计划内容
```

**关键设计**：
- **模式恢复**：智能恢复到进入 Plan 前的模式。如果 auto 模式的 gate 在此期间被关闭了，回退到 `default` 而不是 `auto`
- **团队审批流**：队友的计划需要领导审批，审批通过后才能开始执行
- **CCR 编辑支持**：用户可在 Web UI 中编辑计划，编辑后会重新快照到远端存储

---

### 3.13.3 EnterWorktreeTool — 创建 Git Worktree 隔离

```tsx
export const EnterWorktreeTool = buildTool({
  name: 'EnterWorktree',
  async call({ name? })
  isEnabled: () => isWorktreeModeEnabled(),
  shouldDefer: true,
})
```

**流程**：

```
① 验证当前不在 Worktree 中（禁止嵌套）
② findCanonicalGitRoot() → 找到主仓库根目录
③ createWorktreeForSession(sessionId, slug)
    → git worktree add + 新分支
④ process.chdir(worktreePath)
⑤ 三级缓存全清：
    ├── clearSystemPromptSections()    — env_info 需要重算
    ├── clearMemoryFileCaches()        — claude.md 需要重新读
    └── getPlansDirectory.cache.clear() — 计划目录需要重定位
⑥ saveWorktreeState() → 记录元数据供退出时使用
```

**名称校验**：
- 最大 64 字符
- 每个 `/` 分隔段只允许字母、数字、点、下划线、连字符
- 禁止 `.` 和 `..` 路径段

---

### 3.13.4 ExitWorktreeTool — 退出 Worktree

```tsx
export const ExitWorktreeTool = buildTool({
  name: 'ExitWorktree',
  async call({ action, discard_changes? })
  isDestructive: (input) => input.action === 'remove',
})
```

**两种模式**：

| 模式 | 行为 |
|------|------|
| `keep` | 保留 Worktree 和分支，只切回原目录。返回 tmux session 名供用户重新 attach |
| `remove` | 杀 tmux session → 删 Worktree + 分支 → 恢复原目录 |

**安全机制（remove 模式）**：

```
统计变更：git status --porcelain + git rev-list --count
  │
  ├── 有未提交文件 或 有未合并 commit
  │   └── discard_changes === true ? → 继续删除
  │       discard_changes !== true ? → 拒绝，报告变更数
  │
  └── git status 失败 → 返回 null（fail-closed）
      → 假设有变更，拒绝删除
```

> **双阶段校验**：validateInput 阶段统计一次，call 阶段再统计一次。因为两次之间可能有并发写入。

**恢复流程**：
- `setCwd(originalCwd)` + `setOriginalCwd(originalCwd)`
- 只在 `projectRootIsWorktree` 时恢复 projectRoot（维护项目标识稳定性）
- 与 EnterWorktree 对称的三级缓存清除

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

**完整抓取流程**：

```
URL 输入
  │
  ① URL 校验
  │   ├── 长度 ≤ 2000 字符
  │   ├── 不含用户凭证（阻止 user:pass 格式）
  │   └── HTTP → 自动升级 HTTPS
  │
  ② 域名预检（DOMAIN_CHECK_CACHE, 5 分钟 TTL）
  │   └── 调 api.anthropic.com/api/web/domain_info
  │       ├── 在黑名单 → DomainBlockedError
  │       └── 通过 → 继续
  │
  ③ 缓存查找（URL_CACHE, 15 分钟 TTL, 50MB 上限）
  │   ├── 命中 → 直接返回
  │   └── 未命中 → fetch
  │
  ④ HTTP 抓取
  │   ├── 超时 60 秒，内容上限 10MB
  │   ├── 重定向处理（最多 10 次）
  │   │   ├── www 变体 → 允许
  │   │   └── 跨域重定向 → 阻止，返回重定向 URL 让用户手动重试
  │   └── 检测 X-Proxy-Error 头（egress 代理阻断）
  │
  ⑤ 内容处理
  │   ├── HTML → Turndown 转 Markdown（懒加载，1.4MB 库）
  │   ├── 二进制 → 存磁盘，返回文件路径
  │   └── Markdown 截断到 100K 字符
  │
  ⑥ AI 处理
  │   └── 用 Haiku 对 Markdown 应用用户的 prompt
  │
  ⑦ 缓存写入 + 返回结果
```

**预批准域名白名单**：~130 个开发者常用域名（Python、React、AWS 等）预批准，跳过权限确认。支持路径级白名单。

---

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

**核心实现**：委托给 Claude API 的 `BetaWebSearchTool20250305` beta 工具。

```
用户 query
  │
  ① 校验：query ≥ 2 字符，allowed/blocked 域名互斥
  ② 构建 beta tool 请求
  │   └── schema: web_search_20250305, max 8 次搜索
  ③ 流式处理 API 响应
  │   ├── text block → 普通文本
  │   ├── server_tool_use → 搜索开始（提取 query 关键词）
  │   └── web_search_tool_result → 搜索结果
  ④ 实时进度追踪
  │   └── 从流式 JSON 中用正则提取搜索关键词
  │       /"query"\s*:\s*"((?:[^"\\]|\\.)*)"/
  │       → 去重后更新 UI
  ⑤ 结果末尾追加 "必须引用来源" 提醒
```

**关键设计**：
- **流式架构**：搜索结果实时处理，不等批量返回
- **双模型策略**：主模型执行搜索，可选用 Haiku 处理结果
- **单次搜索上限 8 次**：防止过度搜索消耗 token
- **进度事件去重**：追踪 query 变化，避免重复 UI 更新

---

### 3.14.3 ToolSearchTool — 延迟工具发现

```tsx
export const ToolSearchTool = buildTool({
  name: 'ToolSearch',
  async call({ query, max_results = 5 }, context)
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
})
```

**三种查询模式**：

```
query 类型判断
  │
  ├── "select:ToolName" 或 "select:A,B,C"
  │   → 精确选择模式：按名字直接查找
  │   → 找不到的工具记 debug 日志，部分匹配仍返回
  │
  ├── "mcp__slack__*" 前缀
  │   → MCP 前缀匹配：匹配所有 mcp__slack__* 工具
  │
  └── 关键词搜索（默认）
      → 评分排序
```

**评分算法**：

```
每个搜索词分别打分，汇总：

名称部分精确匹配：
  MCP 工具 +12 / 普通工具 +10
名称部分子串匹配：
  MCP 工具 +6 / 普通工具 +5
全名回退匹配：+3
searchHint 匹配：+4
描述匹配（词边界）：+2
```

**工具名解析**：
- MCP 工具：`mcp__server__action` → 按 `__` 再按 `_` 拆分 → `["server", "action"]`
- 普通工具：`FileReadTool` → CamelCase 拆分 → `["file", "read", "tool"]`

**必需词支持**：`+keyword` 前缀标记为必需项，必须全部匹配才返回结果。

> **存在的原因**：不是所有工具都需要每次加载到 System Prompt。不常用的工具设为 `shouldDefer: true`，只在 Claude 需要时通过 ToolSearch 发现并加载。省 Token，也让 System Prompt 更聚焦。

**变更检测**：通过 `tools.map(t=>t.name).sort().join(',')` 生成签名，检测 deferred 工具集合是否变化，触发缓存失效。

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

这是一个**模板工具**——一个 "壳"。它本身的 `call()` 是占位符，运行时由 `mcpClient.ts` 动态覆盖。

**实例化流程**：

```
MCP 服务器连接成功
  │
  ▼
获取服务器工具列表（tools/list RPC）
  │
  ▼
为每个 MCP 工具创建 MCPTool 实例
  │
  ▼
覆盖核心属性：
  ├── name → "mcp__slack__post_message"
  ├── description() → MCP schema 描述
  ├── prompt() → 工具特定 prompt
  ├── call() → 调用 MCP RPC 请求
  ├── inputSchema → MCP JSON Schema → Zod 转换
  └── userFacingName() → 人类可读名
  │
  ▼
注册到工具池
```

**关键设计**：
- **Schema 透传**：`z.object({}).passthrough()` 接受任意输入，交由 MCP 服务器校验
- **避免 N 个文件**：一个模板工具覆盖所有 MCP 工具，而非每个 MCP 工具一个文件
- **不假设只读**：MCP 工具可能有副作用，默认 `isReadOnly: false`

---

### 3.15.2 McpAuthTool — MCP OAuth 认证

**伪工具模式**：未认证的 MCP 服务器会生成一个 McpAuthTool 作为占位。

```
MCP 服务器需要认证
  │
  ▼
生成 McpAuthTool 占位
  ├── 名字格式：mcp__serverName__auth
  └── 用户触发认证 → OAuth 回调后台运行
  │
  ▼
认证完成
  │
  ▼
McpAuthTool 被移除，替换为真正的 MCP 工具
```

OAuth 回调在后台运行，不阻塞主流程。

---

### 3.15.3 ListMcpResourcesTool / ReadMcpResourceTool

**ListMcpResourcesTool**：

```tsx
export const ListMcpResourcesTool = buildTool({
  name: 'ListMcpResources',
  async call({ server? }, context)
})
```

- 从连接的 MCP 服务器获取资源列表
- **双重缓存**：
  - `fetchResourcesForClient()` — LRU 缓存，按服务器名
  - `ensureConnectedClient()` — 连接 memoize，断线时失效
- 启动时预取，`resources/list_changed` 通知触发缓存失效
- 一个服务器失败不影响其他

**ReadMcpResourceTool**：

```tsx
export const ReadMcpResourceTool = buildTool({
  name: 'ReadMcpResource',
  async call({ server, uri }, context)
})
```

- 通过 MCP 协议读取指定资源
- **二进制处理**：Base64 blob → 解码 → 根据 MIME 推导扩展名 → 存磁盘 → 返回路径
- **文本处理**：直接内联返回
- **容错设计**：磁盘持久化失败不抛错，转为文本消息

---

### 3.15.4 LSPTool — 语言服务器协议（9 种操作）

```tsx
export const LSPTool = buildTool({
  name: 'LSP',
  async call({ operation, filePath, line, character })
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
  isEnabled: () => isLspConnected(),
  shouldDefer: true,
})
```

**9 种操作**：

| 操作 | LSP 方法 | 用途 |
|------|---------|------|
| `goToDefinition` | textDocument/definition | 跳转到定义 |
| `findReferences` | textDocument/references | 查找所有引用 |
| `hover` | textDocument/hover | 悬浮信息（类型、文档） |
| `documentSymbol` | textDocument/documentSymbol | 文件内符号列表 |
| `workspaceSymbol` | workspace/symbol | 工作区符号搜索 |
| `goToImplementation` | textDocument/implementation | 跳转到实现 |
| `prepareCallHierarchy` | textDocument/prepareCallHierarchy | 调用层级入口 |
| `incomingCalls` | callHierarchy/incomingCalls | 谁调用了这个函数 |
| `outgoingCalls` | callHierarchy/outgoingCalls | 这个函数调用了谁 |

**执行流程**：

```
① 等待 LSP 初始化完成
② 检查文件是否已打开（isFileOpen）
│   └── 未打开 → 读取文件内容（< 10MB 校验）→ textDocument/didOpen
③ 坐标转换
│   └── 用户 1-based 行号/列号 → LSP 0-based（line-1, character-1）
│   └── 路径 → file:// URI（pathToFileURL，处理 Windows 路径）
④ 调用对应 LSP 方法
⑤ gitignore 过滤（仅位置类操作）
│   └── 批量 50 个路径 → git check-ignore → 过滤掉 ignored 的结果
⑥ 格式化返回
│   └── 每种操作有专属的结果格式化器
```

**调用层级的两步走**：
```
prepareCallHierarchy(position)
  → 返回 CallHierarchyItem
  → 用这个 item 调 incomingCalls/outgoingCalls
```

**错误码体系**：
- 1: 文件不存在
- 2: 非普通文件
- 3: 无效 Schema
- 4: 文件系统不可访问

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
  ① getAllCommands() — 合并本地 + MCP skills
  │   └── 本地 skills: getCommands(projectRoot)
  │   └── MCP skills: appState.mcp.commands (仅 loadedFrom === 'mcp')
  │   └── uniqBy(name) 去重
  ② findCommand(skillName, allCommands) — 精确匹配
  │   └── 匹配 name / getCommandName() / aliases
  ③ 获取 Skill 的 Prompt 文本
  ④ prepareForkedCommandContext() — 准备子 Agent 上下文
  ⑤ runAgent() — 在隔离的子 Agent 中执行
  │   └── 独立 Token 预算，不消耗主会话上下文
  ⑥ 收集结果并返回
```

> **Skill ≠ Tool**：Skill 本质是一段预写好的 Prompt，通过 SkillTool 这个 Tool 来执行。每个 Skill 在独立的子 Agent 中运行，有自己的 Token 预算。

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

**Skill 发现机制**：详见 [2.4.8 Skill 的发现机制](../c02-core-engine/24-query-loop.md#skill-的发现机制模型就是搜索引擎)。

**实验性远程搜索**：`EXPERIMENTAL_SKILL_SEARCH` feature flag 下有一套语义搜索 + 远程加载机制（`remoteSkillLoader`、`remoteSkillState`），用于解决全量列表塞 prompt 的瓶颈。

---

### 3.16.2 AskUserQuestionTool — 向用户提问

```tsx
export const AskUserQuestionTool = buildTool({
  name: 'AskUserQuestion',
  async call({ questions, answers?, annotations?, metadata? }, context)
  shouldDefer: true,
})
```

**参数结构**：

```tsx
questions: Array<{
  question: string        // 完整问题（以 ? 结尾）
  header: string          // 短标签（chip 形式，限宽）
  options: Array<{
    label: string         // 1-5 词的简洁选项
    description: string   // 选项含义说明
    preview?: string      // 可选预览（代码片段、mockup）
  }>                      // 2-4 个选项（自动添加 "Other"）
  multiSelect?: boolean   // 允许多选
}>
```

**关键设计**：
- **唯一性约束**：问题和选项标签不能重复（`UNIQUENESS_REFINE` 自定义校验）
- **多选支持**：多选答案用逗号拼接
- **预览系统**：每个选项可附带 preview（支持 html、markdown 格式）
- **Channel 禁用**：Telegram/Discord 模式下禁用（用户不在键盘前）
- **渲染分离**：`renderToolUseMessage()` 返回 null，问题通过权限 UI 展示
- **注解支持**：用户可附加 preview 选择和自由文本 notes

---

### 3.16.3 ConfigTool — 读写配置

```tsx
export const ConfigTool = buildTool({
  name: 'Config',
  async call({ setting, value? }, context)
  isReadOnly: (input) => input.value === undefined,
})
```

**Nested Path 支持**：

```
setting: "permissions.defaultMode"
  → 路径数组: ["permissions", "defaultMode"]
  → getValue() 遍历嵌套对象
  → buildNestedObject() 构建写入结构
```

**SET 操作的六步流程**：

```
① 支持性检查 → SUPPORTED_SETTINGS 查表
② 类型强制转换 → "true"/"false" 字符串 → boolean
③ 选项校验 → getOptionsForSetting() 动态选项列表
④ 异步校验 → validateOnWrite()（如验证模型名是否可用，需调 API）
⑤ 特殊预检 → 语音模式：检查 Anthropic 认证、录音可用性、麦克风权限
⑥ 存储写入 → global config / user settings + AppState 同步
```

**权限自动放行**：GET 操作自动允许（只读），SET 操作需要用户确认。

---

### 3.16.4 BriefTool — 向用户发送消息（Feature-gated）

```tsx
export const BriefTool = buildTool({
  name: 'SendUserMessage',
  async call({ message, attachments?, status }, context)
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
})
```

**双层激活机制**：

```
Entitlement（资格）：feature('KAIROS') || feature('KAIROS_BRIEF')
  │                   || getKairosActive() || CLAUDE_CODE_BRIEF 环境变量
  │                   || GrowthBook gate（5 分钟刷新）
  │
  └── AND
  │
Opt-in（启用）：--brief CLI 标志
  │             || defaultView: 'chat' 设置
  │             || /brief 命令
  │             || SDK tools 选项
```

**附件处理**：
- 校验文件存在、是普通文件（非目录/符号链接）、有读权限
- 图片检测基于扩展名（非 MIME 类型）
- 串行 stat（快）→ 并行上传到 Bridge（慢）
- 上传失败不抛错，附件保留本地元数据

**status 参数**：
- `normal` — 回复用户输入
- `proactive` — 主动通知（状态更新、阻塞、任务完成）

---

### 3.16.5 SleepTool — 等待（Feature-gated）

```tsx
// Feature gate: feature('PROACTIVE') || feature('KAIROS')
const SleepTool = feature('PROACTIVE')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

**Tick 机制**：

```
SleepTool 开始睡眠
  │
  ├── 用户发送 <TICK> → 模型被唤醒
  │   ├── 有有用的事做 → 执行
  │   └── 没什么事 → 必须再次调 SleepTool（不能空回复）
  │
  ├── 用户主动中断 → 立刻唤醒
  │
  └── 超时 → 自然醒
```

**成本意识**：
- 每次唤醒 = 一次 API 调用
- Prompt Cache 5 分钟不活动后过期
- 需要平衡睡眠时长 vs 缓存过期 vs 唤醒成本

> SleepTool 优于 `Bash(sleep ...)`：不占用 shell 进程，支持用户中断。

---

> **下一节**：[3.18-3.22 调度、远程与工程启示](./36-scheduling-and-insights.md)
