# 11.9 跨系统对比：Claude Code vs Hermes Agent

> 两个系统解决的是同一个根本问题——"有限的窗口，无限的记忆"。但它们的架构选择折射出截然不同的设计哲学：Claude Code 是一家精密的日本料亭（固定套餐、精确控制每一道），Hermes 是一个模块化的乐高系统（可拆可换、适配不同场景）。

## 11.9.1 全景架构对比

```
Claude Code                              Hermes Agent
─────────────────────                    ─────────────────────
┌─────────────────────┐                  ┌─────────────────────┐
│ 五层压缩管线         │                  │ 三层工具结果防线     │
│ L1: 大结果落盘       │                  │ L1: 工具内部截断     │
│ L2: Snip 去重        │                  │ L2: 单结果持久化     │
│ L3: Micro-Compact    │                  │ L3: 聚合预算执行     │
│ L4: Context Collapse │                  │                     │
│ L5: Auto-Compact     │                  │ ContextCompressor    │
│     (Fork 子 Agent)  │                  │  (辅助模型摘要)      │
└─────────┬───────────┘                  └─────────┬───────────┘
          │                                        │
┌─────────▼───────────┐                  ┌─────────▼───────────┐
│ Session Memory       │                  │ _previous_summary    │
│ (10 节结构化笔记)     │                  │ (迭代式摘要更新)      │
│  独立于压缩的旁注     │                  │  压缩层内部连续性     │
└─────────┬───────────┘                  └─────────────────────┘
          │
┌─────────▼───────────┐                  ┌─────────────────────┐
│ MemDir               │                  │ 内置记忆              │
│ 每条记忆一个 .md 文件  │                  │ MEMORY.md + USER.md  │
│ 4 种封闭分类          │                  │ 全量注入 (~3.5KB)     │
│ Sonnet 三阶段检索     │                  │                      │
│ extractMemories 写入  │                  │ Session Search (FTS5)│
│ autoDream 后台整合    │                  │ SQLite 全文搜索       │
└─────────────────────┘                  │ 辅助模型摘要          │
                                         │                      │
                                         │ Honcho (可选)         │
                                         │ AI-native 用户建模    │
                                         │ 语义搜索+辩证推理     │
                                         └─────────────────────┘
```

## 11.9.2 上下文压缩对比

### 设计哲学

| 维度 | Claude Code | Hermes Agent |
|------|------------|--------------|
| 核心理念 | "能不压就不压" — 五层管线从轻到重 | "独立防线" — 工具结果和对话压缩解耦 |
| 压缩层数 | 5 层（同一管线内） | 3 层工具防线 + 1 层对话压缩（两套独立系统） |
| 压缩阈值 | ~83.5%（尽量推迟） | ~50%（提前压缩，保留更多尾部） |
| 压缩执行者 | Fork 子 Agent（共享 Prompt Cache） | 辅助模型（便宜/快速，独立调用） |
| 去重 | 有（Layer 2 Snip） | 无显式去重 |
| 部分压缩 | 有（Context Collapse，变体 B） | 有（头尾保护 + 中间摘要） |

### 摘要结构

```
Claude Code (9 节)                    Hermes Agent (8 节)
─────────────────                    ─────────────────
1. Primary Request                   ## Goal
2. Technical Concepts                ## Constraints & Preferences
3. Files and Code                    ## Progress (Done/In Progress/Blocked)
4. Errors                            ## Key Decisions
5. Problem Solving                   ## Relevant Files
6. User Messages ← 逐条引用原话      ## Next Steps
7. Pending Tasks                     ## Critical Context
8. Current Work                      ## Tools & Patterns ← 独有
9. Next Step
```

**关键差异**：
- Claude Code 有 `User Messages` 节——逐条引用用户原话，防止改写漂移。Hermes 没有这个强制要求
- Hermes 有 `Tools & Patterns` 节——记录工具使用经验，Claude Code 没有
- Hermes 的 `Progress` 节分为 Done/In Progress/Blocked 三个子节，比 Claude Code 的 `Pending Tasks` + `Current Work` 更结构化

### 跨压缩连续性

这是两者差异最大的地方：

```
Claude Code:
  压缩 #1 → 全量重写 9 节摘要
  Session Memory 独立记录 10 节笔记 ← 跨压缩的保险杠
  压缩 #2 → 又全量重写，但 Session Memory 仍保留之前的积累

Hermes:
  压缩 #1 → 生成摘要，保存到 _previous_summary
  压缩 #2 → "PREVIOUS SUMMARY: {old} + NEW TURNS: {new}"
           → 迭代更新，不是全量重写 ← 在压缩层内部解决连续性
```

Claude Code 用**独立的侧通道**（Session Memory）保证连续性，Hermes 在**压缩算法本身**中保证。两种方案各有优劣：

| 维度 | Claude Code (Session Memory) | Hermes (_previous_summary) |
|------|------------------------------|---------------------------|
| 信息独立性 | 高——Session Memory 独立于压缩过程 | 低——摘要更新依赖前一次摘要质量 |
| 累积能力 | 强——Session Memory 跨多次压缩持续更新 | 中——每次摘要更新可能丢失前次细节 |
| 额外成本 | 高——需要 Fork 子 Agent 更新 10 节笔记 | 零——迭代更新不产生额外 API 调用 |
| 实现复杂度 | 高——独立的服务、Prompt、更新逻辑 | 低——只是一个字符串变量 |

