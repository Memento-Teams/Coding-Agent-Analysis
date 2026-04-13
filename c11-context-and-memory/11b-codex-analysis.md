# 11.B Codex (OpenAI) 分析：两阶段记忆管线 + 渐进式检索

> Codex 的记忆系统是四个系统中最工业化的——它用 **Rust + SQLite** 构建了一个完整的记忆管线（Phase 1 提取 + Phase 2 整合），产出物包含 `MEMORY.md` 手册、`memory_summary.md` 摘要、`rollout_summaries/` 每次会话回顾、和 `skills/` 可复用流程。这不是一个"让 AI 记住偏好"的轻量方案，而是一个"让 AI 随时间自我改进"的系统性工程。

## 11.B.1 架构概览

```
Session (会话)
  │
  ├── Context Manager ──── 上下文窗口管理
  │   ├── history.rs       item 存储、token 估算
  │   ├── updates.rs       增量状态 diff
  │   ├── normalize.rs     call/output 配对修复
  │   └── compact.rs       Auto-Compact 压缩
  │
  ├── Rollout Recorder ──── 会话持久化
  │   └── sessions/YYYY-MM-DD_HH-MM-SS.jsonl
  │
  └── Memories Pipeline ──── 跨 Session 记忆（后台异步）
      ├── Phase 1: 逐 rollout 提取（并行）
      │   └── → stage1_outputs (DB)
      └── Phase 2: 全局整合（串行）
          └── → MEMORY.md + memory_summary.md
              + rollout_summaries/ + skills/
```

## 11.B.2 上下文窗口管理

Codex 的上下文管理由 Rust 实现的 `ContextManager`（`context_manager/history.rs`）驱动，核心是四个子系统的协作：**Token 预算控制**、**工具输出截断**、**增量状态 Diff**、**Auto-Compact 压缩**。

### Token 预算策略

Codex 用三层 token 阈值控制窗口（`openai_models.rs:273-308`、`codex.rs:888`）：

```
context_window (模型总容量，如 128K)
  │
  ├── × effective_context_window_percent (默认 95%)
  │   = 有效窗口 (121,600 tokens) ← 实际可用空间
  │
  └── × 90% = auto_compact_token_limit (115,200 tokens)
              ← 超过此值触发 Auto-Compact
```

**Token 估算**用字节启发式（`truncate.rs`）：

```rust
const APPROX_BYTES_PER_TOKEN: usize = 4;  // 4 字节 ≈ 1 token

fn approx_token_count(text: &str) -> usize {
    text.len().saturating_add(3) / 4  // 向上取整
}
```

对特殊类型有专门处理（`history.rs:488-517`）：
- **推理 item**：Base64 解码后减 650 字节开销
- **图片**：
  - `detail: "original"` → 根据图片尺寸和 32px patch 大小动态计算
  - 其他 detail 级别 → 固定 7,373 字节估算
  - 结果缓存在 LRU cache 中（按 SHA1 去重）

Token 使用量通过四维度追踪（`history.rs:52-57`）：

```rust
struct TotalTokenUsageBreakdown {
    last_api_response_total_tokens: i64,           // API 上次报告的总量
    all_history_items_model_visible_bytes: i64,     // 全历史字节估算
    estimated_tokens_of_items_added_since_last_successful_api_response: i64,  // 新增 items 的 token 估算
    estimated_bytes_of_items_added_since_last_successful_api_response: i64,   // 新增 items 的字节数
}
```

这使得 Codex 能在不调用 API 的情况下估算当前上下文占用。

### 工具输出截断（`output-truncation` crate）

Codex 对工具输出使用**中间截断**策略——保留开头和结尾，截掉中间（`truncate.rs:38-69`）：

```
原始输出（假设 10000 行）：
┌─────────────────────────────────┐
│ 前 N 行（开头保留）              │ ← 通常包含关键上下文
├─────────────────────────────────┤
│ [... truncated 8000 chars ...]  │ ← 中间截断标记
├─────────────────────────────────┤
│ 后 M 行（结尾保留）              │ ← 通常包含结果/错误
└─────────────────────────────────┘
```

截断预算分配：

