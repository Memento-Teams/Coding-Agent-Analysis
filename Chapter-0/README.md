# Chapter 0: Claude Code 项目概览

> 本章是整个 Claude Code 源码分析系列的起点。我们将从"上帝视角"俯瞰整个项目的架构，理解各层之间的关系，为后续深入每个子系统打下基础。

## 本章结构

| 节 | 主题 | 核心问题 |
|---|------|---------|
| [0.0 全局架构总览](./00-architecture-overview.md) | Top-Level Architecture | 整个项目由哪些层组成？它们如何协作？ |
| [0.1 启动流程](./01-boot-sequence.md) | Boot Sequence | 从 `$ claude` 命令到 UI 就绪，经历了什么？ |
| [0.2 工具系统](./02-tool-architecture.md) | Tool Architecture | 42 个工具是如何注册、发现和执行的？ |
| [0.3 多 Agent 系统](./03-agent-system.md) | Agent/Team Architecture | Agent 如何生成、通信和协作？ |
| [0.4 UI 渲染管线](./04-ui-rendering.md) | Ink Rendering Pipeline | 终端 UI 是如何用 React 渲染的？ |
| [0.5 服务层与扩展机制](./05-services-and-extensions.md) | Services & Extensions | API 调用、MCP、Skills、Plugins 如何运作？ |
| [0.6 数据流全景](./06-data-flow.md) | End-to-End Data Flow | 一次完整的用户交互，数据如何流转？ |

## 阅读建议

**如果你是第一次接触 Claude Code 源码**，建议按顺序阅读：
1. 先看 **0.0 全局架构**，建立整体认知
2. 再看 **0.1 启动流程**，理解程序如何跑起来
3. 然后看 **0.6 数据流全景**，理解一次交互的完整链路
4. 最后根据兴趣深入 0.2-0.5 的具体子系统

**如果你已有一定了解**，可以直接跳到感兴趣的章节。

## 项目规模速览

Claude Code 是一个大型 TypeScript 项目，以下数字可以帮助你建立直觉：

- **42 个工具** — 从文件读写到 Agent 生成，覆盖了开发者日常所需
- **101 个命令** — Slash Command 体系，支持 Git、Session、配置等操作
- **17 个内建 Skill** — 更高层的能力抽象，如 commit、review、pr 等
- **144 个 UI 组件** — 基于自定义 React 渲染器（Ink）的终端 UI
- **85+ 个 Hooks** — React Hooks 驱动的状态和行为管理
- **329+ 个工具函数** — 覆盖 auth、config、shell、tokens 等方方面面
