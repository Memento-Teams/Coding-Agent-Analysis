# 0.7 六大扩展机制的演进时间线

> Claude Code 的扩展能力不是一次性设计出来的，而是在两年间逐步叠加的。理解它们的出现顺序，有助于理解为什么架构是今天这个样子。

## 时间线

```
2024-11  MCP ─────────── 最先出现，解决"连接外部服务"的问题
           │
2025-07  Subagents ───── 让 Claude 可以生成子 Agent 并行工作
           │
2025-09  Hooks ───────── 让用户在工具执行前后插入自定义逻辑
           │
2025-10  Plugins ─────── 标准化的第三方扩展包（NPM/Git）
         Skills ──────── 最轻量的扩展——一个 Markdown 文件就是一个能力
           │
2026-02  Agent Teams ─── 多 Agent 团队协作（Coordinator + Swarm）
```

## 六大机制速览

| 机制 | 一句话解释 | 本质是什么 | 复杂度 |
|------|-----------|-----------|--------|
| **MCP** | 连接外部服务（数据库、Slack、GitHub…） | 通信协议（JSON-RPC） | 最重（119KB client.ts） |
| **Subagents** | Claude 生成子 Claude 并行干活 | 进程/线程管理 | 重（14 个文件） |
| **Hooks** | 在工具执行前后插入自定义脚本 | 生命周期拦截器 | 中（4 种 Hook 类型） |
| **Plugins** | 可安装的第三方扩展包 | 包管理器（类似 npm） | 中（40+ 文件） |
| **Skills** | 一个 Markdown 文件 = 一个能力 | Prompt 注入 | 最轻（~500 行） |
| **Agent Teams** | 多 Agent 组队协作 | 分布式编排 | 重（信箱 + Coordinator） |

## 它们之间的关系

```
用户写了一个 Skill（/deploy）
  → Skill 内部可以调用 Plugin 提供的命令
    → Plugin 内部可以自带 MCP Server 连接数据库
      → 执行过程中 Hooks 在每一步前后拦截检查
        → 如果任务复杂，Skill 可以生成 Subagent
          → 多个 Subagent 组成 Team 并行工作
```

它们不是互相替代的关系，而是**层层嵌套、互相组合**的关系。

## 各机制的详细章节

| 机制 | 详细讲解位置 |
|------|------------|
| MCP | [c05 服务层 - MCP](../c05-services/53-mcp-and-tool-execution.md) / [c08 扩展机制 - MCP](../c08-extensions/83-mcp.md) |
| Subagents | [c00 Agent 系统](./03-agent-system.md) / [c03 AgentTool](../c03-tool-system/34-agent-and-task-tools.md) |
| Hooks | [c00 Hooks 专题](./08-hooks-deep-dive.md) |
| Plugins | [c05 服务层 - Plugin](../c05-services/57-analytics-and-infra.md) / [c08 扩展机制 - Plugins](../c08-extensions/82-plugins.md) |
| Skills | [c08 扩展机制 - Skills](../c08-extensions/81-skills.md) |
| Agent Teams | [c00 Agent 系统](./03-agent-system.md) / [c03 Team 工具](../c03-tool-system/34-agent-and-task-tools.md) |
