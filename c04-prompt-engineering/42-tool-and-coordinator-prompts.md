# 4.2-4.3 工具 Prompt 与 Coordinator Prompt

> 每个工具都有一个 `prompt.ts` 文件，教 Claude 什么时候该用、怎么用、什么时候不该用。Coordinator Prompt 则把 Claude 变成一个调度员。

## 4.2 工具 Prompt

### 4.2.1 BashTool — 最长的工具 Prompt（370 行）

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

> **Prompt 技巧 — 分层指令**：BashTool 的 Prompt 有三个层次：① 原则层（什么时候用 Bash）② 策略层（多命令如何编排）③ 规则层（Git 操作的具体禁令）。从抽象到具体，每层解决不同级别的问题。

### 4.2.2 AgentTool Prompt（287 行）

```
When NOT to use the Agent tool:
- If you want to read a specific file path → use Read
- If you are searching for "class Foo" → use Glob
- If you are searching within 2-3 files → use Read
```

> **Prompt 技巧 — 决策矩阵**：不是给一个模糊的"视情况而定"，而是给出精确的二选一标准。LLM 在有明确判断标准时表现更好。

### 4.2.3 SkillTool Prompt（242 行）

```
When a skill matches the user's request, this is a
BLOCKING REQUIREMENT: invoke the relevant Skill tool
BEFORE generating any other response about the task.
```

> **Prompt 技巧 — BLOCKING REQUIREMENT**：普通的 "should" 经常被 LLM 忽略，但 "BLOCKING REQUIREMENT" 结合全大写，几乎不会被跳过。

### 4.2.4 TodoWriteTool Prompt（181 行）

```
When to use: 3+ steps, non-trivial tasks, user asks for tracking
When NOT to use: single task, trivial changes, conversation

CRITICAL: Two forms required:
- content: imperative ("Fix the bug")
- activeForm: present continuous ("Fixing the bug")
```

> **Prompt 技巧 — When to / When NOT to 对比**：同时给出正面和反面场景，消除边界情况的歧义。

### 4.2.5 FileEditTool / GrepTool Prompt

**FileEditTool**：

```
- You must use Read at least once before editing.
- Preserve exact indentation as it appears AFTER the line number prefix.
- The edit will FAIL if old_string is not unique.
```

**GrepTool**：

```
- ALWAYS use Grep for search tasks. NEVER invoke grep or rg as Bash command.
- Pattern syntax: Uses ripgrep (not grep)
  - literal braces need escaping (use interface\{\} to find interface{})
```

> **Prompt 技巧 — 预告失败条件 + 纠正常见混淆**："The edit will FAIL if..." 让 Claude 在调用前就知道什么会失败；显式纠正 grep vs Grep、grep 语法 vs ripgrep 语法的混淆。

### 4.2.6 其余工具 Prompt 摘要

| 工具 | Prompt 核心要点 |
|------|----------------|
| FileReadTool | 绝对路径、支持 PDF/图片/Notebook、默认 2000 行 |
| FileWriteTool | 必须先读后写、不主动创建 md 文件 |
| WebSearchTool | 必须包含 Sources 节、用当前日期搜索 |
| EnterPlanModeTool | 内外用户两套 Prompt（用户分群） |
| BriefTool | "工具外的文字大多数用户不会看到"、Ack-Work-Result 模式 |
| ConfigTool | 读取免授权，写入需确认、动态生成设置文档 |
| CronCreateTool | cron 表达式用本地时间、避免 :00/:30 整点 |

---

## 4.3 Coordinator Prompt — 多 Agent 编排

Coordinator 模式让 Claude 变成一个**调度员**，不亲自干活，只负责分配和综合。

```
你是一个编排者。你的工作是：
1. 理解用户的任务
2. 分解为子任务
3. 分配给 Worker Agent
4. 综合结果
5. 向用户报告
```

**关键规则**：

```
Parallelism is your superpower.
Never write "based on your findings, fix it" — that's lazy delegation.
Synthesize worker findings BEFORE directing follow-up work.
```

**决策矩阵**：

```
Continue existing worker vs. Spawn new one?
  - Context overlap high → Continue (SendMessage)
  - Context overlap low  → Spawn new (AgentTool)
  - Direction changed    → Stop old (TaskStop) + Spawn new
```

> **Prompt 技巧 — 命名反模式**："Never write 'based on your findings, fix it' — that's **lazy delegation**"。给坏行为一个**名字**，让 Claude 能识别和避免它。人类也是这样学习的——知道一个反模式的名字，就更容易避免。

> **下一节**：[4.4-4.5 Compact 与记忆系统 Prompt](./44-compact-and-memory-prompts.md)
