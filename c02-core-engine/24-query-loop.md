# 2.4 query.ts — 引擎的心脏

> 这是整个 Claude Code 最核心的文件。一个 `while(true)` 循环驱动了所有的智能行为。本节逐一拆解循环的四个阶段。

## 2.4.1 循环的骨架

把 1700 行代码浓缩一下，核心结构是这样的：

```tsx
async function* queryLoop(params) {
  while (true) {

    // ═══ 阶段 1：压缩管线 ═══
    // 对话太长了？先压缩一下再发给 API
    await compactPipeline(messages)

    // ═══ 阶段 2：调 API ═══
    // 把对话历史 + 工具列表发给 Claude
    for await (const message of callModel({ messages, tools, systemPrompt })) {
      yield message                    // 流式吐出 AI 的回复
      detectToolUseBlocks(message)     // 检测 AI 是否要调用工具
    }

    // ═══ 阶段 3：执行工具 ═══
    // AI 说要读文件？执行 FileReadTool
    // AI 说要编辑代码？执行 FileEditTool
    for await (const result of executeTools(toolUseBlocks)) {
      yield result                     // 吐出工具执行结果
      messages.push(result)            // 追加到对话历史
    }

    // ═══ 阶段 4：决定是否继续 ═══
    if (没有 tool_use) break           // AI 只回了文字，循环结束
    if (出错且无法恢复) break           // 遇到不可恢复的错误
    if (Token 预算耗尽) break          // 对话太长了
    // 否则 → 回到阶段 1，带着工具结果再调一次 API
  }
}
```

> **注意这里的 `async function*`** —— 这和 2.3 讲的生成器是同一个模式。`query.ts` 就是那个"生成方"，每次 `yield` 一条消息，QueryEngine（调用方）就拿到一条、渲染一条。

这个循环看起来简单，但每个阶段背后都藏着不少设计决策。下面逐一展开。

---

## 2.4.2 阶段 1：压缩管线 — 对话太长怎么办？

Claude 的上下文窗口有限（比如 200K tokens）。一次复杂任务可能产生几十轮工具调用，每轮都带着文件内容和命令输出，很快就撑满了。

query.ts 在每次调 API 之前，会经过一条**五层压缩管线**，从轻量到重量逐级尝试：

```
大结果落盘 → Snip 去重 → Microcompact 摘要 → Context Collapse 归档 → Auto-Compact 全文压缩
免费、即时、无损 ───────────────────────────────────────→ 昂贵、耗时、有损
```

大部分情况下前几层就够了，只有真的塞满才会动用最后一层（让 Claude 再调一次 API 对整个对话做摘要）。

> 这个主题涉及的代码量很大（`services/compact/` 6 个文件、`memdir/` 8 个文件、`services/SessionMemory/` 6 个文件），详细实现将在**上下文与记忆**章节中展开。

---

## 2.4.3 阶段 2：调 API — 流式响应的处理

### Claude API 返回什么？

API 不是一次性返回完整回复，而是**一个字一个字地流出来**（Streaming）：

```
API 流式返回：
  event: message_start    → 新消息开始
  event: content_block    → "我"
  event: content_block    → "来"
  event: content_block    → "看"
  event: content_block    → "看"
  event: content_block    → "这个"
  event: content_block    → "文件"
  event: content_block    → tool_use { name: "FileRead", input: { path: "src/app.ts" } }
  event: message_delta    → usage: { input_tokens: 1500, output_tokens: 200 }
  event: message_stop     → 消息结束
```

query.ts 的处理逻辑：

```tsx
for await (const message of callModel({ messages, tools, systemPrompt })) {
  // 1. 文字块 → 直接 yield 给用户看（实时显示打字效果）
  // 2. tool_use 块 → 记录下来，等消息结束后执行
  // 3. message_delta → 记录 token 用量
  // 4. message_stop → 累计总用量
}
```

### StreamingToolExecutor — 流水线执行

Claude Code 有一个精妙的优化：**不等 API 完全返回就开始执行工具**。

