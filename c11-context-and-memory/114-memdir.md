# 11.4 MemDir — 文件系统持久化记忆

> Compact 和 Session Memory 解决的是**单次会话内**的记忆问题。但用户今天说了"不要 mock 数据库"，明天开新会话时 Claude 还记得吗？MemDir 就是跨会话记忆的解决方案——把值得长期保留的信息存到文件系统中。

## 11.4.1 设计哲学：记忆是文件，不是数据库

MemDir 没有用 SQLite、Redis 或任何数据库。每条记忆就是一个 Markdown 文件，放在 `~/.claude/memory/` 目录下：

```
~/.claude/memory/                    ← 用户级记忆（全局）
├── MEMORY.md                        ← 索引文件（始终注入 System Prompt）
├── user_role.md                     ← "我是后端工程师，熟悉 Go 和 Postgres"
├── feedback_testing.md              ← "不要 mock 数据库"
├── feedback_git.md                  ← "PR 不要拆太细"
├── project_auth_rewrite.md          ← "认证重构由合规驱动"
└── ref_linear_tickets.md            ← "Bug 追踪在 Linear 项目 INGEST"

.claude/memory/                      ← 项目级记忆（随代码库）
└── project_specific_context.md
```

为什么用文件而不用数据库？

1. **透明性**：用户可以直接 `cat` 查看、`vim` 编辑自己的记忆。没有二进制格式、没有 API、没有查询语言
2. **版本控制**：项目级记忆可以跟代码一起提交到 git——团队成员共享同一份项目上下文
3. **LLM 原生**：Claude 本身就擅长读写 Markdown。存为 Markdown 意味着读写记忆不需要额外的序列化/反序列化
4. **调试友好**：出了问题？直接打开文件看——不需要 `sqlite3` 或数据库客户端

## 11.4.2 四种记忆类型

MemDir 使用**封闭分类法**——只有 4 种类型，不多不少：

### user — 用户画像

```markdown
---
name: user_role
description: 高级后端工程师，Go 和 Postgres 专家，React 新手
type: user
---

Senior backend engineer. Deep Go expertise (10 years).
First time touching React and this project's frontend.

Frame frontend explanations in terms of backend analogues.
```

**何时保存**：用户自述身份、表达偏好、展示知识水平
**如何使用**：个性化回复的风格和复杂度

### feedback — 纠正与确认

```markdown
---
name: feedback_testing
description: 测试必须用真实数据库，不用 mock
type: feedback
---

Don't mock the database in tests.

**Why:** We got burned last quarter when mocked tests passed
but the prod migration failed — mock/prod divergence masked
a broken migration.

**How to apply:** When writing or reviewing tests that touch data,
always spin up a real database (docker-compose) instead of mocking.
```

**何时保存**：用户纠正了做法，或者确认了一个非显然的做法
**如何使用**：避免重复犯错，保持已验证的做法

> **注意**：feedback 不只记录纠正，也记录确认。如果 Claude 做了一个非显然的选择（比如把重构放在一个大 PR 里），用户说"对，就这样"——这也值得记录。纠正容易注意到，确认容易遗漏。

### project — 项目状态

```markdown
---
name: project_auth_rewrite
description: 认证中间件重构，由法规合规驱动，非技术债清理
type: project
---

Auth middleware rewrite is driven by legal/compliance requirements
around session token storage, not tech-debt cleanup.

**Why:** Legal flagged the old middleware for storing session tokens
in a way that doesn't meet new compliance requirements.

**How to apply:** Scope decisions should favor compliance over ergonomics.
When in doubt about auth changes, prioritize meeting the compliance
spec over developer experience.
```

**何时保存**：学到了项目的目标、决策背景、里程碑、截止日期
**如何使用**：理解需求背后的动机，做出更符合上下文的建议

### reference — 外部系统指针

```markdown
---
name: ref_linear_tickets
description: Bug 追踪在 Linear 项目 INGEST
type: reference
---

Pipeline bugs are tracked in Linear project "INGEST".
Check there for context on pipeline-related tickets.
```

**何时保存**：学到了信息在外部系统（Linear、Slack、Confluence 等）的位置
**如何使用**：当用户提到外部系统时，知道去哪里找信息

### 为什么只有 4 种？

开放式分类会导致"什么都存"——Claude 面对模糊指令时倾向于过度保存。4 种类型的封闭分类法给它一个明确的边界：**如果信息不属于这 4 类，就不该存**。

同样重要的是"什么不该存"：

