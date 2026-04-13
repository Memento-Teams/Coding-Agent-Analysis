# 11.A MemPalace 分析：向量检索 + 知识图谱的记忆方案

> Claude Code 的 MemDir 用"Markdown 文件 + Sonnet 排序"解决跨 Session 记忆，MemPalace 则走了一条完全不同的路——**"存一切原文，用向量检索找"**。两者折射出 AI 记忆领域的核心辩论：该让 AI 决定记什么（摘要派），还是什么都存然后靠搜索（原文派）？

## 11.A.1 设计哲学："Store everything, then make it findable"

MemPalace 的核心信条与 Claude Code 的 MemDir 截然相反：

| | Claude Code MemDir | MemPalace |
|---|---|---|
| 核心理念 | AI 提炼关键信息，丢弃原文 | 存一切原文，搜索来找 |
| 存储内容 | 高度压缩的总结（~150 字/条） | 逐字逐句的对话原文（800 字符/chunk） |
| 信息损失点 | extractMemories 提取时有损 | 入库时无损，检索时可能遗漏 |
| 检索方式 | LLM 语义排序（Sonnet） | 向量嵌入 + 余弦相似度（ChromaDB） |
| 外部依赖 | 无（纯文件系统） | ChromaDB + SQLite |

## 11.A.2 Palace 隐喻：空间化记忆组织

MemPalace 借用了古希腊"记忆宫殿"的概念，用空间隐喻组织记忆：

```
Palace (整座宫殿)
├── Wing: wing_kai          ← 一个人或一个项目
│   ├── Room: auth-migration    ← 一个具体话题
│   │   ├── Closet: (摘要指针)  ← 指向原文的缩写（计划用 AAAK 编码）
│   │   └── Drawer: (原文)      ← 最小存储单元，逐字保存
│   ├── Room: graphql-switch
│   └── ...
├── Wing: wing_driftwood
│   ├── Room: auth-migration    ← 同名 Room 跨 Wing → 自动产生 Tunnel
│   └── ...
└── Halls (走廊): 贯穿所有 Wing 的分类维度
    ├── hall_facts        — 已锁定的决策
    ├── hall_events       — 会话、里程碑
    ├── hall_discoveries  — 突破性发现
    ├── hall_preferences  — 习惯、偏好
    └── hall_advice       — 建议、解决方案
```

**Tunnel（隧道）** 是最有趣的设计：当同一个 Room（如 `auth-migration`）出现在多个 Wing（如 `wing_kai` 和 `wing_driftwood`）中时，系统自动创建跨 Wing 的连接。这使得 "哪些人/项目共享了同一个话题？" 成为可查询的结构。

与 MemDir 的对比：MemDir 只有扁平的文件列表 + 4 种 type 标签。MemPalace 有三维空间结构（Wing × Room × Hall）+ 自动图连接。

## 11.A.3 存储：ChromaDB 向量数据库

### 后端架构

```
~/.mempalace/
├── palace/                    ← ChromaDB 持久化目录（SQLite + HNSW 索引）
│   └── mempalace_drawers      ← 默认 collection
├── knowledge_graph.sqlite3    ← 知识图谱（实体-关系三元组）
├── identity.txt               ← Layer 0 身份文件
└── wal/
    └── write_log.jsonl        ← 写前日志（审计追踪）
```

**关键参数**（`backends/chroma.py`）：
- 向量索引：HNSW（Hierarchical Navigable Small World）
- 距离度量：cosine（`hnsw:space=cosine`）
- Embedding 模型：ChromaDB 默认（本地嵌入，零 API 调用）

### 数据入库方式

两种 Miner 负责不同类型的数据：

**Project Miner**（`miner.py`）— 代码和文档：
```python
CHUNK_SIZE = 800       # 每个 Drawer 800 字符
CHUNK_OVERLAP = 100    # 相邻 Drawer 重叠 100 字符
MIN_CHUNK_SIZE = 50    # 跳过过小的片段
MAX_FILE_SIZE = 10MB   # 跳过过大的文件
```

**Conversation Miner**（`convo_miner.py`）— 对话记录：
- 按 exchange pair 分块（一个用户发言 + 一个 AI 回复 = 一个 Drawer）
- 支持 5 种格式：Claude Code、ChatGPT、Slack、纯文本等
- 通过 `normalize.py` 统一为标准 transcript 格式

