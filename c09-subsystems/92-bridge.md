# 9.2 Bridge — IDE 集成（36 个文件）

> Bridge 是 Claude Code 和 IDE（VSCode、JetBrains、Claude.ai）之间的双向通信层。它把 Claude Code 变成一个**无头 AI 后端**，让 IDE 插件直接调用 Claude 的能力。

## Bridge 的两种用途

### 用途 1：IDE 插件（remote-control 模式）

```
VSCode 插件
  │ HTTP/WebSocket（经由 claude.ai Bridge 服务）
  ▼
Claude Code 后台进程（无头，没有终端 UI）
  ├── 接收 IDE 指令（打开文件、执行命令、查询 AI）
  ├── 把 AI 回复发回 IDE（diff 预览、代码建议）
  └── 权限审批通过 IDE 原生对话框（非终端 UI）
```

### 用途 2：Always-On REPL Bridge（持久连接）

```
claude.ai 网页端
  │ WebSocket
  ▼
Claude Code 进程（正常运行，带终端 UI）
  ├── 接收来自网页的指令
  └── 把结果推送到网页
```

## 连接生命周期

```tsx
// bridge/bridgeMain.ts
const DEFAULT_BACKOFF = {
  connInitialMs: 2_000,    // 首次重连等待 2 秒
  connCapMs: 120_000,      // 最大退避 2 分钟
  connGiveUpMs: 600_000,   // 10 分钟后放弃
}

async function runBridge(): Promise<void> {
  let backoffMs = DEFAULT_BACKOFF.connInitialMs

  while (Date.now() - startTime < DEFAULT_BACKOFF.connGiveUpMs) {
    try {
      await connectAndRun()   // 建立连接，处理消息
      backoffMs = DEFAULT_BACKOFF.connInitialMs  // 成功后重置退避
    } catch (err) {
      if (isFatalError(err)) throw err  // 认证失败等致命错误不重试
      await sleep(backoffMs)
      backoffMs = Math.min(backoffMs * 2, DEFAULT_BACKOFF.connCapMs)  // 指数退避
    }
  }
}
```

## JWT Token 刷新调度器

```tsx
// bridge/jwtUtils.ts
function createTokenRefreshScheduler({ getAccessToken, onRefresh }) {
  let generation = 0  // Generation counter 防竞态

  async function scheduleRefresh(token: string) {
    const gen = ++generation
    const expiry = decodeJwtExpiry(token)

    // 在过期前 5 分钟刷新（非阻塞式）
    const refreshAt = expiry - 5 * 60 * 1000
    const delay = Math.max(0, refreshAt - Date.now())

    await sleep(delay)

    if (gen !== generation) return  // 被新的刷新覆盖了，放弃

    const newToken = await getAccessToken()
    if (newToken !== token) {
      onRefresh(newToken)
      scheduleRefresh(newToken)  // 刷新后重新调度
    }
  }

  return { start: (token) => scheduleRefresh(token) }
}
```

> **Generation Counter 防竞态**：如果用户快速重连（网络抖动），可能有多个 `scheduleRefresh` 同时在跑。每次调用记下当前 generation，等待后检查是否还是自己的——不是就放弃。这比 `clearTimeout/setTimeout` 更简洁，也不需要管理定时器 ID。

## Bridge 模式 vs 终端模式

Bridge 模式下，Claude Code 是无头后台进程，大量模块不需要加载：

| 模块 | 终端模式 | Bridge 模式 |
|------|:--------:|:-----------:|
| Ink 渲染器（终端 UI） | 必须 | 不需要 |
| React 组件（144 个） | 必须 | 不需要 |
| Vim 模式 | 必须 | 不需要 |
| 键盘事件处理 | 必须 | 不需要 |
| Commander.js 参数解析 | 必须 | 不需要 |
| 42 个工具 | 必须 | 由 Bridge 层处理 |
| 配置加载 | 必须 | 必须 |
| OAuth Token | 必须 | 必须 |
| Bridge 核心逻辑 | 不走 | 必须 |

Fast Path 在 cli.tsx 中确保 Bridge 模式跳过所有不必要的加载（见 [Chapter 1](../c01-boot-sequence/11-cli-entry.md)）。

> **下一节**：[9.3-9.4 Buddy 与 MemDir](./93-buddy-and-memdir.md)