| 不该存 | 原因 |
|-------|------|
| 代码规范、架构模式 | 从代码中直接推导 |
| Git 历史、谁改了什么 | `git log` / `git blame` 是权威来源 |
| 调试方案、修复步骤 | 修复已在代码中，commit message 有上下文 |
| CLAUDE.md 中已有的 | 已经存在 |
| 临时任务细节 | 用 Tasks/Plans，不用记忆 |

设计哲学：**只存"代码里看不出来的信息"**。

## 11.4.3 MEMORY.md — 索引文件

MEMORY.md 是 MemDir 的"入口"——它在**每次 API 调用时都注入到 System Prompt 中**。这意味着 Claude 永远能看到这个索引。

```markdown
- [User Role](user_role.md) — 高级工程师，专注后端，熟悉 Go 和 Postgres
- [Testing Feedback](feedback_testing.md) — 用真实数据库，不 mock
- [Auth Rewrite](project_auth_rewrite.md) — 认证重构由合规要求驱动
- [Linear Issues](ref_linear_tickets.md) — Bug 追踪在 Linear 项目 INGEST
```

### 硬性约束

```tsx
const MAX_ENTRYPOINT_LINES = 200   // 不超过 200 行
const MAX_ENTRYPOINT_BYTES = 25_000  // 不超过 25KB
```

MEMORY.md 每次 API 调用都注入，如果不限制大小，记忆增长会导致 Token 成本失控。每条索引控制在 ~150 字符，200 行上限约等于 200 条记忆——对大多数用户足够了。

### MEMORY.md 只存索引，不存内容

MEMORY.md 只是指针，不是记忆本身。实际内容在各个 `.md` 文件中。这样做的好处：

1. **MEMORY.md 始终很小**——每次 API 调用只消耗 ~2K tokens
2. **按需读取**——Claude 看到索引后，只 Read 需要的记忆文件
3. **独立更新**——修改一条记忆不需要重写索引

## 11.4.4 记忆相关性筛选：三阶段检索

当记忆文件很多时，不能全部注入对话——会撑满 Token 预算。MemDir 用**三阶段流程**完成检索，核心是用 Sonnet 做相关性排序。

### 阶段 1：扫描（memoryScan.ts）

递归读取 memory 目录下所有 `.md` 文件（排除 MEMORY.md），**只读每个文件的前 30 行来解析 frontmatter**：

```tsx
// memoryScan.ts
const MAX_MEMORY_FILES = 200
const FRONTMATTER_MAX_LINES = 30

async function scanMemoryFiles(memoryDir, signal): Promise<MemoryHeader[]> {
  const entries = await readdir(memoryDir, { recursive: true })
  const mdFiles = entries.filter(f => f.endsWith('.md') && basename(f) !== 'MEMORY.md')

  // 对每个文件：只读前 30 行，解析 frontmatter
  // 提取：filename, filePath, mtimeMs, description, type
  // 按修改时间降序排列，取最新的 200 个
}
```

产出：一个 `MemoryHeader[]` 数组，每条包含文件名、路径、修改时间、description、type。**不读文件正文。**

然后将 headers 格式化为 **manifest 文本**，交给下一阶段：

```tsx
// memoryScan.ts
function formatMemoryManifest(memories: MemoryHeader[]): string {
  // 每条格式：- [type] filename (ISO时间戳): description
  // 例如：
  // - [feedback] feedback_testing.md (2026-04-10T...): 测试必须用真实数据库
  // - [user] user_role.md (2026-04-08T...): 高级后端工程师，Go 专家
}
```

> **注意**：Sonnet 看到的是从 frontmatter 扫描生成的 manifest，**不是 MEMORY.md 索引**。MEMORY.md 是给主模型在 System Prompt 中看的全局概览；manifest 是给 Sonnet 筛选用的实时清单。两者内容可能不同步——比如用户手动创建了记忆文件但忘了更新 MEMORY.md，manifest 仍然能扫描到。

### 阶段 2：Sonnet 选择（findRelevantMemories.ts）

用 **sideQuery**（独立 API 调用，不进入主对话流）调用 Sonnet，输入是用户消息 + manifest，输出是最多 5 个文件名：

```tsx
// findRelevantMemories.ts
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be useful
to Claude Code as it processes a user's query. You will be given the user's query and
a list of available memory files with their filenames and descriptions.

Return a list of filenames for the memories that will clearly be useful (up to 5).
Only include memories that you are certain will be helpful.
- If unsure, do not include.
- If none useful, return empty list.
- If recently-used tools provided, do not select their usage docs
  (but DO select warnings/gotchas about those tools).`

