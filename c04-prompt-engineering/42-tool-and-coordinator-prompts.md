# 4.2-4.3 工具 Prompt 与 Coordinator Prompt

> 每个工具都有一个 `prompt.ts` 文件，教 Claude 什么时候该用、怎么用、什么时候不该用。Coordinator Prompt 则把 Claude 变成一个调度员。

---

## 4.2 工具 Prompt

### 4.2.1 BashTool — 最长的工具 Prompt（369 行）

> **源文件**：`src/tools/BashTool/prompt.ts`

BashTool 的 Prompt 是所有工具中最长的，因为 shell 命令的自由度最高、风险最大。它不是一个静态字符串，而是一个**动态生成函数** `getSimplePrompt()`，根据运行环境拼装不同的指令段。

**动态组装结构**：

```
getSimplePrompt()
  │
  ├── 基础描述（固定）：执行 bash 命令、返回输出
  ├── 工具优先规则（固定）：Avoid Bash when dedicated tool exists
  ├── 多命令策略（固定）：parallel / && / ; 的选择
  │
  ├── 条件段 1：Git 安全协议
  │   └── getCommitAndPRInstructions()
  │       ├── 外部用户 → 完整的 7 步 commit 流程 + PR 创建指南
  │       └── 内部用户（ant）→ 简化版
  │
  ├── 条件段 2：后台任务说明
  │   └── getBackgroundUsageNote()
  │
  └── 条件段 3：超时配置
      └── getDefaultTimeoutMs() / getMaxTimeoutMs()
```

**工具优先规则**：

```
IMPORTANT: Avoid using Bash when a dedicated tool exists.
- To read files use Read instead of cat, head, tail
- To edit files use Edit instead of sed or awk
- To search for files use Glob instead of find
```

**多命令策略**：

```
When issuing multiple commands:
  - Independent → make multiple Bash tool calls in parallel
  - Sequential dependent → single call with && chains
  - Sequential non-critical → use ; (don't care if earlier fail)
  - DO NOT use newlines to separate commands
```

**Git 安全协议**（最详细的部分）：

```
- NEVER update the git config
- NEVER run destructive git commands unless explicitly requested
- NEVER skip hooks (--no-verify)
- NEVER force push to main/master
- CRITICAL: Always create NEW commits rather than amending
- When staging, prefer specific files over "git add -A"
- NEVER commit unless explicitly asked
```

**用户分群设计**：BashTool 通过 `process.env.USER_TYPE === 'ant'` 区分内外用户。外部用户得到完整的 7 步 commit 流程（包括 `Co-Authored-By` 签名模板），内部用户得到简化版。这是因为内部用户更熟悉 Git 操作，不需要手把手教。

> **Prompt 设计特点 — 分层指令**：BashTool 的 Prompt 有三个层次：① 原则层（什么时候用 Bash）② 策略层（多命令如何编排）③ 规则层（Git 操作的具体禁令）。从抽象到具体，每层解决不同级别的问题。

> **Prompt 设计特点 — 条件组装**：不是一个巨大的静态字符串，而是多个函数拼接。好处是：不同环境看到不同指令，减少无关信息的 token 消耗。

---

### 4.2.2 AgentTool Prompt（287 行）

> **源文件**：`src/tools/AgentTool/prompt.ts`

AgentTool 的 Prompt 也是动态生成的（`getPrompt()`），根据 feature flag 控制不同能力段的注入。

**动态注入结构**：

```
getPrompt()
  │
  ├── Agent 类型列表（动态）
  │   └── formatAgentLine() 为每个 agent 生成描述行
  │       ├── 内建 Agent（explore, plan, general-purpose...）
  │       └── 用户自定义 Agent（从 .claude/agents/ 加载）
  │
  ├── 使用指南（固定）
  │   ├── 何时使用 / 何时不使用
  │   ├── 描述要求（3-5 词摘要）
  │   └── 并发启动指南（多个 Agent 同时发）
  │
  ├── 条件段：fork subagent 说明
  │   └── feature('FORK_SUBAGENTS') 控制
  │
  └── 条件段：worktree 隔离说明
      └── isWorktreeModeEnabled() 控制
```

**何时不使用的决策矩阵**：

```
When NOT to use the Agent tool:
- If you want to read a specific file path → use Read
- If you are searching for "class Foo" → use Glob
- If you are searching within 2-3 files → use Read
```

**消息注入机制**：`shouldInjectAgentListInMessages()` 决定是否在对话消息中插入 Agent 列表（而非仅在 system prompt 中）。当用户自定义 Agent 存在时，会额外注入到消息流中，确保模型看到最新的 Agent 列表。

> **Prompt 设计特点 — 决策矩阵**：不是给一个模糊的"视情况而定"，而是给出精确的二选一标准。LLM 在有明确判断标准时表现更好。

> **Prompt 设计特点 — Fleet 控制**：通过 GrowthBook feature flag（`getFeatureValue_CACHED_MAY_BE_STALE`）可以在不发版的情况下控制全球所有实例的 Agent 行为。