```rust
fn split_budget(max_bytes: usize) -> (left_budget, right_budget)
// 按比例分配给头部和尾部
```

两种截断策略（`TruncationPolicy`）：
- `Bytes(n)` — 按字节预算截断
- `Tokens(n)` — 按 token 预算截断（内部转换为字节）

截断后自动添加统计信息：`"Total output lines: {total_lines}"`。

**与 Claude Code 的对比**：

| 维度 | Claude Code | Codex |
|------|------------|-------|
| 大结果处理 | 落盘到文件（30K 字符阈值） | 中间截断（保留首尾） |
| 截断方式 | 替换为文件路径引用 | 就地截断，保留在上下文中 |
| 去重 | Layer 2 Snip（重复 tool 输出去重） | 无显式去重 |
| 截断标记 | 一行摘要 | `[truncated N chars/tokens]` + 总行数 |

### 增量状态 Diff（`updates.rs`）

这是 Codex 上下文管理中最独特的设计。`ContextManager` 维护一个 `reference_context_item` 基准快照（`history.rs:38-48`），每轮只发送**变化的部分**而非全量上下文：

```rust
// history.rs:34
struct ContextManager {
    items: Vec<ResponseItem>,              // 历史消息
    token_info: Option<TokenUsageInfo>,    // Token 使用追踪
    reference_context_item: Option<TurnContextItem>,  // ← 基准快照
}
```

每轮开始时，`build_settings_update_items()`（`updates.rs:188-223`）对比基准和当前状态，只对变化的维度发送更新消息：

```
reference_context_item (上一轮快照)
        ↓ diff 6 个维度
current_turn_context
        ↓
增量更新消息（只发变化的部分）：
  1. 环境上下文变更 → user message     ← equals_except_shell() 判断
  2. 模型指令变更 → developer message  ← 切换模型时
  3. 权限/沙箱策略变更 → developer msg ← sandbox_policy 或 approval_policy 变化
  4. 协作模式变更 → developer message  ← collaboration_mode 变化
  5. 实时状态转换 → developer message  ← realtime_active 变化
  6. 人格调整 → developer message      ← personality 变化
```

**清空基准的场景**（触发下轮完整重注入）：
- Pre-turn / 手动 compact 后（`compact.rs:213`）
- Rollback 剪切到混合初始上下文 bundle 时（`history.rs:431`）

**与 Claude Code 的对比**：

| 维度 | Claude Code | Codex |
|------|------------|-------|
| System Prompt | 每轮完整注入（~5K tokens） | 首轮注入，后续只发 diff |
| 上下文变更 | 无 diff 机制 | 6 维度精确 diff |
| 清空重置 | 无此概念 | compact/rollback 后清空基准 |
| 节省量 | 0 | 稳态下每轮节省数千 tokens |

这是一个显著的效率优势。Claude Code 在每次 API 调用时都完整注入 System Prompt 和 MEMORY.md 索引（~7K tokens），Codex 只在首轮或 compact 后完整注入，其余轮次只发增量 diff。

### Auto-Compact 压缩

Codex 在三个时机触发压缩（`codex.rs:5667-6165`）：

| 时机 | 触发条件 | InitialContextInjection | 效果 |
|------|---------|------------------------|------|
| **Pre-turn** | `total_tokens >= auto_compact_limit` | `DoNotInject` | 压缩后清空 reference，下轮完整重注入 |
| **Mid-turn** | 流式处理中 `ContextWindowExceeded` | `BeforeLastUserMessage` | 在最后用户消息前插入初始上下文 |
| **Post-turn** | 轮结束后检查累积 | `DoNotInject` | 同 Pre-turn |

**压缩提示极简**（`templates/compact/prompt.md`，仅 4 行）：

```
You are performing a CONTEXT CHECKPOINT COMPACTION.
Create a handoff summary for another LLM that will resume the task.
Include:
- Current progress and key decisions made
- Important context, constraints, or user preferences
- What remains to be done (clear next steps)
- Any critical data, examples, or references needed to continue
```

这与 Claude Code 的 9 节结构化压缩模板（~200 行 Prompt）形成鲜明对比。Codex 依赖模型自身判断该保留什么，Claude Code 用详细模板确保不遗漏。