**核心原则：原文逐字存储，不做任何摘要或提取。**

### 与 MemDir 的对比

| 维度 | Claude Code MemDir | MemPalace |
|------|-------------------|-----------|
| 存储格式 | Markdown 文件 + frontmatter | ChromaDB 向量 + 元数据 |
| 单条大小 | ~500 字（人工总结） | 800 字符（机械分块） |
| 总容量 | ~200 条记忆 | 理论无上限 |
| 入库方式 | LLM 提取（extractMemories） | 机械分块（Miner） |
| 信息损失 | 提取时有损 | 入库时无损 |

## 11.A.4 检索：四层记忆栈

MemPalace 的检索设计核心是 **按需加载**（`layers.py`）：

```
┌──────────────────────────────────────────────────────┐
│ Layer 0 — Identity (~100 tokens)                     │
│   来源：~/.mempalace/identity.txt                     │
│   内容："我是 Atlas，Alice 的 AI 助手"                  │
│   加载时机：始终                                       │
├──────────────────────────────────────────────────────┤
│ Layer 1 — Essential Story (~500-800 tokens)           │
│   来源：按 importance 元数据 + 修改时间排序             │
│         取 top 15 个 Drawer，截断到 3200 chars          │
│   内容：高权重记忆的原文摘录                            │
│   加载时机：始终                                       │
├──────────────────────────────────────────────────────┤
│ Layer 2 — On-Demand (~200-500 tokens)                │
│   来源：ChromaDB 按 wing/room 元数据过滤               │
│   内容：特定话题的 Drawer（最多 10 个）                  │
│   加载时机：当话题被提及                                │
├──────────────────────────────────────────────────────┤
│ Layer 3 — Deep Search (不限)                         │
│   来源：ChromaDB 全库语义搜索（余弦相似度）              │
│   内容：与查询最相关的 Drawer（默认 top 5）              │
│   加载时机：显式搜索（"为什么我们当时…？"）              │
└──────────────────────────────────────────────────────┘
```

**启动成本**：L0 + L1 ≈ 600-900 tokens，留 95%+ 窗口给对话。

### 搜索实现（`searcher.py`）

```python
# searcher.py — 核心搜索函数
def search_memories(query, wing=None, room=None, n=5, max_distance=1.5):
    where = build_where_filter(wing, room)     # 元数据过滤
    results = collection.query(
        query_texts=[query],                   # ChromaDB 自动嵌入查询
        n_results=n,
        where=where,
        include=["documents", "metadatas", "distances"]
    )
    # 返回：文档内容 + 元数据 + 余弦距离
```

**Wing/Room 元数据过滤使检索召回率提升 34%**：从无过滤的搜索到 wing+room 过滤后，LongMemEval R@10 从 ~70% 升到 94.8%。

### 与 MemDir 检索的对比

```
Claude Code MemDir:                    MemPalace:
────────────────                       ────────
扫描 .md frontmatter                   L0+L1 始终注入（600 tokens）
    ↓                                      ↓
Sonnet sideQuery                       话题触发 → L2 元数据过滤
  "选 ≤5 个最相关文件"                      ↓
    ↓                                  显式搜索 → L3 语义搜索
读取全文 → system-reminder                 ↓
                                       返回原文 Drawer
```

| 维度 | MemDir | MemPalace |
|------|--------|-----------|
| 检索模型 | LLM (Sonnet) | 向量嵌入 (ChromaDB 本地) |
| 每轮检索成本 | ~1K tokens（Sonnet 调用） | 0（向量计算，无 LLM） |
| 语义理解力 | 极强（LLM 理解上下文） | 中（嵌入模型的能力上限） |
| 返回数量 | ≤5 条 | 可配置（默认 5） |
| 分层加载 | 无（每轮独立选） | 有（L0-L3 按需递进） |
| 基准分数 | 未公开 | 96.6% LongMemEval R@5 |

## 11.A.5 知识图谱：时态实体关系

MemPalace 除了向量搜索外，还有一个**独立的知识图谱**（`knowledge_graph.py`），用 SQLite 存储实体关系三元组：