```
没有流水线：
  API 流式返回完毕 ──等待──→ 开始执行 Tool 1 ──→ Tool 2 ──→ 把结果发回 API
                  浪费时间 ↑

有流水线：
  API 还在返回... ──同时──→ Tool 1 已经开始执行了！
  API 返回完毕    ──同时──→ Tool 1 完成，Tool 2 开始
                           → 立刻把结果发回 API
```

当 API 流式返回中出现第一个 `tool_use` 块时，StreamingToolExecutor 立刻开始执行这个工具，不等后面还有没有更多工具。等 API 返回完毕时，第一个工具可能已经执行完了。

```tsx
// API 流式返回中检测到 tool_use
if (streamingToolExecutor) {
  streamingToolExecutor.addTool(toolUseBlock)  // 立刻提交执行，不等待
}

// API 返回完毕后，收集所有结果
const toolUpdates = streamingToolExecutor.getRemainingResults()
for await (const update of toolUpdates) {
  yield update.message       // 吐出工具结果
  toolResults.push(update)   // 追加到对话
}
```

---

## 2.4.4 阶段 3：工具执行 — 并行还是串行？

### 问题

Claude 可能在一次回复中调用多个工具：

```
AI 回复：
  "我需要读两个文件来理解这个 bug"
  [tool_use: FileRead("src/app.ts")]
  [tool_use: FileRead("src/utils.ts")]
  [tool_use: BashTool("npm test")]
```

这三个工具调用，应该并行还是串行执行？

### 答案：让工具自己决定

每个工具实现一个方法 `isConcurrencySafe(input)`：

```tsx
// FileReadTool: 读文件不会修改任何东西，可以并行
isConcurrencySafe() { return true }

// BashTool: 执行命令可能有副作用，不安全
isConcurrencySafe() { return false }

// FileEditTool: 修改文件，不安全
isConcurrencySafe() { return false }
```

### 编排器如何分批执行

`toolOrchestration.ts` 把工具调用分成批次：

```
输入：[FileRead, FileRead, BashTool, Grep, Grep, FileEdit]

分批：
  Batch 1: [FileRead, FileRead]  ← 都是并发安全的 → 并行执行
  Batch 2: [BashTool]            ← 不安全 → 单独串行
  Batch 3: [Grep, Grep]          ← 并发安全 → 并行执行
  Batch 4: [FileEdit]            ← 不安全 → 单独串行

执行顺序：Batch 1 全部完成 → Batch 2 → Batch 3 全部完成 → Batch 4
```

> **设计美学 — 控制反转 (IoC)**
>
> 编排器不需要知道每个工具的具体行为。它只问一个问题：`isConcurrencySafe()?`。工具自己最清楚自己是否安全。
>
> - 编排器里写 `if (tool === 'FileRead') { 并行 } else if (tool === 'Bash') { 串行 }`
> - 编排器问工具 `你能并行吗？`，工具自己回答
>
> 好处：新增工具时，编排器的代码不用改。

---

## 2.4.5 阶段 4：决定是否继续

每轮工具执行完后，引擎要决定：**继续循环，还是停下来？**

### 四种停止条件

```
工具执行完毕
     │
     ▼
┌── 检查 1：有没有 tool_use？────────────────────┐
│   API 回复只有文字，没有 tool_use               │
│   → 说明 Claude 认为任务完成了                  │
│   → 停止循环                                   │
└────────────────────────────────────────────────┘
     │ 有 tool_use
     ▼
┌── 检查 2：有没有不可恢复的错误？────────────────┐
│   → Prompt-Too-Long (413)                      │
│     → 先尝试压缩恢复                           │
│     → 压缩后仍然太长 → 停止循环                │
│   → 其他致命错误 → 停止循环                    │
└────────────────────────────────────────────────┘
     │ 没有错误
     ▼
┌── 检查 3：Stop Hooks ──────────────────────────┐
│   → Teammate 空闲通知？                        │
│   → 任务被外部取消？                           │
│   → 用户按了 Ctrl+C？                          │
│   → 任何一个触发 → 停止循环                    │
└────────────────────────────────────────────────┘
     │ 没有触发
     ▼
┌── 检查 4：Token 预算 ──────────────────────────┐
│   这一轮用了多少 Token？收益在递减吗？          │
│   → 连续 3+ 轮，每轮增量 < 500 tokens          │
│   → 说明 Claude 在"原地打转"，强制停止         │
└────────────────────────────────────────────────┘
     │ 没有超预算
     ▼
  继续循环 → 回到阶段 1
```

