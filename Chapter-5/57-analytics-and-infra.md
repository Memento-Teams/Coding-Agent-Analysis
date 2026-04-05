# 5.7-5.12 Analytics、OAuth、LSP、Plugin 与智能后台服务

> 本节覆盖服务层的剩余部分：可观测性、安全认证、代码智能、插件管理，以及几个令人惊喜的"智能后台"服务。

## 5.7 Analytics 服务 — 可观测性

### 双轨导出

```
事件发生（如工具调用）
  │
  ├── logEvent('tool_use', { tool_name: 'Bash', duration_ms: 150 })
  │
  ├── 路由 1：Datadog
  │     非 PII 指标：耗时、成功率、模型分布
  │
  └── 路由 2：First-Party Event Logging
        PII 标记字段：_PROTO_skill_name（用于内部分析）
```

### GrowthBook Feature Gate

```tsx
export function isFeatureEnabled(flag: string): boolean
export function getFeatureValue(flag: string): unknown
```

运行时 Feature Gate，用于灰度发布新功能、A/B 测试、紧急关闭有问题的功能。

**缓存策略**：启动时从磁盘加载上次的值（避免阻塞），后台异步刷新。

### 事件队列模式（Queue-and-Drain）

```tsx
// 事件在 sink 初始化之前就可以记录
logEvent('startup', { ... })  // sink 还没初始化，进队列
// ... 稍后 ...
initSinks()  // sink 初始化，队列中的事件被导出
```

> 如果要求 sink 初始化后才能记录事件，就会丢失启动阶段的关键遥测。Queue-and-Drain 模式让事件记录和 sink 初始化**解耦**——想记就记，sink 准备好后自然排空。

---

## 5.8 OAuth 服务 — 安全认证

```
用户首次登录
  │
  ├── 生成 PKCE 验证器（crypto.ts）
  │     code_verifier = random(43 chars)
  │     code_challenge = SHA256(code_verifier)  // base64url
  │
  ├── 打开浏览器 → Claude.ai 授权页面
  │     URL 包含 code_challenge + state 参数
  │
  ├── 启动本地 HTTP 服务器等待回调（auth-code-listener.ts）
  │     监听 localhost:${random_port}/callback
  │
  ├── 用户在浏览器中授权
  │
  ├── 回调到达本地服务器
  │     ├── 验证 state 参数（防 CSRF）
  │     ├── 用 authorization_code + code_verifier 换 Token
  │     └── 存入 Keychain / Windows Credential Manager
  │
  └── 后续请求用 Token 认证（自动刷新过期 Token）
```

> **为什么不用简单的 API Key？** Claude Code 支持 API Key 和 OAuth 两种方式。OAuth 更安全（Token 有过期时间、可撤销），且支持组织级权限控制。但对开发者来说 API Key 更方便。两种方式共存，用户选择。

---

## 5.9 LSP 服务 — 代码智能

```
Claude Code
  │
  └── LSPServerManager（单例）
        │
        ├── LSPServerInstance（TypeScript）
        │     └── LSPClient → tsserver
        │
        ├── LSPServerInstance（Python）
        │     └── LSPClient → pyright
        │
        └── LSPDiagnosticRegistry
              └── 聚合所有语言的诊断信息
```

### Generation Counter 防竞态

```tsx
class LSPManager {
  private generation = 0  // 每次重启递增

  async initialize() {
    const gen = ++this.generation
    // ... 初始化 ...
    if (gen !== this.generation) return  // 被新的初始化覆盖了，丢弃
  }
}
```

> 如果用户快速切换项目，可能触发多次 LSP 初始化。没有 generation counter，先发起的初始化可能在后发起的之后完成，导致状态错乱。每次初始化递增计数器，完成时检查是否还是自己的 generation——不是就丢弃结果。

---

## 5.10 Plugin 服务 — 插件管理

```tsx
// pluginOperations.ts — 纯函数，无副作用
export function installPlugin(name, scope): PluginResult
export function uninstallPlugin(name, scope): PluginResult
export function enablePlugin(name, scope): PluginResult
export function disablePlugin(name, scope): PluginResult
export function updatePlugin(name, scope): PluginResult
```

