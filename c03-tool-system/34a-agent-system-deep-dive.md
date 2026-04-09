# 3.8a 多 Agent 系统深度剖析

> 本节是 [3.8-3.12 Agent 协作与任务管理](./34-agent-and-task-tools.md) 的深度展开。前一节介绍了 AgentTool、SendMessage、Team 系列工具的接口和用法；本节深入这些工具背后的**执行引擎、通信基础设施、编排模式和生命周期管理**——从 AgentTool.call() 的第一行代码，到 teammate 优雅关机的最后一条消息。

---

## 4.1 Agent 生命周期

### AgentTool.call() — 多 Agent 的入口

`AgentTool.call()` 是整个多 Agent 系统的入口。根据参数不同，它分化出三种截然不同的执行路径：

```
AgentTool.call(input)
  │
  ├── input.run_in_background === false
  │   → 模式一：同步前台（Foreground）
  │
  ├── input.run_in_background === true
  │   → 模式二：异步后台（Background）
  │
  └── fork 模式（隐式分叉）
      → 模式三：Fork
```

---

### 模式一：同步前台（Foreground）

`run_in_background=false`（默认），父会话阻塞式迭代 `runAgent()` 生成器：

```
父会话                          子 Agent (runAgent)
  │                                │
  ├── for await (msg of runAgent)  │
  │   │                            ├── yield 消息 1（进度）
  │   ├── 收到消息 1，显示进度      │
  │   │                            ├── yield 消息 2（工具调用）
  │   ├── 收到消息 2，显示工具结果  │
  │   │                            ├── yield 消息 3（最终回复）
  │   ├── 收到消息 3，渲染回复      │
  │   │                            └── return（生成器结束）
  │   └── 循环结束                  │
  └── 继续主会话                    ×
```

**自适应转换机制**：

| 时间 | 行为 | 常量 |
|------|------|------|
| 0-2 秒 | 正常前台执行 | — |
| 2 秒后 | 显示 "background hint"（提示可转后台） | `PROGRESS_THRESHOLD_MS = 2000` |
| 120 秒后 | **自动转后台**，无缝切换 | `getAutoBackgroundMs() = 120_000` |

这意味着用户不需要提前判断任务会持续多久——短任务直接看到结果，长任务自动转后台不阻塞。

**父会话逐条消费子 agent 消息时做了什么**：
- 进度传递：子 agent 的工具执行进度实时显示在父 UI
- Token 计数：累加子 agent 的 token 消耗到父会话统计
- Bash 进度转发：子 agent 执行 shell 命令时的 stdout 实时转发

---

### 模式二：异步后台（Background）

`run_in_background=true`，`runAsyncAgentLifecycle()` 以 fire-and-forget 方式启动：

```
父会话                           子 Agent (后台)
  │                                │
  ├── runAsyncAgentLifecycle()     │
  │   │                            │ (独立运行)
  ├── 立即返回:                    │
  │   {status: 'async_launched',   │
  │    agentId, outputFile}        │
  │                                │
  ├── 继续其他工作...              ├── ...执行任务...
  │                                │
  │   ← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ├── 完成！
  │                                │   注入 <task-notification> XML
  │                                │   到父会话的下一轮对话
  └── 在下一轮看到通知             ×
```

父会话不等待，立即拿到 `agentId` 和 `outputFile` 路径。子 agent 完成后通过 `<task-notification>` XML 块注入父会话的下一轮对话消息，实现异步通知。

---

### 模式三：Fork（隐式分叉）

Fork 模式是一种特殊的执行路径，子代理共享父会话的 prompt 前缀：

```
父会话 Prompt: [System Prompt] + [对话历史 1-20]
                    │
                    ▼
Fork 子代理:   [System Prompt] + [对话历史 1-20] + [Fork 指令]
                    │
                    共享相同 prefix → 最大化 Prompt Cache 命中
```

**关键优化**：Fork 子代理与父会话的 prompt 前缀完全一致，Claude API 的 Prompt Cache 按前缀匹配，这意味着 Fork 子代理几乎不产生额外的输入 Token 费用——只需要为 Fork 指令和输出 Token 付费。

---

### runAgent() 异步生成器 — 执行引擎

> **源文件**：`src/tools/AgentTool/runAgent.ts`

`runAgent()` 是所有三种模式共用的底层执行引擎，签名为 `AsyncGenerator<Message, void>`，调用方按需消费。

#### 初始化阶段（7 步）

