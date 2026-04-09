# 11.2 Auto-Compact 深入

> Auto-Compact 是五层管线中最重的一层——Fork 一个子 Agent，让 Claude 总结自己的对话。这一节拆解它的完整流程、三种变体、以及 Prompt 中的精妙设计。

## 11.2.1 压缩的完整流程

```tsx
export async function compactConversation(
  messages: Message[],
  systemPrompt: SystemPrompt,
  options: CompactOptions
): Promise<CompactResult>
```

```
原始对话（假设 180K tokens）
  │
  ├── 1. 消息分组 (grouping.ts)
  │     按 API 轮次分组：
  │       每个 assistant 回复 + 对应的 tool_result = 一组
  │       tool_use → tool_result 配对不可拆散（拆开会导致 API 报错）
  │
  │     示例：
  │     ┌── Group 1 ──────────────────────────┐
  │     │ User: "修复 auth bug"                │
  │     │ Assistant: (text + tool_use:FileRead) │
  │     │ User: (tool_result: 文件内容)         │
  │     └─────────────────────────────────────┘
  │     ┌── Group 2 ──────────────────────────┐
  │     │ Assistant: (text + tool_use:Edit)     │
  │     │ User: (tool_result: 编辑成功)         │
  │     └─────────────────────────────────────┘
  │
  ├── 2. Fork 子 Agent 做摘要
  │     ├── 共享父会话的 Prompt Cache（关键优化）
  │     ├── 注入 Compact Prompt 模板
  │     └── Claude 输出结构化摘要
  │
  ├── 3. Token 估算验证
  │     压缩后的摘要是否低于阈值？
  │       是 → 继续
  │       否 → 再压缩一次（Recompaction）
  │
  ├── 4. 清理
  │     ├── 清除图片缓存（图片 Token 大，压缩后不再需要）
  │     ├── 清除文件元数据
  │     └── 重置 Recompaction 信息
  │
  └── 5. 返回压缩后的消息列表
```

### 为什么 Fork 子 Agent？

三个原因：

1. **共享 Prompt Cache**：子 Agent 与父会话共享相同的系统 Prompt 和对话历史前缀。API 不会重复计算这部分 Token 的费用，摘要操作几乎只付"输出 Token"的钱。
2. **不阻塞主线程**：压缩在后台进行，用户的交互流程不被打断。
3. **隔离副作用**：子 Agent 只能输出文本，不能调用工具。Compact Prompt 的开头和结尾都写了 `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.`

### 消息分组的约束

```
消息序列：
  User → Assistant(text + tool_use) → User(tool_result) → Assistant → ...

分组规则：
  ┌── Group 1 ──────────────────────────┐
  │ User message                        │
  │ Assistant message (text + tool_use) │
  │ User message (tool_result)          │
  └─────────────────────────────────────┘
  ┌── Group 2 ──────────────────────────┐
  │ Assistant message (pure text)        │
  │ User message (new input)             │
  └─────────────────────────────────────┘
```

**关键约束**：tool_use 和对应的 tool_result 必须在同一组。Claude API 要求 tool_use 后紧跟对应的 tool_result，拆开会报错。分组算法必须保证这个不变量。

## 11.2.2 三种压缩变体

不同场景对"保留什么"的需求完全不同。Claude Code 设计了三种变体：

| 变体 | 函数 | 触发场景 | 压缩范围 | 输出差异 |
|------|------|---------|---------|---------|
| A：完整压缩 | `getCompactPrompt()` | `/compact` 或 Auto-Compact | 整个对话 | 9 节完整摘要 |
| B：部分压缩 | `getPartialCompactPrompt()` | Context Collapse 归档旧消息 | 只压缩旧消息 | 增量摘要，保留近消息 |
| C：截止点压缩 | `getPartialCompactUpTo()` | 恢复会话时 | 到当前轮为止 | `Current Work` → `Work Completed` |

### 变体 A：完整压缩

最常见的变体。当 Token 超过阈值或用户手动 `/compact` 时触发。要求输出严格遵循 **9 个固定章节**：

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

**为什么固定 9 个章节？** 自由格式的总结质量参差不齐。Claude 在自由总结时倾向于：

