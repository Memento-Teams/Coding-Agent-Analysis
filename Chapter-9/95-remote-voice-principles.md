# 9.5-9.7 Remote Session、Voice Mode 与共同设计原则

> 本节覆盖跨机器协作、语音输入，以及所有特色子系统共享的设计原则。

## 9.5 Remote Session — 跨机器协作

`src/remote/` 和 `src/server/` 让多人可以连接同一个 Claude Code Session，或让 Claude Code 连接远程的 Claude.ai Assistant 会话。

### RemoteSessionManager

```tsx
class RemoteSessionManager {
  constructor(
    config: {
      sessionId: string
      getAccessToken: () => string
      orgUuid: string
      viewerOnly?: boolean     // 纯查看模式（不能发消息）
    },
    callbacks: {
      onMessage: (message: SDKMessage) => void
      onPermissionRequest: (request, requestId) => void
      onConnected?: () => void
      onDisconnected?: () => void
      onReconnecting?: () => void
    }
  ) {}

  async connect(): Promise<void>        // WebSocket 订阅实时事件流
  async sendMessage(message: string): Promise<void>  // HTTP POST 发送消息
  async respondToPermission(requestId: string, decision: PermissionDecision): Promise<void>
}
```

**Viewer-only 模式**：当 `viewerOnly: true` 时：
- 禁用 Ctrl+C（不能中断远程 Agent）
- 不更新终端标题
- 连接超时从 60 秒延长到无限（远程 Agent 可能长时间运行）

### 跨会话权限转发

```
远程 Agent 需要权限（如要运行 rm 命令）
  │
  ├── 远程 Agent 发送 permission_request 消息
  ├── RemoteSessionManager 触发 onPermissionRequest 回调
  ├── 本地终端弹出权限对话框
  ├── 用户确认/拒绝
  └── 把决定发回给远程 Agent（via HTTP）
```

即使跨机器，权限审批也必须经过"本地用户"。这保证了安全性：远程 Agent 无法在没有本地用户意识的情况下执行危险操作。

---

## 9.6 Voice Mode — 语音输入

```tsx
// voice/voiceModeEnabled.ts
export function isVoiceModeEnabled(): boolean {
  // 三重检查：
  return (
    feature('VOICE_MODE') &&           // 1. 构建时 Feature Flag
    hasVoiceAuth() &&                  // 2. 需要 Anthropic OAuth Token
    isVoiceGrowthBookEnabled()         // 3. GrowthBook 运行时开关（kill-switch）
  )
}
```

语音模式需要 Anthropic OAuth 认证（不是 API Key），这将它限制在使用 Anthropic 账号的用户。GrowthBook kill-switch 允许在出现问题时无需发布新版本就能关闭语音功能。

**流程**：麦克风录音 → 音频缓冲 → STT API（Whisper 系列）→ 转录文字 → 注入 BaseTextInput → 用户可继续编辑后提交。

---

## 9.7 子系统的共同设计原则

这些特色子系统看起来互不相关，但有一个共同的设计原则：**高内聚、低耦合、可独立测试**。

```
Vim 模式：
  输入：按键序列
  输出：(新文本, 新光标位置, 是否进入插入模式)
  无外部依赖：不需要 AppState、不需要 API 调用
  → 可以用纯函数单元测试

Bridge：
  输入：IDE 指令 / Claude.ai 事件
  输出：AI 回复 / 权限审批请求
  无 UI 依赖：不需要 Ink 渲染器
  → 可以用 mock HTTP 服务器测试

Buddy：
  输入：userId（字符串）
  输出：Companion 对象（外观 + 属性）
  无副作用：纯确定性生成
  → 同一 userId 永远生成相同结果，trivial 测试

MemDir：
  输入：记忆文件目录路径
  输出：格式化的记忆内容（注入 System Prompt 用）
  文件系统操作 + LLM 调用（相关性筛选）
  → 可以 mock 文件系统独立测试
```

> **每个子系统都是"小王国"**
>
> "小王国"的比喻准确描述了这种设计：每个子系统有自己的领土（职责范围）、边境（接口定义）、内政（实现细节）。
>
> 只要接口不变，内政怎么改都不影响邻国。Vim 模式升级到支持更多命令，不需要修改 Bridge 的代码。Bridge 切换通信协议，不需要修改 MemDir 的代码。
>
> 这是模块化设计的终极目标：**修改一个子系统，不需要阅读其他子系统的代码**。

---

> 返回 [Chapter 9 目录](./README.md) | 返回 [Chapter 0 总览](../Chapter-0/README.md)
