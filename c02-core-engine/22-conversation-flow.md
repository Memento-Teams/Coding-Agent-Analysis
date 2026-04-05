# 2.2 一次对话到底发生了什么？

> 在深入代码之前，先用一个具体例子完整走一遍引擎的工作流程。理解了这个流程，后面的代码就不再抽象。

## 完整示例

```
你输入：帮我修 src/app.ts 里的 TypeError bug

Claude 的内部流程：
  Round 1:
    → 调 API："用户说要修 bug"
    ← API 返回：我需要先看看代码 → [tool_use: FileRead("src/app.ts")]
    → 执行 FileReadTool → 读到文件内容
    → 把文件内容作为 tool_result 追加到对话

  Round 2:
    → 调 API：之前的对话 + 文件内容
    ← API 返回：找到问题了，需要修改第 42 行 → [tool_use: FileEdit(...)]
    → 执行 FileEditTool → 修改文件
    → 把修改结果作为 tool_result 追加到对话

  Round 3:
    → 调 API：之前的对话 + 修改结果
    ← API 返回：已修复！问题是类型不匹配，我把 string 改成了 number。
    → 没有 tool_use → 循环结束，返回文字给用户
```

## ReAct 模式

这个"调 API → 执行工具 → 再调 API"的循环，就是整个引擎的核心。学术上这叫 **ReAct**（Reasoning + Acting）模式，由 Shunyu Yao 等人在 2022 年提出（[论文链接](https://arxiv.org/abs/2210.03629)）。

核心思想是：让 LLM 交替进行"思考"和"行动"，而不是一口气想完再做。Claude Code 的实现就是这个模式的工程化落地。

```
传统方式：  思考思考思考思考思考 ──→ 一次性行动 ──→ 结果

ReAct 方式：思考 → 行动 → 观察 → 思考 → 行动 → 观察 → ... → 结果
              │      │      │
              │      │      └── 工具的执行结果
              │      └── 调用工具（FileRead、Bash 等）
              └── Claude API 的推理
```

每一轮"思考 → 行动 → 观察"对应 query.ts 的一次循环迭代。Claude 根据观察到的结果决定下一步做什么，而不是一开始就制定完整计划。

> **下一节**：[2.3 QueryEngine.ts — 引擎的外壳](./23-query-engine.md)
