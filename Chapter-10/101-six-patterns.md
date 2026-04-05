# 10.1 贯穿始终的六个模式

> 本节提炼出整个代码库中反复出现的六个核心设计模式。理解了它们，你就理解了 Claude Code 的设计语言。

## 涉及的文件

| 模式 | 核心文件 | 体现位置 |
|------|---------|---------|
| 注册表模式 | `src/tools.ts`、`src/commands.ts`、`src/skills/bundledSkills.ts` | 工具/命令/Skill 注册 |
| Feature Gate | `src/entrypoints/cli.tsx`、`src/services/analytics/growthbook.ts` | 构建时/运行时/动态开关 |
| 懒加载 | `src/entrypoints/cli.tsx`、`src/skills/loadSkillsDir.ts` | 入口/Skill 文档 |
| 异步生成器 | `src/QueryEngine.ts`、`src/query.ts`、`src/services/tools/StreamingToolExecutor.ts` | 整条调用链 |
| 接口即解耦 | `src/Tool.ts`、`src/utils/swarm/backends/types.ts`、`src/cli/transports/` | 工具/Agent/Transport |
| fail-closed | `src/Tool.ts`（buildTool 默认值）、`src/utils/worktree.ts` | 安全默认值 |

---

## 模式 1：注册表（Registry Pattern）

整个代码库有三个核心注册表：

```
tools.ts                → 42 个工具注册（始终 / Feature-gated / 懒加载）
commands.ts             → 101 个命令注册（始终 / Feature-gated）
skills/bundledSkills.ts → 17 个内建 Skill
```

**注册表模式的威力**：

```
第 1 个工具时，tools.ts 是 50 行
第 42 个工具时，tools.ts 是 390 行

引擎代码（query.ts、toolOrchestration.ts）：
  第 1 个工具时 ≈ 第 42 个工具时
  （几乎没有增长）

结论：工具增加了 42 倍，引擎复杂度增加了 0 倍
```

---

## 模式 2：三层 Feature Gate

Feature Gate 在 Claude Code 里用了三种不同的机制，选哪种是有意义的：

```tsx
// 机制 1：构建时消除（Bun bundle flag）
// 用途：不应该在外部版本里存在的代码
const SleepTool = feature('PROACTIVE') ? loadSleepTool() : null
// 构建时 feature('PROACTIVE') === false → DCE 消除 require(...)
// 外部 bundle 中连工具名字都不存在

// 机制 2：运行时环境变量
// 用途：可配置的功能（用户可以开关）
const TeamTools = process.env.CLAUDE_CODE_AGENT_SWARMS
  ? loadTeamTools() : []
// 代码在 bundle 里，但非必要时不执行

// 机制 3：GrowthBook（动态 A/B 测试）
// 用途：渐进推出的新功能（不用发版就能开关）
const isAutoModeEnabled = growthbook.isEnabled('tengu_auto_mode')
// 服务端控制，可以针对特定用户/组织开放
```

三种机制分别对应三种发布场景：**永久隔离**（构建时）、**用户配置**（环境变量）、**灰度发布**（GrowthBook）。

---

## 模式 3：懒加载在所有层次

```
层次                  懒加载方式                    收益
─────────────────────────────────────────────────────────
入口（cli.tsx）       Fast Path + lazy import        启动时间 < 1ms vs ~135ms
工具注册（tools.ts）  Feature Flag / 惰性 getter      避免循环依赖 + 减少内存
Skill 文档           首次调用才解压                   /claude-api 的 247KB 不影响启动
工具搜索（ToolSearch）shouldDefer 工具不加 System Prompt  省 Token
React 渲染           Dirty Flag 跳过未变化节点        渲染开销与变化量成正比
Yoga 布局            calculateLayout() 增量重算       避免全量重布局
```

**核心思想**：每个懒加载都是一个"直到需要才付费"的决定。

---

## 模式 4：异步生成器统一流控

整个调用链都用异步生成器：

```
CLI → main.tsx      → submitMessage(AsyncGenerator)
  → QueryEngine     → query.ts(AsyncGenerator)
    → runTools()    → toolOrchestration(AsyncGenerator)
      → BashTool    → call()(AsyncGenerator with onProgress)
        → API       → streamMessage(AsyncGenerator)
```

> 生成器有一个独特的特性：**把控制权交还给调用方**（通过 yield），调用方决定何时继续（通过 for-await）。
>
> 这在整条调用链上都是需要的：
> - UI 层需要逐条展示消息（不等 30 秒）
> - 工具层需要流式显示进度（不等工具完成）
> - API 层需要流式传输 Token（不等完整回复）
>
> 生成器让每一层都能"交出控制权"，上层按自己的节奏消费。这比回调地狱简洁，比 Promise 灵活，比事件系统线性。

---

## 模式 5：接口即解耦（IoC 的彻底实践）

Claude Code 里有三个关键的接口：

```tsx
// 接口 1：Tool 接口（42 个工具的共同契约）
type Tool = {
  call(args, ctx): Promise<ToolResult>
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  checkPermissions(input): Promise<PermissionResult>
  renderToolUseMessage(input): React.ReactNode
}
// 引擎只认识 Tool 接口，不知道 BashTool 的存在

// 接口 2：AgentBackend 接口（进程内 vs tmux 的共同契约）
type AgentBackend = {
  spawn(config: AgentConfig): Promise<AgentHandle>
  sendMessage(agentId: AgentId, message: Message): Promise<void>
  stopAgent(agentId: AgentId): Promise<void>
}
// Swarm 系统只认识 AgentBackend，不知道是哪种后端

// 接口 3：Transport 接口（输出目标的共同契约）
type Transport = {
  write(data: string): void
  moveCursorUp(lines: number): void
  clear(): void
}
// 渲染器只认识 Transport，不知道是终端还是文件
```

每个接口的存在，都代表一个**可以独立替换的边界**。

---

## 模式 6：fail-closed 安全默认值

在每个有安全含义的地方，默认值都偏向"安全"而非"方便"：

```tsx
// buildTool() 的默认值
const TOOL_DEFAULTS = {
  isConcurrencySafe: () => false,  // 默认不并行（安全）
  isReadOnly: () => false,         // 默认假设会写入（安全）
  isDestructive: () => false,      // 默认假设有影响
}
// 忘写 → 工具执行慢一点，不会出安全问题

// ExitWorktreeTool 的 git status 失败
// 失败时假设有变更（安全）而非假设无变更
if (gitStatusFailed) return null  // null = 假设有变更

// AgentTool 的隔离模式
// isolation: 'worktree' = 创建副本，主仓库安全
```

> **fail-closed 的哲学**：
> - **fail-open**：不确定时允许（"可能没问题，先放行"）
> - **fail-closed**：不确定时拒绝（"可能有问题，先阻止"）
>
> 代价是有时候"不必要地谨慎"，收益是从不发生"意外的破坏性操作"。

> **下一节**：[10.2-10.4 反直觉决策与框架对比](./102-decisions-and-comparisons.md)
