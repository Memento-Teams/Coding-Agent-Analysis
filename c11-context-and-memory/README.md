# Chapter 11: 上下文与记忆 — "有限的窗口，无限的记忆"

> LLM 的上下文窗口是有限的，但用户的工作是连续的。Claude Code 用三层机制解决这个矛盾：**压缩**管理窗口内的信息密度，**Session Memory** 保持压缩边界间的连续性，**MemDir** 提供跨会话的持久化记忆。再加上 **autoDream** 在后台默默整理，四个系统协作形成了一套完整的记忆架构。

## 为什么需要单独讲这个主题？

上下文与记忆管理是 Claude Code 最复杂的横切关注点之一。它的代码分布在 3 个服务目录（`services/compact/`、`services/SessionMemory/`、`services/autoDream/`）和 1 个子系统目录（`memdir/`），总计约 **20+ 个文件、3000+ 行代码**。之前的章节从不同视角零散涉及：

- [2.4 query.ts](../c02-core-engine/24-query-loop.md) — 压缩管线在主循环中的位置
- [4.4-4.5 Compact 与记忆 Prompt](../c04-prompt-engineering/44-compact-and-memory-prompts.md) — Prompt 设计视角
- [5.5-5.6 Compact 与 Session Memory](../c05-services/55-compact-and-memory.md) — 服务实现视角
- [9.3-9.4 Buddy 与 MemDir](../c09-subsystems/93-buddy-and-memdir.md) — 子系统视角

本章将这些散落的内容统一起来，从架构角度系统性地讲清楚整个记忆体系。

## 本章结构

| 节 | 主题 | 核心问题 |
|---|------|---------|
| [11.1 上下文窗口的生命周期](./111-context-lifecycle.md) | Context Lifecycle | Token 如何增长？压力如何积累？五层管线如何逐级应对？ |
| [11.2 Auto-Compact 深入](./112-auto-compact.md) | Auto-Compact | 触发条件、压缩算法、三种变体、Scratchpad 模式 |
| [11.3 Session Memory — 跨压缩连续性](./113-session-memory.md) | Session Memory | 压缩会丢信息，Session Memory 如何补救？ |
| [11.4 MemDir — 文件系统持久化记忆](./114-memdir.md) | Filesystem Memory | 跨会话记忆的完整读写流程、相关性检索、四种记忆类型 |
| [11.5 autoDream — 后台记忆整合](./115-auto-dream.md) | Auto Dream | 触发条件、四阶段流程、与 MemDir 的交互 |
| [11.6 四层协作：全景视图](./116-cooperation.md) | System Cooperation | 四个机制如何分工？信息在哪一层被保留、在哪一层被丢弃？ |

## 核心概念：三种时间尺度的记忆

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户的工作（连续的）                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─── 秒级：上下文窗口 ───┐                                        │
│  │ 原始对话消息            │  ← 完整保真，但容量有限                 │
│  │ 工具调用 + 结果          │                                       │
│  │ 200K tokens 上限        │                                        │
│  └────────┬───────────────┘                                        │
│           │ 快满了                                                   │
│           ▼                                                         │
│  ┌─── 分钟级：压缩 + Session Memory ───┐                           │
│  │ 五层管线逐级压缩                      │  ← 有损，但保留关键信息   │
│  │ Session Memory 保留 10 节结构化笔记   │                          │
│  │ 跨 Compact 边界保持连续性             │                          │
│  └────────┬──────────────────────────┘                              │
│           │ 会话结束                                                │
│           ▼                                                         │
│  ┌─── 天级：MemDir + autoDream ───┐                                │
│  │ 文件系统持久化记忆               │  ← 高度压缩，只留核心          │
│  │ 四种类型：user/feedback/         │                               │
│  │   project/reference              │                               │
│  │ autoDream 后台整合修剪           │                               │
│  └──────────────────────────────┘                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 涉及的代码

| 目录 | 文件数 | 行数 | 职责 |
|------|-------|------|------|
| `src/services/compact/` | 6 | ~800 | 压缩算法、自动触发、消息分组 |
| `src/services/SessionMemory/` | 6 | ~600 | 会话笔记的更新与恢复 |
| `src/services/autoDream/` | 3 | ~400 | 后台记忆整合 |
| `src/services/extractMemories/` | 3 | ~300 | 会话结束时提取记忆 |
| `src/memdir/` | 8 | ~1200 | 持久化记忆的存储与检索 |

## 前置知识

- [Chapter 2: 核心引擎](../c02-core-engine/README.md) — 主循环如何驱动压缩
- [Chapter 4: Prompt 工程](../c04-prompt-engineering/README.md) — 压缩和记忆的 Prompt 设计
- [Chapter 5: 服务层](../c05-services/README.md) — 服务的通用架构模式

---

> **下一节**：[11.1 上下文窗口的生命周期](./111-context-lifecycle.md)
