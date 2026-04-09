# 11.3 Session Memory — 跨压缩连续性

> 压缩解决了"窗口不够用"的问题，但压缩是有损的——每次压缩都会丢失细节。Session Memory 的角色是在压缩发生时，**独立地**维护一份结构化的"会话笔记"，确保关键信息不随压缩一起消失。

## 11.3.1 问题：压缩边界的信息断崖

考虑这个场景：

```
原始对话（30 轮工具调用，180K tokens）
  │
  ├── 用户说："用 tabs 缩进，不要 spaces"  ← 第 3 轮
  ├── 遇到 bug：auth.test.ts 第 34 行      ← 第 12 轮
  ├── 用户说："不要改 utils/ 目录的文件"    ← 第 18 轮
  ├── 当前正在修改 src/app.ts              ← 第 29 轮
  │
  ▼ Auto-Compact 触发
  
压缩后的摘要（9 节结构化文本，~8K tokens）
```

压缩后，Claude 看到的只是一份摘要。如果摘要质量不够好（漏掉了"用 tabs 缩进"这个要求），Claude 在后续操作中就会用 spaces——而用户以为它已经知道了。

**Session Memory 是压缩的"保险杠"**——它独立于压缩过程，持续维护一份更持久的笔记。即使压缩摘要遗漏了某些信息，Session Memory 中可能还保留着。

## 11.3.2 Session Memory 的结构

Session Memory 是一个 Markdown 文件，包含 **10 个固定章节**：

```markdown
# Session Title
_简短的会话标题_

## Current State         ← 最关键，确保连续性
_当前正在做什么、做到哪一步了_

## Task Specification
_用户的原始需求、约束条件_

## Files and Functions
_操作过的文件列表、关键函数_

## Workflow
_工作流程、决策链_

## Errors & Corrections
_遇到的错误、用户的纠正_

## Codebase and System Documentation
_代码库的关键信息_

## Learnings
_在这次会话中学到的东西_

## Key Results
_已完成的关键成果_

## Worklog
_按时间顺序的工作日志_
```

### 为什么是 10 节而非自由格式？

和 Compact Prompt 的 9 节设计同理——固定章节是一个"提取清单"，防止遗漏关键维度。但 Session Memory 的 10 节和 Compact 的 9 节侧重点不同：

| Session Memory | Compact Prompt | 区别 |
|---------------|----------------|------|
| Current State | Current Work | Session Memory 更关注"做到哪了" |
| Errors & Corrections | Errors | Session Memory 额外记录用户纠正 |
| Learnings | — | Compact 没有这个维度 |
| Worklog | — | Session Memory 保留时间线 |
| — | User Messages | Compact 逐条引用用户原话 |

两者互补：Compact 的摘要替换了原始对话，Session Memory 独立保留了一份"旁注"。

## 11.3.3 更新机制

Session Memory 的更新也是 Fork 子 Agent 完成的——不在主线程做，不打断用户交互：

```
触发条件：
  基于 Token 消耗量和工具调用次数的复合条件
  （不是简单的"每 N 轮更新一次"）

更新流程：
  ├── Fork 子 Agent（共享 Prompt Cache）
  ├── 注入 Session Memory 更新 Prompt
  ├── 子 Agent 阅读最近的对话
  ├── 增量更新 10 个章节
  │     ├── 不重写整个文件，只更新变化的部分
  │     ├── Current State 每次都更新
  │     └── 其他章节按需更新
  └── 写回磁盘
```

### 更新 Prompt 的关键约束

```
IMPORTANT: This message and these instructions are NOT part of
the actual user conversation. They are a system directive.

Rules:
- NEVER modify section headers or italic descriptions
- Each section < 2000 tokens
- Total < 12000 tokens
- Current State section is MOST IMPORTANT for continuity
```

三个设计要点：

1. **元指令隔离**：开头明确声明"这不是用户对话的一部分"。没有这句话，Claude 可能把更新指令和用户消息混淆，导致 Session Memory 中出现对更新指令的"回复"。

2. **结构保护**：`NEVER modify section headers or italic descriptions`。Session Memory 会被多次更新——如果不保护结构，几轮更新后章节标题可能被改写甚至删除，整个文件变成非结构化的乱文本。

3. **大小限制**：每节 < 2000 tokens，总量 < 12000 tokens。Session Memory 会被注入到后续的对话中——如果不限制大小，它自己就会成为 Token 负担。

## 11.3.4 Session Memory 在恢复会话时的作用

当用户关闭 Claude Code 后重新打开，继续之前的会话时：

```
恢复会话
  │
  ├── 从磁盘加载对话历史
  ├── 对话可能很长（上次没有压缩就退出了）
  │
  ├── 用变体 C（截止点压缩）压缩到当前轮
  │     ├── Current Work → Work Completed（语态转换）
  │     └── 生成摘要
  │
  ├── 注入 Session Memory
  │     ├── Current State 告诉 Claude "上次做到哪了"
  │     ├── Task Specification 告诉 Claude "用户要什么"
  │     └── Errors & Corrections 告诉 Claude "哪些坑已经踩过"
  │
  └── Claude 可以无缝继续工作
```

这就是 Session Memory 最大的价值——**会话恢复时的快速回忆**。没有 Session Memory，Claude 只能看到一份压缩摘要；有了 Session Memory，它还能看到一份独立维护的、更详细的笔记。

## 11.3.5 Session Memory vs Compact 的关系

两者不是替代关系，而是互补关系：

```
                    Compact                    Session Memory
─────────────────────────────────────────────────────────────
触发时机        Token 超过阈值               基于消耗量和调用数
产出物          替换原始对话的摘要            独立的笔记文件
信息损失        原始对话被删除               原始对话不受影响
更新方式        全量重写                     增量更新
保留位置        作为新的对话起点             注入到系统 Prompt
生命周期        每次 Compact 重写            跨多次 Compact 持续累积
```

关键差异在最后一行：**Compact 每次都是全量重写，Session Memory 是跨 Compact 累积的。** 这意味着第一次 Compact 丢失的信息，如果 Session Memory 之前已经记录了，后续仍然可用。

### 信息流向

```
原始对话
  │
  ├──────→ Compact 摘要（替换对话）
  │             │
  │             └──→ 下一轮对话的起点
  │
  └──────→ Session Memory（独立更新）
                │
                └──→ 注入系统 Prompt，作为旁注
```

---

> **下一节**：[11.4 MemDir — 文件系统持久化记忆](./114-memdir.md)