```
runAgent() 启动
  │
  ├── ① 生成唯一 agentId
  │     注册 Perfetto trace（层级可视化调试）
  │
  ├── ② 克隆父的 FileStateCache
  │     LRU 文件缓存隔离——子 agent 的文件读取不污染父缓存
  │
  ├── ③ 获取 user/system context
  │     只读 agent 省略 CLAUDE.md + gitStatus
  │     （全集群每周省 ~5-15 Gtok）
  │
  ├── ④ 覆盖权限模式
  │     agent permissionMode 覆盖父的（如 Explore agent 强制 plan 模式）
  │
  ├── ⑤ resolveAgentTools() 解析工具集
  │     初始化 agent 专属 MCP 服务器
  │
  ├── ⑥ 组装系统提示
  │     fork → 直接使用父的渲染字节（共享 prompt cache）
  │     普通 → getSystemPrompt() 全新构建
  │
  └── ⑦ 执行 SubagentStart hooks
        预加载技能
```

**步骤 ③ 的成本意识设计**：只读 agent（Explore、Plan）不需要 CLAUDE.md 和 git status 上下文——这些信息只对写入操作有用。省略它们意味着每个只读子 agent 的 system prompt 缩短几 K tokens。在大规模部署中（每周数百万次子 agent 调用），累积节省达到 5-15 Gtokens。

#### 主查询循环

初始化完成后，`runAgent()` 进入内层 `query()` 函数驱动的 LLM 交互循环——与 [2.4 query.ts 主循环](../c02-core-engine/24-query-loop.md) 完全相同的 ReAct 循环，只是运行在子 agent 的隔离上下文中。

每条消息通过 `yield` 逐条吐出，调用方（前台模式的 `for await` 或后台模式的 `runAsyncAgentLifecycle`）按需消费。

---

## 4.2 Team 配置与后端

### TeamFile — 团队的声明式配置

> **存储路径**：`~/.claude/teams/{team-name}/config.json`

```json
{
  "teamName": "refactor-team",
  "leadAgentId": "team-lead@refactor-team",
  "members": [
    {
      "agentId": "researcher@refactor-team",
      "name": "researcher",
      "agentType": "explore",
      "model": "haiku",
      "prompt": "Search the codebase for...",
      "color": "#4CAF50"
    },
    {
      "agentId": "coder@refactor-team",
      "name": "coder",
      "agentType": "general-purpose",
      "model": "sonnet",
      "prompt": "Implement the changes...",
      "color": "#2196F3"
    }
  ],
  "cwd": "/Users/dev/project",
  "worktreePath": "/Users/dev/project-worktrees/coder",
  "backendType": "in_process",
  "planModeRequired": true,
  "subscriptions": ["*"],
  "mode": "default"
}
```

**关键设计决策**：

| 字段 | 设计意图 |
|------|---------|
| `leadAgentId: "team-lead@{teamName}"` | 确定性 ID——进程重启后仍然稳定，可预测、可推导 |
| `members` 数组 | 每个成员完整声明 agentId / name / agentType / model / prompt / color |
| `worktreePath`（可选） | 需要代码隔离的成员才设置，搜索/分析角色可共享主仓库 |
| `planModeRequired` | 设为 true 时，teammate 的计划需要 leader 审批后才能执行 |
| `subscriptions` | 消息订阅规则，`["*"]` 表示接收所有广播 |

**配置即代码**：团队全部元数据以声明式 JSON 存储，可版本控制和复现。创建一个完全相同的团队只需复制这个文件。

---

### TeammateExecutor — 统一执行接口

> **源文件**：`src/utils/swarm/backends/types.ts`

```tsx
interface TeammateExecutor {
  spawn(config: TeammateConfig): Promise<TeammateHandle>
  sendMessage(agentId: AgentId, message: Message): Promise<void>
  terminate(agentId: AgentId): Promise<void>   // 协商关机
  kill(agentId: AgentId): Promise<void>        // 强制终止
  isActive(agentId: AgentId): boolean
}
```

五个核心方法定义了 agent 后端必须实现的"合同"。目前有两种后端实现：

#### 后端一：InProcessBackend

> **源文件**：`src/utils/swarm/backends/InProcessBackend.ts`

```
┌──────────────────────────────────┐
│        Node.js 进程              │
│                                  │
│  ┌─── Leader ───┐               │
│  │  主查询循环    │               │
│  └───────────────┘               │
│                                  │
│  ┌─── Teammate A ─┐  ┌── B ──┐ │
│  │ AsyncLocalStorage│  │ ALS  │ │
│  │ TeammateContext  │  │  TC  │ │
│  └─────────────────┘  └──────┘ │
│                                  │
│  共享：API 连接、MCP 连接         │
└──────────────────────────────────┘
```

