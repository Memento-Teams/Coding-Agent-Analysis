# 4.6-4.7 内建 Agent 与 Skill 的 Prompt

> 5 个内建 Agent 和 17 个内建 Skill 的 Prompt 设计，展示了如何为不同角色定制行为。其中 Verification Agent 的 Prompt 是整个系统中最精妙的。

---

## 4.6 内建 Agent 的 Prompt

> **源目录**：`src/tools/AgentTool/built-in/`

每个内建 Agent 是一个 TypeScript 对象，包含 `agentType`、`whenToUse`、`disallowedTools`、`source`、`model`、`getSystemPrompt()` 等字段。`getSystemPrompt()` 返回的就是 Agent 的核心 Prompt。

### 4.6.1 Explore Agent — 快速搜索专家

> **源文件**：`src/tools/AgentTool/built-in/exploreAgent.ts`（83 行）

```
You are a file search specialist that excels at thoroughly navigating
codebases. Your strengths: glob patterns, regex search, file reading.

Thoroughness scaling:
- "quick": basic searches
- "medium": moderate exploration
- "very thorough": comprehensive analysis across multiple locations
```

| 属性 | 值 |
|------|------|
| 模型 | Haiku（最快、最便宜） |
| 可用工具 | 只读（Glob、Grep、Read、WebFetch、WebSearch） |
| 禁用工具 | Agent、ExitPlanMode、Edit、Write、NotebookEdit |

**设计特点**：
- **只读强制**：不是在 Prompt 里说"不要修改文件"，而是在 `disallowedTools` 中直接移除所有写入工具。双重保证。
- **三级搜索深度**：`quick/medium/very thorough` 不是让 Claude 自己判断，而是由调用方指定。这避免了模型在"搜索多深"上的犹豫。
- **Haiku 模型**：搜索任务不需要推理深度，用 Haiku 既快又省钱。模型选择是 Agent 设计中最重要的决策之一。

---

### 4.6.2 Plan Agent — 只读规划者

> **源文件**：`src/tools/AgentTool/built-in/planAgent.ts`

```
CRITICAL: You must NOT modify any files.
  - Do NOT use FileEdit, FileWrite, or NotebookEdit
  - Do NOT run Bash commands that create/modify/delete files

Process: Understand → Explore → Design → Detail
Output: 3-5 critical files + implementation plan
```

| 属性 | 值 |
|------|------|
| 模型 | 继承父会话 |
| 可用工具 | 只读工具（同 Explore） |
| 禁用工具 | Agent、ExitPlanMode、Edit、Write、NotebookEdit |

**设计特点**：
- **三次重复禁令**："不能修改文件"在 Plan Agent 的 Prompt 中出现了**三次**，用不同的措辞（"must NOT modify"、"Do NOT use"、"Do NOT run"）。这不是冗余——对 LLM 来说，关键约束的重复是确保遵守的最可靠方法。
- **四步流程**：`Understand → Explore → Design → Detail` 给了 Claude 一个思考框架，避免它直接跳到"给方案"而跳过理解和探索。
- **输出约束**：要求输出 3-5 个关键文件 + 实现计划。数量约束（3-5）防止了过于笼统或过于琐碎的输出。

---

### 4.6.3 Verification Agent — 对抗性测试者（152 行）

> **源文件**：`src/tools/AgentTool/built-in/verificationAgent.ts`（152 行）

这是所有 Agent 中 Prompt 最精妙的——152 行，是 Explore Agent（83 行）的近两倍。长度本身就说明了验证任务的复杂性。

**两个已知失败模式**：

```
Failure Pattern 1: Verification Avoidance
  → 读代码而不是运行测试
  → "看起来没问题" 而不是 "测试通过了"

Failure Pattern 2: Seduced by First 80%
  → 精美的 UI 掩盖了 bug
  → 前 80% 的完美让你放松了后 20% 的检查
```

**心理防御（预防接种）**：

```
You will feel the urge to skip checks. You will rationalize:
"The code looks clean" or "The tests would probably pass."
RESIST THIS URGE.
```

**证据要求（PASS 格式）**：

```
Every PASS must include:
  ### Check: [what you're verifying]
  **Command run:** [exact command]
  **Output observed:** [copy-paste, not paraphrased]
  **Result: PASS/FAIL**
```

**对抗性探测清单**：

```
- Concurrency: parallel create-if-not-exists
- Boundary values: 0, -1, empty, long, unicode
- Idempotency: same request twice
- Orphan operations: delete non-existent IDs
```

**设计特点**：

1. **命名失败模式**：不是笼统说"要仔细验证"，而是给出两个具体的、有名字的失败模式。Claude 在识别具体模式时比遵守抽象原则更可靠。

2. **预言内心挣扎**（Inoculation Theory）："You will feel the urge to skip checks. You will rationalize..." 这提前预告了 Claude 会经历的"偷懒冲动"，让它在真正遇到时能识别。类似于疫苗——提前暴露弱化的威胁，增强免疫力。

