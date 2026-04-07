# 3.3-3.4 工具注册与权限系统

> 42 个工具不是全部一股脑加载的——它们经过三层过滤才最终进入 Claude 的工具池。权限系统则提供四层防御，确保危险操作必须经过人类确认。

---

## 3.3 工具注册 — tools.ts 的三层过滤

### 第一层：基础注册（哪些工具存在？）

`getAllBaseTools()` 返回所有工具，用三种策略控制：

```tsx
// 策略 1：始终可用
import { FileReadTool } from './tools/FileReadTool/FileReadTool.js'
import { BashTool } from './tools/BashTool/BashTool.js'
// ... 直接 import，一定存在

// 策略 2：Feature Flag（构建时消除）
const SleepTool = feature('PROACTIVE')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
// 外部构建中 SleepTool 的代码根本不存在于 bundle 里

// 策略 3：惰性 Getter（打破循环依赖）
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool
// 延迟到首次使用时才加载
```

### 第二层：模式过滤（当前模式下哪些可用？）

`getTools(permissionContext)` 根据运行模式过滤：

```
完整模式（正常使用）        → 42 个工具
简单模式（--bare）          → 只保留 Bash、FileRead、FileEdit
Coordinator 模式           → 只保留 AgentTool、TaskStopTool、SendMessage
REPL 模式                 → 隐藏被 REPL 包装的底层工具
```

### 第三层：权限过滤（用户禁止了哪些？）

`filterToolsByDenyRules()` 移除用户明确禁止的工具：

```json
// 用户的 settings.json
{
  "permissions": {
    "deny": ["WebSearch", "WebFetch"]   // 禁止联网
  }
}
```

### 最终组装

```tsx
function assembleToolPool(permissionContext, mcpTools) {
  const builtInTools = getTools(permissionContext)   // 42 个内建工具（已过滤）
  const allTools = [...builtInTools, ...mcpTools]    // + MCP 工具
  return deduplicateByName(allTools)                 // 去重（内建优先）
}
```

> **Prompt Cache 稳定性**：工具列表的顺序会影响 Claude API 的 Prompt Cache 命中率。`assembleToolPool()` 对内建工具和 MCP 工具**分别排序**，内建工具作为连续前缀。这保证了内建工具的顺序永远一致 → 缓存命中；MCP 工具加在后面，不影响前缀 → 缓存仍然命中。

---

## 3.4 权限系统 — 四层防御

```
Claude 说要用某个工具
     │
     ▼
  ① validateInput()         输入合法吗？（fail-closed）
     │
     ▼
  ② checkPermissions()      工具自己判断安不安全
     │ → allow / deny / ask
     ▼
  ③ 规则匹配                用户预设的 always_allow / always_deny
     │
     ▼
  ④ 用户交互                弹出权限对话框
```

### 权限模式

| 模式 | 行为 |
|------|------|
| `default` | 每次都问 |
| `auto` | 大部分自动批准 |
| `plan` | 只允许只读 |
| `bypass` | 跳过所有检查 |
| `deny` | 拒绝一切 |

> **下一节**：[3.5-3.7 文件、搜索与命令执行](./33-file-and-bash-tools.md)
