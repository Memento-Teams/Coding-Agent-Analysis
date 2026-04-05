# 4.6-4.7 内建 Agent 与 Skill 的 Prompt

> 5 个内建 Agent 和 17 个内建 Skill 的 Prompt 设计，展示了如何为不同角色定制行为。其中 Verification Agent 的 Prompt 是整个系统中最精妙的。

## 4.6 内建 Agent 的 Prompt

### 4.6.1 Explore Agent — 快速搜索专家

```
You are a file search specialist that excels at thoroughly navigating
codebases. Your strengths: glob patterns, regex search, file reading.

Thoroughness scaling:
- "quick": basic searches
- "medium": moderate exploration
- "very thorough": comprehensive analysis across multiple locations
```

模型：Haiku（快速）。工具：只读（Glob、Grep、Read）。

### 4.6.2 Plan Agent — 只读规划者

```
CRITICAL: You must NOT modify any files.
  - Do NOT use FileEdit, FileWrite, or NotebookEdit
  - Do NOT run Bash commands that create/modify/delete files

Process: Understand → Explore → Design → Detail
Output: 3-5 critical files + implementation plan
```

> **Prompt 技巧 — 三次重复关键禁令**："不能修改文件"这个禁令在 Plan Agent 的 Prompt 中出现了**三次**，用不同的措辞。这不是冗余——对 LLM 来说，关键约束的重复是确保遵守的最可靠方法。

### 4.6.3 Verification Agent — 对抗性测试者（130 行）

这是所有 Agent 中 Prompt 最精妙的。

**两个已知失败模式**：

```
Failure Pattern 1: Verification Avoidance
  → 读代码而不是运行测试
  → "看起来没问题" 而不是 "测试通过了"

Failure Pattern 2: Seduced by First 80%
  → 精美的 UI 掩盖了 bug
  → 前 80% 的完美让你放松了后 20% 的检查
```

**心理防御**：

```
You will feel the urge to skip checks. You will rationalize:
"The code looks clean" or "The tests would probably pass."
RESIST THIS URGE.
```

**证据要求**：

```
Every PASS must include:
  ### Check: [what you're verifying]
  **Command run:** [exact command]
  **Output observed:** [copy-paste, not paraphrased]
  **Result: PASS/FAIL**
```

**对抗性探测**：

```
- Concurrency: parallel create-if-not-exists
- Boundary values: 0, -1, empty, long, unicode
- Idempotency: same request twice
- Orphan operations: delete non-existent IDs
```

> **Prompt 技巧 — 命名认知偏误**："You will feel the urge to skip checks. You will rationalize..." 这是最高级的技巧之一。它不只是说"不要跳过检查"，而是**预言 Claude 会怎么给自己找借口**，然后让 Claude 识别这个模式。类似于人类心理学中的"预防接种理论"——提前暴露弱论点，增强对它的抵抗力。

### 4.6.4 General Purpose Agent

```
You are an agent for Claude Code.
When you complete the task, respond with a concise report.
Don't gold-plate, but don't leave it half-done.
```

工具：全部。模型：继承父会话。

### 4.6.5 Claude Code Guide Agent

动态注入用户的配置（skills、agents、MCP servers），然后：

```
Approach:
1. Determine which domain the question belongs to
2. Fetch the docs map from official URLs
3. Identify relevant docs
4. Fetch specific pages
5. Provide guidance with citations
```

---

## 4.7 内建 Skill 的 Prompt

### 4.7.1 /simplify — 代码审查（3 个并行 Agent）

```
Phase 1: Run git diff to identify changes
Phase 2: Launch THREE review agents in parallel:
  - Agent 1: Code Reuse Review (search for existing utilities)
  - Agent 2: Code Quality Review (redundant state, parameter sprawl, copy-paste)
  - Agent 3: Efficiency Review (N+1 patterns, missed concurrency, memory leaks)
Phase 3: Aggregate findings and fix
```

> **Prompt 技巧 — 结构化分工**：不是让一个 Agent 做所有审查，而是分成 3 个视角（复用、质量、效率），每个 Agent 只关注一个方面。这比"做一次全面审查"更彻底，因为每个 Agent 的注意力更集中。

### 4.7.2 /batch — 大规模并行重构

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
   5. Commit
   6. Create PR
   7. Output 'PR: <url>'"
```

### 4.7.3 /debug — 调试辅助

```
1. Enable debug logging if not active
2. Read last 20 lines of debug log
3. Offer to spawn claude-code-guide for feature questions
4. Suggest fixes
```

### 4.7.4 /stuck — 进程诊断（ant-only）

```
Investigation steps:
1. ps -axo pid,pcpu,rss,etime,state,comm,command
2. Detect process states (D=disk sleep, T=stopped, Z=zombie)
3. Stack sampling for hung processes
4. Do NOT kill or signal processes - diagnostic only
```

### 4.7.5 /remember — 记忆整理

```
Classification Table:
- CLAUDE.md: Project conventions for all contributors
- CLAUDE.local.md: Personal instructions
- Team memory: Org-wide knowledge
- Auto-memory: Working notes, temporary context
```

### 4.7.6 /update-config — 配置管理（149 行 Hook 文档）

包含完整的 Hook 系统参考：

```
Hook Events: PreToolUse, PostToolUse, Notification, Stop
Hook Types: command, prompt, agent

4-Step Hook Construction:
1. Write the hook script
2. Test with synthetic stdin
3. Add to settings.json
4. Verify it fires correctly
```

### 4.7.7 /claude-api — API 文档注入

```
Dynamic detection:
  - Detect project language (py, ts, java, go, ruby...)
  - Lazy-load 247KB of docs
  - Inline language-specific sections
```

### 4.7.8 其余 Skill 摘要

| Skill | Prompt 核心 |
|-------|------------|
| /verify | 验证执行结果（ant-only） |
| /loop | 解析时间间隔 → cron 表达式 |
| /keybindings | 快捷键参考 + JSON Schema |
| /schedule | 远程 Agent 调度向导 |
| /skillify | 4 轮面试式 Skill 定义生成（ant-only） |
| /loremIpsum | 测试用文本生成 |
| /claudeInChrome | Chrome 扩展指引 |

> **下一节**：[4.8-4.9 服务级 Prompt 与技巧总结](./48-service-prompts-and-summary.md)
