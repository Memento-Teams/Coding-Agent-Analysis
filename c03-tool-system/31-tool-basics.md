# 3.1-3.2 工具基础 — 一个工具长什么样？

> 在讲 42 个工具之前，先看一个最简单的例子理解工具的骨架，然后看完整的 Tool 接口。

## 3.1 涉及的文件

| 文件/目录 | 职责 | 大小 |
|-----------|------|------|
| `src/Tool.ts` | Tool 接口定义、buildTool() 工厂函数 | ~800 行 |
| `src/tools.ts` | 42 个工具的注册表 | ~390 行 |
| `src/tools/BashTool/` | Shell 命令执行（最复杂的工具） | 18 个文件 |
| `src/tools/FileEditTool/` | 文件局部编辑 | 6 个文件 |
| `src/tools/AgentTool/` | 子 Agent 生成 | 14 个文件 |
| `src/tools/GrepTool/` | 代码搜索（最简单的工具之一） | 3 个文件 |
| `src/hooks/useCanUseTool.tsx` | 权限检查 UI 层 | 40KB |
| `src/constants/tools.ts` | 工具白名单/黑名单 | ~100 行 |

## 3.2 一个工具长什么样？

### 目录结构（以 GrepTool 为例）

```
src/tools/GrepTool/
├── GrepTool.ts    ← 核心逻辑：调 ripgrep、处理结果
├── UI.tsx         ← 终端渲染：显示搜索结果
└── prompt.ts      ← 给 Claude 看的描述：告诉它什么时候该用 Grep
```

每个工具都是这个模式：**逻辑 + UI + 描述**，三件套装在一个独立目录里。

### 用 buildTool() 创建

```tsx
// GrepTool.ts（简化版）
export const GrepTool = buildTool({
  name: 'Grep',

  // 输入格式（用 Zod 定义）
  get inputSchema() {
    return z.strictObject({
      pattern: z.string(),           // 正则表达式
      path: z.string().optional(),   // 搜索路径
      glob: z.string().optional(),   // 文件过滤
      output_mode: z.enum(['content', 'files_with_matches', 'count']).optional(),
      // ... 更多参数
    })
  },

  // 核心执行
  async call(args, context) {
    const results = await ripGrep(args)   // 调用 ripgrep
    return { data: formatResults(results) }
  },

  // 安全属性
  isConcurrencySafe: () => true,   // 只读操作，可以并行
  isReadOnly: () => true,          // 不修改任何文件
})
```

### buildTool() 工厂函数的秘密

所有 42 个工具都通过 `buildTool()` 创建。这个函数的核心作用是**填充安全默认值**：

```tsx
const TOOL_DEFAULTS = {
  isConcurrencySafe: () => false,  // 默认不能并行（安全优先）
  isReadOnly: () => false,         // 默认假设会写入
  isDestructive: () => false,      // 默认无破坏性
  checkPermissions: () => 'allow', // 默认允许
}
```

> **设计哲学：fail-closed**。不声明并发安全就默认不安全，不声明只读就默认可写。忘写一个属性，最坏的结果是工具执行慢一点（不并行），而不是出安全问题。

## 3.2c Tool 接口全貌

Tool 接口有 40+ 个字段和方法，按职责分成六类：

```
┌─────────────────────────────────────────────────────┐
│                    Tool 接口                         │
│                                                     │
│  ┌─── 身份 ───┐  ┌─── 数据契约 ───┐               │
│  │ name       │  │ inputSchema    │               │
│  │ aliases    │  │ outputSchema   │               │
│  │ searchHint │  │                │               │
│  └────────────┘  └────────────────┘               │
│                                                     │
│  ┌─── 核心能力 ───┐  ┌─── 安全属性 ─────────────┐ │
│  │ call()         │  │ isConcurrencySafe()       │ │
│  │ description()  │  │ isReadOnly()              │ │
│  │ prompt()       │  │ isDestructive()           │ │
│  │                │  │ checkPermissions()        │ │
│  │                │  │ validateInput()           │ │
│  └────────────────┘  └──────────────────────────┘ │
│                                                     │
│  ┌─── UI 渲染 ────────────────┐                    │
│  │ renderToolUseMessage()       │  正在执行...       │
│  │ renderToolResultMessage()    │  显示结果          │
│  │ renderToolUseRejectedMessage()│  权限被拒         │
│  │ renderToolUseErrorMessage()  │  出错了           │
│  │ renderToolUseProgressMessage()│ 实时进度          │
│  └──────────────────────────────┘                  │
│                                                     │
│  ┌─── 元数据 ──────────────────────┐               │
│  │ maxResultSizeChars   结果大小上限 │               │
│  │ getToolUseSummary()  简短摘要    │               │
│  │ isSearchOrReadCommand()  分类标记│               │
│  │ extractSearchText()  搜索索引    │               │
│  └─────────────────────────────────┘               │
└─────────────────────────────────────────────────────┘
```

### 工具的完整生命周期

为什么需要这么多方法？因为一个工具的生命周期不仅仅是"执行"：

```
Claude 说要用某个工具
     │
     ▼
  ① validateInput()        输入合法吗？
     │
     ▼
  ② checkPermissions()     需要用户授权吗？
     │ ├── allow → 继续
     │ ├── deny  → renderToolUseRejectedMessage()
     │ └── ask   → description() 展示给用户看，等用户决定
     │
     ▼
  ③ call()                 执行！
     │ ├── onProgress()  → renderToolUseProgressMessage() 实时更新
     │ └── 出错          → renderToolUseErrorMessage()
     │
     ▼
  ④ renderToolResultMessage()   显示结果给用户
     │
     ▼
  ⑤ mapToolResultToToolResultBlockParam()   结果转成 API 格式发回给 Claude
     │
     ▼
  ⑥ extractSearchText()   提取关键词，用于会话搜索索引
```

每个方法对应生命周期中的一个阶段。工具只需要实现它关心的阶段，其余用 `buildTool()` 的默认值。

> **关注点分离到了方法级别**：权限系统只调 `checkPermissions()`，不关心执行细节；UI 层只调 `renderXxx()`，不关心权限逻辑；引擎只调 `call()`，不关心怎么渲染。改权限逻辑不影响 UI，改 UI 不影响执行。

> **下一节**：[3.3-3.4 注册与权限](./32-registration-and-permissions.md)