3. **证据链要求**：PASS 格式要求"命令 → 输出 → 判断"的完整链条。这防止了最常见的验证失败——"看了代码就说通过了"。`copy-paste, not paraphrased` 进一步阻止 Claude 用自己的话"美化"输出。

4. **对抗性探测**：不是"测试一下"，而是给出具体的边界条件探测清单。这确保验证不只是"happy path 通过了"。

> **Prompt 设计特点 — 认知偏误预防**：Verification Agent 的 Prompt 本质上是一份"认知偏误清单"——它不是在教 Claude 如何测试，而是在教 Claude 如何**对抗自己偷懒的倾向**。这是整个 Claude Code Prompt 系统中最高级的技巧。

---

### 4.6.4 General Purpose Agent

> **源文件**：`src/tools/AgentTool/built-in/generalPurposeAgent.ts`

```
You are an agent for Claude Code.
When you complete the task, respond with a concise report.
Don't gold-plate, but don't leave it half-done.
```

| 属性 | 值 |
|------|------|
| 模型 | 继承父会话 |
| 可用工具 | 全部（* 减去 Agent 自身） |

**设计特点**：
- **双向约束**："Don't gold-plate, but don't leave it half-done" 同时限制过度和不足。比单方向的 "be thorough" 或 "be concise" 更精确。这给了 Claude 一个"完成度光谱"上的两个边界。
- **极简 Prompt**：只有几行。因为 General Purpose Agent 继承了主会话的大部分上下文，不需要额外的行为指导。

---

### 4.6.5 Claude Code Guide Agent

> **源文件**：`src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts`

```
Approach:
1. Determine which domain the question belongs to
2. Fetch the docs map from official URLs
3. Identify relevant docs
4. Fetch specific pages
5. Provide guidance with citations
```

| 属性 | 值 |
|------|------|
| 模型 | Haiku |
| 可用工具 | Glob、Grep、Read、WebFetch、WebSearch |

**设计特点**：
- **动态配置注入**：`getSystemPrompt()` 会注入用户当前的 skills、agents、MCP servers 列表。这让 Guide Agent 能回答"我有哪些 skill"这类问题。
- **5 步引导法**：不是直接回答，而是先确定领域 → 查文档 → 定位 → 抓取 → 引用回答。这确保答案基于最新的官方文档，而非训练数据中的过时信息。
- **Haiku 模型**：查文档不需要深度推理，Haiku 足够且更快。

---

## 4.7 内建 Skill 的 Prompt

> **源目录**：`src/skills/bundled/`

每个 Skill 是一个 TypeScript 文件，通过 `registerBundledSkill()` 注册。核心是一个 `buildPrompt(instruction)` 函数，接收用户指令并拼装完整的执行 Prompt。

### 4.7.1 /simplify — 代码审查（3 个并行 Agent）

> **源文件**：`src/skills/bundled/simplify.ts`（69 行）

```
Phase 1: Run git diff to identify changes
Phase 2: Launch THREE review agents in parallel:
  - Agent 1: Code Reuse Review (search for existing utilities)
  - Agent 2: Code Quality Review (redundant state, parameter sprawl, copy-paste)
  - Agent 3: Efficiency Review (N+1 patterns, missed concurrency, memory leaks)
Phase 3: Aggregate findings and fix
```

**设计特点**：
- **结构化分工 vs 全面审查**：不是让一个 Agent 做"全面审查"，而是拆成 3 个正交视角（复用/质量/效率）。每个 Agent 的注意力更集中，审查更彻底。这也利用了并行执行——3 个 Agent 同时工作。
- **69 行极简实现**：Skill 文件本身很短，因为大部分复杂度在 Agent 的 Prompt 中。Skill 只负责编排。

---

### 4.7.2 /batch — 大规模并行重构

> **源文件**：`src/skills/bundled/batch.ts`

```
Phase 1 (Plan Mode): Decompose into 5-30 independent units
Phase 2: Spawn workers with isolation: "worktree" + run_in_background
Phase 3: Track progress, render status table

WORKER_INSTRUCTIONS template:
  "You are worker N of M. Your unit is: [description]
   1. Create worktree branch
   2. Apply changes per unit specification
   3. Run /simplify
   4. Run tests
   5. Commit + Create PR
   6. Output 'PR: <url>'"
```

**设计特点**：
- **Plan Mode 强制**：Phase 1 要求进入 Plan Mode，确保在动手之前有完整的分解方案。这是 `/batch` 独有的——其他 Skill 不要求。
- **Worker 隔离**：每个 Worker 在独立的 git worktree 中工作 + `run_in_background`。这实现了真正的并行：多个 Worker 同时修改代码，互不干扰。
- **端到端闭环**：Worker 不只是修改代码，还要 `/simplify` 审查 → 运行测试 → 提交 → 创建 PR。每个 Worker 输出一个 PR URL，Coordinator 收集所有 URL 作为最终报告。
- **动态 Agent 数量**：`MIN_AGENTS=5, MAX_AGENTS=30`，根据分解出的单元数自动调整。

---

### 4.7.3 /debug — 调试辅助

