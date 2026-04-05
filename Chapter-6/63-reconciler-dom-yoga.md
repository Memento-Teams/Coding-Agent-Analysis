# 6.3-6.5 Reconciler、Ink DOM 与 Yoga 布局

> React 的渲染后端是可插拔的。浏览器里 `react-dom` 映射到 HTML DOM；Claude Code 里 `reconciler.ts` 映射到 Ink DOM——一个只有 5 种节点类型的极简终端节点树。

## 6.3 reconciler.ts — React 和 Ink 的桥梁

实现了 React Reconciler Host Config 中的 30+ 个回调：

```tsx
const reconciler = ReactReconciler({
  // ── 创建阶段 ──
  createInstance(type, props, container) {
    const node = createDOMElement(type)  // 'ink-box' | 'ink-text' | ...
    node.yogaNode = createLayoutNode()   // 绑定 Yoga 布局节点
    applyProps(node, props)
    return node
  },
  createTextInstance(text) {
    return { nodeName: '#text', nodeValue: text }
  },

  // ── 树操作 ──
  appendChild(parent, child) {
    parent.childNodes.push(child)
    parent.yogaNode?.insertChild(child.yogaNode, parent.childNodes.length - 1)
    markDirty(parent)   // 标记：这个节点及其祖先需要重新布局
  },
  removeChild(parent, child) {
    parent.yogaNode?.removeChild(child.yogaNode)
    parent.childNodes.splice(parent.childNodes.indexOf(child), 1)
    markDirty(parent)
  },

  // ── 提交阶段（React Fiber 工作完成后调用）──
  commitUpdate(node, _, __, ___, newProps) {
    applyProps(node, newProps)
    markDirty(node)
  },
  commitTextUpdate(node, _, newText) {
    node.nodeValue = newText
    markDirty(node.parentNode)
  },

  // ── 性能分析 ──
  recordYogaMs(ms)  { yMs += ms }            // Yoga 布局耗时追踪
  markCommitStart() { commitStart = now() }  // 提交阶段耗时追踪
})
```

> **开放封闭原则（Open-Closed Principle）**：React 的协调算法（Fiber 调度、优先级、diff 算法）**对修改封闭**——你不能也不需要改它。但渲染后端**对扩展开放**——只要实现 Host Config 接口。上层 144 个 React 组件完全不需要知道自己是在终端里运行还是在浏览器里运行。

---

## 6.4 dom.ts — Ink DOM 的最小化设计

```tsx
// 浏览器 DOM：8 种节点类型，100+ 个属性
// Ink DOM：5 种节点类型，够用就行

type InkNodeName =
  | 'ink-root'          // 根容器（全局唯一）
  | 'ink-box'           // 布局容器（对应 <Box/>）
  | 'ink-text'          // 文本节点（对应 <Text/>）
  | 'ink-virtual-text'  // 不占空间的文本（用于覆盖层）
  | 'ink-newline'       // 强制换行

type DOMElement = {
  nodeName: InkNodeName
  attributes: Record<string, unknown>  // flexDirection, color, bold, wrap, ...
  childNodes: DOMNode[]
  yogaNode?: LayoutNode                // Yoga 布局节点
  dirty: boolean                       // 是否需要重新渲染？
  // 滚动相关（ScrollBox 专用）
  scrollTop?: number
  pendingScrollDelta?: number
  scrollHeight?: number
  // 焦点管理（根节点专用）
  focusManager?: FocusManager
  // 事件处理器
  _eventHandlers?: Map<string, Function>
  // 调试用（DEBUG_REPAINTS 模式）
  debugOwnerChain?: string[]           // React 组件调用链
}
```

> **最小化 DOM**：浏览器 DOM 背负着 20 年的历史包袱。Ink DOM 从零设计，只保留终端渲染真正需要的字段。内存占用比 HTMLElement 小 10 倍以上。

---

## 6.5 layout/yoga.ts — Flexbox 布局引擎

Yoga 是 Meta 开发的跨平台 Flexbox 布局引擎（同时用于 React Native 和 Ink），以 WASM 模块运行：

```tsx
type LayoutNode = {
  // 设置样式属性
  setWidth(value: number | 'auto' | `${number}%`): void
  setFlexDirection(value: 'row' | 'column' | 'row-reverse' | 'column-reverse'): void
  setPadding(edge: Edge, value: number): void
  // ...30+ 个 setter

  // 触发布局计算
  calculateLayout(availableWidth: number, availableHeight?: number): void

  // 读取计算结果
  getComputedTop(): number
  getComputedLeft(): number
  getComputedWidth(): number
  getComputedHeight(): number

  // 文本测量（由 Ink 提供回调）
  setMeasureFunc(fn: (width, widthMode, height, heightMode) => { width, height }): void

  // 清理
  freeRecursive(): void
}
```

### 文本测量回调

Yoga 不知道如何计算中文字符的宽度（汉字是 2 字宽），所以 Ink 注入 `setMeasureFunc`，Yoga 在布局时回调询问文本的实际尺寸。这样 Flexbox 布局就能正确处理宽字符。

### Yoga 节点只创建一次

每个 DOMElement 在 `createInstance()` 时创建一个 Yoga 节点，此后复用。只有属性变化（`commitUpdate()`）时才更新属性，不重新分配。每帧调用一次 `calculateLayout()` 触发增量重算。

> **下一节**：[6.6-6.8 绘制引擎、双缓冲与增量渲染](./66-rendering-engine.md)