**压缩后的历史构建**（`compact.rs:191-222`）：

```
压缩后替换历史 =
  1. 初始上下文 items（仅 Mid-turn 模式注入）
  2. 最近用户消息（倒序选取，上限 20,000 tokens）
  3. 摘要前缀 + 压缩摘要（作为最后一条消息）
  4. Ghost Snapshots（保留，模型不可见）

摘要前缀（summary_prefix.md）：
"Another language model started to solve this problem and produced
a summary of its thinking process. You also have access to the state
of the tools that were used by that language model. Use this to build
on the work that has already been done and avoid duplicating work."
```

**失败重试机制**（`compact.rs:154-169`）：

```
如果压缩本身超出窗口 (ContextWindowExceeded):
  → 移除最旧的 history item（从头部删）
  → 重置重试计数
  → 继续循环
  如果只剩 1 个 item 仍然超出：
  → 标记上下文为满，返回错误
```

**压缩后警告**（`compact.rs:227-230`）：

```
"Heads up: Long threads and multiple compactions can cause the model
to be less accurate. Start a new thread when possible to keep threads
small and targeted."
```

这暗示了 Codex 团队对多次压缩后质量退化的经验认知。

### 历史归一化（`normalize.rs`）

确保发送给模型的历史是合法的（`normalize.rs`，346 行）：
- 为孤立的 tool_call 添加合成 `"aborted"` 输出（`lines 14-120`）
- 移除没有对应 call 的孤立 output（`lines 122-195`）
- 模型不支持图片时剥离图片内容（`lines 293-345`）
- 删除 item 时自动清理对应的 call/output 配对（`remove_corresponding_for()`）

**与 Claude Code 的对比**：

| 维度 | Claude Code | Codex |
|------|------------|-------|
| Tool Pair 策略 | **预防式** — `grouping.ts` 保证分组不拆散 | **修复式** — 事后添加/删除配对 |
| 孤立 call 处理 | 不会产生（分组保护） | 添加合成 `"aborted"` output |
| 孤立 output 处理 | 不会产生 | 移除 |
| 图片降级 | N/A | 模型不支持时自动剥离 |

### Rollback 机制（`history.rs:228-251`）

Codex 支持回退最近 N 轮用户对话：

```rust
fn drop_last_n_user_turns(&mut self, num_turns: u32) {
    // 1. 找到所有用户消息的位置
    // 2. 从末尾回退 N 个
    // 3. 如果回退切到了混合 context bundle
    //    → 清空 reference_context_item（下轮完整重注入）
    // 4. 保留第一条用户消息之前的 items（session 前缀）
}
```

这与 Claude Code 没有的显式 rollback 机制形成对比。Claude Code 的压缩是单向的——一旦压缩，旧消息就没了。Codex 允许在压缩前回退，更灵活。

### Ghost Snapshots

Codex 有一种特殊的 `ResponseItem::GhostSnapshot` 类型：
- **模型不可见**：发送给模型前被过滤掉（`history.rs:120`）
- **压缩保留**：compact 时被完整保留（`compact.rs:207-212`）
- **用途**：rollout 重建——允许从 rollout 文件重放 session 时恢复内部状态，但不影响模型行为

这是一种"隐式元数据通道"——在对话历史中嵌入模型不可见的状态快照。

### 与 Claude Code 上下文管理的全面对比

```
Claude Code 上下文管理                    Codex 上下文管理
─────────────────────                    ─────────────────
五层管线（从轻到重）                      三时机 Compact + 截断策略
  L1: 大结果落盘 (30K)                     工具输出中间截断（保留首尾）
  L2: Snip 去重                            无显式去重
  L3: Micro-Compact                        无等价物
  L4: Context Collapse (部分压缩)           Mid-turn Compact
  L5: Auto-Compact (Fork 子 Agent)         Pre/Post-turn Compact

压缩提示                                 压缩提示
  9 节结构化模板 (~200 行)                  4 行自由格式
  逐条引用用户原话                          模型自行判断保留内容

上下文注入                               上下文注入
  每轮完整注入 System Prompt               首轮全量 + 后续 Diff
  无增量 diff                              6 维度精确 diff

跨压缩连续性                             跨压缩连续性
  Session Memory (独立 10 节笔记)           无等价物（依赖摘要质量）
  autoDream (后台整合)                     用户消息保留 (最近 20K tokens)

Tool Pair                               Tool Pair
  预防式（分组不拆散）                      修复式（事后合成/清理配对）

Rollback                                Rollback
  无                                      支持回退最近 N 轮

隐式元数据                               隐式元数据
  无                                      Ghost Snapshots（模型不可见）
```

