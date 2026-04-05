# 7.6-7.7 关键 Hooks 与状态层整体架构

> 85+ 个 Hooks 中，最复杂的是 useCanUseTool（40KB 的权限决策中枢）。本节深入几个关键 Hook，并展示状态层的四层整体架构。

## 7.6 关键 Hooks 深度解析

### useCanUseTool — 权限决策中枢（40KB）

每次 Claude 要调用工具时，都会经过这个 Hook：

```
Claude 说要调用 BashTool("rm -rf /tmp/test")
     │
     ▼
① 校验输入（validateInput）
     │ 输入格式不对 → 直接拒绝
     ▼
② 查配置规则（always_allow / always_deny）
     │ 匹配 → 直接决定（不弹对话框）
     ▼
③ 工具的 checkPermissions() 方法
     │ allow → 继续
     │ deny  → 拒绝 + renderRejected()
     │ ask   → 继续（需要用户确认）
     ▼
④ 权限模式判断
     │ plan 模式 + 非只读工具 → 拒绝
     │ bypass 模式 → 直接允许
     │ deny 模式 → 拒绝所有
     ▼
⑤ Swarm 中？
     │ 是 Worker → 发请求给 Leader 审批（通过 Mailbox）
     │ 是 Leader → 正常流程
     ▼
⑥ 显示权限对话框（仅 ask 时）
     │ 用户点"允许" → 继续
     │ 用户点"拒绝" → 拒绝
     │ 用户点"始终允许" → 更新 settings + 继续
     ▼
⑦ 记录决策到 Analytics
     └── decision reason: 'classifier' | 'explicit_allow' | 'explicit_deny' | 'config'
```

**决策缓存**：同一个 toolUseID 不会弹两次对话框，即使工具在 StreamingToolExecutor 中被多次检查。

---

### useInboxPoller — 信箱轮询

```tsx
function useInboxPoller(agentId: AgentId) {
  useInterval(async () => {
    const messages = await readMailbox(agentId)
    for (const msg of messages) {
      switch (msg.type) {
        case 'plain_text':
          appendToInbox(msg)
          break
        case 'permission_request':
          const decision = await showPermissionDialog(msg)
          await writeToMailbox(msg.from, { type: 'permission_response', decision })
          break
        case 'shutdown_request':
          await handleShutdownRequest(msg)
          break
        case 'idle_notification':
          markTeammateIdle(msg.agentId)
          break
      }
    }
  }, POLL_INTERVAL_MS)  // 默认 500ms
}
```

### useBackgroundTaskNavigation — 后台任务切换

```tsx
function useBackgroundTaskNavigation() {
  const tasks = useAppState(s => s.tasks)

  // Shift+Tab 在 Teammates 之间循环
  useInput((input, key) => {
    if (key.shift && key.tab) {
      const sortedAgents = Object.keys(tasks).sort()
      const currentIdx = sortedAgents.indexOf(currentAgentId)
      const nextIdx = (currentIdx + 1) % sortedAgents.length
      switchToAgent(sortedAgents[nextIdx])
    }
  })
}
```

### useArrowKeyHistory — 命令历史导航

```tsx
function useArrowKeyHistory(currentInput: string) {
  const [historyIndex, setHistoryIndex] = useState(-1)

  useInput((_, key) => {
    if (key.upArrow) {
      const prev = getHistoryEntry(historyIndex + 1)
      if (prev !== undefined) {
        setHistoryIndex(i => i + 1)
        setInput(prev)
      }
    }
    if (key.downArrow) {
      if (historyIndex > 0) {
        setHistoryIndex(i => i - 1)
        setInput(getHistoryEntry(historyIndex - 1) ?? '')
      } else {
        setHistoryIndex(-1)
        setInput('')  // 回到用户正在输入的内容
      }
    }
  })
}
```

---

## 7.7 状态层的整体架构

```
┌──────────────────────────────────────────────────────────┐
│                      全局状态层                            │
│                                                          │
│  AppStateStore (Zustand-like)                           │
│  ├── getState() → AppState (DeepImmutable)              │
│  ├── setState(updater) → 通知订阅者                       │
│  └── subscribe(listener) → () => void                   │
│                                                          │
│  副作用层 (onChangeAppState.ts)                          │
│  ├── 磁盘持久化（settings.json）                         │
│  ├── Bridge 状态同步                                     │
│  ├── 桌面通知触发                                         │
│  └── Analytics 遥测                                     │
│                                                          │
├──────────────────────────────────────────────────────────┤
│                     局部上下文层                           │
│                                                          │
│  React Context Providers (9 个)                         │
│  ├── NotificationsContext — 通知队列                     │
│  ├── MailboxContext — 进程间消息                         │
│  ├── StatsContext — 会话统计                             │
│  └── ... 其余 6 个                                       │
│                                                          │
├──────────────────────────────────────────────────────────┤
│                      Hook 层                              │
│                                                          │
│  85+ 个 React Hooks                                     │
│  ├── useCanUseTool — 权限决策（40KB）                    │
│  ├── useInboxPoller — 信箱轮询                           │
│  ├── useBackgroundTaskNavigation — 后台任务切换           │
│  ├── useArrowKeyHistory — 命令历史                        │
│  └── ... 81+ 个其他 Hook                                 │
│                                                          │
├──────────────────────────────────────────────────────────┤
│                      组件层                               │
│                                                          │
│  144 个 React 组件                                       │
│  只通过 useAppState(selector) 订阅需要的状态切片          │
│  只通过 appStateStore.setState() 修改状态                │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 为什么 CLI 也需要 React 状态管理？

一个简单的 CLI 工具，`global.currentModel = 'opus'` 就够了。但 Claude Code 不简单：

- 144 个 UI 组件需要响应状态变化
- 85+ 个 Hook 共享数据
- 多 Agent 并发修改同一份状态
- 状态需要持久化到磁盘
- 某些状态变化需要触发副作用（通知、磁盘写入）

这些需求让全局变量的做法失控。单向数据流（状态 → UI，而不是 UI 直接修改状态）让每个状态变化都有明确的来源，调试时可以精确追踪"谁、什么时候、改了什么"。

---

> 返回 [Chapter 7 目录](./README.md) | 返回 [c00 总览](../c00-project-overview/README.md)
