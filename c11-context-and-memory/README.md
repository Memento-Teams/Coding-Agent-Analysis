# Chapter 11: 上下文与记忆 — "有限的窗口，无限的记忆"

> LLM 的上下文窗口是有限的，但用户的工作是连续的。本章分析两个代表性系统如何解决这个矛盾：**Claude Code** 用五层压缩管线 + Session Memory + MemDir + autoDream 四层协作；**Hermes Agent** 用三层工具防线 + 可插拔 ContextEngine + 内置记忆 + FTS5 会话搜索 + Honcho 用户建模。两者的架构选择折射出截然不同的设计哲学。

## 为什么需要单独讲这个主题？

上下文与记忆管理是 Coding Agent 最复杂的横切关注点之一。

**Claude Code** 的相关代码分布在 3 个服务目录（`services/compact/`、`services/SessionMemory/`、`services/autoDream/`）和 1 个子系统目录（`memdir/`），总计约 **20+ 个文件、3000+ 行代码**。之前的章节从不同视角零散涉及：

- [2.4 query.ts](../c02-core-engine/24-query-loop.md) — 压缩管线在主循环中的位置
- [4.4-4.5 Compact 与记忆 Prompt](../c04-prompt-engineering/44-compact-and-memory-prompts.md) — Prompt 设计视角
- [5.5-5.6 Compact 与 Session Memory](../c05-services/55-compact-and-memory.md) — 服务实现视角
- [9.3-9.4 Buddy 与 MemDir](../c09-subsystems/93-buddy-and-memdir.md) — 子系统视角

**Hermes Agent** 的相关代码分布在 `agent/` 目录（`context_engine.py`、`context_compressor.py`、`memory_manager.py`、`memory_provider.py`）、`tools/` 目录（`tool_result_storage.py`、`memory_tool.py`、`session_search_tool.py`）和 `plugins/memory/honcho/` 目录，总计约 **15+ 个文件、2500+ 行代码**。

本章将两个系统的设计统一起来，从架构角度系统性地讲清楚记忆体系的设计空间和权衡。

## 本章结构

### Part I: Claude Code

| 节 | 主题 | 核心问题 |
|---|------|---------|
| [11.1 上下文窗口的生命周期](./111-context-lifecycle.md) | Context Lifecycle | Token 如何增长？压力如何积累？五层管线如何逐级应对？ |
| [11.2 Auto-Compact 深入](./112-auto-compact.md) | Auto-Compact | 触发条件、压缩算法、三种变体、Scratchpad 模式 |
| [11.3 Session Memory — 跨压缩连续性](./113-session-memory.md) | Session Memory | 压缩会丢信息，Session Memory 如何补救？ |
| [11.4 MemDir — 文件系统持久化记忆](./114-memdir.md) | Filesystem Memory | 跨会话记忆的完整读写流程、相关性检索、四种记忆类型、会话历史召回机制 |
| [11.5 autoDream — 后台记忆整合](./115-auto-dream.md) | Auto Dream | 触发条件、四阶段流程、与 MemDir 的交互 |
| [11.6 四层协作：全景视图](./116-cooperation.md) | System Cooperation | 四个机制如何分工？信息在哪一层被保留、在哪一层被丢弃？ |

### Part II: Hermes Agent

| 节 | 主题 | 核心问题 |
|---|------|---------|
| [11.7 Hermes — 上下文管理](./117-hermes-context-management.md) | Context Management | 可插拔 ContextEngine、三层工具结果防线、ContextCompressor 压缩算法、迭代式摘要 |
| [11.8 Hermes — 记忆系统](./118-hermes-memory-system.md) | Memory System | 内置记忆（冻结快照）、FTS5 会话搜索、Honcho AI-native 用户建模、MemoryManager 编排 |

### Part III: 跨系统对比

| 节 | 主题 | 核心问题 |
|---|------|---------|
| [11.9 跨系统对比](./119-cross-system-comparison.md) | Cross-System | 架构哲学差异、压缩策略对比、记忆存储/检索/写入模型对比、成本分析 |

## 核心概念：三种时间尺度的记忆

两个系统都围绕"秒级 → 分钟级 → 天级"三种时间尺度组织记忆，但具体实现截然不同：

```
Claude Code                              Hermes Agent
─────────────────────────────           ─────────────────────────────
┌─ 秒级：上下文窗口 ──────┐             ┌─ 秒级：上下文窗口 ──────┐
│ 200K tokens 上限         │             │ 模型 context_length      │
│ 五层管线从轻到重         │             │ 三层工具防线 + 压缩器    │
└────────┬────────────────┘             └────────┬────────────────┘
         │                                       │
┌─ 分钟级：压缩 + 连续性 ─┐             ┌─ 分钟级：压缩 + 连续性 ─┐
│ Auto-Compact (Fork 子Agent)│            │ ContextCompressor       │
│ Session Memory (10节笔记)  │            │ _previous_summary 迭代  │
│ 独立侧通道保连续性        │            │ 压缩层内部保连续性       │
└────────┬────────────────┘             └────────┬────────────────┘
         │                                       │
┌─ 天级：持久化 + 整合 ───┐             ┌─ 天级：持久化 + 搜索 ───┐
│ MemDir (.md 文件)        │             │ 内置记忆 (3.5KB 全量注入)│
│ 4种封闭分类 + Sonnet检索  │             │ FTS5 会话搜索 (SQLite)  │
│ extractMemories 提取      │             │ Honcho 用户建模 (可选)  │
│ autoDream 后台整合        │             │ MemoryManager 编排      │
└──────────────────────────┘             └──────────────────────────┘
```

## 涉及的代码

### Claude Code

| 目录 | 文件数 | 行数 | 职责 |
|------|-------|------|------|
| `src/services/compact/` | 6 | ~800 | 压缩算法、自动触发、消息分组 |
| `src/services/SessionMemory/` | 6 | ~600 | 会话笔记的更新与恢复 |
| `src/services/autoDream/` | 3 | ~400 | 后台记忆整合 |
| `src/services/extractMemories/` | 3 | ~300 | 会话结束时提取记忆 |
| `src/memdir/` | 8 | ~1200 | 持久化记忆的存储与检索 |

### Hermes Agent

| 目录 | 关键文件 | 职责 |
|------|---------|------|
| `agent/` | `context_engine.py`, `context_compressor.py` | 可插拔上下文引擎、有损摘要压缩 |
| `agent/` | `memory_manager.py`, `memory_provider.py` | 记忆 Provider 编排、抽象接口 |
| `tools/` | `tool_result_storage.py`, `budget_config.py` | 三层工具结果持久化防线 |
| `tools/` | `memory_tool.py` | 内置记忆（MEMORY.md + USER.md） |
| `tools/` | `session_search_tool.py` | FTS5 跨会话搜索 |
| `hermes_state.py` | — | SQLite 会话存储（sessions + messages + FTS5） |
| `plugins/memory/honcho/` | `__init__.py` | Honcho AI-native 用户建模 |

## 前置知识

- [Chapter 2: 核心引擎](../c02-core-engine/README.md) — 主循环如何驱动压缩
- [Chapter 4: Prompt 工程](../c04-prompt-engineering/README.md) — 压缩和记忆的 Prompt 设计
- [Chapter 5: 服务层](../c05-services/README.md) — 服务的通用架构模式

---

> **下一节**：[11.1 上下文窗口的生命周期](./111-context-lifecycle.md)
