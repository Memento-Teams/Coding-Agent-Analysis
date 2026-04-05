# 4.4-4.5 Compact Prompt 与记忆系统 Prompt

> 对话压缩和记忆系统是 Claude Code 对抗 LLM 上下文限制的两大武器。它们的 Prompt 设计体现了一些最高级的技巧。

## 4.4 Compact Prompt — 对话压缩

当对话太长需要压缩时，让 Claude 自己总结之前的对话。3 个变体：

- **变体 A：完整压缩** — 输出 9 个固定章节（Primary Request、Technical Concepts、Files and Code、Errors、Problem Solving、User Messages、Pending Tasks、Current Work、Next Step）
- **变体 B：部分压缩** — 只总结最近的消息
- **变体 C：截止点压缩** — 总结到当前轮次，把 Current Work 改为 Work Completed

### Scratchpad 模式

```xml
请先在 <analysis> 标签中做草稿分析：
<analysis>
  ...你的分析过程（不会出现在最终摘要中）...
</analysis>

然后在 <summary> 标签中输出最终摘要：
<summary>
  ...正式的总结内容...
</summary>
```

> **Prompt 技巧 — Scratchpad 模式**：`<analysis>` 是 Claude 的"草稿纸"，用于思考和组织，但被系统丢弃，不进入最终摘要。这是工业级 Prompt 中常用的模式：给 AI 一个"不计入结果"的思考空间。

### 首尾强化

工具禁止指令在 Prompt 的**开头和结尾**各出现一次（NO_TOOLS_PREAMBLE + NO_TOOLS_TRAILER）。因为 LLM 对中间部分的记忆最弱（Lost in the Middle 问题）。

---

## 4.5 记忆系统 Prompt

### 四种记忆类型

```
- user: 用户角色、偏好、知识水平
- feedback: 用户的纠正和确认（"不要 mock 数据库"）
- project: 进行中的工作、目标、截止日期
- reference: 外部系统中信息的指针
```

每种类型都有详细的：`when_to_save`（什么时候该保存）、`how_to_use`（什么时候该使用）、`examples`（2-3 个具体示例）。

### 不该保存什么

```
- Code patterns, conventions, architecture → 从代码中推导
- Git history → git log 是权威来源
- Debugging solutions → 修复已在代码中
- Anything in CLAUDE.md files → 已有
- Ephemeral task details → 用 Tasks/Plans
```

> **Prompt 技巧 — 封闭分类法（Closed Taxonomy）**：只有 4 种记忆类型，不允许新增。这防止了"什么都存"的倾向。LLM 面对开放式指令时容易过度热心，封闭分类法给它一个明确的边界。

### 记忆可靠性警告

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

> **Prompt 技巧 — 教 AI 怀疑自己的记忆**：这段 Prompt 教 Claude 区分"记忆中说存在"和"现在确实存在"。这对避免幻觉（hallucination）非常有效——不是禁止 Claude 使用记忆，而是要求它**验证后再使用**。

> **下一节**：[4.6-4.7 Agent 与 Skill Prompt](./46-agent-and-skill-prompts.md)
