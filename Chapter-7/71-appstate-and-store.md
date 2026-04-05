# 7.1-7.3 AppState 与 AppStateStore

> AppState 是整个应用的"单一数据源"——200+ 字段，用 `DeepImmutable` 保证不可变性。AppStateStore 是一个 40 行代码的自制 Zustand。

## 7.1 涉及的文件

| 文件 | 职责 | 大小 |
|------|------|------|
| `src/state/AppState.tsx` | 中心化状态类型定义（200+ 字段） | ~400 行 |
| `src/state/AppStateStore.ts` | Observable 状态存储 + 订阅 + 持久化 | 569 行 |
| `src/state/onChangeAppState.ts` | 状态变化的副作用（磁盘写入、Bridge 同步） | 171 行 |
| `src/state/selectors.ts` | 便捷 selector 函数 | 76 行 |
| `src/state/teammateViewHelpers.ts` | 团队视图状态辅助 | 141 行 |
| `src/context/` | 9 个 React Context Provider | ~800 行 |
| `src/hooks/` | 85+ 个 React Hooks | ~20,000 行 |

## 7.2 AppState — 200+ 字段的单一数据源

```tsx
type AppState = DeepImmutable<{
  // ── 基础配置 ──
  settings: SettingsJson             // ~/.claude/settings.json
  verbose: boolean                   // --verbose 标志
  mainLoopModel: ModelSetting        // 当前使用的模型

  // ── 对话历史 ──
  messages: Message[]                // 所有消息（含工具调用/结果）

  // ── 任务与 Agent ──
  tasks: Record<AgentId, TaskState>  // 所有后台任务状态
  expandedView: 'none' | 'tasks' | 'teammates'

  // ── 权限系统 ──
  toolPermissionContext: ToolPermissionContext
  // permissionMode: default/auto/plan/bypass/deny
  // alwaysAllow: string[], alwaysDeny: string[]

  // ── Agent 团队 ──
  teamContext?: TeamContext
  selectedTeammateId?: AgentId

  // ── 投机执行 ──
  speculation: SpeculationState      // 边用户说话边预生成回复

  // ── MCP 集成 ──
  mcp: { clients, tools, resources }

  // ── 插件 ──
  plugins: { enabled, disabled, commands }

  // ── 通知与权限请求队列 ──
  notifications: { current, queue }
  elicitation: { queue }

  // ── 进程间通信 ──
  inbox: { messages }
  agentNameRegistry: Map<string, AgentId>

  // ── Bridge/IDE 集成 ──
  replBridgeEnabled: boolean
  replBridgeConnected: boolean

  // ── 特殊运行模式 ──
  kairosEnabled: boolean             // 自主模式
  thinkingEnabled?: boolean          // 扩展思考模式
  remoteSessionUrl?: string          // 远程 Session URL
}>
```

### DeepImmutable 的意义

```tsx
type DeepImmutable<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? DeepImmutable<T[K]>
    : Readonly<T[K]>
}
```

效果：`state.messages.push(...)` → **编译报错**。`state.tasks['abc'].status = 'done'` → **编译报错**。

所有修改都必须走 `setState()` 这条正门。在有 85+ 个 Hook 和 144 个组件的代码库里，这个约束防止了大量难以追踪的 Bug。

---

## 7.3 AppStateStore — 自制的 Zustand

```tsx
type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: () => void) => () => void
}

function createStore<T>(initialState: T): Store<T> {
  let state = initialState
  const listeners = new Set<() => void>()

  return {
    getState: () => state,

    setState: (updater) => {
      const next = updater(state)
      if (Object.is(next, state)) return  // 引用相同 → 跳过通知
      state = next
      listeners.forEach(fn => fn())       // 通知所有订阅者
    },

    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

40 行代码，实现了 Zustand 的核心功能。React 层用 `useAppState` hook 订阅：

```tsx
// 只订阅 messages 字段，settings 变化不触发重渲
const messages = useAppState(state => state.messages)
```

### Selector 的意义

144 个组件，每次状态更新如果全部重渲，是 144 次 React reconciliation。实际情况：

- `messages` 变化 → 只有 `MessageList` 及其子树重渲（约 20-30 个组件）
- `tasks` 变化 → 只有 `AgentProgressLine` 等重渲（约 5 个组件）
- `settings` 变化 → 只有 `SettingsPanel` 重渲（约 3 个组件）

Selector 把每次状态更新的重渲范围从 144 个降到 5-30 个。

> **下一节**：[7.4-7.5 副作用与 Context](./74-side-effects-and-context.md)