---

### 4.2.3 SkillTool Prompt（241 行）

> **源文件**：`src/tools/SkillTool/prompt.ts`

SkillTool 的 Prompt 最特殊——它不只教 Claude 怎么用工具，还要**动态列出所有可用的 Skill**。

**核心函数 `formatCommandsWithinBudget()`**：

```
formatCommandsWithinBudget(commands, contextWindow)
  │
  ① 计算预算 = 上下文窗口的 1%（200K → ~8000 字符）
  ② 分类：bundled skills vs 其他 skills
  ③ 优先保证 bundled skills 的完整描述
  ④ 剩余预算分配给其他 skills
  ⑤ 预算不够时的降级策略：
      ├── 截断描述到 250 字符
      ├── 极端情况只保留名字
      └── 最极端情况完全省略
```

**强制触发指令**：

```
When a skill matches the user's request, this is a
BLOCKING REQUIREMENT: invoke the relevant Skill tool
BEFORE generating any other response about the task.
```

**Skill 列表注入格式**：

```
The following skills are available for use with the Skill tool:
- commit: Create a git commit following the repository's conventions...
- review-pr: Review a pull request...
- simplify: Review changed code for reuse, quality, efficiency...
```

> **Prompt 设计特点 — BLOCKING REQUIREMENT 信号词**：普通的 "should" 经常被 LLM 忽略，但 "BLOCKING REQUIREMENT" 结合全大写，几乎不会被跳过。这是 Claude Code 中最强的优先级信号。

