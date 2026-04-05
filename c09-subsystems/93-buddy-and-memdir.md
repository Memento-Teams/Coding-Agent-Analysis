# 9.3-9.4 Buddy 与 MemDir

> Buddy 是 Claude Code 里一只有统计属性的陪伴精灵（默认是鸭子）。MemDir 是一个完整的持久化记忆管理子系统，让 Claude 跨会话保持上下文。

## 9.3 Buddy — 有统计属性的陪伴精灵

### 确定性生成

```tsx
// buddy/companion.ts
function generateCompanion(userId: string): Companion {
  // 用 FNV-1a 哈希把 userId 转成一个种子数
  const seed = fnv1a(userId)

  // 用 mulberry32 确定性 PRNG（伪随机数生成器）
  const rng = mulberry32(seed)

  // 确定稀有度（概率：common > uncommon > rare > epic > legendary）
  const rarity = rollRarity(rng)

  // 根据稀有度生成属性
  return {
    species: 'duck',         // 当前只有鸭子 :)
    rarity,
    eye: pickEye(rng),       // 眼睛类型（圆/方/星形/...）
    hat: pickHat(rng),       // 帽子类型（无/礼帽/皇冠/...）
    shiny: rollShiny(rng),   // 1% 概率
    stats: {
      grace: generateStat(rng, rarity),    // 优雅
      vigor: generateStat(rng, rarity),    // 活力
      wisdom: generateStat(rng, rarity),   // 智慧
      charm: generateStat(rng, rarity),    // 魅力
      luck: generateStat(rng, rarity),     // 幸运
    },
    name: '',  // 用户命名
    soul: { inspirationSeed: rng() * 1000000 | 0 }  // 个性种子
  }
}
```

> **为什么用确定性 PRNG？** 每次启动都用 userId 作为 PRNG 种子，生成的 Companion **永远是同一只**。今天打开 Claude Code，明天打开，还是那只戴礼帽的稀有蓝色鸭子。这带来了归属感：这是"你的"鸭子，不是每次随机生成的临时访客。
>
> 用户的 name 和 soul 存在配置文件里（持久化），但 bones（外观和属性）每次从 userId 重新生成。即使删除了配置，鸭子的外观也不会变——身份由 userId 决定，不由存储决定。

### Buddy 的角色边界 Prompt

```
A small ${species} named ${name} sits beside the user's input box
and occasionally comments in a speech bubble.

You're not ${name} — it's a separate watcher.

When the user addresses ${name} directly (by name), the bubble will answer.
Your job in that moment is to stay out of the way:
  respond in ONE line or less.
  Don't explain that you're not ${name} — they know.
  Don't narrate what ${name} might say — the bubble handles that.
```

第一句话确立角色分离：Buddy 是"另一个观察者"，不是 Claude 的化身。最后两句防止 Claude 的两个常见失误：过度解释和代替 Buddy 说话。

---

## 9.4 MemDir — 结构化持久化记忆

### 目录结构

```
~/.claude/memory/                    ← 用户级记忆
├── MEMORY.md                        ← 索引文件（始终注入 System Prompt）
├── user_role.md                     ← 用户身份类
├── user_preferences.md              ← 用户偏好类
├── feedback_testing.md              ← 反馈类（测试相关）
├── feedback_git.md                  ← 反馈类（git 相关）
├── project_auth_rewrite.md          ← 项目信息类
└── ref_linear_tickets.md            ← 外部系统引用类

.claude/memory/                      ← 项目级记忆（随代码库存储）
└── project_specific_context.md
```

每个记忆文件都有标准 frontmatter：

```markdown
---
name: feedback_testing
description: 测试相关的反馈——用真实数据库，不用 mock
type: feedback
---

Don't mock the database in tests.

**Why:** We got burned last quarter when mocked tests passed but
the prod migration failed — mock/prod divergence masked a broken migration.

**How to apply:** When writing or reviewing tests that touch data,
always spin up a real database (docker-compose) instead of mocking.
```

### MEMORY.md 的硬性约束

```tsx
const MAX_ENTRYPOINT_LINES = 200   // 不超过 200 行
const MAX_ENTRYPOINT_BYTES = 25_000  // 不超过 25KB

// MEMORY.md 在每次 API 调用时都注入 System Prompt
// 如果不限制大小，记忆增长会导致 Token 成本失控
```

MEMORY.md 只存索引，每行约 150 字符：

```markdown
- [User Role](user_role.md) — 高级工程师，专注后端，熟悉 Go 和 Postgres
- [Testing Feedback](feedback_testing.md) — 用真实数据库，不 mock
- [Auth Rewrite](project_auth_rewrite.md) — 认证重构由合规要求驱动
- [Linear Issues](ref_linear_tickets.md) — Bug 追踪在 Linear 项目 INGEST
```

### 记忆相关性筛选

当记忆文件很多时，不能全部注入（会撑满 Token 预算）。用 Sonnet 模型快速筛选：

```tsx
async function findRelevantMemories(
  query: string,          // 用户的当前请求
  allMemories: Memory[],  // 所有记忆的摘要
  maxCount = 5            // 最多选 5 条
): Promise<Memory[]> {
  // 用 Sonnet（快速、便宜）做相关性判断
  const selected = await callModelForSelection(query, allMemories, maxCount)

  // 过滤：如果刚刚用过某个工具，不要注入那个工具的"使用参考"
  // 但保留关于那个工具的"警告和注意事项"
  return filterRecentlyUsedToolDocs(selected)
}
```

### 记忆生命周期

```
会话中学到了新信息（用户纠正了某个做法）
  │
  ├── 会话结束时（stop hook 触发）
  ├── extractMemories 子 Agent 扫描对话
  ├── 识别值得记住的内容（按 4 种类型分类）
  ├── 生成/更新记忆文件
  └── 更新 MEMORY.md 索引

Claude Code 空闲时（Proactive 模式的 CronTool 调度）
  │
  ├── autoDream 子 Agent 启动
  ├── 四个阶段：Orient → Gather → Consolidate → Prune
  └── 整合记忆，修剪索引（保持 < 200 行）
```

> **下一节**：[9.5-9.7 Remote、Voice 与共同设计原则](./95-remote-voice-principles.md)