**作用域**：

| Scope | 位置 | 共享范围 |
|-------|------|---------|
| project | .claude/plugins/ | 通过 VCS 共享 |
| managed | Enterprise 管控 | 组织级别 |

**依赖管理**：卸载时检查反向依赖——如果其他插件依赖于要卸载的插件，拒绝操作。

> **纯函数设计**：`pluginOperations.ts` 中的所有函数都是纯函数——不打印日志、不修改全局状态，只返回结果对象。调用方决定如何展示结果。同一套逻辑可以在 CLI、SDK、测试中复用。

---

## 5.11 智能后台服务

### 5.11.1 extractMemories — 自动记忆提取

```
每次对话结束时（stop hook 触发）：
  │
  ├── Fork 子 Agent（共享 Prompt Cache）
  ├── 扫描最近的消息
  ├── 提取值得记住的信息
  │     ├── user 类型：用户偏好、角色
  │     ├── feedback 类型：纠正和确认
  │     ├── project 类型：项目状态、决策
  │     └── reference 类型：外部资源指针
  └── 写入 ~/.claude/memory/
```

### 5.11.2 autoDream — 记忆整合（"做梦"）

```
Claude Code 空闲时（通过 CronTool 调度）：
  │
  ├── 获取整合锁（防止并发做梦）
  ├── Fork 子 Agent
  ├── 四个阶段：
  │     ① Orient    — 审视现有记忆
  │     ② Gather    — 从日志中收集新信号
  │     ③ Consolidate — 合并、更新记忆文件
  │     ④ Prune     — 修剪索引，保持 <200 行
  └── 释放锁
```

### 5.11.3 MagicDocs — 自动文档更新

```
代码中有 "# MAGIC DOC: " 标题的文档会被自动更新：
  │
  ├── 检测到代码变更涉及文档化的模块
  ├── Fork 子 Agent
  ├── 用 "HIGH SIGNAL ONLY" 原则更新文档
  │     ├── 只写：架构、入口、WHY/HOW/WHERE
  │     ├── 不写：显而易见的代码、详尽的 API
  │     └── 要求：TERSE（简洁）
  └── 原地更新文档，移除过时内容
```

### 5.11.4 tips — 上下文提示

```tsx
export function getTipForContext(context: TipContext): Tip | null
// 根据当前场景返回相关提示
// 有冷却期——同一个提示不会连续出现
```

---

## 5.12 基础设施服务

### 5.12.1 policyLimits — 组织策略

```tsx
export async function isPolicyAllowed(feature: string): Promise<boolean>
// 检查组织是否允许某功能
// 例如：'allow_remote_control'、'allow_web_search'
```

**ETag 缓存**：策略不经常变化，用 ETag 避免重复下载。

### 5.12.2 tokenEstimation — Token 计数

```tsx
export function tokenCountWithEstimation(messages: Message[]): number
// 估算消息列表的 Token 数量
// 支持 thinking blocks、图片 Token 估算
// 移除 tool-search 相关字段（不计入 Token）
```

**三个时机**的 Token 追踪：

1. **Pre-request**：`tokenCountWithEstimation()` 估算，决定是否需要压缩
2. **During streaming**：从 API 流式事件中实时读取 Token 数
3. **Post-request**：从 API 响应的 `usage` 字段中获取精确数字

### 5.12.3 notifier — 终端通知

```
任务完成通知：
  │
  ├── Terminal Bell（\a）          ← 所有终端
  ├── iTerm2 通知 API             ← iTerm2 专用
  ├── Kitty 通知协议               ← Kitty 专用
  └── Ghostty 通知协议             ← Ghostty 专用
```

### 5.12.4 rateLimitMessages — 限流消息

集中管理所有限流消息的生成，避免在多处重复编写相同的用户提示。

---

> 返回 [Chapter 5 目录](./README.md) | 返回 [Chapter 0 总览](../Chapter-0/README.md)
