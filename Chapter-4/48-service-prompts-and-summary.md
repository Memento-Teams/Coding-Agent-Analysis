# 4.8-4.9 服务级 Prompt 与技巧总结

> 本节覆盖隐藏在实现文件中的精彩 Prompt——包括"做梦"、陪伴精灵、自主模式——以及全部 Prompt 工程技巧的总结。

## 4.8 服务级 Prompt

### 4.8.1 Session Memory 模板

```
9 个固定章节：
- Session Title
- Current State         ← 最关键，确保连续性
- Task Specification
- Files and Functions
- Workflow
- Errors & Corrections
- Documentation
- Learnings
- Worklog

规则：保留章节标题、每节 <2000 tokens、总量 <12000 tokens
```

### 4.8.2 Memory 提取 Prompt

```
两步流程：
1. 写入记忆文件（带 frontmatter）
2. 在 MEMORY.md 中添加索引

MEMORY.md 规则：
- 仅索引，不放内容
- 每条 <150 字符
- 无 frontmatter
- 检查重复后再写
```

### 4.8.3 Magic Docs Prompt

```
Philosophy: HIGH SIGNAL ONLY
- Architecture, entry points, WHY/HOW/WHERE/PATTERNS
- Skip: obvious code, exhaustive APIs, implementation steps
- Be TERSE
- Update in-place, remove outdated info
```

---

## 4.8b 隐藏在实现文件中的 Prompt

以下 Prompt 不在标准的 `prompt.ts` 文件中，而是嵌入在各种实现文件里。它们同样精彩。

### 4.8b.1 Dream Prompt — 记忆整合"做梦"

这是整个系统中最诗意的 Prompt 名称。当 Claude Code 空闲时，它会"做梦"——回顾和整理记忆。

```
You are performing a dream — a reflective pass over your memory files.
Synthesize what you've learned recently into durable, well-organized
memories so that future sessions can orient quickly.
```

**四个阶段**：

1. **Orient** — 审视现有记忆文件
2. **Gather recent signal** — 从日志、漂移的记忆、对话中收集新信息
3. **Consolidate** — 写入/更新记忆文件，合并信号，转换相对日期为绝对日期
4. **Prune and index** — 保持索引在 200 行、~25KB 以内

> **Prompt 技巧 — 隐喻命名**：把记忆整合叫"做梦"不只是好听——它精确传达了这个过程的本质：无意识的、反思性的、在空闲时发生的。隐喻帮助 Claude 理解这个任务的"气质"：不是紧急的执行，而是从容的回顾。

### 4.8b.2 记忆相关性选择 Prompt

用 Sonnet 模型快速筛选最相关的 5 条记忆：

```
You are selecting memories that will be useful to Claude Code
as it processes a user's query.
```

**关键规则**：最多选 5 条。过滤掉最近使用过的工具的"使用参考"类文档（避免重复），但保留关于那些工具的**警告和注意事项**。

### 4.8b.3 Proactive/Autonomous Mode Prompt

当 Claude Code 在自主模式下运行时的行为指令：

**首次唤醒**：

```
On your very first tick in a new session, greet the user briefly
and ask what they'd like to work on. Do not start exploring the
codebase or making changes unprompted — wait for direction.
```

**后续唤醒**：

```
Look for useful work. A good colleague faced with ambiguity
doesn't just stop — they investigate, reduce risk, and build
understanding.
```

**空闲协议**：

```
If a tick arrives and you have no useful action to take,
call Sleep immediately. Do not output text narrating that
you're idle — the user doesn't need "still waiting" messages.
```

**终端焦点感知**：
- 用户离开（Unfocused）→ 大胆自主行动，做决定，探索，提交，推送
- 用户在看（Focused）→ 展示选择，大改动前先问，保持输出简洁

> **Prompt 技巧 — 情境感知指令**：根据用户是否在看屏幕给出不同行为指令。Claude 在用户离开时像一个独立工作的同事，在用户回来时像一个汇报进展的助手。

### 4.8b.4 Token Budget 续写 Prompt

当用户设置了 Token 目标（如 "+500k"）时：

