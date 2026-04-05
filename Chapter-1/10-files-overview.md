# 1.0 涉及的文件与目录

> 在深入代码之前，先了解启动流程涉及哪些文件，以及它们各自的职责。这张表可以作为后续阅读的速查参考。

## 文件清单

| 文件/目录 | 职责 | 行数 |
|-----------|------|------|
| `src/entrypoints/cli.tsx` | 真正的入口：环境变量、Fast Path、Lazy Import | 303 |
| `src/entrypoints/init.ts` | 配置加载、TLS/代理、迁移、安全检查 | ~290 |
| `src/main.tsx` | Commander.js 路由、状态初始化、启动 REPL | ~3400 |
| `src/bootstrap/state.ts` | 全局 Session 状态定义（200+ 字段） | ~250 |
| `src/context.ts` | System Prompt 上下文收集（Git、环境） | ~110 |
| `src/tools.ts` | 42 个工具的条件注册 | ~150 |
| `src/commands.ts` | 101 个命令的条件注册 | ~100+ |
| `src/migrations/` | 11 个版本迁移脚本 | ~600 |
| `src/utils/startupProfiler.ts` | 性能打点，毫秒级启动追踪 | ~100 |

## 调用关系

```
cli.tsx  ──(lazy import)──>  init.ts  ──>  migrations/
   │                                         
   └──(lazy import)──>  main.tsx  ──>  bootstrap/state.ts
                            │              │
                            │              └──>  Session 历史加载
                            │
                            ├──>  tools.ts (42 工具注册)
                            ├──>  commands.ts (101 命令注册)
                            ├──>  context.ts (环境信息收集)
                            └──>  React.render(<App/>)
```

> **注意 `main.tsx` 的体量**：~3400 行，是整个项目中最大的单文件之一。它承担了"总指挥"的角色——几乎所有启动后的初始化工作都在这里编排。

> **下一节**：[1.1 cli.tsx — 真正的第一行代码](./11-cli-entry.md)