> **Prompt 设计特点 — 预算感知**：Skill 列表不是无脑塞入，而是根据上下文窗口大小动态调整。这是因为 Skill 列表属于 system prompt，占用 Prompt Cache 的"黄金位置"，必须精打细算。详见 [2.4.8 Skill 的发现机制](../c02-core-engine/24-query-loop.md#skill-的发现机制模型就是搜索引擎)。

---

### 4.2.4 TodoWriteTool Prompt（184 行）

> **源文件**：`src/tools/TodoWriteTool/prompt.ts`

TodoWriteTool 的 Prompt 是所有工具中**例子最多的**——包含 10 个完整的 Example/Anti-Example 对。

**When to / When NOT to 双向界定**：

```
When to use: 3+ steps, non-trivial tasks, user asks for tracking
When NOT to use: single task, trivial changes, conversation
```

**双形式要求**：

```
CRITICAL: Two forms required:
- content: imperative ("Fix the bug")
- activeForm: present continuous ("Fixing the bug")
```

**状态管理规则**：

```
- Exactly ONE task in_progress at any time
- Mark tasks complete IMMEDIATELY after finishing
- ONLY mark completed when FULLY accomplished
- If blocked → keep as in_progress, create new task for blocker
```

**大量示例的设计意图**：184 行 Prompt 中约 100 行是 `<example>` 和 `<reasoning>` 标签包裹的示例。每个示例不只展示"怎么做"，还解释"为什么这样做"（通过 `<reasoning>` 标签）。

> **Prompt 设计特点 — When to / When NOT to 对比**：同时给出正面和反面场景，消除边界情况的歧义。这比单独说"在复杂任务时使用"更有效。

> **Prompt 设计特点 — Example + Reasoning 模式**：不只给 few-shot 示例，还给出推理过程。这让 Claude 学到的是**判断标准**，而不只是模式匹配。

---

### 4.2.5 FileEditTool Prompt（28 行 — 最短的工具 Prompt）

> **源文件**：`src/tools/FileEditTool/prompt.ts`

与 BashTool 的 369 行形成鲜明对比。FileEditTool 只需 28 行，因为它的 Schema 本身就已经约束了大部分行为。

```
- You must use Read at least once before editing.
- Preserve exact indentation as it appears AFTER the line number prefix.
- The edit will FAIL if old_string is not unique.
- Use replace_all for renaming variables across file.
- Only use emojis if user explicitly requests.
```

**为什么这么短？** FileEditTool 的安全性主要靠**工具实现**（唯一性校验、编码检测）而非 Prompt 约束。BashTool 无法在实现层做安全限制（shell 命令太自由），只能靠长 Prompt 教 Claude "什么不该做"。

> **Prompt 设计特点 — 预告失败条件**："The edit will FAIL if..." 让 Claude 在调用前就知道什么会失败，主动避免。比"请确保 old_string 唯一"更有效，因为 Claude 会为了避免工具报错而更仔细。

---

### 4.2.6 GrepTool Prompt

> **源文件**：`src/tools/GrepTool/prompt.ts`

```
- ALWAYS use Grep for search tasks. NEVER invoke grep or rg as Bash command.
- Pattern syntax: Uses ripgrep (not grep)
  - literal braces need escaping (use interface\{\} to find interface{})
```

> **Prompt 设计特点 — 纠正常见混淆**：显式纠正 grep vs Grep（工具名 vs 命令名）、grep 语法 vs ripgrep 语法的混淆。这是因为 Claude 的训练数据中充满了 `grep` 命令行用法，很容易混淆。

---

### 4.2.7 WebSearchTool Prompt（34 行）

> **源文件**：`src/tools/WebSearchTool/prompt.ts`

```
CRITICAL REQUIREMENT - You MUST follow this:
  After answering, include a "Sources:" section
  List all relevant URLs as markdown hyperlinks: [Title](URL)

IMPORTANT - Use the correct year in search queries:
  The current month is {month} {year}. You MUST use this year.
```

**设计特点**：
- **强制引用来源**：`Sources:` 节是 CRITICAL REQUIREMENT，不是建议。搜索引擎的核心价值是可追溯。
- **时间注入**：动态注入当前年月，因为 Claude 的训练数据截止时间可能导致搜索过时的内容。

---

### 4.2.8 WebFetchTool Prompt（46 行）

> **源文件**：`src/tools/WebFetchTool/prompt.ts`

WebFetchTool 的 Prompt 有两层——外层教 Claude 何时使用，内层是**二级模型的 Prompt**：

```
外层（教 Claude 使用工具）：
  - URL must be fully-formed
  - HTTP → auto upgrade to HTTPS
  - For GitHub URLs, prefer gh CLI

内层（教 Haiku 处理抓取结果）：
  - 版权保护：最多 1 条 < 15 词的引用
  - 不复制歌词
  - 不做 30+ 词的替代性摘要
```

> **Prompt 设计特点 — 嵌套 Prompt**：两层 Prompt 各自面向不同的模型。外层给主模型看，内层给处理网页内容的 Haiku 看。内层包含严格的版权合规指令。

---

### 4.2.9 其余工具 Prompt 摘要

| 工具 | 源文件 | 行数 | Prompt 核心要点 |
|------|--------|------|----------------|
| FileReadTool | `src/tools/FileReadTool/prompt.ts` | ~40 | 绝对路径、支持 PDF/图片/Notebook、默认 2000 行、空文件警告 |
| FileWriteTool | `src/tools/FileWriteTool/prompt.ts` | ~30 | 必须先读后写、不主动创建 md/README 文件、不用 emoji |
| EnterPlanModeTool | `src/tools/EnterPlanModeTool/prompt.ts` | ~60 | 内外用户两套 Prompt（用户分群）、7 种触发场景 |
| BriefTool | `src/tools/BriefTool/prompt.ts` | ~50 | "工具外的文字大多数用户不会看到"、proactive 节、Ack-Work-Result 模式 |
| ConfigTool | `src/tools/ConfigTool/prompt.ts` | ~30 | 读取免授权写入需确认、动态生成设置文档 |
| CronCreateTool | `src/tools/ScheduleCronTool/prompt.ts` | ~40 | cron 表达式用本地时间、避免 :00/:30 整点（fleet 负载均衡） |
| SendMessageTool | `src/tools/SendMessageTool/prompt.ts` | ~30 | Agent 间通信格式、广播 vs 定向 |
| SleepTool | `src/tools/SleepTool/prompt.ts` | ~30 | Tick 机制、成本意识、优于 Bash sleep |

---

## 4.3 Coordinator Prompt — 多 Agent 编排

> **源文件**：`src/coordinator/coordinatorMode.ts`

Coordinator 模式让 Claude 变成一个**调度员**，不亲自干活，只负责分配和综合。

**角色定义**：

```
你是一个编排者。你的工作是：
1. 理解用户的任务
2. 分解为子任务
3. 分配给 Worker Agent
4. 综合结果
5. 向用户报告
```

**工具限制**：Coordinator 只有 3 个工具：`AgentTool`、`TaskStopTool`、`SendMessage`。没有文件操作、没有 Bash——这从工具层保证了 Coordinator 不会"亲自下场干活"。

**并行优先原则**：

```
Parallelism is your superpower.
Never write "based on your findings, fix it" — that's lazy delegation.
Synthesize worker findings BEFORE directing follow-up work.
```

**继续 vs 新建的决策矩阵**：

```
Continue existing worker vs. Spawn new one?
  - Context overlap high → Continue (SendMessage)
  - Context overlap low  → Spawn new (AgentTool)
  - Direction changed    → Stop old (TaskStop) + Spawn new
```

> **Prompt 设计特点 — 工具限制 + Prompt 指令的双重保证**：不只在 Prompt 里说"你不要亲自干活"，还在工具注册层移除了所有文件操作工具。Prompt 可以被模型忽略，工具限制不能。

> **Prompt 设计特点 — 命名反模式**："Never write 'based on your findings, fix it' — that's **lazy delegation**"。给坏行为一个**名字**，让 Claude 能识别和避免它。人类也是这样学习的——知道一个反模式的名字，就更容易避免。

> **下一节**：[4.4-4.5 Compact 与记忆系统 Prompt](./44-compact-and-memory-prompts.md)