```
Stopped at ${pct}% of token target (${fmt(turnTokens)} / ${fmt(budget)}).
Keep working — do not summarize.
```

配套指令：

```
When the user specifies a token target, your output token count
will be shown each turn. Keep working until you approach the target.
The target is a hard minimum, not a suggestion.
If you stop early, the system will automatically continue you.
```

### 4.8b.5 Buddy/Companion Prompt — 陪伴精灵

```
A small ${species} named ${name} sits beside the user's input box
and occasionally comments in a speech bubble. You're not ${name} —
it's a separate watcher.

When the user addresses ${name} directly (by name), its bubble
will answer. Your job in that moment is to stay out of the way:
respond in ONE line or less. Don't explain that you're not ${name}
— they know. Don't narrate what ${name} might say — the bubble
handles that.
```

> **Prompt 技巧 — 角色边界**：精确定义了 Claude 和 Buddy 之间的"领土"。"Don't explain that you're not ${name} — they know" 阻止了 Claude 过度解释的倾向。

### 4.8b.6 Default Agent Prompt

```
You are an agent for Claude Code, Anthropic's official CLI for Claude.
Complete the task fully — don't gold-plate, but don't leave it half-done.
```

> **Prompt 技巧 — 双向约束**："Don't gold-plate, but don't leave it half-done" — 同时限制过度和不足。比单方向的 "be thorough" 或 "be concise" 更精确。

### 4.8b.7 System Prompt Section 缓存机制

```tsx
// 安全的：memoized，直到 /clear 或 /compact 才重新计算
systemPromptSection('env_info', async () => { ... })

// 危险的：每轮都重新计算，破坏 Prompt Cache
DANGEROUS_uncachedSystemPromptSection('current_date', async () => { ... }, '日期每天变')
```

> **为什么 "DANGEROUS"？** 每次内容变化都会导致 Prompt Cache 失效。开发者必须显式标记为 DANGEROUS 并写明原因——用命名来阻止滥用。

---

## 4.9 Prompt 工程技巧总结

### 结构技巧

| 技巧 | 用在哪里 | 效果 |
|------|---------|------|
| **安全红线前置** | 主 Prompt 开头 | 利用 Primacy Effect |
| **首尾强化** | Compact Prompt | 对抗 Lost in the Middle |
| **三次重复禁令** | Plan Agent | 确保关键约束被遵守 |
| **XML 标签分区** | `<analysis>` `<summary>` | 草稿和输出分离 |
| **BLOCKING REQUIREMENT** | SkillTool | 强信号词防止忽略 |

### 行为技巧

| 技巧 | 用在哪里 | 效果 |
|------|---------|------|
| **反面指令** | 任务执行原则 | 对抗 LLM 的过度 helpful |
| **决策框架** | 谨慎行动 | 覆盖未知场景 |
| **一对一映射** | 工具优先级 | 消除模糊性 |
| **When to / When NOT to** | Plan Mode、Todo | 双向界定使用范围 |
| **预告失败条件** | FileEdit | 让 AI 主动避免错误 |

### 认知技巧

| 技巧 | 用在哪里 | 效果 |
|------|---------|------|
| **命名反模式** | Coordinator ("lazy delegation") | AI 能识别并避免 |
| **预言内心挣扎** | Verification Agent | 预防接种式防护 |
| **教 AI 怀疑自己** | 记忆系统 | "记忆说存在 ≠ 现在存在" |
| **封闭分类法** | 记忆类型（4 种） | 防止开放式过度存储 |
| **用户分群** | Plan Mode（内外版） | 不同熟练度不同 Prompt |

### 效率技巧

| 技巧 | 用在哪里 | 效果 |
|------|---------|------|
| **静态/动态分界** | 主 Prompt | 最大化 Prompt Cache |
| **Scratchpad 模式** | Compact | 思考空间不污染输出 |
| **条件注入** | Feature gate | 不需要的指令不占 Token |
| **截断 + 警告** | Git 状态 | 省 Token + 提供降级方案 |
| **表格化决策** | Coordinator | LLM 读表格比读段落更准确 |

---

> 返回 [Chapter 4 目录](./README.md) | 返回 [Chapter 0 总览](../Chapter-0/README.md)