| 特性 | 说明 |
|------|------|
| **执行方式** | 同一 Node.js 进程内并发 |
| **隔离机制** | `AsyncLocalStorage` 提供 `TeammateContext`（agentId / teamName / color / permissionMode） |
| **资源共享** | 共享 API 连接和 MCP 连接（进程内复用） |
| **优点** | 启动极快（无进程 spawn 开销）、资源高效 |
| **风险** | 内存共享——一个 teammate 的内存泄漏影响整个进程 |

**内存教训**：早期测试中，2 分钟内启动 292 个 agent 导致 36.8GB 内存占用。解决方案：引入 `TEAMMATE_MESSAGES_UI_CAP = 50`——UI 层只保留最近 50 条消息用于显示，完整消息在本地变量中保留供上下文使用。

#### 后端二：PaneBackendExecutor（tmux / iTerm2）

```
┌─── 主进程 ───┐    ┌─── tmux pane A ───┐    ┌─── tmux pane B ───┐
│   Leader      │    │   Teammate A       │    │   Teammate B       │
│               │    │   独立 CLI 进程     │    │   独立 CLI 进程     │
│               │    │   独立 API 连接     │    │   独立 API 连接     │
└───────────────┘    └───────────────────┘    └───────────────────┘
        │                     │                        │
        └─────── 文件 Mailbox 通信（统一）──────────────┘
```

| 特性 | 说明 |
|------|------|
| **执行方式** | 外部 CLI 进程（tmux pane 或 iTerm2 tab） |
| **隔离机制** | 进程级隔离（天然沙箱） |
| **终止方式** | kill-pane（杀掉整个终端面板） |
| **资源** | 独立的 API 和 MCP 连接 |

**两种后端的统一**：无论是 InProcess 还是 Pane，都使用文件 mailbox 通信。通信层完全统一，上层编排代码无需区分后端类型。

> **模块化启示**：新增第三种后端（如容器化 Docker 后端）只需实现 `TeammateExecutor` 接口的五个方法，核心编排逻辑零修改。

---

### 身份解析三级优先级

> **源文件**：`src/utils/swarm/teammateContext.ts`

当代码需要知道"我是谁"时，按以下优先级解析身份：

```
获取当前 agent 身份
  │
  ├── 优先级 1：AsyncLocalStorage
  │     进程内 teammate 通过 ALS 获取 TeammateContext
  │     （最精确：每个并发任务独立的上下文）
  │
  ├── 优先级 2：dynamicTeamContext
  │     tmux teammate 通过 CLI 参数传入
  │     （进程启动时确定，整个进程生命周期不变）
  │
  └── 优先级 3：AppState.teamContext
        Leader 回退——如果不是 teammate，就是 leader
        （兜底策略）
```

三级优先级覆盖了所有场景：进程内并发（ALS）、跨进程（CLI 参数）、单进程（回退到 leader）。

---

## 4.3 信箱系统（Mailbox）

> **源文件**：`src/utils/teammateMailbox.ts`

### 物理结构

```
~/.claude/teams/{team-name}/inboxes/
├── team-lead@refactor-team.json      ← Leader 的收件箱
├── researcher@refactor-team.json     ← Researcher 的收件箱
└── coder@refactor-team.json          ← Coder 的收件箱
```

每个 agent 有一个独立的 JSON 文件作为收件箱。消息追加写入，读取后标记为已读。

### 锁策略：不对称读写

```
写入（writeToMailbox）              读取（readMailbox）
─────────────────────              ─────────────────
① 加锁（proper-lockfile）           ① 直接读取（无锁）
② 读取当前文件内容                  ② 过滤已读消息
③ 追加新消息                        ③ 返回未读消息
④ 写回文件
⑤ 释放锁
```

**为什么读取不加锁？** 读取是幂等的——即使读到中间状态（写入只完成了一半），最坏的结果是少看到一条消息，下次轮询就能看到。而写入必须加锁，否则两个并发写入会互相覆盖。

**并发性能**：使用 `proper-lockfile`（基于 `mkdir` 原子性 + `stale` 超时检测），10 路 swarm 并发最坏等待约 2.6 秒。

### 核心操作三件套

