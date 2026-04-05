# 9.1 Vim 模式 — 从零实现的完整状态机

> Claude Code 不依赖任何 Vim 库，完全从头实现了 Vim 的 Normal/Insert 模式、operators、motions、text objects、寄存器、点重复（`.`）、查找重复（`;`/`,`）。

## 目录结构

```
src/vim/
├── types.ts        ← 状态机类型定义（所有状态和转换的"合同"）
├── transitions.ts  ← 状态转换函数（接收按键，返回新状态）
├── motions.ts      ← 光标移动实现（h/j/k/l/w/b/0/$...）
├── operators.ts    ← 操作符实现（d/c/y 的执行逻辑）
└── textObjects.ts  ← 文本对象实现（iw/aw/i"/a"...）
```

## 类型即文档

```tsx
// vim/types.ts — 通过读类型就能知道支持哪些 Vim 功能

type VimState =
  | { mode: 'INSERT'; insertedText: string }      // 插入模式
  | { mode: 'NORMAL'; command: CommandState }     // 普通模式

// 普通模式的子状态机
type CommandState =
  | { type: 'idle' }                              // 等待命令
  | { type: 'count'; digits: string }             // 输入数字（如 "3" in 3dd）
  | { type: 'operator'; op: Operator; count: number }
    // 已输入操作符，等待动作（如 d 后等 w/b/iw/...）
  | { type: 'operatorCount'; op: Operator; count: number; digits: string }
    // d3 ← 操作符后跟数字
  | { type: 'operatorFind'; op: Operator; count: number; find: FindType }
    // df ← 操作符 + f/F/t/T，等待目标字符
  | { type: 'operatorTextObj'; op: Operator; count: number; scope: TextObjScope }
    // di ← 操作符 + i/a，等待文本对象类型
  | { type: 'find'; find: FindType; count: number }  // f/F/t/T 独立使用
  | { type: 'g'; count: number }                     // g 前缀命令（gg、gq...）
  | { type: 'replace'; count: number }               // r ← 等待替换字符
  | { type: 'indent'; dir: '>' | '<'; count: number } // >> / << 缩进

// 跨命令持久化状态（. 重复、寄存器等）
type PersistentState = {
  lastChange: RecordedChange | null    // 供 . 重复上次修改
  lastFind: { type: FindType; char: string } | null  // 供 ; , 重复查找
  register: string                     // 默认寄存器（"未命名寄存器"）
  registerIsLinewise: boolean          // 是否行级复制
}
```

> **类型即文档**：Vim 有几十个命令，状态转换极其复杂。传统实现用一堆 `if-else`，逻辑散落各处。Claude Code 的做法是：先把所有可能的状态都列在 TypeScript 类型里。类型本身就是文档——读 `CommandState` 就知道 Vim 模式支持什么。更重要的是：TypeScript 编译器会在编译时检查所有状态分支是否都被处理。

## 状态机转换示例

```
用户按 "3dw"（删除 3 个词）：

按键 '3'：
  当前：{ mode: 'NORMAL', command: { type: 'idle' } }
  调用：transition(state, '3', ctx)
  结果：{ mode: 'NORMAL', command: { type: 'count', digits: '3' } }

按键 'd'：
  当前：{ command: { type: 'count', digits: '3' } }
  调用：transition(state, 'd', ctx)
  结果：{ command: { type: 'operator', op: 'delete', count: 3 } }
    （数字 "3" 被合并进 count 字段）

按键 'w'：
  当前：{ command: { type: 'operator', op: 'delete', count: 3 } }
  调用：transition(state, 'w', ctx)
  立即执行：executeOperatorMotion('delete', 'word', 3, ctx)
  结果：{ command: { type: 'idle' } }
    （执行完毕，回到空闲）
    lastChange 记录为 { type: 'operator', op: 'delete', motion: 'w', count: 3 }
    （供 . 重复使用）
```

## 操作符（operators.ts）— 纯函数

```tsx
// 所有操作符都是纯函数——接受上下文，返回结果，不直接修改状态
type OperatorContext = {
  cursor: { line: number; col: number }
  text: string
  setText: (text: string) => void       // 修改文本的回调
  setOffset: (offset: number) => void  // 移动光标的回调
  enterInsert: (offset: number) => void // 进入插入模式的回调
  getRegister: () => string
  setRegister: (content: string, linewise: boolean) => void
  recordChange: (change: RecordedChange) => void
}

// d2w：删除 2 个词
function executeOperatorMotion(
  op: Operator,     // 'delete' | 'change' | 'yank'
  motion: string,   // 'w' | 'b' | 'e' | 'h' | 'l' | '$' | ...
  count: number,
  ctx: OperatorContext
): void {
  const start = ctx.cursor
  const end = applyMotion(motion, start, count, ctx.text)
  const isInclusive = isInclusiveMotion(motion)

  const [from, to] = orderPositions(start, end)
  const deleted = extractRange(ctx.text, from, to, isInclusive)

  if (op !== 'yank') ctx.setText(removeRange(ctx.text, from, to, isInclusive))
  if (op !== 'yank') ctx.setOffset(positionToOffset(from, ctx.text))
  if (op === 'change') ctx.enterInsert(positionToOffset(from, ctx.text))

  ctx.setRegister(deleted, false)  // 存入寄存器
  ctx.recordChange({ type: 'operator', op, motion, count })
}
```

## 文本对象（textObjects.ts）

```
ciw（change inner word，修改内部词）：
  1. 找到当前词的起点和终点（跳过空白）
  2. 删除这个范围的内容
  3. 进入插入模式

daw（delete around word，删除含空白的词）：
  1. 找到当前词 + 周围空白的范围
  2. 删除整个范围

di"（delete inside quotes，删除引号内容）：
  1. 向左找最近的 "
  2. 向右找对应的 "
  3. 删除两个 " 之间的内容

da(（delete around parentheses，删除含括号的内容）：
  1. 支持嵌套括号（从内向外匹配）
  2. 删除括号和内容
```

> **下一节**：[9.2 Bridge](./92-bridge.md)