```sql
-- 实体
CREATE TABLE entities (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  type TEXT,           -- person, project, tool, ...
  properties TEXT,     -- JSON
  created_at TEXT
);

-- 三元组（带时态）
CREATE TABLE triples (
  subject TEXT NOT NULL,
  predicate TEXT NOT NULL,    -- works_on, assigned_to, uses, ...
  object TEXT NOT NULL,
  valid_from TEXT,            -- 事实生效时间
  valid_to TEXT,              -- 事实失效时间（NULL = 仍有效）
  confidence REAL,
  source_closet TEXT          -- 溯源到 Drawer
);
```

**时态查询能力**——这是 MemDir 完全没有的：

```python
# "Kai 现在在做什么？"
kg.query_entity("Kai")
# → [Kai → works_on → Orion (2025-06-01 ~ 至今)]

# "一月份 Maya 在做什么？"  
kg.query_entity("Maya", as_of="2026-01-20")
# → [Maya → assigned_to → auth-migration (2026-01-15 ~ 2026-03-01)]

# "Orion 项目的时间线"
kg.timeline("Orion")
# → 按时间排列的所有相关三元组
```

**与 MemDir 的对比**：MemDir 没有结构化的实体关系。`project` 类型记忆只是自然语言描述（"认证重构由合规驱动"），无法做 "谁在什么时间段参与了什么" 这类结构化查询。

## 11.A.6 MCP 集成：19 个工具

MemPalace 通过 MCP Server（`mcp_server.py`，1400+ 行）向 AI Agent 暴露全部能力：

```bash
# 安装
claude mcp add mempalace -- python -m mempalace.mcp_server
```

| 类别 | 工具数 | 代表性工具 |
|------|--------|-----------|
| Palace 读取 | 7 | `mempalace_search`, `mempalace_get_taxonomy`, `mempalace_list_wings` |
| Palace 写入 | 4 | `mempalace_add_drawer`, `mempalace_update_drawer`, `mempalace_delete_drawer` |
| 知识图谱 | 5 | `mempalace_kg_query`, `mempalace_kg_add`, `mempalace_kg_invalidate`, `mempalace_kg_timeline` |
| 图导航 | 3 | `mempalace_traverse`, `mempalace_find_tunnels`, `mempalace_graph_stats` |
| Agent 日记 | 2 | `mempalace_diary_write`, `mempalace_diary_read` |

**Memory Protocol**（MCP 内置指令）：
1. 启动时调 `mempalace_status` 加载概览
2. 回答关于人/项目的问题前，先 `mempalace_kg_query` 或 `mempalace_search` 验证
3. 事实变化时：`kg_invalidate` 旧的 + `kg_add` 新的
4. 会话结束时：`mempalace_diary_write` 记录本次对话

**Write-Ahead Log**（写前日志）：每次写操作先记录到 `~/.mempalace/wal/write_log.jsonl`，用于审计和防记忆投毒。

## 11.A.7 AAAK 方言（实验性）

AAAK 是 MemPalace 自创的一种有损压缩方言（`dialect.py`，650+ 行），用于在 token 受限时压缩大量记忆：

```
原文：
  Alice told Jordan she was worried about Riley's college applications.
  Jordan said Max's swimming competitions are next month.

AAAK：
  FAM: ALC→♡JOR | 2D(kids): RIL(18,sports) MAX(11,chess+swimming) | BEN(contributor)
```

**编码规则**：
- 实体 → 3 字母大写代码（Alice→ALC, Jordan→JOR）
- 情感 → 3 字母小写代码（vulnerability→vul, joy→joy）
- 结构 → 管道分隔，连字符 slug
- 标志 → ORIGIN, CORE, SENSITIVE, PIVOT 等

**当前状态**：实验阶段。LongMemEval 基准测试中 AAAK 模式 **84.2%** 远低于原文模式 **96.6%**——压缩导致了 12.4 个百分点的召回损失。README 中作者已坦承这一问题。

## 11.A.8 三方对比：Claude Code vs Hermes vs MemPalace

### 架构定位

