# Chapter 9: 特色子系统 — "每个都是独立的小王国"

> 有些功能足够特别，值得单独拆开讲。Vim 模式是一个完整的状态机实现；Bridge 把 Claude Code 变成无头 AI 后端；Buddy 是一只有统计属性的小鸭子；MemDir 是结构化的持久化记忆系统。这些子系统共同的特点：**高度内聚，低度耦合**。

## 本章结构

| 节 | 主题 | 核心问题 |
|---|------|---------|
| [9.1 Vim 模式](./91-vim-mode.md) | Vim State Machine | 如何从零实现完整的 Vim 状态机？ |
| [9.2 Bridge](./92-bridge.md) | IDE Integration | 如何把 CLI 变成 IDE 的无头后端？ |
| [9.3-9.4 Buddy 与 MemDir](./93-buddy-and-memdir.md) | Buddy & MemDir | 确定性生成的陪伴精灵 + 结构化持久化记忆 |
| [9.5-9.7 Remote、Voice 与共同设计原则](./95-remote-voice-principles.md) | Remote, Voice & Principles | 跨机器协作、语音输入、子系统的共同设计原则 |

## 前置知识

- [Chapter 0: 全局架构](../Chapter-0/00-architecture-overview.md) — 特色子系统在架构图中的位置
- [Chapter 1: 启动流程](../Chapter-1/11-cli-entry.md) — Bridge 模式的 Fast Path

## 涉及的文件

| 文件/目录 | 职责 | 规模 |
|-----------|------|------|
| `src/vim/` | Vim 模式状态机 | 5 文件，~2,500 行 |
| `src/bridge/` | IDE 双向通信 | 31 文件，~8,000 行 |
| `src/buddy/` | 陪伴精灵 UI | 6 文件，~500 行 |
| `src/memdir/` | 结构化持久化记忆 | 8 文件，~1,200 行 |
| `src/remote/` | 远程 Session 管理 | 4 文件，~800 行 |
| `src/server/` | Direct Connect WebSocket Server | 3 文件，~400 行 |
| `src/voice/` | 语音输入模式 | 1 文件，~50 行 |
| `src/services/voice.ts` | 语音服务核心 | ~200 行 |
