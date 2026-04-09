# 11.1 上下文窗口的生命周期

> 上下文窗口就像一间固定面积的办公桌。工作越多，桌面越乱。Claude Code 不是等桌面满了才收拾，而是从一开始就按"从轻到重"五个层级逐步管理桌面空间。

## 11.1.1 Token 是怎么增长的？

一次典型的编程任务，Token 消耗的增长曲线大致是这样的：

```
Token
消耗     ┃
200K ━━━━╋━━━━━━━━━━━━━━━━━━━━━ ← API 硬限制
         ┃                    ╱
180K ────╂─ · · · · · · · · ╱── ← 自动压缩阈值 (effectiveWindow - 13K)
         ┃               ╱╱
167K ────╂─ · · · · · ·╱╱───── ← 警告阈值 (effectiveWindow)
         ┃           ╱╱
         ┃         ╱╱
         ┃       ╱╱  ← 工具结果是 Token 增长的主力
         ┃     ╱╱      一次 FileRead 可能带来 5-10K tokens
         ┃   ╱╱        一次 Bash 输出可能带来 20-50K tokens
         ┃  ╱
         ┃╱ ← 用户消息 + AI 回复（相对较少）
         ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━
              对话轮次 →
```

**Token 增长的主要来源**（按贡献排序）：

| 来源 | 单次贡献 | 特点 |
|------|---------|------|
| Bash 工具输出 | 5K-50K+ | 最大的 Token 消耗者，npm test 等命令输出巨大 |
| FileRead 结果 | 2K-10K | 大文件一次读取就消耗大量 Token |
| AI 回复 + 思考 | 1K-5K | 含 thinking blocks 时更大 |
| 系统 Prompt | ~5K | 固定成本，每轮都有 |
| 用户消息 | 0.1K-1K | 通常最小 |

一个关键洞察：**工具结果占对话 Token 的大头**。用户可能只说了一句"修复这个 bug"，但 Claude 调用了 10 次工具，每次带回几 K 的文件内容和命令输出，很快就积累到 100K+ tokens。

## 11.1.2 五层压缩管线

query.ts 在每次调用 API 之前，都会通过一条**五层压缩管线**。这个设计遵循一个原则：**能不压就不压，能少压就少压**。

```
输入：当前对话消息列表
  │
  │  Layer 1: 大结果落盘 (Result Offloading)
  │  ────────────────────────────
  │  成本：几乎为零（磁盘 I/O）
  │  损失：无（原始数据仍可访问）
  │  
  │  单个工具输出超过 30K 字符？
  │    → 存到磁盘 ~/.claude/tool-results/{hash}.txt
  │    → 消息中替换为一行摘要 + 文件路径
  │  
  ▼
  │  Layer 2: Snip 去重 (Deduplication)
  │  ────────────────────────────
  │  成本：O(n) 扫描
  │  损失：无（去掉的是重复信息）
  │  
  │  同一个文件被读了 3 次？
  │    → 只保留最后一次的完整内容
  │    → 前两次替换为 "[已在后续消息中更新]"
  │  同一个命令输出重复？
  │    → 去重
  │  
  ▼
  │  Layer 3: Micro-Compact (单结果摘要)
  │  ────────────────────────────
  │  成本：低（可能调用 LLM 做单条摘要）
  │  损失：轻微（丢失原始输出的细节）
  │  
  │  工具输出超过阈值但不值得落盘？
  │    → 用 LLM 生成一行摘要
  │    → "执行了 npm test，47 个测试通过，2 个失败（auth.test.ts:34, db.test.ts:78）"
  │  
  ▼
  │  Layer 4: Context Collapse (部分归档)
  │  ────────────────────────────
  │  成本：中等（LLM 对旧消息做摘要）
  │  损失：中等（旧对话被压缩为摘要）
  │  
  │  对话太长了但还没到临界点？
  │    → 把"旧消息"（距当前较远的）归档为摘要
  │    → 保留"近消息"（最近几轮）的完整内容
  │    → 用 Partial Compact Prompt（变体 B）
  │  
  ▼
  │  Layer 5: Auto-Compact (全文压缩)
  │  ────────────────────────────
  │  成本：高（Fork 子 Agent，LLM 处理整个对话）
  │  损失：显著（整个对话被压缩为结构化摘要）
  │  
  │  消息超过 167K tokens？
  │    → Fork 子 Agent（共享 Prompt Cache，省钱）
  │    → 整个对话压缩为 9 节结构化摘要
  │    → 触发 Session Memory 更新
  │
  ▼
输出：压缩后的消息列表（保证在 Token 预算内）
```