### "没有 tool call 但任务没完成"怎么办？

检查 1 的逻辑是：**没有 tool_use → 停止循环**。但现实中，"没有 tool call"并不等于"任务完成了"。常见的翻车场景：

- **模型"偷懒"** — 用文字描述它会怎么做，而不是真的调用工具去做
- **模型误判任务已完成** — 觉得做完了，实际没做完
- **模型主动提问** — 需要用户澄清，返回纯文本问题
- **模型 hallucinate** — 幻觉自己已经完成了操作

Claude Code 对此的策略是 **human-in-the-loop**，而非自动重试：

```
循环结束（没有 tool_use）
  │
  ▼
文本展示给用户
  │
  ▼
用户发现任务没完成
  │
  ├── "继续"
  ├── "你还没做完，请实际执行"
  └── "请用工具去做，不要光说"
  │
  ▼
新一轮 queryLoop 开始 → 带着用户的纠正消息再调 API
```

也就是说，Claude Code **不在引擎层做自动 nudge**，而是依赖两道防线：

1. **System Prompt 的约束** — 明确告诉模型"要用工具去做，不要只描述"
2. **用户的交互式纠正** — 循环结束后，用户可以立刻继续对话

这是一个有意识的设计选择：与其让引擎猜测模型是"说完了"还是"偷懒了"，不如把判断权交给用户。

---

### Token 预算：防止无限循环

有时候 Claude 会陷入"我再检查一下" → "再看看" → "确认一下"的循环，每轮做的事越来越少。Token Budget 检测这种模式：

```tsx
// tokenBudget.ts
if (turnTokens < budget * 0.9 && deltaSinceLastCheck > 500) {
  // 这一轮还有内容产出，继续
  continuationCount++
  continue  // 带一条"请继续"的消息重新循环
}

if (isDiminishing || continuationCount > 0) {
  // 产出在递减，或者已经"催"过一次了
  stop  // 停止，不再浪费 Token
}
```

没有这个机制，一个"纠结"的 Claude 可能在 20 轮循环里花掉 $2 却没有实质产出。Token Budget 像一个"监工"，发现产出递减就叫停。

---

## 2.4.6 错误恢复 — 优雅地处理故障

### Prompt-Too-Long (HTTP 413)

当对话总 Token 超过 API 限制时，API 返回 413 错误。query.ts 不会直接崩溃，而是尝试恢复：

```
收到 413 错误
  │
  ├── 尝试 1：Context Collapse（把旧对话归档）
  │   成功？→ 重试 API 调用
  │
  ├── 尝试 2：Reactive Compact（紧急压缩）
  │   成功？→ 重试 API 调用
  │
  └── 都失败了 → 告诉用户"上下文太长了，请用 /compact 手动压缩"
```

### Max Output Tokens（回复太长被截断）

有时候 Claude 的回复还没说完就被截断了（达到输出 Token 上限）：

```
收到 max_output_tokens 截断
  │
  ├── 尝试 1：提升上限（8K → 64K）
  │   成功？→ 重试
  │
  ├── 尝试 2：多轮恢复（最多 3 次）
  │   → 把截断的回复作为上下文
  │   → 告诉 Claude "你的回复被截断了，请继续"
  │   → Claude 接着说
  │
  └── 3 次都截断 → 放弃，返回已有内容
```

### 模型降级（Fallback）

如果主模型出错（比如负载过高），可以自动降级到备用模型：

```
主模型 (Opus) 报错
  │
  └── 切换到 fallbackModel (Sonnet)
      → 清理不兼容的消息格式（比如 Opus 特有的思考签名）
      → 用 Sonnet 重试
```

### "错误扣留"模式（Error Withholding）

query.ts 有一个很有意思的设计：当它收到 413 或 max_output_tokens 错误时，不立刻报错，而是先"扣留"（withhold），看看能不能自动恢复。只有恢复失败了，才把错误抛出来。

