# Chapter 7: 状态与上下文 — "React 单向数据流在 CLI 的实践"

> 大部分 CLI 工具用全局变量管理状态——改哪儿就改哪儿，谁都能写。Claude Code 选了另一条路：React 的单向数据流、不可变状态、selector 订阅。这一章讲为什么这个选择在 CLI 里也是对的。

## 本章结构

| 节 | 主题 | 核心问题 |
|---|------|---------|
| [7.1-7.3 AppState 与 Store](./71-appstate-and-store.md) | State & Store | 200+ 字段的 DeepImmutable 状态 + 40 行自制 Zustand |
| [7.4-7.5 副作用与 Context](./74-side-effects-and-context.md) | Side Effects & Context | 集中管理的副作用 + 9 个 React Context Provider |
| [7.6-7.7 关键 Hooks 与整体架构](./76-hooks-and-architecture.md) | Hooks & Architecture | useCanUseTool 权限决策 + 四层状态架构 |

## 前置知识

- [Chapter 6: 终端 UI](../Chapter-6/README.md) — UI 组件如何消费状态

## 核心设计

```
AppStateStore (全局单例，Zustand-like)
        │
        ├── getState() → DeepImmutable<AppState>
        ├── setState(updater) → 通知订阅者
        └── subscribe(listener)
                │
    ┌───────────┼───────────┐
    ▼           ▼           ▼
  组件A       组件B       组件C
  selector    selector    selector
  (messages)  (tasks)     (settings)
```

40 行代码实现 Zustand 核心功能。Selector 把每次状态更新的重渲范围从 144 个组件降到 5-30 个。
