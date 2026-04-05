# 6.6-6.8 绘制引擎、双缓冲与增量渲染

> 从 Ink DOM 树到终端上的像素（字符），中间经过了 1,462 行的绘制引擎、三级对象内化的 Screen Buffer、以及利用硬件滚动的增量渲染。

## 6.6 render-node-to-output.ts — 1,462 行的绘制引擎

把 Ink DOM 树遍历一遍，把每个节点的内容"画"到一个字符矩阵里。

### 核心遍历

```tsx
function renderNode(
  node: DOMElement,
  x: number, y: number,
  maxWidth: number, maxHeight: number,
  output: Output,
  options: RenderOptions
): void {
  // 1. 从 Yoga 读取布局结果
  const nodeX = x + node.yogaNode.getComputedLeft()
  const nodeY = y + node.yogaNode.getComputedTop()
  const width = node.yogaNode.getComputedWidth()

  // 2. 处理边框（如果有）
  if (node.attributes.borderStyle) {
    renderBorder(node, nodeX, nodeY, width, height, output)
  }

  // 3. 递归渲染子节点
  for (const child of node.childNodes) {
    renderNode(child, nodeX + paddingLeft, nodeY + paddingTop, ...)
  }
}
```

### ScrollBox 的三遍渲染

这是绘制引擎里最复杂的部分：

```
ScrollBox 渲染流程：
  遍历 1：正常渲染所有子节点到 offscreen buffer
  遍历 2：
    如果终端支持 DECSTBM（硬件滚动）→ 发送 scroll region + scroll 命令
    否则 → 把 buffer 中对应行贴到 output（软件 blit）
  遍历 3：修补被 blit 覆盖的绝对定位元素（overlays、badges）
```

### 硬件滚动（DECSTBM）

当用户滚动对话历史时，朴素的实现是重新绘制整个可见区域（50+ 行），发送 2,000+ 个字符。DECSTBM 的做法是：

```
ESC[5;48r   → 设置滚动区域：第 5 行到第 48 行
ESC[3S      → 硬件滚动：向上 3 行（终端自己滚，不用我们重绘）
```

2 个转义序列 vs 2,000+ 个字符。在大型对话历史中，**滚动性能提升约 10 倍**。

支持 DECSTBM 的终端：iTerm、WezTerm、Ghostty、Kitty、foot。不支持的（tmux、部分旧终端）退回软件 blit。

### 文本压扁优化

```tsx
// 相邻的文本节点合并为一个带样式的 segment 数组
// 避免逐字符渲染（每个字符一次 output.write() 调用）
const segments = squashTextNodesToSegments(textNode)
// segments: [{ text: "Hello ", style: { bold: true } }, { text: "World", style: {} }]
```

---

## 6.7 screen.ts — 双缓冲与三级内化

Screen Buffer 把抽象的"字符矩阵"转换成实际的终端输出序列，并且只输出**变化的部分**。

### 三级对象内化（Interning）

内化是一种技术：把频繁重复的对象存入全局池，用小整数 ID 代替对象引用，既省内存又加速相等比较。

**CharPool — 字符内化**：

```tsx
class CharPool {
  // 快速路径：ASCII 字符直接用数组查（避免 Map.get 的哈希开销）
  private asciiCache = new Uint32Array(128)

  // 慢速路径：非 ASCII（汉字、emoji）用 Map
  private pool = new Map<string, number>()

  intern(char: string): number {
    if (char.length === 1 && char.charCodeAt(0) < 128) {
      return this.asciiCache[char.charCodeAt(0)]  // ASCII 快速路径
    }
    if (!this.pool.has(char)) this.pool.set(char, this.pool.size + 128)
    return this.pool.get(char)!
  }
}
```

特殊值：`0 = 空格`（最常见），`1 = 空字符串`（宽字符的"尾"格）。

**StylePool — 样式内化**：

每个 Cell 存样式 ID（4 字节整数），不是样式对象。关键优化——预计算样式"转换字符串"：

```tsx
class StylePool {
  // styleA → styleB 的 ANSI 转换序列（预计算并缓存）
  getTransition(fromStyle: number, toStyle: number): string {
    const key = (fromStyle << 16) | toStyle
    if (!this.transitions.has(key)) {
      this.transitions.set(key, computeANSITransition(fromStyle, toStyle))
    }
    return this.transitions.get(key)!
  }
}
```

**HyperlinkPool — 超链接内化**：对话中可能出现几十个相同的 URL（如源码引用），内化后每个唯一 URL 只存一份。

### 双缓冲 Diff

```
上一帧（frontBuffer）：
  行 3: ["H","e","l","l","o"," ","W","o","r","l","d"]

本帧（backBuffer）：
  行 3: ["H","e","l","l","o"," ","W","o","r","l","d","!"]  ← 新增 "!"

差分输出：
  ESC[3;12H  ← 移动光标到第 3 行第 12 列
  !          ← 只写这一个字符
```

> **三级内化的意义**：200×80 终端有 16,000 个 Cell。用整数 ID 代替对象引用后，Cell 大小从 ~200 字节降到 ~12 字节（char ID 4 + style ID 4 + hyperlink ID 4）。Diff 变成整数比较（1 纳秒 vs 深比较的微秒级开销）。

---

## 6.8 log-update.ts — 增量渲染与防闪烁

### 同步输出（DEC 2026 Synchronized Output）

某些终端在收到部分输出时会立即刷新，导致用户看到"半帧"（闪烁）。DEC 2026 解决这个问题：

```
ESC[?2026h  ← 开始同步输出（终端缓冲，不立即刷新）
...很多输出内容...
ESC[?2026l  ← 结束同步输出（终端一次性刷新，原子操作）
```

支持的终端：iTerm2、WezTerm、Ghostty、Kitty、foot、contour、VSCode Terminal。不支持的：tmux（分块机制会破坏原子性）。

### 渲染性能数字

| 阶段 | 典型耗时 |
|------|---------|
| React reconciliation | 1–3ms |
| Yoga layout | 1–5ms |
| render-node-to-output | 3–8ms |
| screen diff + output | 2–5ms |
| **总计** | **~10–20ms/帧（50–100 FPS）** |

密集更新（如工具执行时）靠 Dirty Flag 跳过未变节点，总渲染量和变化量成正比而非和总节点数成正比。

> **下一节**：[6.9-6.10 键盘事件与组件库](./69-events-and-components.md)
