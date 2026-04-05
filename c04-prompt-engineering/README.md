# Chapter 4: Prompt 工程 — "如何教会 Claude 做事"

> Claude Code 不只是调 API，它用精心设计的 Prompt 教会 Claude 什么时候该小心、什么时候该大胆、怎么用工具、怎么和用户交互。这一章逐条分析每一个 Prompt，揭示工业级 Prompt Engineering 的技巧。

## 本章结构

| 节 | 主题 | 核心问题 |
|---|------|---------|
| [4.0-4.1 主系统 Prompt](./40-system-prompt.md) | System Prompt | Claude Code 的"宪法"长什么样？静态/动态如何分界？ |
| [4.2-4.3 工具与 Coordinator Prompt](./42-tool-and-coordinator-prompts.md) | Tool & Coordinator | 怎么教 Claude 用 42 个工具？Coordinator 如何编排多 Agent？ |
| [4.4-4.5 Compact 与记忆系统 Prompt](./44-compact-and-memory-prompts.md) | Compact & Memory | 对话压缩的 Scratchpad 模式、四种记忆类型的封闭分类法 |
| [4.6-4.7 Agent 与 Skill Prompt](./46-agent-and-skill-prompts.md) | Agent & Skill | 5 个内建 Agent 和 17 个 Skill 的 Prompt 设计 |
| [4.8-4.9 服务级 Prompt 与技巧总结](./48-service-prompts-and-summary.md) | Service Prompts & Summary | 隐藏在实现中的精彩 Prompt + 全部技巧总结 |

## 前置知识

- [Chapter 2: 核心引擎](../c02-core-engine/README.md) — 理解 System Prompt 在哪里被注入
- [Chapter 3: 工具系统](../c03-tool-system/README.md) — 理解每个工具的 prompt.ts 如何生效

## 为什么 Prompt 值得单独一章？

在 Claude Code 中，Prompt 不是附属品——它们是**核心产品逻辑**。主系统 Prompt 约 15,000 tokens，加上 42 个工具的 Prompt、5 个 Agent、17 个 Skill、各种服务级 Prompt，总量超过 50,000 tokens。这些 Prompt 的设计质量直接决定了 Claude Code 的行为质量。
