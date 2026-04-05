# 7.4-7.5 副作用管理与 React Context Providers

> 状态变化不只是 UI 更新——还需要写磁盘、同步 Bridge、发通知。所有副作用集中在一个文件里管理。9 个 Context Provider 处理更局部的跨组件数据。

## 7.4 状态变化的副作用 — onChangeAppState.ts

```tsx
appStateStore.subscribe(() => {
  const state = appStateStore.getState()
  const prev = previousState

  // 1. 设置变化 → 写入磁盘
  if (state.settings !== prev.settings) {
    writeSettingsToDisk(state.settings)
  }

  // 2. Bridge 连接状态变化 → 同步给 CCR
  if (state.replBridgeConnected !== prev.replBridgeConnected) {
    notifyBridgeStatusChange(state.replBridgeConnected)
  }

  // 3. 任务完成 → 发桌面通知
  for (const [id, task] of Object.entries(state.tasks)) {
    const prevTask = prev.tasks[id]
    if (task.status === 'completed' && prevTask?.status === 'running') {
      showNotification(`任务完成：${task.subject}`)
    }
  }

  // 4. 更新 Analytics 遥测
  logStateMetrics(state)

  previousState = state
})
```

> **副作用集中管理**：所有状态变化的副作用都在 `onChangeAppState.ts` 里统一管理，不散落在各个组件中。想知道磁盘写入在什么时候发生，只看这一个文件。

---

## 7.5 React Context Providers — 9 个跨组件数据通道

AppStateStore 处理应用级状态（消息、任务、权限），Context Provider 处理更局部的、需要跨层传递的数据：

| Provider | 提供数据 | 使用场景 |
|---------|---------|---------|
| `NotificationsContext` | 通知队列（toast/banner） | 任意组件显示通知 |
| `MailboxContext` | 进程间消息订阅 | Teammate 消息轮询 |
| `StatsContext` | 会话统计（Token、耗时、成本） | 状态栏、统计面板 |
| `VoiceContext` | 语音输入状态机 | 语音模式 UI |
| `ModalContext` | 全局 Modal 状态 | 协调 Escape 键行为 |
| `OverlayContext` | 活跃覆盖层追踪 | 防止 Escape 穿透 |
| `PromptOverlayContext` | 建议覆盖层可见性 | 自动补全提示 |
| `QueuedMessageContext` | 流式消息装配队列 | 工具结果延迟显示 |
| `FpsMetricsContext` | 帧率和渲染性能 | 性能调试面板 |

### AppState vs Context 的分工

两者都能跨组件共享数据，为什么需要两个系统？

| 维度 | AppStateStore | React Context |
|------|--------------|---------------|
| **作用域** | 全局单例，任何地方都能访问 | 局部子树共享 |
| **订阅方式** | 需要 selector，精确控制重渲 | 直接提供值，整颗子树重渲 |
| **适用数据** | 需要持久化或跨会话的数据 | 生命周期和组件树绑定的数据 |
| **示例** | 消息历史、权限设置、Agent 状态 | 通知队列、当前 Modal 状态 |

两者互补，各司其职。

> **下一节**：[7.6-7.7 关键 Hooks 与整体架构](./76-hooks-and-architecture.md)
