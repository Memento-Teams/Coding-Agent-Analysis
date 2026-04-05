# 6.1-6.2 文件与渲染管线总览

> 本节列出终端 UI 涉及的所有文件，并展示从 React 组件到 ANSI 转义序列的完整渲染管线。

## 6.1 涉及的文件

| 文件 | 职责 | 大小 |
|------|------|------|
| `src/ink/reconciler.ts` | React ↔ Ink DOM 桥接，协调器实现 | 350+ 行 |
| `src/ink/dom.ts` | Ink DOM 节点类型，脏标记，滚动状态 | 484 行 |
| `src/ink/layout/yoga.ts` | Yoga (WASM Flexbox) 适配层 | ~200 行 |
| `src/ink/layout/node.ts` | 布局节点接口 | ~100 行 |
| `src/ink/render-node-to-output.ts` | DOM → 字符缓冲区（绘制引擎） | 1,462 行 |
| `src/ink/screen.ts` | 终端 Screen Buffer，双缓冲，样式内化 | 1,486 行 |
| `src/ink/output.ts` | Buffer → ANSI 字符串，字形聚类 | 797 行 |
| `src/ink/log-update.ts` | 增量差分渲染，硬件滚动优化 | 300+ 行 |
| `src/ink/parse-keypress.ts` | 键盘事件解析，修饰键，Kitty 协议 | 801 行 |
| `src/ink/termio/` | ANSI 序列词法分析（CSI/DEC/OSC/SGR） | ~1,600 行 |
| `src/ink/events/` | 事件捕获/冒泡系统 | ~1,100 行 |
| `src/ink/terminal.ts` | 终端能力探测（同步输出、进度条） | ~100 行 |

## 6.2 整体渲染管线

```
React 组件树（<App/> → <MessageList/> → ...）
     │
     ▼ reconciler.ts（React 协调器）
     │  createInstance() / appendChild() / commitUpdate()
     │
Ink DOM 树（DOMElement 节点）
     │  nodeName: 'ink-box' | 'ink-text' | 'ink-root' | ...
     │  attributes: { flexDirection, color, bold, ... }
     │  yogaNode: LayoutNode
     │  dirty: boolean
     │
     ▼ layout/yoga.ts（Yoga WASM Flexbox 布局引擎）
     │  calculateLayout(availableWidth)
     │  getComputedTop() / getComputedLeft() / getComputedWidth() / getComputedHeight()
     │
布局结果（每个节点的 x, y, width, height）
     │
     ▼ render-node-to-output.ts（绘制引擎，1,462 行）
     │  squashTextNodesToSegments() — 合并相邻文本节点
     │  renderBorder() — Unicode 边框字符
     │  ScrollBox 三遍渲染（子节点 → DECSTBM 硬件滚动 → 绝对定位修补）
     │
Output Buffer（字符 + 样式 二维矩阵）
     │
     ▼ screen.ts（Screen Buffer，双缓冲）
     │  CharPool: 字符串内化（space=0, 常用 ASCII 直接查表）
     │  StylePool: ANSI 代码内化（缓存样式转换字符串）
     │  HyperlinkPool: OSC 8 URL 内化
     │
增量变化（新帧 vs 旧帧 diff）
     │
     ▼ log-update.ts（差分渲染 + 硬件滚动优化）
     │  DECSTBM: CSI r 设置滚动区域，CSI S/T 滚动
     │  Synchronized Output: DEC 2026（防闪烁）
     │
     ▼ output.ts（ANSI 编码器）
     │  字形聚类（grapheme clustering）处理宽字符
     │
ANSI 转义序列 → 终端 stdout
```

> **下一节**：[6.3-6.5 Reconciler、DOM 与 Yoga](./63-reconciler-dom-yoga.md)