- 过度关注最近的内容（Recency Bias）
- 丢失用户的具体措辞（Paraphrase Drift）
- 忽略错误和失败（Positivity Bias）

固定章节相当于一个"提取清单"，**强制**覆盖每个关键维度。特别是第 6 节 `User Messages`——要求逐条引用用户原话，防止压缩过程把"请用 tabs 缩进"改写为"用户要求使用特定缩进风格"。

### 变体 B：部分压缩

Context Collapse 触发。只压缩距离当前较远的旧消息，保留最近几轮的完整上下文：

```
对话：[旧消息 1-20] [近消息 21-25]
                │              │
                ▼              │
        压缩为摘要           保留原样
                │              │
                └──────┬───────┘
                       ▼
           [摘要] + [近消息 21-25]
```

好处：近消息是最有价值的（包含当前任务的细节），保留它们避免了完整压缩带来的信息丢失。

### 变体 C：截止点压缩

恢复会话时触发。关键差异：把 `Current Work`（正在做的）改写为 `Work Completed`（已经做完的），因为恢复时上次的"正在做"已经过去了。

## 11.2.3 Scratchpad 模式

Compact Prompt 中使用了一个高级技巧——给 Claude 一个"草稿空间"：

```
请先在 <analysis> 标签中做草稿分析：
<analysis>
  ...你的分析过程（不会出现在最终摘要中）...
</analysis>

然后在 <summary> 标签中输出最终摘要：
<summary>
  ...正式的总结内容...
</summary>
```

系统只保留 `<summary>` 中的内容，丢弃 `<analysis>`。

**为什么需要草稿空间？** 压缩是一个高认知负荷的任务——要在大量信息中做取舍判断。`<analysis>` 让 Claude 先把所有候选信息列出来、比较重要性、权衡取舍，然后在 `<summary>` 中输出精选结果。就像写文章先列大纲再正式写一样。

> 这是工业级 Prompt 中的常见模式：把"思考过程"和"最终输出"分离到不同的标签中，既提升输出质量（允许先想后写），又控制结果格式（只取 summary）。

## 11.2.4 首尾双重强化

```tsx
// prompt.ts 开头
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY.
Do NOT call any tools. Do NOT use tool_use blocks.`

// prompt.ts 结尾
const NO_TOOLS_TRAILER = `Remember: TEXT ONLY response.
Do NOT call any tools or use tool_use blocks.`
```

同一条指令在 Prompt 的**开头和结尾**各出现一次。这是因为 LLM 对 Prompt 中间部分的记忆最弱（**Lost in the Middle** 现象），而 Compact 任务绝对不能让 Claude 调用工具——否则压缩过程本身会产生新的对话内容，造成无限递归。

## 11.2.5 压缩结果的格式

```tsx
// formatCompactSummary()
`<summary>
${summaryContent}
</summary>

[The conversation above has been summarized to save context.
Previous messages have been replaced with this summary.]`
```

压缩结果不是裸文本替换到对话里，而是：

1. 包裹在 `<summary>` 标签中——让后续的 Claude 调用能识别"这是压缩过的历史，不是用户说的话"
2. 附加一条说明消息——显式告知上下文已被压缩，防止 Claude 以为自己"记不清了"而产生幻觉

## 11.2.6 Micro-Compact：轻量级单结果压缩

不是所有压缩都需要处理整个对话。当单个工具输出过大时，Micro-Compact 只压缩那一条：

```
一个 BashTool 输出了 50K 字符
  │
  ├── 超过 maxResultSizeChars (30K)？
  │     → 存到磁盘 ~/.claude/tool-results/{hash}.txt
  │     → 消息中替换为摘要 + 文件路径
  │
  └── 生成摘要示例：
      "执行了 npm test，输出 50K 字符，已存至 ~/.claude/tool-results/abc123.txt"
```

Micro-Compact 是 Layer 3 的实现。它比完整的 Auto-Compact 轻量得多——不需要 Fork 子 Agent，不需要处理整个对话，只是针对单条超大结果做"摘要+落盘"。

---

> **下一节**：[11.3 Session Memory — 跨压缩连续性](./113-session-memory.md)
