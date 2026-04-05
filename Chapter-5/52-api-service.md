# 5.2 API 服务 — 和 Claude 对话的管道

> 这是最核心的服务——所有与 Claude API 的通信都经过这里。从 SDK 客户端创建到请求编排、智能重试、错误分类、日志记录，一条龙完成。

## 5.2.1 client.ts — SDK 客户端工厂

```tsx
// 支持 4 种 API 提供商
function createClient(provider: 'direct' | 'bedrock' | 'foundry' | 'vertex'): AnthropicClient
```

| 提供商 | SDK | 认证方式 |
|--------|-----|---------|
| Direct | `@anthropic-ai/sdk` | API Key / OAuth |
| AWS Bedrock | `@anthropic-ai/bedrock-sdk` | AWS IAM |
| Google Vertex | `@anthropic-ai/vertex-sdk` | Google Service Account |

**关键设计**：懒加载 SDK（每个提供商的 SDK 只在首次使用时 `require()`）、自动配置 HTTP/HTTPS 代理、每个请求附带 `x-client-request-id` 头用于追踪。

---

## 5.2.2 claude.ts — 请求编排（3,600+ 行）

这是 API 层最大的文件，负责组装和发送每个请求：

```
claude.ts 的工作：
  │
  ├── 1. 构建请求
  │     ├── System Prompt（静态 + 动态）
  │     ├── 消息历史（可能已压缩）
  │     ├── 工具定义（42 个工具的 JSON Schema）
  │     ├── 模型选择（Opus/Sonnet/Haiku）
  │     └── 配置（thinking、effort level、max tokens）
  │
  ├── 2. 发送请求（流式）
  │     ├── 调用 withRetry() 包装
  │     └── 逐 chunk 处理响应
  │
  ├── 3. 处理响应
  │     ├── 提取 tool_use 块
  │     ├── 统计 Token 用量
  │     ├── 处理 Prompt Cache 指标
  │     └── 触发日志记录
  │
  └── 4. 错误恢复
        ├── Prompt-Too-Long → 压缩后重试
        ├── Rate Limit → withRetry 处理
        └── Model Fallback → 降级到备用模型
```

---

## 5.2.3 withRetry.ts — 智能重试（823 行）

这是整个 API 层最精巧的文件。注意它是一个**异步生成器**——重试过程中可以 yield 系统消息（如"正在重试..."），让 UI 保持更新。

```tsx
async function* withRetry(
  fn: () => Promise<Response>,
  options: RetryOptions
): AsyncGenerator<SystemMessage | Response>
```

### 重试策略

```
请求失败
  │
  ├── 错误类型分析
  │     ├── 401 Unauthorized → 不重试，让用户重新登录
  │     ├── 429 Rate Limited → 重试 + 触发 Fast Mode 冷却
  │     ├── 529 Overloaded   → 特殊处理
  │     ├── 500 Server Error → 重试
  │     ├── Network Error    → 重试
  │     └── 其他             → 检查 x-should-retry 头
  │
  ├── 退避计算
  │     BASE_DELAY = 500ms
  │     delay = min(BASE_DELAY * 2^attempt + random_jitter, MAX_DELAY)
  │     MAX_DELAY = 32s（正常）或 6h（持久模式）
  │
  └── 重试或放弃
        ├── 可重试 → yield "正在重试..." + sleep(delay) + 重试
        └── 不可重试 → throw 错误
```

### 529 Overloaded 的特殊处理

```
529 错误（Claude API 过载）
  ├── 前台会话：正常退避重试，连续 3+ 次 → 触发模型降级（如 Opus → Sonnet）
  └── 后台/无人值守会话：启用"持久重试模式"，最大退避时间从 32s 提升到 6 小时
```

> **前台 vs 后台的差异化重试**：用户在屏幕前等待时，32 秒已经是极限。但后台 Agent 没人看——等 6 小时也没关系，关键是最终完成。同一个重试逻辑，根据**使用场景**选择不同策略。

### Fast Mode 冷却

收到 429 时暂时禁用 Fast Mode，冷却期从 retry-after 头提取，解除后自动恢复。Fast Mode 使用更快但有更严格限额的端点，被限流时切回普通模式，避免浪费等待时间。

---

## 5.2.4 errors.ts — 错误分类（41KB）

把 API 错误分成精确的类别，让 withRetry 和 QueryEngine 做出正确决策：

```tsx
type APIErrorCategory =
  | 'auth'             // 认证失败（不重试）
  | 'rate_limit'       // 限流（重试 + 冷却）
  | 'overloaded'       // 服务过载（重试 + 可能降级）
  | 'context_too_long' // 上下文超限（压缩后重试）
  | 'max_output'       // 输出超限（提升限额或多轮恢复）
  | 'transient'        // 瞬时错误（重试）
  | 'user'             // 用户错误（不重试）
  | 'system'           // 系统错误（可能重试）
```

每个类别从 API 响应中提取关键信息：

```
"prompt is too long: 215,000 tokens > 200,000 token limit"
  → category: 'context_too_long'
  → tokenOverage: 15,000
  → tokenLimit: 200,000
```

---

## 5.2.5 logging.ts — 可观测性（789 行）

每次 API 调用记录三个事件：

| 事件 | 时机 | 记录内容 |
|------|------|---------|
| `logAPIQuery` | 发送请求时 | 模型、beta headers、thinking 类型、fast mode |
| `logAPIError` | 错误发生时 | 错误分类、重试次数、client request ID |
| `logAPISuccessAndDuration` | 成功返回时 | Token 用量、成本、缓存命中率、首 Token 延迟 |

**网关检测**：自动检测 LiteLLM、Helicone、Portkey、Cloudflare 等代理网关，在日志中标注。

---

## 5.2.6 usage.ts + bootstrap.ts

- **usage.ts** — 获取 Claude.ai 订阅限额和额外用量状态（5 秒超时，处理过期 Token）
- **bootstrap.ts** — 启动时预取客户端数据和模型选项，用 ETag 缓存避免重复下载，OAuth 过期时自动刷新重试

> **下一节**：[5.3-5.4 MCP 与工具执行](./53-mcp-and-tool-execution.md)
