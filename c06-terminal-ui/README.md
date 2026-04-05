# Chapter 6: 终端 UI — "用 React 思维构建终端"

> 为什么一个命令行工具要用 React？这是 Claude Code 最反直觉、也最聪明的工程决策之一。这一章拆解 Ink 渲染器的每一层，从 React 协调器到最终输出的 ANSI 转义序列。

## 本章结构

| 节 | 主题 | 核心问题 |
|---|------|---------|
| [6.1-6.2 文件与渲染管线](./61-pipeline-overview.md) | Pipeline Overview | 从 React 组件到 ANSI 转义序列的完整链路 |
| [6.3-6.5 Reconciler、DOM 与 Yoga](./63-reconciler-dom-yoga.md) | Reconciler & Layout | React 如何映射到终端？Flexbox 如何处理宽字符？ |
| [6.6-6.8 绘制引擎、双缓冲与增量渲染](./66-rendering-engine.md) | Rendering Engine | 1,462 行绘制引擎 + 三级内化 + 硬件滚动 |
| [6.9-6.10 键盘事件与组件库](./69-events-and-components.md) | Events & Components | 字节流解析、事件冒泡、144 个组件、Transport 抽象 |

## 前置知识

- [c00: 0.4 UI 渲染管线概览](../c00-project-overview/04-ui-rendering.md) — 本章是这张图的"放大版"

## 关键数字

- **144 个 React 组件** + **85+ 个 Hooks**
- 渲染管线总耗时：**~10-20ms/帧（50-100 FPS）**
- Screen Buffer 三级内化：Cell 大小从 ~200 字节降到 ~12 字节
- DECSTBM 硬件滚动：2 个转义序列 vs 2,000+ 个字符
