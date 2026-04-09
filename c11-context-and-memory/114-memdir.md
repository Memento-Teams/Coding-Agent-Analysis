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

## 11.4.4 记忆相关性筛选

当记忆文件很多时，不能全部注入对话——会撑满 Token 预算。MemDir 用 **Sonnet 模型**做快速相关性筛选：

```tsx
async function findRelevantMemories(
  query: string,          // 用户的当前请求
  allMemories: Memory[],  // 所有记忆的摘要（来自 MEMORY.md 索引）
  maxCount = 5            // 最多选 5 条
): Promise<Memory[]> {
  // 用 Sonnet（快速、便宜）做相关性判断
  const selected = await callModelForSelection(query, allMemories, maxCount)

  // 过滤：如果刚刚用过某个工具，不注入那个工具的"使用参考"
  // 但保留关于那个工具的"警告和注意事项"
  return filterRecentlyUsedToolDocs(selected)
}
```

### 为什么用 LLM 做筛选而不是向量检索？

1. **记忆数量少**（通常 < 50 条）——LLM 直接判断相关性比向量检索更准确
2. **语义理解强**——"修 auth bug" 和 "认证重构由合规驱动" 之间的关联，向量检索不一定能捕捉
3. **Sonnet 又快又便宜**——一次筛选 < 0.5 秒，< $0.01

### 为什么最多 5 条？

更多记忆 = 更多 Token = 更多成本 + 更多噪音。5 条是经验值——足以提供关键上下文，又不会淹没当前对话。

## 11.4.5 记忆的写入：两阶段批处理

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

## 11.4.6 记忆的可靠性：教 AI 怀疑自己

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

---

> **下一节**：[11.5 autoDream — 后台记忆整合](./115-auto-dream.md)