### Tool Pair 完整性

```
Claude Code:
  预防式——消息分组（grouping.ts）保证 tool_use + result 不被拆散
  从不产生孤立的 tool pair

Hermes:
  修复式——先压缩，再 _sanitize_tool_pairs() 事后修复
  删除孤立 result，为缺失 result 插入 stub
  容错性更强但逻辑更复杂
```

### 工具结果落盘

| 维度 | Claude Code | Hermes |
|------|------------|--------|
| 落盘阈值 | 30K 字符 | 100K 字符 |
| 写入方式 | 本地文件直写 | `env.execute()` 沙箱写入 |
| 多后端支持 | 仅本地 | Docker/SSH/Modal/Daytona |
| 预览 | 一行摘要 | 1500 chars 完整预览 |
| 聚合预算 | 无 | 200K chars 单轮预算 |
| 死循环防护 | 无显式处理 | `read_file` 豁免（threshold=inf） |

## 11.9.3 持久化记忆对比

### 存储设计

```
Claude Code:                          Hermes Agent:
~/.claude/memory/                     ~/.hermes/memories/
├── MEMORY.md (索引,≤200行)            ├── MEMORY.md (条目,≤2200 chars)
├── user_role.md                      └── USER.md (条目,≤1375 chars)
├── feedback_testing.md
├── project_auth.md                   ~/.hermes/state.db (SQLite + FTS5)
└── ref_linear.md                       sessions + messages + messages_fts

                                      Honcho 云端 (可选)
                                        User Representation + Peer Card
```

### 检索策略

这是两者最本质的差异之一：

```
Claude Code — "LLM 做检索"
───────────────────────
  扫描所有 .md frontmatter → manifest
  Sonnet sideQuery → 选 ≤5 个文件
  读取选中文件全文 → 注入 system-reminder

Hermes — "分层全量 + 按需搜索"
───────────────────────
  内置记忆: 全量注入 system prompt (3.5KB 很小)
  Session Search: agent 调 session_search tool
    → FTS5 关键词搜索 → 辅助模型摘要 → 返回
  Honcho: 自动 prefetch 或 agent 调工具
    → 语义搜索 / 辩证推理 → 返回
```

Claude Code 用 Sonnet 做相关性排序（**语义理解强，但每轮都有 LLM 成本**）。Hermes 的内置记忆直接全量注入（**零检索成本，但容量极小**），FTS5 搜索是关键词匹配（**快但无语义理解**），Honcho 提供语义搜索（**最强但需要外部服务**）。

### 写入模型

```
Claude Code — "事后批量提取"
───────────────────────
  会话进行中: agent 不写记忆（极少例外）
  会话结束后: extractMemories 子 Agent
    Phase 1: 读取所有现有记忆
    Phase 2: 批量写入新/更新的记忆
  后台: autoDream 整合碎片、修剪过时

Hermes — "实时主动写入"
───────────────────────
  会话进行中: agent 随时调 memory tool (add/replace/remove)
  每轮: 自动 sync 到 Honcho（如果启用）
  无 autoDream: agent 手动管理，无后台整合
```

| 维度 | Claude Code (事后提取) | Hermes (实时写入) |
|------|----------------------|------------------|
| 遗漏风险 | 中——extractMemories 可能漏提 | 高——完全依赖 agent 自觉 |
| 中途退出保护 | 低——会话未正常结束则不提取 | 高——每次 tool 调用立即持久化 |
| 重复写入 | 低——先读后写的两阶段避免重复 | 中——agent 可能重复写入 |
| 写入质量 | 较高——专门的提取 Prompt | 参差——取决于 agent 当时的判断 |
| 后台维护 | autoDream 定期整合 | 无——记忆只增不减（除非 agent 手动删） |

### 类型系统

```
Claude Code — 4 种封闭分类
  user      — 用户画像
  feedback  — 纠正与确认
  project   — 项目状态
  reference — 外部系统指针

  每种有结构化要求（feedback 要有 Why + How to apply）
  frontmatter 元数据（name, description, type）
  "什么不该存"有明确规定

Hermes — 2 种目标
  memory    — agent 的个人笔记
  user      — 关于用户的信息

  无子分类，无结构化要求
  纯文本条目，§ 分隔
  通过 tool schema description 引导 agent 的写入行为
```

Claude Code 的类型系统更严格——每条记忆有 frontmatter 元数据和结构化内容。Hermes 更简单——两个纯文本文件，Agent 自由写入。

### 安全考虑

| 维度 | Claude Code | Hermes |
|------|------------|--------|
| 记忆注入扫描 | 无 | prompt injection + exfiltration 模式检测 |
| 不可见字符检测 | 无 | 零宽空格、RTL 覆盖等 |
| 上下文文件扫描 | 有（AGENTS.md 等外部文件） | 有（同样的模式） |
| 记忆内容隔离 | 作为 system-reminder 注入 | `<memory-context>` fence + 系统注释 |

