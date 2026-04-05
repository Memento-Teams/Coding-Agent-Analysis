# Chapter 2: 核心引擎 — "Claude 是怎么思考和行动的"

> 当你对 Claude 说"帮我修这个 bug"，Claude 不只是回一段文字。它会读代码、编辑文件、运行测试——这一切的背后，是一个循环引擎在驱动。这一章拆解这个引擎的每一个零件。

## 本章结构

| 节 | 主题 | 核心问题 |
|---|------|---------|
| [2.1 涉及的文件与目录](./21-files-overview.md) | Files Overview | 核心引擎由哪些文件组成？ |
| [2.2 一次对话到底发生了什么](./22-conversation-flow.md) | Conversation Flow | "调 API → 执行工具 → 再调 API" 的循环全景 |
| [2.3 QueryEngine.ts — 引擎的外壳](./23-query-engine.md) | QueryEngine | 异步生成器如何同时解决流式输出和流程控制？ |
| [2.4 query.ts — 引擎的心脏](./24-query-loop.md) | Query Loop | while(true) 四阶段循环的完整拆解 |

## 前置知识

阅读本章前，建议先完成：
- [Chapter 0: 项目概览](../Chapter-0/README.md) — 理解引擎在整体架构中的位置
- [0.1 启动流程](../Chapter-0/01-boot-sequence.md) — 理解引擎如何被启动

## 本章的核心概念

本章围绕一个核心循环展开——学术上叫 **ReAct**（Reasoning + Acting）模式：

```
用户输入 → 构建 Prompt → 调 Claude API → 解析响应
                                              │
                            ┌─────────────────┤
                            │ 有 tool_use     │ 纯文本
                            ▼                 ▼
                        执行工具          返回给用户
                            │              (循环结束)
                            ▼
                    结果追加到对话
                            │
                            └──→ 回到 "调 Claude API"
```

理解了这个循环，你就理解了 Claude Code 的灵魂。
