# 4.4-4.5 Compact Prompt 与记忆系统 Prompt

> 对话压缩和记忆系统是 Claude Code 对抗 LLM 上下文限制的两大武器。它们的 Prompt 设计体现了一些最高级的技巧。

---

## 4.4 Compact Prompt — 对话压缩

> **源文件**：`src/services/compact/prompt.ts`（374 行）

当对话太长需要压缩时，让 Claude 自己总结之前的对话。这个文件是所有服务级 Prompt 中最长的（374 行），因为它要精确控制压缩的质量和格式。

### 三个变体及其设计意图

| 变体 | 函数 | 触发场景 | 压缩范围 | 设计意图 |
|------|------|---------|---------|---------|
| A：完整压缩 | `getCompactPrompt()` | 用户手动 `/compact` 或 Auto-Compact | 整个对话 | 保留全局信息，适合长会话 |
| B：部分压缩 | `getPartialCompactPrompt()` | Context Collapse 归档旧消息 | 只压缩最近 N 条 | 保留旧压缩结果，增量更新 |
| C：截止点压缩 | `getPartialCompactUpTo()` | 恢复会话时压缩到当前轮 | 到当前轮为止 | `Current Work` → `Work Completed` |

**为什么需要三个变体？** 不同场景对"保留什么"的需求不同：

- **变体 A**：用户说"太长了，压缩一下"——需要全面总结
- **变体 B**：引擎自动触发——只需增量归档旧消息，保留最近的完整上下文
- **变体 C**：从磁盘恢复会话——需要把"正在做的事"改为"已经做完的事"

### 9 个固定输出章节

变体 A 要求输出严格遵循 9 个章节：

```
1. Primary Request      ← 用户最初要求什么
2. Technical Concepts   ← 涉及的技术概念
3. Files and Code       ← 操作过的文件、代码片段
4. Errors               ← 遇到的错误及解决方案
5. Problem Solving      ← 决策过程和策略
6. User Messages        ← 用户的原话（逐条引用）
7. Pending Tasks        ← 还没完成的事
8. Current Work         ← 正在进行的事
9. Next Step            ← 下一步计划
```

**章节固定的原因**：自由格式的总结质量参差不齐。固定章节相当于一个"提取模板"，确保每次压缩都覆盖关键维度。特别是 `User Messages` 章节——要求逐条引用用户原话，防止压缩过程丢失用户的具体指令。

### Scratchpad 模式

```
DETAILED_ANALYSIS_INSTRUCTION:

请先在 <analysis> 标签中做草稿分析：
<analysis>
  ...你的分析过程（不会出现在最终摘要中）...
</analysis>

然后在 <summary> 标签中输出最终摘要：
<summary>
  ...正式的总结内容...
</summary>
```

**为什么需要草稿空间？** 压缩任务要求 Claude 在大量信息中做取舍判断。`<analysis>` 标签给 Claude 一个"不计入结果"的思考空间——可以列出所有候选信息，比较重要性，然后在 `<summary>` 中只输出精选后的结果。系统会丢弃 `<analysis>` 的内容，只保留 `<summary>`。

> **Prompt 设计特点 — Scratchpad 模式**：这是工业级 Prompt 中常用的模式。把"思考过程"和"最终输出"分离到不同的 XML 标签中，既提升了输出质量（允许 Claude 先想后写），又控制了结果格式（只取 summary 部分）。

### 首尾强化

```tsx
// prompt.ts 开头
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY.
Do NOT call any tools. Do NOT use tool_use blocks.`

// prompt.ts 结尾
const NO_TOOLS_TRAILER = `Remember: TEXT ONLY response.
Do NOT call any tools or use tool_use blocks.`
```

工具禁止指令在 Prompt 的**开头和结尾**各出现一次。这是因为 LLM 对中间部分的记忆最弱（Lost in the Middle 问题），而 Compact 任务绝对不能让 Claude 调用工具——否则压缩过程本身会产生新的对话内容。

> **Prompt 设计特点 — 首尾双重强化**：不是"在 Prompt 里写一次就够了"，而是利用 LLM 的注意力分布特征，在最容易被记住的位置（开头和结尾）放置最关键的约束。

### 用户摘要消息

```tsx
// formatCompactSummary() 生成的消息格式
`<summary>
${summaryContent}
</summary>

[The conversation above has been summarized to save context.
Previous messages have been replaced with this summary.]`
```

压缩结果不是直接替换对话，而是包裹在 `<summary>` 标签中，并附加一条说明消息。这让后续的 Claude 调用能识别"这是压缩过的历史"，而不是"这是用户说的话"。

---

## 4.5 记忆系统 Prompt

### 4.5.1 记忆提取 Prompt

> **源文件**：`src/services/extractMemories/prompts.ts`（154 行）

**两个构建函数**：
- `buildExtractAutoOnlyPrompt()` — 仅自动记忆（无团队作用域）
- `buildExtractCombinedPrompt()` — 自动 + 团队记忆

**角色声明**：

```
You are now acting as the memory extraction subagent.
Your job is to extract useful memories from the conversation
and save them to the appropriate files.
```

### 四种记忆类型（封闭分类法）

> **源文件**：`src/memdir/memoryTypes.ts`

```
- user: 用户角色、偏好、知识水平
    when_to_save: 用户自述身份、表达偏好、展示知识水平
    how_to_use: 个性化回复风格和复杂度