```tsx
// 不是立刻抛出：
// throw new Error("Prompt too long")  ❌

// 而是先扣留，尝试恢复：
withheldError = error        // 先记下来
// ... 尝试 compact ...
if (恢复成功) {
  withheldError = null       // 恢复了，假装没事
  continue                   // 继续循环
} else {
  throw withheldError        // 真的没办法了，再抛出
}
```

这让错误恢复对调用方透明——调用方根本不知道中间出过一次 413。

---

## 2.4.7 Tool 接口 — 工具的契约

每个工具都要实现 `Tool` 接口。这个接口定义了引擎和工具之间的"合同"：

```tsx
type Tool = {
  // ── 身份 ──
  name: string                    // "FileRead"、"Bash" 等
  aliases?: string[]              // 别名

  // ── 数据契约 ──
  inputSchema: ZodSchema          // 输入格式（用 Zod 校验）
  outputSchema?: ZodSchema        // 输出格式

  // ── 核心能力 ──
  call(args, context): Promise<ToolResult>          // 实际执行
  description(input): Promise<string>               // 描述（给 Claude 看的）

  // ── 安全属性 ──
  isConcurrencySafe(input): boolean    // 能并行吗？
  isReadOnly(input): boolean           // 只读吗？
  isDestructive?(input): boolean       // 有破坏性吗？
  checkPermissions(input): Promise<PermissionResult>  // 需要用户授权吗？

  // ── UI 渲染 ──
  renderToolUseMessage(input): React.ReactNode       // 在终端显示"正在读文件..."
  renderToolResultMessage(output): React.ReactNode   // 在终端显示结果
}
```

> **模块化 — 接口即解耦**
>
> 引擎只依赖 `Tool` 接口，不依赖任何具体工具。这就像一个插座标准：
>
> ```
> 引擎（插座）
>   ├── 知道怎么给电（调用 call()）
>   ├── 知道怎么检查安全（调用 checkPermissions()）
>   └── 不知道也不关心插的是什么电器
>
> 工具（电器）
>   ├── FileReadTool（台灯）
>   ├── BashTool（洗衣机）
>   ├── AgentTool（空调）
>   └── 任何符合接口的新工具
> ```
>
> 只要实现了这个接口，任何工具都能"插入"引擎。这就是为什么 Claude Code 能有 42 个工具而引擎代码不需要知道任何一个的细节。

---

## 2.4.8 Skill 在哪里？

> **读者问**：Claude Code 有 `/commit`、`/review`、`/simplify` 这些 Skill 对吧？我翻遍了 query.ts 的主循环，根本没看到任何关于 Skill 的处理逻辑。它们不是和 Tool 平级的吗？

你找不到是对的——因为 **Skill 不是和 Tool 平级的概念，它被包在一个叫 SkillTool 的 Tool 里面。**

```
引擎只认识 Tool
  │
  ├── FileReadTool      → 直接干活（读文件）
  ├── BashTool          → 直接干活（执行命令）
  ├── GrepTool          → 直接干活（搜索代码）
  │
  └── SkillTool         → 它本身是一个 Tool
        │
        │  内部再去查找和执行具体的 Skill：
        ├── /commit      → 一段 Prompt，教 Claude 如何做 git commit
        ├── /simplify    → 一段 Prompt，启动 3 个 Agent 做代码审查
        ├── /review      → 一段 Prompt，教 Claude 如何做 code review
        └── ...17 个内建 Skill
```

### Skill 的本质

Skill 的本质就是**一段预写好的 Prompt**。当 Claude 调用 `SkillTool({ skill: "simplify" })` 时，SkillTool 内部做的事情是：

1. 根据名字找到对应的 Skill 定义（`skills/bundled/simplify.ts`）
2. 取出里面的 Prompt 文本
3. 用这段 Prompt 启动一个**子 Agent**（通过 `runAgent()`）去执行

### 为什么不把每个 Skill 单独做成 Tool？

如果把每个 Skill 都做成单独的 Tool，42 个工具就会变成 59 个，而且这些 Skill 其实共享同一套执行逻辑（加载 Prompt → 启动 Agent）。用一个 SkillTool 包裹它们，引擎只看到一个 Tool，干净得多。

