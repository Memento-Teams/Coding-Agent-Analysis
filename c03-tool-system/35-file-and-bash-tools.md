# 3.5-3.6 文件操作与命令执行工具

> 文件操作类和 BashTool 是 Claude Code 中使用频率最高的工具。本节深入它们的实现细节。

## 3.5 文件操作类

### 3.5.1 FileReadTool — 读一切文件

**目录**：`src/tools/FileReadTool/`（8 个文件）

```tsx
export const FileReadTool = buildTool({
  name: 'Read',
  async call({ file_path, offset = 1, limit = undefined, pages }, context)
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
})
```

不只是读文本——根据文件类型返回不同格式：

```
file_path 的扩展名是什么？
  │
  ├── .ts/.js/.py/...  → 文本：返回带行号的内容 + 范围信息
  ├── .png/.jpg/...    → 图片：Base64 编码 + 自动缩放 + 尺寸信息
  ├── .ipynb           → Notebook：解析 cells + outputs
  ├── .pdf             → PDF：提取指定页面，转为图片
  └── 二次读取同一文件   → file_unchanged：文件没变，返回空占位（省 token）
```

**安全设计**：
- **文件未变检测**：文件自上次读取后没有变化时，返回 `file_unchanged` 占位符，避免浪费 Token
- **UNC 路径阻断**：Windows 上阻止 `\\server\share` 路径，防止 NTLM 凭证泄露
- **设备路径阻断**：阻止 `/dev/zero`、`/dev/stdin` 等设备文件
- **macOS 截图路径**：自动处理 macOS 截图文件名中的 thin space（U+202F）

---

### 3.5.2 FileWriteTool — 原子写入

**目录**：`src/tools/FileWriteTool/`（8 个文件）

```tsx
export const FileWriteTool = buildTool({
  name: 'Write',
  async call({ file_path, content }, context)
  // isConcurrencySafe: 默认 false（写入操作）
  // isReadOnly: 默认 false
})
```

**写入流程**：

```
① 发现并激活相关 Skills（根据文件路径）
② 文件历史备份（记录修改前内容，支持撤销）
③ 过期检查（文件自上次读取后是否被外部修改？）
④ 原子写入（LF 换行符标准化）
⑤ 通知 LSP 服务器（didChange + didSave）
⑥ 通知 VSCode SDK（更新 diff 视图）
⑦ 更新读取时间戳（供后续过期检查使用）
```

**关键设计**：
- **过期检查**：比较 mtime 和上次读取时间。如果文件被外部修改了，拒绝写入，避免覆盖用户修改
- **团队记忆密钥检查**：防止写入包含团队密钥（`checkTeamMemSecrets()`）
- **LSP 集成**：写入后通知语言服务器，确保代码诊断即时更新

---

### 3.5.3 FileEditTool — 精确的文本替换

**目录**：`src/tools/FileEditTool/`（6 个文件）

```tsx
export const FileEditTool = buildTool({
  name: 'Edit',
  async call({ file_path, old_string, new_string, replace_all? }, context)
})
```

**校验流程**：

```
① 团队记忆密钥检查      → 防止泄露密钥
② 空操作检测            → old_string === new_string → 拒绝
③ 文件大小检查           → > 1 GiB → 拒绝（防 OOM）
④ 唯一性检查（核心！）    → 匹配 0 次 → "找不到"
                         → 匹配 1 次 → 通过 ✅
                         → 匹配 2+ 次 → "请提供更多上下文"
⑤ 编码检测              → UTF-8 / UTF-16LE
⑥ 换行符保留            → 保持 \n / \r\n / \r
⑦ 执行替换 + 生成 Diff
```

> **唯一性检查**是 FileEditTool 最核心的设计。如果 `old_string` 在文件中匹配多处，工具会拒绝执行并要求 Claude 提供更多上下文。这**用限制避免歧义**——看似增加了 Claude 的负担，实际上消除了"改错地方"的风险。

---

### 3.5.4 NotebookEditTool — Jupyter 编辑

**目录**：`src/tools/NotebookEditTool/`（6 个文件）