- feedback: 用户的纠正和确认
    when_to_save: "不要 mock 数据库"、"用 tabs 不用 spaces"
    how_to_use: 避免重复犯错

- project: 进行中的工作、目标、截止日期
    when_to_save: 长期项目的状态、里程碑、外部依赖
    how_to_use: 提供项目上下文

- reference: 外部系统中信息的指针
    when_to_save: "文档在 Confluence"、"CI 用的 GitHub Actions"
    how_to_use: 指引去正确的地方查找
```

每种类型都有详细的：`when_to_save`、`how_to_use`、`examples`（2-3 个具体示例）。

**为什么只有 4 种？** 开放式分类会导致"什么都存"——Claude 面对模糊指令时倾向于过度保存。4 种类型的封闭分类法给它一个明确的边界：如果信息不属于这 4 类，就不该存。

### 不该保存什么

```
- Code patterns, conventions, architecture → 从代码中推导
- Git history → git log 是权威来源
- Debugging solutions → 修复已在代码中
- Anything in CLAUDE.md files → 已有
- Ephemeral task details → 用 Tasks/Plans
```

**设计哲学**：记忆系统只存"代码里看不出来的信息"。代码规范、架构模式这些信息已经存在于代码本身中，重复存储只会造成同步困难和过期风险。

### 两阶段执行策略

```
Phase 1: Read ALL existing memory files (single turn)
Phase 2: Write ALL new/updated memories (single turn)
```

**为什么不交替读写？** 每次 LLM 调用都有成本（API 调用 + 延迟）。两阶段策略把所有读操作集中在一轮，所有写操作集中在下一轮，最小化 API 调用次数。同时，先读后写确保不会覆盖已有的记忆。

### 记忆可靠性警告

> **源文件**：`src/memdir/memdir.ts`

```
A memory that names a specific function, file, or flag is a CLAIM
that it existed WHEN THE MEMORY WAS WRITTEN. It may have been renamed,
removed, or never merged.

Before recommending:
- If memory names a file path → check the file exists
- If memory names a function → grep for it
- If user will act on recommendation → verify first

"The memory says X exists" is not the same as "X exists now."
```

> **Prompt 设计特点 — 教 AI 怀疑自己的记忆**：这段 Prompt 教 Claude 区分"记忆中说存在"和"现在确实存在"。这对避免幻觉（hallucination）非常有效——不是禁止 Claude 使用记忆，而是要求它**验证后再使用**。这是整个 Prompt 系统中最高级的认知校准技巧之一。

### 4.5.2 Session Memory 模板

> **源文件**：`src/services/SessionMemory/prompts.ts`（324 行）

**核心函数**：
- `loadSessionMemoryTemplate()` — 加载 10 节模板
- `buildSessionMemoryUpdatePrompt()` — 构建更新指令
- `truncateSessionMemoryForCompact()` — 压缩时截断

**10 个固定章节**：

```
- Session Title
- Current State         ← 最关键，确保连续性
- Task Specification
- Files and Functions
- Workflow
- Errors & Corrections
- Codebase and System Documentation
- Learnings
- Key Results
- Worklog
```

**更新指令的关键约束**：

```
IMPORTANT: This message and these instructions are NOT part of
the actual user conversation. They are a system directive.

Rules:
- NEVER modify section headers or italic descriptions
- Each section < 2000 tokens
- Total < 12000 tokens
- Current State section is MOST IMPORTANT for continuity
```

> **Prompt 设计特点 — 元指令隔离**：更新指令开头明确声明"这不是用户对话的一部分"，防止 Claude 把系统指令和用户消息混淆。

> **Prompt 设计特点 — 章节保护**："NEVER modify section headers or italic descriptions" 确保模板结构在多次更新后仍然完整。这是结构化数据在自然语言更新下的持久性保障。

> **下一节**：[4.6-4.7 Agent 与 Skill Prompt](./46-agent-and-skill-prompts.md)