```tsx
// 写入：加锁 → 读取 → 追加 → 写回 → 释放锁
async function writeToMailbox(agentId: AgentId, message: MailboxMessage): Promise<void>

// 读取：直接读取（无锁），返回未读消息
async function readMailbox(agentId: AgentId): Promise<MailboxMessage[]>

// 标记已读：加锁更新单条 read 标志
async function markMessageAsReadByIndex(agentId: AgentId, index: number): Promise<void>
```

### 消息类型体系（5 种）

| 类型 | 用途 | 典型场景 |
|------|------|---------|
| **① 纯文本消息** | Teammate 间直接通信 | Leader 给 Researcher 分配任务 |
| **② 关机协议** | 三阶段握手 | `shutdown_request` → `shutdown_approved` / `shutdown_rejected` |
| **③ 空闲通知** | Teammate 报告状态 | `idle_notification { idleReason: 'available' \| 'interrupted' \| 'failed' }` |
| **④ 权限请求** | 冒泡审批 | `permission_request { tool_name, tool_use_id, description, input }`（`permissionMode='bubble'` 时冒泡到 leader） |
| **⑤ 计划审批** | Leader 审批计划 | `plan_approval_request` → `plan_approval_response` |

### 消息优先级（4 级）

```
信箱中有多条未读消息时，处理顺序：
  │
  ├── 优先级 1（最高）：关机请求
  │     防止被其他消息淹没，确保优雅退出
  │
  ├── 优先级 2：Team-lead 消息
  │     代表用户意图，不被 peer 消息饿死
  │
  ├── 优先级 3：同级消息
  │     FIFO 顺序处理
  │
  └── 优先级 4（最低）：未认领任务
        claimTask() 拉取任务列表
```

> **设计美学**：文件即消息队列——zero-dependency，无 IPC / socket / 中间件，故障域最小。控制信号（关机、权限）永远优先于数据信号（普通消息），保证系统在任何负载下都能响应管理命令。

---

## 4.4 Coordinator 编排模式

### 编排哲学

Coordinator（team-lead）的编排策略**不依赖硬编码工作流引擎、DAG 调度器或状态机**。编排逻辑完全由 LLM 推理能力驱动——通过精心设计的系统 prompt 教 coordinator 如何分配和监控工作。

```
传统编排：
  ┌─────────────────────┐
  │  硬编码 DAG 调度器    │
  │  节点 A → 节点 B      │
  │  节点 B → 节点 C, D   │  ← 编排逻辑在代码里
  │  节点 C, D → 节点 E   │
  └─────────────────────┘

Claude Code 编排：
  ┌─────────────────────┐
  │  LLM + 系统 Prompt    │
  │  "你是编排者，         │
  │   分解任务，           │  ← 编排逻辑在 Prompt 里
  │   分配给 Worker，      │
  │   综合结果"            │
  └─────────────────────┘
```

### 系统 Prompt 中的编排策略

