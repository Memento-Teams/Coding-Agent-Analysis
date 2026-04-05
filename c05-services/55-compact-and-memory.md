# 5.5-5.6 Compact 服务与 Session Memory

> Compact 服务解决 LLM 最根本的限制之一——上下文窗口有限。Session Memory 则为对话创建自动"笔记"，让 Claude Code 在恢复会话时快速回忆上下文。

## 5.5 Compact 服务 — 对话记忆管理

### 5.5.1 compact.ts — 核心压缩算法

```tsx
export async function compactConversation(
  messages: Message[],
  systemPrompt: SystemPrompt,
  options: CompactOptions
): Promise<CompactResult>
```

**压缩过程**：

```
原始对话（200K tokens）
  │
  ├── 1. 消息分组（grouping.ts）
  │     按 API 轮次分组：每个 assistant 回复 + 对应的 tool_result = 一组
  │     确保 tool_use → tool_result 配对不被拆散
  │
  ├── 2. Fork 子 Agent 做摘要
  │     共享父会话的 Prompt Cache（省钱）
  │     用 Compact Prompt 模板（9 节结构）
  │
  ├── 3. Token 估算验证
  │     压缩后必须低于阈值，否则再压缩一次
  │
  ├── 4. 清理
  │     清除图片缓存
  │     清除文件元数据
  │     重置 Recompaction 信息
  │
  └── 5. 返回压缩后的消息列表
```

### 5.5.2 autoCompact.ts — 自动触发

```tsx
// 触发条件
const effectiveWindow = getContextWindowForModel(model) - WARNING_THRESHOLD_BUFFER_TOKENS  // -20K
const threshold = effectiveWindow - AUTOCOMPACT_BUFFER_TOKENS  // 再 -13K

// 例：200K 模型
// effectiveWindow = 200,000 - 20,000 = 180,000
// threshold = 180,000 - 13,000 = 167,000
// 当消息超过 167K tokens → 触发 auto-compact
```

- **熔断器**：连续 3 次压缩失败后停止自动压缩（防止无限循环）
- **环境变量覆盖**：`CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=80` → 在上下文窗口 80% 时触发

### 5.5.3 microCompact.ts — 微压缩

比完整压缩更轻量——只压缩单个工具的输出，不涉及整个对话：

```
一个 BashTool 输出了 50K 字符
  │
  ├── 超过 maxResultSizeChars (30K)？
  │     → 存到磁盘，消息中只保留摘要
  │
  └── 生成一行摘要：
      "执行了 npm test，输出 50K 字符，已存至 ~/.claude/tool-results/abc123.txt"
```

### 5.5.4 grouping.ts — 消息分组

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

> **关键约束**：tool_use 和对应的 tool_result 必须在同一组。拆开会导致 API 报错。

---

## 5.6 Session Memory 服务 — 对话笔记

### 自动维护的对话笔记

```
每隔一段时间（基于 Token 消耗和工具调用数）：
  │
  ├── Fork 一个子 Agent
  ├── 让它阅读最近的对话
  ├── 更新一个 Markdown 文件，包含 9 个章节：
  │     ├── Session Title
  │     ├── Current State         ← 最重要
  │     ├── Task Specification
  │     ├── Files and Functions
  │     ├── Workflow
  │     ├── Errors & Corrections
  │     ├── Documentation
  │     ├── Learnings
  │     └── Worklog
  └── 这个文件用于 Session 恢复时快速回忆上下文
```

> **为什么用 Fork 而不是直接在主线程做？** Fork 子 Agent 共享父会话的 Prompt Cache，意味着摘要操作几乎不增加 API 成本（只付摘要输出的 Token 费）。如果在主线程做，会打断用户的交互流程。

> **下一节**：[5.7-5.12 Analytics 与基础设施](./57-analytics-and-infra.md)
