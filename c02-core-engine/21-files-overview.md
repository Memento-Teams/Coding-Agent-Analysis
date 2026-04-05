# 2.1 涉及的文件与目录

> 核心引擎的代码量并不大——两个核心文件加上几个辅助模块，总共不到 2000 行。但这不到 2000 行代码驱动了 Claude Code 的所有智能行为。

## 文件清单

| 文件 | 职责 | 大小 |
|------|------|------|
| `src/QueryEngine.ts` | 引擎入口：接收用户输入，编排整个流程 | 46KB |
| `src/query.ts` | 核心循环：调 API → 执行工具 → 再调 API → … | 68KB |
| `src/query/tokenBudget.ts` | Token 预算：决定什么时候该停 | ~90 行 |
| `src/query/stopHooks.ts` | 停止钩子：检查各种终止条件 | ~470 行 |
| `src/query/config.ts` | 查询配置：模型、Token 上限 | ~50 行 |
| `src/Tool.ts` | 工具接口：每个工具必须实现的契约 | ~700 行 |
| `src/services/tools/toolOrchestration.ts` | 工具编排：哪些工具可以并行 | ~115 行 |
| `src/services/tools/StreamingToolExecutor.ts` | 流水线执行：边收 API 回复边执行工具 | ~300 行 |
| `src/services/compact/autoCompact.ts` | 自动压缩：上下文太长时自动缩减 | ~90 行 |

## 调用关系

```
QueryEngine.ts          ← 引擎外壳：准备 Prompt、Memory、工具列表
    │
    └──→ query.ts       ← 引擎心脏：while(true) 核心循环
            │
            ├──→ callModel()                    ← 调 Claude API（流式）
            │       └──→ StreamingToolExecutor  ← 边收回复边执行工具
            │
            ├──→ toolOrchestration.ts           ← 工具并行/串行编排
            │
            ├──→ autoCompact.ts                 ← Token 超限时压缩
            │
            ├──→ tokenBudget.ts                 ← 检查产出是否递减
            │
            └──→ stopHooks.ts                   ← 检查各种终止条件
```

> **下一节**：[2.2 一次对话到底发生了什么](./22-conversation-flow.md)
