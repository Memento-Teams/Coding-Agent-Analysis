# 10.5-10.6 可学走的工程实践与元主题

> 本节从 Claude Code 中提炼出 7 个可以直接应用到你自己项目中的工程实践，以及贯穿全书的一个元主题。

## 10.5 可以学走的七个工程实践

### 实践 1：先写接口，再写实现

`Tool.ts` 用 40+ 个方法定义了工具的全部"合同"，然后 42 个工具各自实现。引擎（query.ts）只调用接口，从不 `instanceof BashTool`。

**可以学走的**：在设计新系统时，先画出"如果有 N 个不同的实现，它们的公共接口是什么？"然后写接口，最后写实现。

### 实践 2：默认值偏向安全（fail-closed）

`buildTool()` 的默认：不并行、假设写入、需要权限。开发者忘写 `isConcurrencySafe: true` → 工具执行慢一点（不是安全问题）。

**可以学走的**：所有安全相关的配置，默认值选"更安全"而非"更方便"。让"偷懒"的代价是性能损失，而不是安全漏洞。

### 实践 3：用命名来强制规范

```tsx
// 普通写法：
systemPromptSection('date', async () => new Date().toISOString())

// Claude Code 的写法：
DANGEROUS_uncachedSystemPromptSection(
  'date',
  async () => new Date().toISOString(),
  '日期每次都变，会破坏 Prompt Cache'
)
```

函数名本身就是警告，调用时开发者被迫看到 `DANGEROUS_` 前缀和强制传入的理由参数。

**可以学走的**：对有副作用的 API，用命名约定（`DANGEROUS_`、`unsafe_`、`_force`）在调用点可见地标注风险。

### 实践 4：用 Prompt 测试假设（AI 系统独有）

Verification Agent 的 Prompt 做了普通单元测试做不到的事：

```
- "你会有冲动跳过检查。你会自我说服：'代码看起来没问题'。抵制这种冲动。"
- "前 80% 的完美会让你放松 20% 的检查——这正是 bug 藏身的地方。"
- 对抗性探测清单：并发、边界值、幂等性、孤儿操作
```

**可以学走的**：为 AI Agent 写 Prompt 时，不只是说"做什么"，还要说"你会倾向于做什么错误的事，以及为什么不该这样做"。预言失败模式比禁止它们更有效。

### 实践 5：懒加载是对用户时间的尊重

```
claude --version → 0 个 import，< 1ms
claude bridge    → 约 50 个 import，~50ms
claude           → 约 200 个 import，~135ms
```

这不是微优化——每天在 CI 里跑几千次的命令，0.1 秒 × 几千次 = 几分钟的浪费。

**可以学走的**：对于频繁调用的工具（CI 命令、开发脚本），做一个启动时间剖析。找出最快路径（`--version`、`--help`），确保它们不触发慢加载。

### 实践 6：把不变的和变的分开

这是整个代码库中重复次数最多的模式：

```
System Prompt：静态部分（Prompt Cache）vs 动态部分（每次重算）
工具注册：始终注册 vs Feature-gated vs 懒加载
Compact：落盘 → Snip → Micro → Collapse → Auto（从轻到重）
内建工具 + MCP 工具：分别排序，保证前缀稳定（Prompt Cache）
```

**可以学走的**：在系统的任何地方，问"哪些东西永远不变（或很少变）？"，把它们和"经常变的"分开处理。不变的部分可以缓存、预计算、合并。

### 实践 7：Prompt Engineering 是系统设计

Claude Code 的 Prompt 文件比很多人的工程代码质量更高：

- **结构化分层**：原则 → 策略 → 规则
- **反面指令**：不只说"做什么"，也说"不做什么"
- **决策框架**：不穷举规则，教 Claude 评估"可逆性和影响范围"
- **命名反模式**：给坏行为一个名字，让 Claude 能识别
- **预防接种**：描述 Claude 会有什么错误倾向，然后要求抵制

**可以学走的**：对 AI 系统，Prompt 的质量和代码质量同等重要。像写高质量代码一样认真写 Prompt：可读、可维护、有结构、有测试（Verification Agent）。

---

## 10.6 一个贯穿全书的元主题

读完这 10 章，你可能注意到一个反复出现的主题：

### 每一层都在为下一层提供更简单的抽象。

```
LLM API（复杂：流式、重试、压缩、降级）
    ↓ QueryEngine 封装
引擎（简单：submitMessage() → AsyncGenerator）
    ↓ Tool 接口封装
工具（简单：call(input) → ToolResult）
    ↓ SkillTool 封装
Skill（最简单：/simplify 就是一段 Prompt）
```

这不是偶然的——这是**有意的分层设计**。每一层都知道下层的细节，但上层完全不需要知道。

- 用户不需要知道 `query.ts` 里的 Error Withholding
- `query.ts` 不需要知道 `BashTool` 的 18 个文件
- `BashTool` 不需要知道 `screen.ts` 的双缓冲优化

每一层的复杂度都被封装在接口之后，对外只暴露一个清晰的 API。

这就是软件工程 50 年来反复证明的核心原则：**好的抽象是系统可以持续演进的原因。**

---

> **尾声**
>
> 42 个工具、101 个命令、36 个服务、144 个组件、85+ 个 Hook、17 个 Skill——这些数字背后是数以百计的工程决策，每一个都有它的道理。
>
> 最好的系统不是没有取舍，而是每一个取舍都是有意识的——知道在牺牲什么、在换取什么。
>
> Claude Code 的代码在说：快速启动比完整加载更重要（Fast Path）；安全默认比方便默认更重要（fail-closed）；接口稳定比实现固定更重要（Tool 接口）；Prompt 质量和代码质量同等重要（Verification Agent）。
>
> 希望这份解析不只告诉你 Claude Code 怎么工作，也让你对"系统设计的取舍"有新的视角。

---

> 返回 [Chapter 10 目录](./README.md) | 返回 [Chapter 0 总览](../Chapter-0/README.md)
