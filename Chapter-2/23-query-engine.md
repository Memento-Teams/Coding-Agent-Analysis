# 2.3 QueryEngine.ts — 引擎的外壳

> QueryEngine 是用户（或 SDK）与核心循环之间的中间层。它负责"准备工作"，然后把真正的活交给 `query.ts`。

## 定位

```
用户 / SDK / IDE
      │
      ▼
  QueryEngine          ← 准备工作（System Prompt、Memory、工具列表）
      │
      ▼
    query()            ← 真正的循环引擎
      │
      ▼
  Claude API + Tools   ← 实际的 AI 交互和工具执行
```

QueryEngine 的核心方法签名：

```tsx
async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean },
): AsyncGenerator<SDKMessage, void, unknown> {
  // ...
}
```

注意这里的 `async *`——这是一个**异步生成器**。它不是一次性返回结果，而是一条一条地吐出消息。

## 为什么要用生成器？

有时候一个 API 的**返回类型**比它的业务逻辑更重要。因为返回类型决定了：用户在等待时看见什么、你能不能优雅地停下来、以及你能不能把流程写得像一条直线。

### 方案 A：Promise（一次性返回）

```tsx
const result = await engine.submitMessage("修 bug")
// 用户等了 30 秒...什么都看不到...
// 30 秒后突然全部出现
console.log(result)
```

问题：用户盯着空白屏幕 30 秒，不知道 Claude 在干什么。

### 方案 B：回调（事件驱动）

```tsx
engine.submitMessage("修 bug", {
  onText: (text) => { /* 显示文字 */ },
  onToolStart: (tool) => { /* 显示"正在读文件..." */ },
  onToolDone: (result) => { /* 显示结果 */ },
  onDone: () => { /* 结束 */ },
  onError: (err) => { /* 报错 */ },
})

// 想中途停下来？没有好办法
// 想等某个特定工具完成再做下一步？很难控制
```

问题：流程控制很乱，想"中途停下"或"等某个结果再继续"很难写。

### 方案 C：生成器（Claude Code 的选择）

```tsx
for await (const message of engine.submitMessage("修 bug")) {
  // 每产生一条就立刻拿到，实时展示
  render(message)

  if (userPressedCtrlC) {
    break // 一个 break 就能停，干净利落
  }

  if (message.type === "某个我关心的结果") {
    // 可以根据内容决定是否继续
    doSomething()
  }
}
```

### 三种方案对比

|  | **Promise** | **回调** | **生成器** |
|---|---|---|---|
| 流式输出 | 一次性返回 | 事件驱动 | yield 逐条吐出 |
| 中途停止 | 无法停止 | 需要额外状态 | break 一行搞定 |
| 流程控制 | 简单直观 | 回调地狱 | 线性流程 |
| 代码可读性 | 简单 | 嵌套深 | for-await 平铺直叙 |

## 生成器的本质：轮流执行

生成器能同时解决两个问题，因为它的本质是：**调用方和生成方轮流执行**。

```
生成器内部（query.ts）               调用方（UI）
    │                                   │
    ├── yield 消息1 ───────────────────> │ 收到消息1，显示给用户
    │   （暂停，等调用方要下一条）        │ 处理完了，要下一条
    │ <───────────────────────────────── │
    ├── yield 消息2 ───────────────────> │ 收到消息2，显示给用户
    │   （暂停）                         │ 用户按了 Ctrl+C → break
    │                                   │ 生成器被销毁，循环结束
    ×                                   ×
```

关键在 `yield` 这个词——它的意思是 **"交出控制权"**：

- 生成器执行到 `yield`，暂停自己，把数据交给调用方 → **流式输出**
- 调用方决定要不要继续（继续 `for` 循环 or `break`） → **流程控制**

> **下一节**：[2.4 query.ts — 引擎的心脏](./24-query-loop.md)