async function selectRelevantMemories(query, memories, signal, recentTools) {
  const manifest = formatMemoryManifest(memories)
  const toolsSection = recentTools.length > 0
    ? `\n\nRecently used tools: ${recentTools.join(', ')}` : ''

  const result = await sideQuery({
    model: getDefaultSonnetModel(),
    system: SELECT_MEMORIES_SYSTEM_PROMPT,
    messages: [{
      role: 'user',
      content: `Query: ${query}\n\nAvailable memories:\n${manifest}${toolsSection}`,
    }],
    max_tokens: 256,
    output_format: {
      type: 'json_schema',
      schema: {
        type: 'object',
        properties: {
          selected_memories: { type: 'array', items: { type: 'string' } },
        },
        required: ['selected_memories'],
      },
    },
    signal,
  })
  // 解析 JSON，过滤无效文件名，返回 string[]
}
```

几个设计要点：

- **强制 JSON Schema 输出**：`output_format` 保证返回结构化结果，不需要正则解析
- **max_tokens: 256**：只需要返回几个文件名，极小的输出预算
- **recentTools 过滤**：如果 Claude 正在用某个工具（如 `mcp__X__spawn`），不再注入该工具的使用文档（已经在用了），但保留其 warnings/gotchas
- **失败容错**：sideQuery 失败或被 abort 时返回空数组，不阻塞主流程

### 阶段 3：读取与注入（attachments.ts）

被选中的记忆文件被完整读取（超 50 行或 8KB 则截断），附加新鲜度元数据（几天前写的），作为 `<system-reminder>` 附件注入当前对话。超过 1 天的记忆会标记 staleness 警告。

### 异步预取：不阻塞用户

整个三阶段流程在 `query.ts` 的每轮对话中**异步触发**（`startRelevantMemoryPrefetch()`），不阻塞主模型的响应生成。如果用户取消或当前轮结束前预取未完成，会被 abort。

```
用户输入
  │
  ├──→ 主模型开始处理（不等记忆）
  │
  └──→ 后台：扫描 → Sonnet 选择 → 读取
         │
         └──→ 结果注入为 system-reminder 附件
```

### 完整流程图

```
memory/
├── MEMORY.md              ──→ 始终注入 System Prompt（全局概览）
├── user_role.md
├── feedback_testing.md         ┐
├── project_auth.md             │ 扫描 frontmatter
├── ref_linear.md               │    ↓
└── ...                         │ manifest 文本
                                │    ↓
                                │ Sonnet sideQuery（选 ≤5 个）
                                │    ↓
                                │ 读取选中文件全文
                                │    ↓
                                └─→ 注入为 system-reminder 附件
```

### 为什么用 LLM 做筛选而不是向量检索？

1. **记忆数量少**（通常 < 50 条）——LLM 直接判断相关性比向量检索更准确
2. **语义理解强**——"修 auth bug" 和 "认证重构由合规驱动" 之间的关联，向量检索不一定能捕捉
3. **Sonnet 又快又便宜**——一次筛选 < 0.5 秒，< $0.01
4. **零基础设施**——不需要向量数据库、不需要 embedding 模型、不需要索引构建

### 为什么最多 5 条？

更多记忆 = 更多 Token = 更多成本 + 更多噪音。5 条是经验值——足以提供关键上下文，又不会淹没当前对话。

## 11.4.5 两个 200 的限制：主题维度，不是对话维度

MemDir 有两个容易混淆的数量限制，都定义在源码中：

```tsx
// memdir.ts
const MAX_ENTRYPOINT_LINES = 200   // MEMORY.md 索引最多 200 行
const MAX_ENTRYPOINT_BYTES = 25_000 // MEMORY.md 最大 25KB

// memoryScan.ts
const MAX_MEMORY_FILES = 200        // 扫描时最多读 200 个记忆文件
```

**关键理解：200 是主题数上限，不是对话数上限。** 记忆按主题聚合（如 `user_role.md`、`feedback_testing.md`），不按对话拆分。同一主题的信息会被更新覆盖（"Do not write duplicate memories. First check if there is an existing memory you can update before writing a new one."），所以实际记忆文件数通常远小于 200。

扫描时按修改时间排序取最新的 200 个文件，再由 Sonnet 从中选出最相关的 5 个注入上下文。真正的瓶颈不在 200 的上限，而在**每轮只选 5 个**的筛选策略。

## 11.4.6 记忆的写入：两阶段批处理

记忆的写入由 `extractMemories` 子 Agent 完成，采用**两阶段策略**：

```
Phase 1: Read ALL existing memory files (single turn)
  ├── 读取所有现有记忆
  ├── 了解已经存了什么
  └── 避免重复存储

Phase 2: Write ALL new/updated memories (single turn)
  ├── 生成新的记忆文件
  ├── 更新已有的记忆文件
  └── 更新 MEMORY.md 索引