### 为什么是五层而不是一层？

如果只有 Auto-Compact（最后一层），每次压缩都要：
1. Fork 一个子 Agent（延迟 2-5 秒）
2. 让 LLM 处理全部对话（消耗大量 Token，即使共享 Cache）
3. 产生有损压缩（丢失细节）

五层管线的好处是：**大部分情况下前三层就够了**。落盘、去重、微压缩几乎零成本且无损。只有真正压不住了，才动用第四、第五层的"重型武器"。

这和操作系统的内存管理策略类似——先回收缓存页，再压缩内存页，最后才 swap 到磁盘。

### 阈值计算

```tsx
// autoCompact.ts
const effectiveWindow = getContextWindowForModel(model) - WARNING_THRESHOLD_BUFFER_TOKENS  // -20K
const threshold = effectiveWindow - AUTOCOMPACT_BUFFER_TOKENS  // 再 -13K

// 例：200K 模型
// effectiveWindow = 200,000 - 20,000 = 180,000
// threshold = 180,000 - 13,000 = 167,000
// 当消息超过 167K tokens → 触发 Layer 5 Auto-Compact
```

为什么留 33K 的缓冲？

- **20K（WARNING_THRESHOLD_BUFFER）**：给 AI 的下一次回复留空间。如果 180K 的上下文，AI 还要回复，很可能超过 200K 限制
- **13K（AUTOCOMPACT_BUFFER）**：压缩本身需要时间，在压缩完成之前，新消息还在进来。这 13K 是"压缩窗口期"的缓冲

### 环境变量覆盖

```bash
# 在上下文窗口 80% 时就触发自动压缩（默认约 83.5%）
CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=80
```

### 熔断器

```tsx
// 连续 3 次压缩失败后停止自动压缩
if (consecutiveFailures >= MAX_CONSECUTIVE_FAILURES) {
  disableAutoCompact()  // 防止无限循环
}
```

压缩失败的原因通常是：压缩后的结果仍然太长（比如对话中有大量图片或结构化数据，很难再压缩）。与其反复重试浪费 Token，不如停下来让用户手动介入。

## 11.1.3 Token 估算：三个时机

准确的 Token 计数是压缩决策的基础。Claude Code 在三个时机追踪 Token：

```
                    对话轮次
                       │
  ┌── Pre-request ─────┤
  │                    │
  │  tokenCountWithEstimation(messages)
  │  → 估算，用于决定是否需要压缩
  │  → 支持 thinking blocks、图片 Token 估算
  │  → 移除 tool-search 相关字段（不计入）
  │                    │
  │              ┌─────▼─────┐
  │              │  API 调用  │
  │              └─────┬─────┘
  │                    │
  ├── During streaming ┤
  │                    │
  │  从 API 流式事件中实时读取
  │  → message_delta 事件包含 usage 字段
  │  → 实时更新 Token 消耗统计
  │                    │
  └── Post-request ────┤
                       │
     从 API 响应的 usage 字段获取精确数字
     → input_tokens: 实际消耗的输入 Token
     → output_tokens: 实际消耗的输出 Token
     → 用于计费统计和下次压缩决策的精确基准
```

> **为什么不只用 Post-request 的精确数字？** 因为 Pre-request 估算决定了"要不要先压缩再发"——这个决定必须在调 API **之前**做。如果估算太保守（常常以为不需要压缩），会频繁遇到 413 错误；如果估算太激进（总是压缩），会浪费 Token 在不必要的压缩上。

## 11.1.4 错误恢复：413 的处理

即使有五层管线，偶尔仍会遇到 Prompt 超长（HTTP 413）。query.ts 不会直接崩溃：

```
收到 413 错误
  │
  ├── 尝试 1：Context Collapse（归档旧对话）
  │   成功？→ 重试 API 调用
  │
  ├── 尝试 2：Reactive Compact（紧急全文压缩）
  │   成功？→ 重试 API 调用
  │
  └── 都失败了 → 告诉用户"上下文太长，请用 /compact 手动压缩"
```

这个恢复过程对调用方透明——query.ts 先"扣留"(withhold) 错误，尝试恢复，只有恢复失败才抛出。调用方可能根本不知道中间出过一次 413。

---

> **下一节**：[11.2 Auto-Compact 深入](./112-auto-compact.md)