```tsx
export const NotebookEditTool = buildTool({
  name: 'NotebookEdit',
  async call({ notebook_path, new_source, cell_id, cell_type, edit_mode }, context)
})
```

- 解析 Notebook JSON → 定位目标 Cell → 执行编辑 → 写回文件
- 编辑代码 Cell 时自动重置 `execution_count` 并清除 `outputs`（旧输出不再有效）
- **Read-before-Edit 约束**：必须先读过 Notebook 才能编辑，防止盲改
- 如果尝试 replace 超出范围的 Cell，自动转为 insert 模式

---

### 3.5.5 GlobTool — 文件模式匹配

```tsx
export const GlobTool = buildTool({
  name: 'Glob',
  async call({ pattern, path? }, context)
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
})
```

执行 glob 匹配，返回文件列表。结果上限 100 个文件，路径相对化以节省 Token。

---

### 3.5.6 GrepTool — 代码搜索（最简单的工具）

```tsx
export const GrepTool = buildTool({
  name: 'Grep',
  async call({ pattern, path, glob, type, output_mode, ... }, context)
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
})
```

调用 ripgrep (`rg`) 执行搜索，自动排除 `.git`、`.svn` 等目录，支持分页。

**关键设计**：
- **`head_limit` 默认 250**：防止搜索结果太大撞爆上下文
- **`--max-columns 500`**：防止 base64 编码的二进制文件产生超长行

---

## 3.6 命令执行类

### 3.6.1 BashTool — 最复杂的工具（18 个文件）

**目录结构**：

```
BashTool.tsx              ← 主实现
bashPermissions.ts        ← 权限：哪些命令需要授权
bashSecurity.ts           ← 安全：AST 分析命令安全性
bashCommandHelpers.ts     ← 解析：拆分复合命令
commandSemantics.ts       ← 语义：命令在"做什么"
sedEditParser.ts          ← sed 命令解析
sedValidation.ts          ← sed 安全校验
modeValidation.ts         ← 权限模式检查
pathValidation.ts         ← 路径访问校验
readOnlyValidation.ts     ← 只读约束检查
shouldUseSandbox.ts       ← 沙箱决策
destructiveCommandWarning.ts ← 破坏性命令警告
commentLabel.ts           ← 注释提取
UI.tsx / BashToolResultMessage.tsx ← 终端渲染
utils.ts / toolName.ts / prompt.ts ← 辅助
```

```tsx
export const BashTool = buildTool({
  name: 'Bash',
  async call({ command, timeout?, description?, run_in_background?,
               dangerouslyDisableSandbox? }, context)
  isConcurrencySafe: (input) => this.isReadOnly?.(input) ?? false,
  isReadOnly: (input) => checkReadOnlyConstraints(input.command),
})
```

**权限检查流程**：

```
"rm -rf /tmp/test && git push origin main"
     │
  ① 拆分子命令 → ["rm -rf /tmp/test", "git push origin main"]
  ② AST 安全分析 → rm: destructive, git push: needs_approval
  ③ 路径校验 → /tmp/test 在允许范围内？
  ④ 规则匹配 → 用户配置的 allow/deny 规则
  ⑤ 最终决定 → allow / deny / ask
```

**关键设计**：
- **子命令上限 50**：超过 50 个子命令不做精细分析，直接回退到 `ask`
- **内外 Schema 分离**：`_simulatedSedEdit` 对 Claude 不可见，防止绕过安全检查
- **自动后台化**：`npm`、`docker`、`python` 等长命令超过 120 秒自动转后台
- **maxResultSizeChars: 30,000**：超过则落盘，只发预览给 Claude

---

### 3.6.2 PowerShellTool — Windows 版的 BashTool

和 BashTool 类似，但针对 Windows PowerShell 环境：
- 支持 PowerShell 别名的大小写不敏感解析
- Windows 沙箱策略校验
- 检测并阻止 `Start-Sleep` 超过 2 秒的命令
- 助手模式下阻塞命令 15 秒后自动后台化

> **下一节**：[3.7-3.8 Agent 协作与任务管理](./37-agent-and-task-tools.md)