```

**为什么不交替读写？** 每次 LLM 调用都有成本。两阶段策略把所有读操作集中在一轮，所有写操作集中在下一轮，最小化 API 调用次数。同时，先读后写确保不会创建重复的记忆。

## 11.4.7 记忆的可靠性：教 AI 怀疑自己

MemDir 的 Prompt 中有一段非常精妙的"认知校准"：

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

这段 Prompt 教 Claude 区分 **"记忆中说存在"** 和 **"现在确实存在"**。

这不是禁止 Claude 使用记忆——而是要求它**验证后再使用**。记忆是有时间戳的：上次会话时 `src/auth.ts` 还在，但可能今天被重命名为 `src/authentication.ts` 了。Claude 不应该盲目推荐一个可能已经不存在的文件。

> 这是整个 Prompt 系统中最高级的认知校准技巧之一——对抗 LLM 把自身记忆当作事实的倾向。

## 11.4.8 会话历史召回：默认不开启的实验功能

一个常见疑问：MemDir 只存高层总结，Claude 能不能回溯之前的**对话原文**？

### 对话历史确实存在

每次会话的完整对话记录以 JSONL 格式存储在项目目录下：

```
~/.claude/projects/<project-slug>/
├── <session-uuid-1>.jsonl    ← 会话 1 的完整记录
├── <session-uuid-2>.jsonl    ← 会话 2 的完整记录
├── ...
└── memory/                   ← MemDir 记忆文件
    ├── MEMORY.md
    └── *.md
```

每个 JSONL 文件可能有数十万 tokens（实测单个文件 60 万+ tokens），包含完整的用户消息、Assistant 回复、工具调用和结果。

### 但默认不搜索

源码中有一个 **"Searching past context"** 功能（`memdir.ts:375`），由 GrowthBook 远程 feature flag `tengu_coral_fern` 控制：

```tsx
// memdir.ts:375-406
export function buildSearchingPastContextSection(autoMemDir: string): string[] {
  if (!getFeatureValue_CACHED_MAY_BE_STALE('tengu_coral_fern', false)) {
    return []  // 默认返回空——不注入搜索指令
  }
  // 开启后，向 System Prompt 注入两级搜索策略：
  // 1. 先搜 memory 目录的 .md 主题文件
  // 2. 最后手段：grep 搜索 .jsonl 会话记录
}
```

开启后，模型被允许按两级策略搜索：

```
搜索优先级：
1. 先搜 memory/*.md 主题文件（快、精确）
2. 最后手段：grep 搜索 *.jsonl 会话记录（慢、大文件）
```

注释明确写道 `"last resort — large files, slow"`——JSONL 搜索是兜底方案。

### 为什么默认不开启？

`tengu_coral_fern` 是 GrowthBook（A/B 测试平台）上的远程 feature flag，默认值 `false`，由 Anthropic 服务端控制。这意味着它还在灰度实验阶段，原因可能包括：

1. **性能问题**：JSONL 文件巨大（单文件 60 万+ tokens），grep 搜索很慢
2. **成本问题**：搜出的对话原文注入上下文会大量消耗 tokens
3. **信噪比低**：对话原文包含大量工具调用结果等噪音，有用信息密度远低于 MemDir 的精炼记忆
4. **质量验证中**：Anthropic 通过灰度发布评估该功能是否值得默认开启

### 实际效果：跨 Session 的信息传递几乎完全依赖 MemDir

```
┌─────────────────────────────────────────────────────────┐
│ Session 1                                               │
│   用户说："不要 mock 数据库"                              │
│   → extractMemories 提取 → feedback_testing.md          │
└───────────────────────┬─────────────────────────────────┘
                        │ 记忆文件（.md）
                        ▼
┌─────────────────────────────────────────────────────────┐
│ Session 2                                               │
│   1. System Prompt 注入 MEMORY.md 索引                  │
│   2. Sonnet 选出相关记忆 → 注入 feedback_testing.md      │
│   3. Claude 看到"不要 mock 数据库"                       │
│                                                         │
│   ✗ 不会自动读取 Session 1 的 JSONL 对话原文              │
│   △ 除非 tengu_coral_fern 开启且模型主动 grep            │
└─────────────────────────────────────────────────────────┘
```

这意味着 MemDir 的记忆质量至关重要——如果 `extractMemories` 在 Session 1 结束时漏掉了某个关键信息，Session 2 默认情况下无法回溯到原始对话来弥补。

---

> **下一节**：[11.5 autoDream — 后台记忆整合](./115-auto-dream.md)
