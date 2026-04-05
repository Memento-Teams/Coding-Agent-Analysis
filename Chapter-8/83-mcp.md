# 8.3 第三层：MCP Servers — 连通异世界的任意门

> MCP 不持有工具，只创造连接。它的本质是一扇跨越两地的"任意门"，只定义了一套严密的 RPC 规范。通过它，Claude 能够不受宿主语言和电脑性能的物理约束，把重型企业应用与异构服务器直接拉到对话框面前。

**核心定位**：彻底跨越宿主语言环境、以巨量代码构建的独立通信层。

仅 `src/services/mcp/` 一个目录就盘踞了 **23 个核心服务文件，纯逻辑代码量累计突破 400 KB**。

## MCP 配置：门把手上的坐标

```json
"mcpServers": {
  // 深核潜入型：基于 Stdio 管道直连本地进程
  "enterprise-db-gateway": {
    "type": "stdio",
    "command": "python",
    "args": ["/var/cluster/db.py"],
    "env": { "DB_SECRET": "${userConfig.API_TOKEN}" }  // 与 Plugin Keychain 无缝衔接
  },
  // 长连心跳型：全双工跨栈网络
  "remote-cloud-agent": {
    "type": "ws",
    "url": "wss://intranet.ag/mcp",
    "headers": { "X-Claude-Code-Ide-Authorization": "Bearer ..." }
  }
}
```

**底层协议大满贯**：`stdio`、`sse`、`sse-ide`、`http`、`ws`、`ws-ide`、`sdk` — 抹平本地与网络的次元壁。

## 宿主侧基建解剖

```
src/services/mcp/
├── client.ts                  # [绝对中枢] 119KB/3500行，全盘接管协议降级与 RPC 拦截
├── auth.ts                    # [安保防线] 88KB，OAuth2 握手、Token 静默刷新、401 拦截
├── config.ts                  # [航线调度] 50KB，多级配置合并与环境变量注射
├── useManageMCPConnections.ts # [生命探针] 44KB，实时监测心跳中断与重连熔断
├── channelNotification.ts     # [逆向神经] 异步事件反向推送接收
└── xaa.ts / xaaIdpLogin.ts    # [企业哨兵] 企业 IdP 身份验证穿透
```

## 切面一：全链路信道大满贯

`client.ts` 的巨型 `if...else if...` 瀑布流完成了一次**概念抽象与解耦**：

```tsx
if (serverRef.type === 'sse') {
  // SSE: 包裹 Timeout + Step-Up 自动探测双层铠甲
  fetch: wrapFetchWithTimeout(
    wrapFetchWithStepUpDetection(createFetchWithInit(), authProvider),
  ),
} else if (serverRef.type === 'sse-ide') {
  // IDE 信任域内免密打通
} else if (serverRef.type === 'ws' || serverRef.type === 'ws-ide') {
  // WebSocket: TLS 防弹衣 + Bun/Node 运行时自动适配
  const tlsOptions = getWebSocketTLSOptions()
  if (typeof Bun !== 'undefined') {
    wsClient = new globalThis.WebSocket(serverRef.url, ...)  // Bun 原生极速
  } else {
    // Node.js 代理兼容路线
  }
}
```

> 无论是 Stdio 的遗留 C++ 节点、需要穿透代理的 SSE 微服务，还是要抹平 Node/Bun 差异的 TLS WebSocket——**一切物理噪音都被无情吞噬**。输出给 AI 的，只有纯净同构的 JSON-RPC 标准调用。

## 切面二：幽灵般的静默重连

当远端 MCP Server 因发版重启或 K8s 意外杀掉时：

```tsx
export function isMcpSessionExpiredError(error: Error): boolean {
  const httpStatus = 'code' in error ? error.code : undefined
  if (httpStatus !== 404) return false  // 第一重滤网：只捕 404
  // 心跳甄别：精准扫描 JSON-RPC 死亡代号 -32001
  return (
    error.message.includes('"code":-32001') ||
    error.message.includes('"code": -32001')
  )
}
// 捕获后：悄无声息清空死链 → 指数退避重连 → 重新分配 Session ID
// 绝不向 AI 和用户暴露任何一次连接失败
```

> **优雅注水 (Graceful Rehydration)**：用极其庞大的本地恢复代码，在风暴废墟中强行续接断桥，换来一个"永远不曾断线"的幻觉护盾。

## 切面三：401 时空倒刺与巨物过载截断

### 防线 A：401 安保时空倒刺

```tsx
const doRequest = async () => {
  await checkAndRefreshOAuthTokenIfNeeded()  // 每次开火前强制验活 Token
  // ...
  if (response.status !== 401) return response
  // 跨维挂起！唤醒浏览器弹窗，在用户授权前整个推理栈被定身
  const tokenChanged = await handleOAuth401Error(sentToken)
  // 换血成功后，原本流产的请求原封不动重新发射
}
```

### 防线 B：体积压强探针

```tsx
const result = await this.client.callTool({ name, arguments: callArgs })
if (mcpContentNeedsTruncation(result.content)) {
  result.content = truncateMcpContentIfNeeded(result.content)  // 超标字符直接砍断
  await persistBinaryContent(result)  // 巨大 Base64/Dump 强储到物理硬盘，只给 AI 短链接
}
```

> MCP 的 `client.ts` 不是单薄的网络转发器——它是一座**堆满重装火力、兼顾断点续传、把揽洗源加密、自带防洪海关查验的终极地堡**。

> **下一节**：[8.4 三层总结](./84-summary.md)