**核心差异总结**：

1. **分层 vs 简洁**：Claude Code 用五层管线从轻到重处理上下文膨胀，Codex 只用截断 + compact 两种机制，但加了增量 Diff 来减少浪费
2. **结构化 vs 自由**：Claude Code 的压缩模板是高度结构化的（9 节），Codex 的压缩提示只有 4 行，把结构判断交给模型
3. **安全网 vs 精简**：Claude Code 有 Session Memory 作为压缩的保险杠，Codex 没有等价物但保留了最近 20K tokens 的用户消息
4. **全量 vs Diff**：Claude Code 每轮全量注入，Codex 用增量 Diff 显著降低稳态成本

## 11.B.3 会话持久化：Rollout

每次会话记录为 JSONL 文件（`sessions/YYYY-MM-DD_HH-MM-SS_xxx.jsonl`），每行是一个 `RolloutItem`：

- `EventMsg` — 协议事件
- `SessionMeta` — 元数据（ID、模型、工作目录、git 信息等）

状态数据库（SQLite，`state/migrations/`，22 个迁移文件）跟踪所有线程：

```sql
CREATE TABLE threads (
  id TEXT PRIMARY KEY,
  rollout_path TEXT,
  created_at TEXT, updated_at TEXT,
  source TEXT, model_provider TEXT, cwd TEXT,
  git_sha TEXT, git_branch TEXT, git_origin_url TEXT,
  tokens_used INTEGER, archived INTEGER
);
```

支持三种会话类型：
- **New** — 空历史
- **Resumed** — 从 rollout 文件重建历史
- **Forked** — 从父会话分叉，继承历史但新建 rollout

## 11.B.4 跨 Session 记忆：两阶段记忆管线

这是 Codex 最有特色的部分。记忆生成完全在后台异步完成，分两个阶段：

### Phase 1：逐 Rollout 提取（并行）

```
触发条件：
  - 非临时会话
  - memory feature 开启
  - 非子 Agent 会话
  - state DB 可用

选取规则：
  - 来自交互式会话源
  - 在配置的时间窗口内
  - 已空闲足够长（避免处理活跃会话）
  - 未被其他 worker 占有

处理流程：
  claim rollout → 过滤为记忆相关 items
  → 发送给模型（并行，有并发上限）
  → 结构化输出：raw_memory + rollout_summary + rollout_slug
  → 脱敏（redact secrets）
  → 存入 state DB (stage1_outputs)
```

**Phase 1 的 Prompt 是整个系统最精心设计的部分**（`stage_one_system.md`，570 行）。它要求模型：

1. **最低信号门控**：先问 "未来 Agent 会因为这些信息做得更好吗？" 如果否，返回全空
2. **任务结果分类**：success / partial / fail / uncertain，基于用户反馈和环境证据
3. **四类高信号记忆**：
   - 稳定的用户操作偏好
   - 高杠杆的程序性知识
   - 可靠的任务地图和决策触发器
   - 用户环境和工作流的持久证据
4. **证据优先级**：用户消息 > 工具输出 > Agent 消息
5. **输出结构**：
   - `rollout_summary`：详细的按任务结构化回顾
   - `raw_memory`：带 frontmatter 的记忆条目（description, task, task_group, task_outcome, cwd, keywords）

### Phase 2：全局整合（串行）