Coordinator 的系统 prompt 教它（详见 [4.3 Coordinator Prompt](../c04-prompt-engineering/42-tool-and-coordinator-prompts.md#43-coordinator-prompt--多-agent-编排)）：

1. **任务分解** — 高层需求拆解为可并行子任务
2. **Agent 选择** — 根据任务特性选择合适的 agent 类型和模型
3. **并行优先** — "Parallelism is your superpower"
4. **综合而非委托** — "Never write 'based on your findings, fix it' — that's lazy delegation"
5. **Continue vs Spawn 决策** — 上下文重叠高 → Continue（SendMessage）；上下文重叠低 → Spawn new（AgentTool）

### 工具限制保证角色纯粹

Coordinator **只有 3 个工具**：`AgentTool`、`TaskStopTool`、`SendMessage`。没有文件操作，没有 Bash。这从工具层保证了 Coordinator 不会"亲自下场干活"——Prompt 可以被模型忽略，工具限制不能。

---

## 4.5 Worktree 隔离

### 独立工作目录

每个需要修改代码的 agent 可获得独立的 git worktree：

```
repo/                              ← 主 worktree（leader）
├── src/
├── .git/                          ← 共享的 Git 目录
└── ...

repo-worktrees/
├── researcher/                    ← Researcher 的 worktree（如果需要）
│   ├── src/                       （同一 Git 仓库的独立检出）
│   └── ...
└── coder/                         ← Coder 的 worktree
    ├── src/                       （独立分支，独立工作树和索引）
    └── ...
```

**TeamFile 中的 `worktreePath` 字段是可选的**——不需要代码隔离的角色（搜索、分析）可以不使用 worktree，共享主仓库。可选绑定避免了不必要的资源开销。

### fail-closed 安全策略

```
删除 worktree
  │
  ├── 检查未提交变更（git status --porcelain + git rev-list --count）
  │
  ├── 有未提交文件 或 有未合并 commit
  │   └── 拒绝删除，报告变更数
  │       （除非显式 discard_changes: true）
  │
  ├── git status 失败
  │   └── 返回 null → 假设有变更 → 拒绝删除（fail-closed）
  │
  └── 无变更
      └── 安全删除 worktree + 分支
```

宁可清理失败（需人工干预），也不静默丢弃工作成果。这和 [10.1 fail-closed 模式](../c10-design-philosophy/101-six-patterns.md#模式-6fail-closed-安全默认值) 一脉相承。

### 模块化解耦

Worktree 与 agent 生命周期是解耦的：

- 可在 agent 启动**前**创建（预分配）
- 可在 agent 终止**后**保留（保存工作）
- 生命周期独立管理，不强绑定

---

## 4.6 生命周期管理

### InProcessTeammate 状态机

```
┌─────────┐     spawn()     ┌─────────┐
│ created │ ──────────────→ │ running │
└─────────┘                 └────┬────┘
                                 │
                    ┌────────────┼────────────┐
                    │            │            │
                    ▼            ▼            ▼
              ┌──────────┐ ┌─────────┐ ┌──────────┐
              │ completed│ │ stopped │ │  failed  │
              └──────────┘ └─────────┘ └──────────┘
                    ▲            ▲
                    │            │
               任务完成      关机/中止
```

**关键状态字段**：

```tsx
type InProcessTeammateTaskState = {
  // 状态
  status: 'running' | 'completed' | 'stopped' | 'failed' | 'paused'

  // 生命周期控制
  abortController: AbortController           // 终止信号
  currentWorkAbortController: AbortController // 中止当前轮次（非终止）
  shutdownRequested: boolean                  // 是否收到关机请求

  // 空闲追踪
  isIdle: boolean                            // 当前是否空闲
  onIdleCallbacks?: Array<() => void>        // 变为 idle 时触发的回调列表

  // 权限
  permissionMode: PermissionMode             // 该 teammate 的权限模式

  // 消息（有上限）
  messages: Message[]                        // UI 显示用，上限 50 条
}
```

### 双层 AbortController

```
abortController                 currentWorkAbortController
─────────────                   ─────────────────────────
整个 teammate 的终止信号         当前轮次的中止信号

terminate() / kill()            用户按 Escape 键
  → abort()                       → abort()
  → teammate 退出，不可恢复       → 当前轮次中止
                                  → teammate 存活，接受新任务

类比：
  "关闭终端 = 进程死亡"          "Ctrl+C = 中止当前命令但不退出 shell"
```

这种设计让用户可以精细控制 agent 的行为——中止一个跑偏的工具调用，而不需要销毁整个 agent 丢失所有上下文。

### 优雅关机协议（协商式）

与 Unix 的 SIGTERM/SIGKILL 双层设计类似，Claude Code 的 agent 关机也有两层：

#### 第一层：协商关机（terminate）

```
Leader                          Teammate
  │                                │
  ├── createShutdownRequestMessage()
  ├── writeToMailbox(teammate, shutdown_request)
  ├── shutdownRequested = true     │
  │                                │
  │                                ├── waitForNextPromptOrShutdown()
  │                                ├── 优先检测 shutdown_request
  │                                ├── 将关机请求传递给 LLM 模型
  │                                │
  │                                ├── 模型自主决定：
  │                                │   ├── approve=true
  │                                │   │   → abortController.abort()
  │                                │   │   → 优雅退出
  │                                │   │
  │                                │   └── approve=false
  │                                │       → 继续工作
  │                                │       → 回复 shutdown_rejected
  │                                │
  │   ← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┤
  └── 收到 approved / rejected     │
```

**模型可以拒绝关机**——如果 teammate 正在执行关键操作（如正在写文件），它可以拒绝关机请求，完成后再接受。这尊重了 agent 的自主性。

#### 第二层：强制终止（kill）

```tsx
kill(): void {
  abortController.abort()  // 直接中止，不经过协商
}
```

不经过信箱，不等待模型决定，直接 abort。用于 teammate 无响应或需要立即停止的紧急场景。

### Idle Tracking + onIdleCallbacks

Teammate 的空闲等待使用**回调驱动**（非轮询）：

```tsx
// teammate 进入 idle 时触发 onIdleCallbacks
function waitForAllTeammatesIdle(teammates: Teammate[], count: number): Promise<void> {
  let remaining = count

  return new Promise((resolve) => {
    for (const teammate of teammates) {
      teammate.onIdleCallbacks.push(() => {
        remaining--                    // decremental 计数器
        if (remaining <= 0) resolve()  // 归零时 Promise resolve
      })
    }
  })
}
```

- **O(1) 通知**：不需要轮询所有 teammate 的状态
- **Decremental 计数器**：每个 teammate 变为 idle 时 `remaining--`，归零时整体 resolve
- 比定时轮询（`setInterval` 检查 `isIdle`）更高效、更及时

### Teammate 执行循环

```
┌──→ 执行任务（query loop）
│         │
│         ▼
│    进入 idle 状态
│    触发 onIdleCallbacks
│         │
│         ▼
│    等待：消息 / 关机请求
│         │
│    ┌────┼────┐
│    │         │
│    ▼         ▼
│  收到消息   收到关机
│    │         │
│    ▼         ▼
│  处理消息   协商关机
│    │         │
└────┘     退出循环
```

### 内存管理

```
task.messages（UI 层）          allMessages（本地变量）
──────────────────             ──────────────────────
上限 50 条                      保存全量消息
(TEAMMATE_MESSAGES_UI_CAP)      供上下文使用
│                               │
└── 超过时截断旧消息             └── 超过 auto-compact 阈值时
    只影响 UI 显示                   触发压缩（摘要替代原始）
```

两层消息存储的设计：UI 层只保留最近 50 条用于渲染（防止内存爆炸），逻辑层保留全量消息供 LLM 上下文使用（超阈值时由 auto-compact 压缩）。

---

## 架构全景

```
┌─────────────────────────────────────────────────────────────┐
│                     AgentTool.call()                         │
│                          │                                   │
│         ┌────────────────┼────────────────┐                 │
│         ▼                ▼                ▼                 │
│    同步前台          异步后台          Fork              │
│    (阻塞迭代)       (fire-and-forget)  (共享prefix)         │
│         │                │                │                 │
│         └────────────────┼────────────────┘                 │
│                          ▼                                   │
│                    runAgent()                                │
│                  (异步生成器)                                 │
│                          │                                   │
│                          ▼                                   │
│                    query() 循环                              │
│                   (ReAct 模式)                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌── TeamFile ──┐  ┌── TeammateExecutor ──┐                │
│  │ config.json   │  │ InProcess / Pane      │                │
│  │ 声明式配置     │  │ 统一接口              │                │
│  └───────────────┘  └──────────────────────┘                │
│                                                              │
│  ┌── Mailbox ───────────────────────────────────────────┐   │
│  │ 文件信箱 + proper-lockfile                            │   │
│  │ 5 种消息类型 · 4 级优先级                              │   │
│  │ 不对称读写（写加锁，读无锁）                           │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌── Coordinator ──┐  ┌── Worktree ──┐  ┌── Lifecycle ──┐  │
│  │ Prompt 驱动编排   │  │ Git 隔离      │  │ 状态机        │  │
│  │ 3 个工具限制      │  │ fail-closed   │  │ 协商式关机    │  │
│  │ LLM 做调度决策    │  │ 可选绑定      │  │ 双层 abort   │  │
│  └──────────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 设计启示

| 原则 | 体现 |
|------|------|
| **接口即解耦** | `TeammateExecutor` 统一了 InProcess 和 Pane 两种后端，新增后端零侵入 |
| **文件即基础设施** | Mailbox 用文件实现消息队列，TeamFile 用 JSON 声明团队，零外部依赖 |
| **fail-closed** | Worktree 删除前检查变更、git status 失败假设有变更、默认拒绝 |
| **自适应降级** | 前台 agent 2 秒显示提示、120 秒自动转后台，无需用户预判 |
| **尊重 agent 自主性** | 协商式关机让模型决定是否接受，不暴力 kill |
| **成本意识** | 只读 agent 省略 CLAUDE.md/gitStatus（每周省 5-15 Gtok）、Fork 共享 prompt cache |

---

> 返回 [3.8-3.12 Agent 协作与任务管理](./34-agent-and-task-tools.md) | 返回 [Chapter 3 目录](./README.md) | 返回 [c00 总览](../c00-project-overview/README.md)
