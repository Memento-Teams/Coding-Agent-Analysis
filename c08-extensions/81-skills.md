# 8.1 第一层：Skills 系统 — 吟游诗人的散文诗

> Skills 的本质，仅仅是一篇篇在内存中飞舞的散文诗。没有沉重的编译，没有复杂的依赖——只有纯文本，和大模型的想象力。

**定位**：原生集成、极度轻量、依附于 Prompt 的本地指令集。

在 Anthropic 的愿景中，Skills 是为了把人类和组织那些"只可意会不可言传的默契规范"，封装成全平台通行的标准化组件（Build once, use everywhere），让 LLM 能像拼搭乐高一样极速堆叠这些 Skills。

## 8.1.1 两种截然不同的物理形态

```
src/skills/
├── bundled/                  # 16 个原生核心内建指令集
│   ├── simplify.ts           # (/simplify) 并发执行代码复杂性审查
│   ├── skillify.ts           # (/skillify) 自动将上文操作包装为新技能
│   ├── batch.ts              # (/batch) 隔离执行批量代理任务
│   └── ...
├── bundledSkills.ts          # 内建法术静态注册表
├── loadSkillsDir.ts          # 轮询式外挂装载流
└── mcpSkillBuilders.ts       # 协议级外置接口转译
```

### 内建技能 (Bundled Skills)

直接硬编码进 CLI 工具的二进制发版内部。冷启动零延迟，往往用于最宏大或危险权限的操作收口（例如 `/simplify` 并行代码审查）。

### 本地外挂技能 (Disk-based Skills)

散落在磁盘深处（`.claude/skills/` 或 `~/.claude/skills/`）的纯 `SKILL.md` 文档。允许开发人员像书写散文一样，自由扩展和固化智能体的复杂流程逻辑，无需 Anthropic 审核升级。

### 殊途同归的无状态理念

无论是繁重的并发本能，还是草草搭建的 Markdown 文件，**所有技能最终都受制于同一种极致轻盈的"无状态理念"**：

**纯粹干净的加载**：

```tsx
// bundledSkills.ts — 内建技能压入常量数组
const bundledSkills: Command[] = []

// loadSkillsDir.ts — 外挂技能仅靠 readFile + parseFrontmatter 裸装载
const content = await fs.readFile(skillFilePath, { encoding: 'utf-8' })
const { frontmatter, content: markdownContent } = parseFrontmatter(content)
```

**随用随弃的执行**：核心本质是一簇无副作用闭包 `getPromptForCommand`——即时调用，把动态参数缝合进自然语言文本，抛给大模型推理：

```tsx
// loadSkillsDir.ts
async getPromptForCommand(args, toolUseContext) {
  finalContent = substituteArguments(finalContent, args, ...)
  return [{ type: 'text', text: finalContent }]  // 直接按普通文本抛给大模型
}
```

**即生即灭的生命周期**：不需要 RwLock 加锁，不需要持久化哈希表。数组重置瞬间排空：

```tsx
export function clearBundledSkills(): void {
  bundledSkills.length = 0  // 极其彻底的技能生命终结
}
```

---

## 8.1.2 `/skillify` 的炼金工厂

`/skillify` 是一个具备"自我繁殖"能力的元 Skill——它能从当前会话中提取可复用的流程，自动封装为新的 Skill。

**两个阶段**：

1. **逻辑提取（战略访谈）** — 4 轮面试式对话：
   - Round 1：高层确认（名称、目标）
   - Round 2：参数确认（输入、提示）
   - Round 3：步骤分解（带成功准则的执行序列）
   - Round 4：触发条件（when_to_use 语义场景）

2. **物理封装（战术模版）** — 生成标准 SKILL.md：
   ```markdown
   ---
   name: {{skill-name}}
   description: {{description}}
   allowed-tools: [...]
   when_to_use: {{detailed...}}
   argument-hint: "{{hint}}"
   context: {{inline or fork}}
   ---
   # {{Skills Title}}
   ## Goal
   ## Steps
   ```

> 这种"内建本能催生外层拓扑"的设计，让系统预置能力通过严密蓝图引导用户开垦出属于自己的 Skills 森林。

---

## 8.1.3 森严而包容的四重覆盖防线

如果任由随处的文本挂载技能，一旦发生指令名字冲突，大模型该听谁的？`loadSkillsDir.ts` 竖立了四重优先级防线：

```tsx
export function getSkillsPath(source: SettingSource | 'plugin', dir: string): string {
  switch (source) {
    case 'policySettings':
      return join(getManagedFilePath(), '.claude', dir)  // 🏛️ 企业强制策略（最高）
    case 'userSettings':
      return join(getClaudeConfigHomeDir(), dir)         // 👤 全局用户级
    case 'projectSettings':
      return `.claude/${dir}`                            // 📁 工程项目级
    case 'plugin':
      return 'plugin'                                    // 🔌 第三方插件（最低）
  }
}
```

| 层级 | 路径 | 特点 |
|------|------|------|
| 🏛️ Policy | `/etc/claude-code/.claude/skills` | 系统只读目录，企业底线规范，无条件向下碾压 |
| 👤 User | `~/.claude/skills` | 个人专属工作习惯，跨项目伴随运行 |
| 📁 Project | `.claude/skills` | 项目定制，提交到 VCS 团队共享 |
| 🔌 Plugin | 第三方包内 | 最低优先级，杜绝外部包篡改核心指令 |

### 隐藏的安全护城河

除路径优先级外，`loadSkillsDir.ts` 还穿插了多道防御：

- **反软链接污染** (`getFileIdentity`)：通过 `realpath` 剔除所有 Symlinks，阻止循环套娃攻击
- **语义审判** (`parseSkillFrontmatterFields`)：严密审计 `allowed-tools`、`paths` 隔离范围和 `hooks` 生命周期
- **Shell 逃逸隔离**：MCP 或 Plugin 来源的技能，**直接绕开** Shell 命令渲染，从根本上拔除远程包挟持终端的可能

```tsx
// 最高权限的 Shell 渲染只授权给本地嫡系技能
if (loadedFrom !== 'mcp') {
  finalContent = await executeShellCommandsInPrompt(finalContent, ...)
}
```

---

## 8.1.4 流派对撞：Claude Code vs Codex CLI

| 维度 | Claude Code (吟游诗人) | Codex CLI (话剧演员) |
|------|----------------------|---------------------|
| 物理形态 | 纯 Markdown 文本 | 二进制序列化编入可执行文件 |
| 加载方式 | `readFile` + 正则提取 | `include_dir!` 宏编译时嵌入 |
| 防篡改 | 路径优先级 + 软链接检查 | 启动时 FNV 哈希指纹校验 |
| 生命周期 | 无状态，闭包即生即灭 | RwLock + 字典索引持久化 |
| 灵活性 | 极高（随时扔个 .md 就行） | 较低（需重新编译） |
| 可靠性 | 依赖路径隔离 | 工业级强类型保障 |

> **下一节**：[8.2 Plugins 体系](./82-plugins.md)