```
Phase 2 独占一个全局锁（同一时间只有一个整合在运行）

输入：
  - stage1_outputs（按 usage_count + 最近使用时间排序选取）
  - 现有的 MEMORY.md、memory_summary.md、skills/

处理流程：
  claim 全局 phase2 job
  → 加载 stage1_outputs（有选择上限）
  → 计算 diff（added / retained / removed）
  → 同步文件系统：
      raw_memories.md（合并所有 raw_memory，最新在前）
      rollout_summaries/（每个 rollout 一个文件）
  → 裁剪过期的 rollout summaries
  → 启动 consolidation 子 Agent
  → 子 Agent 读取 raw_memories.md + 现有记忆
  → 产出更新后的 MEMORY.md + memory_summary.md + skills/
  → 记录 watermark 防止重复处理
```

**整合子 Agent 的 Prompt**（`consolidation.md`，300+ 行）定义了严格的输出格式：

```
memories/
├── memory_summary.md         ← 始终注入 System Prompt（导航摘要）
├── MEMORY.md                 ← 可搜索的手册（grep 关键词检索）
│   └── # Task Group: <cwd/project/workflow>
│       ├── scope: ...
│       ├── applies_to: cwd=...; reuse_rule=...
│       ├── ## Task 1: <description, outcome>
│       │   ├── ### rollout_summary_files
│       │   └── ### keywords
│       ├── ## User preferences
│       ├── ## Reusable knowledge
│       └── ## Failures and how to do differently
├── raw_memories.md           ← Phase 1 原始输出（临时，供 Phase 2 消费）
├── rollout_summaries/        ← 每个 rollout 的详细回顾
│   └── <rollout-slug>.md
└── skills/                   ← 可复用流程
    └── <skill-name>/
        ├── SKILL.md
        ├── scripts/
        ├── templates/
        └── examples/
```

### 遗忘机制

Phase 2 有显式的遗忘能力：
- `max_unused_days` 窗口外的记忆被忽略
- 被移除的线程：从 MEMORY.md 中只删除该线程支持的内容，保留共享内容
- 对应的 `rollout_summaries/` 文件被裁剪
- `memory_summary.md` 同步清理过时引用

## 11.B.5 记忆检索：渐进式披露

Codex 的检索设计（`read_path.md`）采用**渐进式披露**策略：

```
┌────────────────────────────────────────────────────┐
│ 始终加载：memory_summary.md (注入 System Prompt)    │
│   → 导航摘要，帮助判断是否需要查记忆                 │
├────────────────────────────────────────────────────┤
│ Quick Memory Pass（≤4-6 步搜索预算）：              │
│   1. 从 memory_summary 提取关键词                   │
│   2. grep MEMORY.md 搜索关键词                      │
│   3. 如需要，打开 1-2 个 rollout_summaries/         │
│   4. 如需精确证据，搜索原始 rollout                  │
│   5. 无命中则停止                                   │
├────────────────────────────────────────────────────┤
│ 跳过记忆的硬规则：                                  │
│   当前时间、简单翻译、一行命令、简单格式化            │
└────────────────────────────────────────────────────┘
```

**记忆引用系统**——Codex 要求每次使用记忆时附加引用：

```xml
<oai-mem-citation>
<citation_entries>
MEMORY.md:234-236|note=[responsesapi citation extraction code pointer]
rollout_summaries/2026-02-17_weekly_report.md:10-12|note=[weekly report format]
</citation_entries>
<rollout_ids>
019c6e27-e55b-73d1-87d8-4e01f1f75043
</rollout_ids>
</oai-mem-citation>
```

这使得记忆的使用是**可审计的**——可以追踪哪些记忆被引用、哪些 rollout 提供了价值。

### 记忆新鲜度判断

Codex 对记忆新鲜度的处理比 Claude Code 更细致：

| 场景 | 策略 |
|------|------|
| 高漂移 + 低验证成本 | 先验证再回答 |
| 高漂移 + 高验证成本 | 从记忆回答，标注"可能过时"，提供刷新选项 |
| 低漂移 + 低验证成本 | 视情况验证 |
| 低漂移 + 高验证成本 | 直接从记忆回答 |

## 11.B.6 四方对比更新