```
Claude Code MemDir              Hermes Agent                 MemPalace
"日本料亭"                      "模块化乐高"                  "数字记忆宫殿"
精炼总结，LLM 检索              全量注入 + FTS5               原文存储，向量检索
────────────────               ────────────────              ────────────────
.md 文件 + frontmatter          MEMORY.md + USER.md           ChromaDB + SQLite KG
Sonnet 排序 top 5              全量注入（3.5KB）              4 层按需加载
extractMemories 事后提取        Agent 实时写入                Miner 批量入库
autoDream 后台整合              无后台维护                    无后台维护
```

### 核心维度对比

| 维度 | Claude Code MemDir | Hermes Agent | MemPalace |
|------|-------------------|--------------|-----------|
| **存储引擎** | 文件系统（.md） | 文件系统 + SQLite FTS5 | ChromaDB + SQLite KG |
| **存储内容** | AI 提炼的总结 | Agent 手写笔记 | 逐字原文 |
| **存储量级** | ~200 条 × ~500 字 | ~3.5KB 总量 | 理论无上限 |
| **检索方式** | LLM 语义排序 | 全量注入 + FTS5 关键词 | 向量嵌入 + 元数据过滤 |
| **检索成本** | ~1K tokens/轮（Sonnet） | 0（全量注入） | 0（本地向量计算） |
| **语义理解** | 极强（LLM） | 弱（关键词匹配） | 中（嵌入模型） |
| **结构化查询** | 无 | 无 | 知识图谱时态查询 |
| **写入时机** | 会话结束后提取 | 会话中实时写入 | 离线批量入库 |
| **空间组织** | 扁平（type 标签） | 扁平（2 个文件） | 三维（Wing × Room × Hall） |
| **图遍历** | 无 | 无 | Palace Graph（Tunnel 连接） |
| **Agent 协议** | System Prompt 内置指令 | Tool Schema 引导 | MCP 19 个工具 + Memory Protocol |
| **外部依赖** | 无 | 可选（Honcho） | ChromaDB（本地） |
| **基准测试** | 未公开 | 未公开 | 96.6% LongMemEval R@5 |

### 信息流对比

```
Claude Code:
  对话 → [LLM 提取] → .md 精华 → [Sonnet 选 5] → 注入
  损失点：提取时丢弃 95%+ 原文

Hermes:
  对话 → [Agent 判断] → MEMORY.md 便签 → [全量注入] → 可用
  损失点：Agent 遗漏 + 容量极小

MemPalace:
  对话 → [Miner 分块] → ChromaDB 全量 → [向量搜索 top N] → 注入
  损失点：向量搜索可能遗漏语义相关但词汇不同的内容
```

### 各自的优势场景

**Claude Code MemDir 最适合**：
- 长期使用的个人工具（偏好和纠正需要跨月积累）
- 记忆数量适中（< 200 条）
- 不需要精确引用原文

**Hermes Agent 最适合**：
- 轻量级部署（零外部依赖）
- 记忆量极小的场景（便签够用）
- 需要搜索会话历史时（FTS5）

**MemPalace 最适合**：
- 需要精确召回原文（"当时具体怎么说的？"）
- 大量记忆积累（数千条对话）
- 需要结构化查询（"谁在什么时候参与了什么项目？"）
- 多人/多项目交叉引用（Wing × Room 的 Tunnel 机制）

### 根本取舍

三者体现了 AI 记忆设计的三种哲学：

1. **MemDir（提炼派）**：让 AI 决定什么值得记，存精华，搜索时再用 AI 判断相关性。**优点**：精炼、低噪音。**风险**：提取遗漏无法弥补。

2. **Hermes（便签派）**：让 Agent 自己管理小型记忆。**优点**：极简、零成本。**风险**：完全依赖 Agent 自觉，容量有限。

3. **MemPalace（原文派）**：什么都存，靠搜索找。**优点**：无信息损失，高召回率。**风险**：存储膨胀、搜索噪音、需要外部依赖。

> **一句话总结**：MemDir 是"读完书后写读书笔记"，Hermes 是"在书页空白处写批注"，MemPalace 是"把整本书扫描进搜索引擎"。三者不互斥——理想的记忆系统可能需要同时具备精炼总结、便签标注和原文检索的能力。

---

> 返回 [Chapter 11 目录](./README.md)
