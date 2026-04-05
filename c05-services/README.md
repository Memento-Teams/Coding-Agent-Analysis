# Chapter 5: 服务层 — "Claude Code 的神经系统"

> 如果工具是 Claude 的手和脚，服务层就是它的神经系统——负责和 API 通信、处理错误重试、管理上下文压缩、追踪 Token 消耗、连接外部服务。这一章深入每个服务的实现细节。

## 本章结构

| 节 | 主题 | 核心问题 |
|---|------|---------|
| [5.0-5.1 全景与文件](./50-overview.md) | Overview | 20 个服务的分层结构 |
| [5.2 API 服务](./52-api-service.md) | API Service | 请求编排、智能重试、错误分类 |
| [5.3-5.4 MCP 与工具执行](./53-mcp-and-tool-execution.md) | MCP & Tool Execution | MCP 连接/认证、流水线执行、并发编排 |
| [5.5-5.6 Compact 与 Session Memory](./55-compact-and-memory.md) | Compact & Memory | 五层压缩管线、自动对话笔记 |
| [5.7-5.12 Analytics 与基础设施](./57-analytics-and-infra.md) | Analytics & Infrastructure | 可观测性、OAuth、LSP、Plugin、智能后台 |

## 前置知识

- [c00: 0.5 服务层概览](../c00-project-overview/05-services-and-extensions.md) — 服务层全景
- [Chapter 2: 核心引擎](../c02-core-engine/README.md) — 引擎如何调用服务
- [Chapter 3: 工具系统](../c03-tool-system/README.md) — 工具执行的上游

## 关键数字

- **20 个服务模块**，分布在 4 个层次
- **API 服务**：最核心，claude.ts 3,600+ 行
- **MCP 服务**：最大，client.ts 119KB
- **Compact 服务**：最精妙，五层从轻到重的压缩管线
