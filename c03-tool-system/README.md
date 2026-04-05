# Chapter 3: 工具系统 — "Claude 的手和脚"

> Claude 的"大脑"是 LLM，但光有大脑不能改代码、不能跑命令、不能搜文件。工具系统就是 Claude 的手和脚——42 个工具让它能真正"做事"。这一章拆解工具是怎么定义、注册、执行、授权的。

## 本章结构

| 节 | 主题 | 核心问题 |
|---|------|---------|
| [3.1-3.2 工具基础](./31-tool-basics.md) | Tool Anatomy | 一个工具长什么样？buildTool() 做了什么？ |
| [3.3-3.4 注册与权限](./33-registration-and-permissions.md) | Registration & Permissions | 42 个工具如何注册？权限系统如何四层防御？ |
| [3.5-3.6 文件操作与命令执行](./35-file-and-bash-tools.md) | File & Bash Tools | FileRead/Write/Edit 和 BashTool 的内部细节 |
| [3.7-3.8 Agent 协作与任务管理](./37-agent-and-task-tools.md) | Agent & Task Tools | AgentTool、SendMessage、Team、Task 系列 |
| [3.9-3.12 模式切换、网络与元操作](./39-mode-web-meta-tools.md) | Mode, Web & Meta Tools | Plan/Worktree 模式、WebFetch/Search、SkillTool 等 |
| [3.13-3.14 调度远程与工程启示](./313-scheduling-and-insights.md) | Scheduling & Insights | Cron、RemoteTrigger、未开源工具、设计总结 |

## 前置知识

- [c00: 0.2 工具系统概览](../c00-project-overview/02-tool-architecture.md) — 工具分类全景
- [Chapter 2: 2.4.4 工具执行](../c02-core-engine/24-query-loop.md#244-阶段-3工具执行--并行还是串行) — 引擎如何调用工具
- [Chapter 2: 2.4.7 Tool 接口](../c02-core-engine/24-query-loop.md#247-tool-接口--工具的契约) — 工具契约

## 关键数字

- **42 个开源工具** + ~15 个未开源工具
- **最复杂的工具**：BashTool（18 个文件）
- **最简单的工具**：GrepTool（3 个文件）
- **每个工具都有**：逻辑 + UI + 描述，三件套
