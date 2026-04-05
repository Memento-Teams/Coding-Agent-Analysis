# 5.0-5.1 服务层全景

> 本节列出服务层涉及的所有文件，并展示 20 个服务的分层结构。

## 5.0 涉及的文件

| 目录/文件 | 职责 | 规模 |
|-----------|------|------|
| `src/services/api/` | Claude API 客户端、重试、错误分类、日志 | 7 文件，~6,000 行 |
| `src/services/mcp/` | MCP 服务器连接、配置、认证 | 25+ 文件，~5,000 行 |
| `src/services/tools/` | 工具执行编排、流水线、并发控制 | 4 文件，~800 行 |
| `src/services/compact/` | 对话压缩算法、自动触发、微压缩 | 8 文件，~2,500 行 |
| `src/services/SessionMemory/` | 会话记忆维护 | 4 文件，~500 行 |
| `src/services/analytics/` | 事件追踪、Datadog、GrowthBook | 7 文件，~1,500 行 |
| `src/services/oauth/` | OAuth 2.0 + PKCE 认证流程 | 4 文件，~600 行 |
| `src/services/lsp/` | 语言服务器协议集成 | 6 文件，~1,000 行 |
| `src/services/plugins/` | 插件安装、卸载、依赖管理 | 3 文件，~500 行 |
| `src/services/extractMemories/` | 自动记忆提取 | 3 文件，~400 行 |
| `src/services/autoDream/` | 后台记忆整合（"做梦"） | 3 文件，~400 行 |
| `src/services/MagicDocs/` | 自动更新文档 | 2 文件，~300 行 |
| `src/services/tips/` | 上下文提示建议 | 3 文件，~300 行 |
| `src/services/policyLimits/` | 组织策略限制 | 2 文件，~200 行 |
| `src/services/tokenEstimation.ts` | Token 计数估算 | ~150 行 |
| `src/services/notifier.ts` | 终端通知 | ~100 行 |
| `src/services/rateLimitMessages.ts` | 限流消息生成 | ~80 行 |

## 5.1 全景图 — 20 个服务的层次

```
┌─────────────────────────── 外部通信层 ──────────────────────────┐
│                                                                  │
│  ┌── API 服务 ──────────┐  ┌── MCP 服务 ──────┐  ┌── OAuth ──┐│
│  │ client.ts  SDK 工厂   │  │ client.ts 连接    │  │ PKCE 认证 ││
│  │ claude.ts  请求编排   │  │ config.ts 配置    │  │ Token 管理││
│  │ withRetry.ts 重试     │  │ auth.ts   认证    │  └──────────┘│
│  │ errors.ts  错误分类   │  │ types.ts  类型    │               │
│  │ logging.ts 日志       │  └─────────────────┘               │
│  │ usage.ts   用量追踪   │                                     │
│  └──────────────────────┘                                     │
├──────────────────────── 执行编排层 ─────────────────────────────┤
│  ┌── Tool 执行服务 ─────────────┐  ┌── Compact 服务 ──────────┐│
│  │ StreamingToolExecutor 流水线  │  │ compact.ts   压缩算法    ││
│  │ toolOrchestration 并发编排    │  │ autoCompact  自动触发    ││
│  │ toolExecution 单工具执行      │  │ microCompact 微压缩     ││
│  └──────────────────────────────┘  │ grouping     消息分组    ││
│                                     └────────────────────────┘│
├──────────────────────── 智能后台层 ─────────────────────────────┤
│  ┌── SessionMemory ─┐ ┌── extractMemories ─┐ ┌── autoDream ──┐│
│  │ 会话记忆维护      │ │ 自动记忆提取        │ │ 记忆整合"做梦" ││
│  └──────────────────┘ └───────────────────┘ └───────────────┘│
│  ┌── MagicDocs ─────┐ ┌── tips ────────────┐ ┌── AgentSummary┐│
│  │ 自动更新文档      │ │ 上下文提示建议      │ │ Agent 摘要    ││
│  └──────────────────┘ └───────────────────┘ └───────────────┘│
├──────────────────────── 基础设施层 ─────────────────────────────┤
│  ┌── analytics ─────┐ ┌── LSP ────────┐ ┌── plugins ────────┐ │
│  │ 事件追踪         │ │ 语言服务器     │ │ 插件管理          │ │
│  │ Datadog + 1P     │ │ 代码智能       │ │ 安装/卸载/依赖    │ │
│  │ GrowthBook       │ │ 诊断聚合       │ │                  │ │
│  └──────────────────┘ └──────────────┘ └──────────────────┘ │
│  ┌── policyLimits ──┐ ┌── tokenEstimation ┐ ┌── notifier ───┐ │
│  │ 组织策略限制      │ │ Token 计数估算     │ │ 终端通知       │ │
│  └──────────────────┘ └──────────────────┘ └───────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

> **下一节**：[5.2 API 服务](./52-api-service.md)