| 维度 | Claude Code MemDir | Hermes | MemPalace | **Codex** |
|------|-------------------|--------|-----------|-----------|
| **语言** | TypeScript | Python | Python | **Rust** |
| **存储引擎** | 文件系统 (.md) | 文件系统 + SQLite FTS5 | ChromaDB + SQLite KG | **SQLite (state DB) + 文件系统** |
| **存储内容** | AI 提炼的总结 | Agent 手写便签 | 逐字原文 | **结构化记忆 + rollout 回顾** |
| **记忆生成** | extractMemories (会话后) | Agent 实时写入 | Miner 批量入库 | **两阶段管线（Phase 1 提取 + Phase 2 整合）** |
| **检索方式** | Sonnet LLM 排序 | 全量注入 + FTS5 | 向量嵌入搜索 | **memory_summary 注入 + grep MEMORY.md** |
| **检索成本** | ~1K tokens/轮 (Sonnet) | 0 | 0 (本地向量) | **0 (grep，无 LLM)** |
| **结构化查询** | 无 | 无 | 知识图谱时态查询 | **按 Task Group / cwd / keywords grep** |
| **遗忘机制** | autoDream 修剪 | 无 | 无 | **max_unused_days + Phase 2 diff 清理** |
| **可审计性** | 无 | 无 | Write-Ahead Log | **记忆引用 (oai-mem-citation) + rollout_id 追踪** |
| **技能复用** | 无 | 无 | 无 | **skills/ 目录（SKILL.md + scripts/ + templates/）** |
| **压缩** | 五层管线 + Fork 子 Agent | 辅助模型 | N/A | **三时机 Auto-Compact + 增量 Diff** |
| **上下文 Diff** | 无（全量注入） | 无 | N/A | **reference_context_item 基准 Diff** |

### 核心设计差异

**1. 记忆质量控制**

Codex 的 Phase 1 Prompt 是四个系统中对记忆质量控制最严格的（570 行）：
- **最低信号门控**：先判断是否值得记，不值得就返回空
- **任务结果分类**：每个任务标注 success/partial/fail/uncertain
- **证据层级**：用户消息 > 工具输出 > Agent 消息
- **偏好提取**：要求保留用户原话（"when <situation>, the user said: '...'"）

对比 Claude Code 的 extractMemories：也有类似原则但 Prompt 更短、分类更粗（4 种类型 vs Codex 的多维结构）。

**2. 渐进式 vs 选择式检索**

```
Claude Code:  Sonnet 从 200 个文件中选 5 个 → 注入全文
Codex:        memory_summary 始终在 → grep MEMORY.md → 按需打开 rollout_summaries
```

Claude Code 是 "LLM 帮你选"，Codex 是 "模型自己 grep 搜索"。Codex 的方式更透明（可审计引用）但依赖模型的搜索能力。

**3. 增量上下文 Diff**

Codex 通过 `reference_context_item` 实现了增量状态更新——只发送变化的部分。这在多轮对话中显著节省 tokens。Claude Code 没有这个优化，每轮完整注入 System Prompt。

**4. Skills 系统**

Codex 独有的 `skills/` 目录是一种"程序性记忆"——不只记住"知道什么"，还记住"怎么做"。包含入口指令 (SKILL.md)、脚本 (scripts/)、模板 (templates/)、示例 (examples/)。其他三个系统都没有这种结构。

**5. 遗忘的工程化**

Codex 是四个系统中唯一有系统性遗忘机制的：
- `max_unused_days` 自动过期
- Phase 2 diff 计算 added/retained/removed
- 移除时只删该线程支撑的内容，保留共享部分
- 对比 Claude Code 的 autoDream（也有修剪但更粗糙）和 MemPalace/Hermes（无遗忘）

## 11.B.7 一句话总结

**Codex 的记忆系统像一个自动化的知识管理流水线**：每次会话结束后，后台 worker 提取结构化记忆（Phase 1），定期整合成可搜索的手册（Phase 2），模型通过 grep 渐进式检索。它不追求"记住一切"（MemPalace）或"精选几条"（MemDir），而是追求**"随时间让 Agent 越来越好"**——通过偏好积累、失败教训、和可复用技能的系统性沉淀。

> 如果说 MemDir 是"读书笔记"，Hermes 是"便利贴"，MemPalace 是"全文搜索引擎"，那 Codex 就是**"企业知识库"**——有入库流程、有审核标准、有索引结构、有过期清理、有技能模板。

---

> 返回 [Chapter 11 目录](./README.md)