Hermes 对记忆内容的安全处理更严格——因为记忆直接注入 system prompt，如果不扫描，恶意内容可以实现 persistent prompt injection。

## 11.9.4 跨 Session 信息传递的完整路径

### Claude Code

```
Session 1                           Session 2
───────                              ───────
对话进行 (200K tokens)                加载 MEMORY.md 索引
    │                                    │
    ▼                                Sonnet 选 ≤5 个相关记忆
五层压缩管线                             │
Session Memory 更新                  注入 system-reminder 附件
    │                                    │
    ▼ 会话结束                       (tengu_coral_fern 开启时)
extractMemories                      可 grep 搜索 JSONL 原文
    │                                    │
    ▼                                    ▼
MemDir (.md 文件)                    Claude 看到：索引 + 相关记忆
    │
autoDream (后台整合)
```

### Hermes Agent

```
Session 1                           Session 2
───────                              ───────
对话进行                              加载 MEMORY.md + USER.md
    │                                → 冻结快照注入 system prompt
    │  agent 随时调 memory tool
    │  → 实时写入 MEMORY.md/USER.md     agent 调 session_search tool
    │                                → FTS5 搜索 → 辅助模型摘要
    │  每轮自动 sync → Honcho
    │                                Honcho prefetch
    │  每条消息 → state.db           → User Representation 注入
    │                                → 或 agent 调 honcho_* 工具
    ▼
ContextCompressor (对话内压缩)
  _previous_summary 迭代更新
```

## 11.9.5 设计决策总结

| 决策点 | Claude Code | Hermes Agent | 各自的理由 |
|-------|------------|--------------|-----------|
| 压缩是否可插拔 | 否，五层管线硬编码 | 是，ContextEngine 抽象基类 | Claude Code 追求确定性；Hermes 适配多模型多后端 |
| 是否有 Session Memory | 是，独立的 10 节笔记 | 否，靠 _previous_summary 迭代 | Claude Code 用独立侧通道保险；Hermes 追求极简 |
| 记忆写入时机 | 事后提取（extractMemories） | 实时主动（memory tool） | Claude Code 不干扰用户交互；Hermes 防止中途退出丢失 |
| 记忆检索方式 | LLM 语义筛选（Sonnet） | 全量注入 + FTS5 + Honcho | Claude Code 记忆多，需要筛选；Hermes 内置记忆小，不需要 |
| 后台整合 | autoDream（4 阶段） | 无 | Claude Code 记忆会碎片化；Hermes 记忆太少，不需要 |
| 会话历史搜索 | 灰度中（grep JSONL） | 正式功能（FTS5 + 摘要） | Claude Code 担心性能和信噪比；Hermes 用辅助模型摘要解决 |
| 多后端支持 | 仅本地文件系统 | Docker/SSH/Modal/Daytona | Claude Code 定位桌面 CLI；Hermes 定位多环境部署 |
| 外部记忆服务 | 无 | Honcho（可选 Provider） | Claude Code 零依赖哲学；Hermes 允许可选增强 |
| 安全扫描 | 外部文件有，记忆无 | 记忆写入全部扫描 | Claude Code 信任自己的 extractMemories；Hermes 的 agent 可调 memory tool，需要防御 |

## 11.9.6 成本对比

```
                        Claude Code          Hermes Agent
─────────────────────────────────────────────────────────
每轮固定成本
  System Prompt         ~5K tokens           ~5K tokens
  MEMORY.md 注入        ~2K tokens           ~1K tokens (全量,但很小)
  记忆检索              ~1K (Sonnet)         0 (内置全量注入)

压缩触发时
  Auto-Compact          ~5K output           ~3K output (辅助模型)
                        (共享 Cache)          (独立调用)
  Session Memory 更新   ~2K output           0 (无 Session Memory)

会话结束时
  extractMemories       ~3K output           0 (实时写入,无事后提取)

后台
  autoDream             ~5K output           0 (无后台整合)
  Honcho sync           —                    API 调用成本(每轮)

跨 Session 搜索
  JSONL grep            0 (不搜索)           FTS5: 0
                                             辅助模型摘要: ~2K output
```

Claude Code 的总成本更高（Session Memory + extractMemories + autoDream），但信息保留质量也更高。Hermes 在基本配置下更省，但加上 Honcho 后成本取决于 dialectic 调用频率。

## 11.9.7 一句话总结

**Claude Code**：精密的四层记忆体系——压缩管线控窗口、Session Memory 保连续、MemDir 存精华、autoDream 做整合。封闭、确定、零依赖。

**Hermes Agent**：模块化的三系统组合——工具防线 + 压缩器管窗口、内置记忆存便签、FTS5 搜历史、Honcho 做用户建模。开放、可插拔、多后端。

两者的共同信念：**上下文窗口是战术资源，记忆是战略资产。前者用压缩换时间，后者用持久化换空间。**

---

> 返回 [Chapter 11 目录](./README.md) | 返回 [c00 总览](../c00-project-overview/README.md)