所以 Skill 不是一种新的执行机制，而是对 **Prompt + Agent** 的封装。引擎完全不知道 Skill 的存在——它只看到 SkillTool 这一个 Tool。

### Skill 的发现机制：模型就是搜索引擎

> **读者问**：Skill 可以有很多（用户自定义 + 插件 + MCP），当 Skill 数量达到几百甚至上千时，Claude 是怎么找到对的那个的？有向量检索吗？有关键词匹配吗？

**答案是：没有。** 当前稳定版的 Claude Code 没有任何匹配或搜索机制。

#### 当前方案：全量塞进 Prompt

Skill 的"发现"完全依赖一个叫 `skill_listing` 的 attachment：

```
每轮对话开始
  │
  ▼
getSkillListingAttachments()
  │
  ├── 从 5 个来源加载所有 Skill
  │     bundled（内建 ~17 个）
  │     skills/ 目录（用户自定义）
  │     commands/ 目录（遗留格式）
  │     plugin（第三方插件）
  │     mcp（MCP 服务器提供的）
  │
  ├── formatCommandsWithinBudget()
  │     预算 = 上下文窗口的 1%（200K → ~8000 字符）
  │     每条描述最多 250 字符
  │     超预算 → 截断描述 → 极端情况只保留名字
  │
  └── 作为 system-reminder 注入对话
        "The following skills are available for use with the Skill tool:
         - commit: Create a git commit...
         - review-pr: Review a pull request...
         - pdf: Use this skill whenever..."
```

模型看到这个列表后，自己判断该调用哪个 Skill。`findCommand()` 只是个精确匹配——模型已经选好了名字，这一步只是验证名字存不存在：

```tsx
// commands.ts — 没有任何模糊匹配或语义搜索
function findCommand(commandName, commands) {
  return commands.find(_ =>
    _.name === commandName ||
    getCommandName(_) === commandName ||
    _.aliases?.includes(commandName)
  )
}
```

**真正的"搜索引擎"就是 Claude 自己。**

#### 这个方案的瓶颈

当 Skill 数量增长到几百、上千时，问题就出现了：

- **Token 浪费**：1% 的上下文窗口被 Skill 列表占据，skill 越多、留给实际任务的空间越少
- **描述被截断**：预算不够时，描述被砍到 20 字符甚至只剩名字，模型没有足够信息做判断
- **选错或漏选**：模型在一堆名字里"盲猜"，准确率随 Skill 数量增长而下降

#### 实验中的解法：语义搜索发现

Claude Code 在 feature flag `EXPERIMENTAL_SKILL_SEARCH` 后面做了一套新方案：

1. **缩小列表** — `skill_listing` 只注入 bundled + MCP（≤30 个），不再塞全部
2. **语义发现** — 新增 `skill_discovery` 机制，对用户输入做语义搜索，推荐 top-N 相关 Skill
3. **远程加载** — 通过 `remoteSkillLoader` 支持从远端按需下载 Skill

```
稳定版                          实验版
─────────────                   ─────────────
skill_listing（全量）            skill_listing（仅 bundled + MCP）
                                + skill_discovery（语义搜索推荐）
↓                               ↓
模型从完整列表里选               模型从精简列表 + 动态推荐里选
↓                               ↓
findCommand() 精确匹配           findCommand() 精确匹配
```

> **关于 Skill 的大规模检索问题**，我们的工作 [Memento-Skills: Let Agents Design Agents](https://arxiv.org/abs/2603.18743) 提出了一套系统性的解法。Memento-Skills 将 Skill 存储为结构化 Markdown 文件，通过**可训练的 Skill Router** 在 read 阶段根据当前上下文做条件检索，在 write 阶段基于经验持续更新和扩展 Skill 库——形成闭环的持续学习，无需更新 LLM 参数。这直接解决了 Claude Code 当前"全量塞 prompt + 模型盲选"的瓶颈。感兴趣的读者可以参考。

> 关于 Skill 的具体定义方式、用户自定义 Skill、MCP Skill 等内容，详见**第 8 章：扩展机制**。

---

> 返回 [Chapter 2 目录](./README.md) | 返回 [c00 总览](../c00-project-overview/README.md)
