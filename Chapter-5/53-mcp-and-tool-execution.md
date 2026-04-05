# 5.3-5.4 MCP 服务与工具执行服务

> MCP 让 Claude Code 连接外部世界，工具执行服务则是引擎调用工具的中间层——处理并发、流水线和权限检查。

## 5.3 MCP 服务 — 外部工具服务器

MCP（Model Context Protocol）让 Claude Code 能连接外部工具服务器——比如数据库查询服务、Slack 集成等。

### 5.3.1 架构

```
Claude Code
  │
  ├── MCP Client (client.ts)
  │     │
  │     ├── stdio 传输 ← 本地进程（最常见）
  │     ├── SSE 传输   ← HTTP Server-Sent Events
  │     ├── HTTP 传输  ← REST API
  │     ├── WebSocket  ← 实时通信
  │     └── SDK 传输   ← 嵌入式
  │
  └── 连接多个 MCP Server
        ├── Server A（数据库查询）→ 提供 query_db 工具
        ├── Server B（Slack）     → 提供 send_message 工具
        └── Server C（GitHub）    → 提供 create_issue 工具
```

### 5.3.2 配置（config.ts）

MCP 服务器配置从多个来源合并：

```
配置优先级（从高到低）：
  ① .mcp.json（当前目录）         ← 项目级
  ② ~/.claude/mcp.json            ← 用户级
  ③ Enterprise MCP 配置            ← 企业级
  ④ Claude.ai 代理服务器           ← 平台级
```

支持环境变量展开：

```json
{
  "servers": {
    "my-db": {
      "command": "${HOME}/tools/db-server",
      "args": ["--port", "${DB_PORT}"]
    }
  }
}
```

### 5.3.3 认证（auth.ts）

MCP 服务器可能需要 OAuth 认证：

```
用户首次连接需要认证的 MCP Server
  │
  ├── 生成 McpAuthTool（占位工具）
  ├── Claude 调用 McpAuthTool
  ├── 启动 OAuth 流程
  │     ├── 打开浏览器 → 授权页面
  │     ├── 本地 HTTP 服务器等待回调
  │     └── 收到 Token → 存入 Keychain
  ├── 重新连接 MCP Server
  └── 替换 McpAuthTool 为真正的工具
```

### 5.3.4 类型（types.ts）

```tsx
type MCPTransport = 'stdio' | 'sse' | 'http' | 'ws' | 'sdk' | 'sse-ide' | 'ws-ide'

type MCPConfigScope = 'local' | 'user' | 'project' | 'dynamic' |
                      'enterprise' | 'claudeai' | 'managed'

type ServerCapabilities = {
  tools?: boolean        // 提供工具
  resources?: boolean    // 提供资源（文件、数据）
  prompts?: boolean      // 提供 Prompt 模板
}
```

---

## 5.4 工具执行服务 — 并发编排

### 5.4.1 StreamingToolExecutor — 流水线执行

第 2 章提到过的流水线执行器，这里看它的实现细节：

```tsx
class StreamingToolExecutor {
  // API 还在返回时，就开始执行已发现的工具
  addTool(toolUseBlock: ToolUseBlock): void

  // 获取所有结果（包括已完成的和还在执行的）
  async *getRemainingResults(): AsyncGenerator<ToolResult>
}
```

**并发控制**：

```
API 流式返回了 5 个工具调用：
  [FileRead, FileRead, BashTool, Grep, Grep]

StreamingToolExecutor 的处理：
  ① FileRead 到达 → 立即执行（并发安全）
  ② FileRead 到达 → 立即执行（和 ① 并行）
  ③ BashTool 到达 → 等 ①② 完成后执行（非并发安全）
  ④ Grep 到达    → 等 ③ 完成后执行（并发安全）
  ⑤ Grep 到达    → 和 ④ 并行执行

实际执行顺序：
  ┌──①──┐
  │     │→ ③ → ┌──④──┐
  └──②──┘      └──⑤──┘
  并行            串行   并行
```

**兄弟 AbortController**：如果一个 Bash 命令失败了，可以取消同批次中其他正在运行的命令。

### 5.4.2 toolOrchestration.ts — 批次划分

```tsx
export async function* runTools(
  toolUseBlocks: ToolUseBlock[],
  context: ToolUseContext,
  canUseTool: CanUseToolFn
): AsyncGenerator<ToolResult>
```

**划分算法**：

```
输入：[A(safe), B(safe), C(unsafe), D(safe), E(safe), F(unsafe)]

划分规则：相邻的 safe 合并为一批，unsafe 单独成批

结果：
  Batch 1: [A, B]    → 并行执行
  Batch 2: [C]        → 串行执行
  Batch 3: [D, E]    → 并行执行
  Batch 4: [F]        → 串行执行
```

**最大并发度**：`CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`，默认 10。

### 5.4.3 toolExecution.ts — 单工具执行

```
单个工具的执行流程：
  │
  ├── 解析输入（Zod Schema 校验）
  ├── 权限检查（checkPermissions + canUseTool）
  ├── 执行（tool.call()）
  ├── 包装结果为 tool_result 块
  ├── 记录遥测（耗时、权限决定、分析）
  └── 返回结果
```

> **下一节**：[5.5-5.6 Compact 与 Session Memory](./55-compact-and-memory.md)
