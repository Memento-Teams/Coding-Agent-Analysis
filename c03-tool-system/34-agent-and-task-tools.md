# 3.8-3.12 Agent 协作与任务管理工具

> AgentTool 是 Claude Code 实现"多 Agent 协作"的核心，SendMessage 和 Team 系列工具围绕它构建了完整的团队协作能力。Task 系列则提供了结构化的任务管理。

---

## 3.8 Agent 与团队协作类

### 3.8.1 AgentTool — 生成子 Agent（14 个文件）

**目录结构**：

```
AgentTool.tsx        ← 主实现
runAgent.ts          ← 执行引擎
forkSubagent.ts      ← Fork 隔离模式
resumeAgent.ts       ← 恢复后台 Agent
agentToolUtils.ts    ← 进度聚合
agentMemory.ts       ← Agent 记忆管理
agentColorManager.ts ← 颜色分配
loadAgentsDir.ts     ← Agent 定义加载
builtInAgents.ts     ← 内建 Agent 注册
built-in/            ← 5 个内建 Agent 定义
  ├── generalPurposeAgent.ts
  ├── planAgent.ts
  ├── exploreAgent.ts
  ├── claudeCodeGuideAgent.ts
  └── verificationAgent.ts
```

**三种执行模式**：

| 模式 | 触发条件 | 行为 | 适用场景 |
|------|---------|------|---------|
| 同步 | 默认 | 等待子 Agent 完成，返回结果 | 简单查询 |
| 异步 | `run_in_background: true` | 立即返回任务 ID，完成后通知 | 耗时任务 |
| 隔离 | `isolation: 'worktree'` | 创建 git worktree 副本 | 有风险的修改 |

**5 个内建 Agent**：

| Agent | 用途 | 可用工具 |
|-------|------|---------|
| `general-purpose` | 通用任务 | 所有工具 |
| `Explore` | 代码探索 | 只读工具 |
| `Plan` | 架构规划 | 只读工具 |
| `claude-code-guide` | Claude Code 使用指南 | 只读 + Web |
| `verification` | 验证执行结果 | 只读工具 |

**关键设计**：
- **用户自定义 Agent**：可在 `.claude/agents/` 目录下放 markdown 文件定义 Agent
- **进度聚合**：子 Agent 的工具使用进度实时上报给父 Agent
- **MCP 继承**：子 Agent 可继承父 Agent 的 MCP 服务器连接

---

### 3.8.2 SendMessageTool — Agent 间通信

```tsx
export const SendMessageTool = buildTool({
  name: 'SendMessage',
  async call({ to, summary?, message }, context)
  isReadOnly: (input) => typeof input.message === 'string',
})
```

**消息类型**：
- **纯文本消息**：发送到目标 Agent 的信箱
- **广播**：`to: "*"` 发给团队所有成员
- **关机请求**：发送 `shutdown_request`，等待目标 Agent 回复 approved/rejected
- **计划审批**：团队领导审批/拒绝成员的执行计划
- **跨会话通信**：通过 UDS（Unix Domain Socket）或 Bridge 发送给其他进程

**关键设计**：
- **自动唤醒**：如果目标 Agent 已停止，发消息时会自动恢复它
- **三层传输**：进程内 Agent → 团队信箱（文件系统）→ 跨会话（UDS/Bridge）
- **跨机器审批**：Bridge 消息需要用户额外确认

---

### 3.8.3 TeamCreateTool — 创建 Agent 团队

```tsx
export const TeamCreateTool = buildTool({
  name: 'TeamCreate',
  async call({ team_name, description?, agent_type? }, context)
  isEnabled: () => isAgentSwarmsEnabled(),
})
```

**创建流程**：

```
① 生成唯一团队名（如有冲突自动追加随机后缀）
② 生成领导的 ID："team-lead@team-name"
③ 写入团队配置文件：~/.claude/teams/{name}/config.json
④ 创建任务列表目录
⑤ 注册清理回调（session 结束时清理）
⑥ 更新 AppState 中的团队上下文
⑦ 分配 UI 颜色
```

**设计细节**：
- 每个领导只能管一个团队，防止管理冲突
- 领导不是"队友"：`isTeammate()` 对领导返回 false。领导不轮询信箱
- 确定性 ID：`team-lead@team-name` 格式，可预测、可推导

---

### 3.8.4 TeamDeleteTool — 销毁团队

无参数。**拒绝带活跃成员的销毁**——必须先优雅关闭所有成员，再销毁团队。清理：目录、worktree、颜色分配、信箱、AppState。

---

## 3.9 任务管理类

### 3.9.1 TaskCreateTool

```tsx
export const TaskCreateTool = buildTool({
  name: 'TaskCreate',
  async call({ subject, description, activeForm, metadata }, context)
  isConcurrencySafe: () => true,
  isEnabled: () => isTodoV2Enabled(),
})
```

创建新任务（状态 `pending`），执行 task-created 钩子，自动展开 UI 任务列表。

### 3.9.2 TaskGetTool / TaskListTool

- **TaskGetTool**：根据 ID 获取任务详情（subject、description、status、blocking 关系）
- **TaskListTool**：列出所有任务，自动过滤内部任务（`_internal` 标记），清理已完成任务的 `blockedBy` 引用
- 两者都是 `isConcurrencySafe: true`、`isReadOnly: true`

### 3.9.3 TaskUpdateTool

```tsx
export const TaskUpdateTool = buildTool({
  name: 'TaskUpdate',
  async call({ taskId, subject, description, status, owner, addBlocks, addBlockedBy, metadata }, context)
  isConcurrencySafe: () => true,
})
```

**关键设计**：
- **验证提醒**：当 3+ 个任务都标记完成但没有验证任务时，提醒 Claude 需要验证
- **所有权变更通知**：更换 owner 时通过信箱通知新负责人
- `status: 'deleted'` 会删除任务文件
- 队友标记任务为 `in_progress` 时自动设为 owner

### 3.9.4 TaskStopTool / TaskOutputTool

- **TaskStopTool**（别名 `KillShell`）：停止运行中的后台任务
- **TaskOutputTool**（别名 `AgentOutputTool`、`BashOutputTool`）：获取任务输出，支持阻塞/非阻塞模式。已标记为**废弃**，推荐用 Read 工具直接读取输出文件

### 3.9.5 TodoWriteTool — 结构化任务清单

```tsx
export const TodoWriteTool = buildTool({
  name: 'TodoWrite',
  async call({ todos }, context)
  isEnabled: () => !isTodoV2Enabled(),  // 新 Task 系统启用后禁用
})
```

更新 AppState 中的 todo 列表，全部完成时自动清空。

> **TodoWrite vs Task 系统**：TodoWriteTool 是简单的清单（给用户看进度），Task 系统是完整的任务管理（有 ID、阻塞关系、所有权）。

> **下一节**：[3.13-3.17 模式切换、网络与元操作](./35-mode-web-meta-tools.md)