> **源文件**：`src/skills/bundled/debug.ts`

```
1. Enable debug logging if not active
2. Read last 20 lines of debug log
3. Offer to spawn claude-code-guide for feature questions
4. Suggest fixes
```

**设计特点**：
- **先看日志再诊断**：强制先读 debug log，而不是让 Claude 猜测问题。这避免了 LLM 常见的"不看证据就给方案"的倾向。
- **委托给专家**：如果问题是 Claude Code 功能相关的，不自己回答，而是建议启动 `claude-code-guide` Agent。这体现了"专业分工"的设计思想。

---

### 4.7.4 /stuck — 进程诊断

> **源文件**：`src/skills/bundled/stuck.ts`

```
Investigation steps:
1. ps -axo pid,pcpu,rss,etime,state,comm,command
2. Detect process states (D=disk sleep, T=stopped, Z=zombie)
3. Stack sampling for hung processes
4. Do NOT kill or signal processes - diagnostic only
```

**设计特点**：
- **纯诊断，不治疗**："Do NOT kill or signal processes" 是核心约束。`/stuck` 只负责分析进程状态，不负责修复。这避免了误杀进程的风险——诊断和治疗的风险等级完全不同。
- **系统级知识注入**：Prompt 中包含进程状态码（D/T/Z）的含义解释，补充了 Claude 可能不熟悉的操作系统知识。

---

### 4.7.5 /remember — 记忆整理

> **源文件**：`src/skills/bundled/remember.ts`

```
Classification Table:
- CLAUDE.md: Project conventions for all contributors
- CLAUDE.local.md: Personal instructions (gitignored)
- Team memory: Org-wide knowledge shared across agents
- Auto-memory: Working notes, temporary context
```

**设计特点**：
- **四路分类决策**：不是"保存到记忆"这么简单，而是要求 Claude 判断信息应该存到哪里。不同的存储位置有不同的受众（所有人 / 个人 / 团队 / 临时）和不同的生命周期。
- **CLAUDE.md vs CLAUDE.local.md**：前者提交到 Git，后者 gitignored。这让用户能区分"团队约定"和"个人偏好"。

---

### 4.7.6 /update-config — 配置管理（149 行 Hook 文档）

> **源文件**：`src/skills/bundled/updateConfig.ts`

```
Hook Events: PreToolUse, PostToolUse, Notification, Stop
Hook Types: command, prompt, agent

4-Step Hook Construction:
1. Write the hook script
2. Test with synthetic stdin
3. Add to settings.json
4. Verify it fires correctly
```

**设计特点**：
- **内嵌完整参考文档**：149 行中大部分是 Hook 系统的参考文档。这意味着当用户说"帮我设置一个 hook"时，Claude 不需要去查文档——Prompt 里就有完整的 API 参考。
- **4 步安全流程**：不是直接写 settings.json，而是"写脚本 → 测试 → 配置 → 验证"。先测试再配置的策略防止了配置错误导致 Claude Code 行为异常。

---

### 4.7.7 /claude-api — API 文档注入

> **源文件**：`src/skills/bundled/claudeApi.ts`

```
Dynamic detection:
  - Detect project language (py, ts, java, go, ruby...)
  - Lazy-load 247KB of docs
  - Inline language-specific sections
```

**设计特点**：
- **按需加载**：247KB 的 API 文档不是每次都加载，而是检测到项目使用了 `anthropic`/`@anthropic-ai/sdk` 等包时才触发。
- **语言感知**：根据项目语言（Python/TypeScript/Java/Go/Ruby）注入对应的 SDK 文档段。Java 开发者看不到 Python 的 SDK 示例。
- **触发规则**：`TRIGGER when: code imports anthropic / @anthropic-ai/sdk / claude_agent_sdk` + `DO NOT TRIGGER when: code imports openai / other AI SDK`。明确区分了 Claude API 和其他 AI SDK。

---

### 4.7.8 其余 Skill 摘要

| Skill | 源文件 | Prompt 核心 | 设计特点 |
|-------|--------|------------|---------|
| /verify | `skills/bundled/verify.ts` | 验证执行结果 | 复用 Verification Agent 的对抗性测试策略 |
| /loop | `skills/bundled/loop.ts` | 解析时间间隔 → cron 表达式 | 默认 10 分钟间隔，支持 "5m" 等人类友好格式 |
| /keybindings | `skills/bundled/keybindingsHelp.ts` | 快捷键参考 + JSON Schema | 内嵌完整的 keybindings.json Schema |
| /schedule | `skills/bundled/schedule.ts` | 远程 Agent 调度向导 | 引导式创建，而非一步完成 |
| /skillify | `skills/bundled/skillify.ts` | 4 轮面试式 Skill 定义生成 | 交互式收集信息，逐步细化 Skill 定义 |
| /skill-creator | `skills/bundled/skillCreator.ts` | 创建/测试/优化 Skill | 包含 eval 运行和性能基准测试 |

> **下一节**：[4.8-4.9 服务级 Prompt 与技巧总结](./48-service-prompts-and-summary.md)
