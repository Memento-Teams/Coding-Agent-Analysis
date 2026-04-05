# 1.2 main.tsx — 3400 行的"总指挥"

> `cli.tsx` 只是门卫，`init.ts` 只是安检。真正的重头戏在 `main.tsx`——它编排了从配置加载到 REPL 就绪的所有初始化工作。这个文件有两个你在其他 CLI 工具里几乎看不到的设计。

## main.tsx 做了什么？

```
cli.tsx 说："检查完毕，交给你了"
  │
  ▼ main.tsx 接手
  │
  ├── 1. 并行预取（Parallel Prefetch）     ← 本节重点
  ├── 2. 迁移检查（11 个版本迁移脚本）
  ├── 3. 信任检查（才敢跑 git 命令）       ← 本节重点
  ├── 4. 注册 42 个工具 + 101 个命令
  ├── 5. 连接 MCP Servers
  ├── 6. 加载 Skills & Plugins
  ├── 7. 启动 React UI
  └── 8. 进入 REPL 循环，等待用户输入
```

其中 1 和 3 是 Claude Code 独有的设计，值得展开讲。

---

## 1.2.1 并行预取 — 把等待时间藏起来

### 问题

启动时需要读取两种东西：
- **Keychain**：从 macOS Keychain / Windows 凭据管理器读取 OAuth Token（约 65ms）
- **企业策略**：从系统目录读取企业设备管理配置（需要 spawn 子进程）

如果串行执行：

```
加载 135ms 的 import → 读 Keychain 65ms → 读企业策略 30ms
总计：~230ms
```

### Claude Code 的做法：在 import 之前就开火

```tsx
// 📜 src/main.tsx（第 1-20 行）—— 文件最顶部，import 之前！

profileCheckpoint('main_tsx_entry')

import { startMdmRawRead } from './utils/settings/mdm/rawRead.js'
startMdmRawRead()    // 🔥 立刻 spawn 子进程读企业策略

import { startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js'
startKeychainPrefetch()  // 🔥 立刻 spawn 两个并行的 keychain 读取

// 然后才是剩下的 135ms 的 import...
import { Commander } from 'commander'
import { QueryEngine } from './QueryEngine.js'
// ... 几十个 import ...
```

时间线对比：

```
之前（串行）：
  [===== import 135ms =====][= keychain 65ms =][= MDM 30ms =]
  总计 ~230ms

之后（并行预取）：
  [===== import 135ms =====]
  [= keychain =][= MDM =]    ← 在 import 期间就跑完了！
  总计 ~135ms（预取被完全隐藏）
```

后来在 `preAction` 阶段 await 这些 Promise：

```tsx
// 📜 src/main.tsx（第 914 行）
await Promise.all([
  ensureMdmSettingsLoaded(),        // 大概率早就完成了
  ensureKeychainPrefetchCompleted() // 大概率早就完成了
])
```

> **这个技巧的核心思想**：`import` 语句在 Node/Bun 里是同步阻塞的——CPU 在那 135ms 里忙着解析和执行 JS。但子进程的 I/O（读 keychain、读系统配置）不需要 CPU，只是在等磁盘/系统调用。所以让它们在 CPU 忙 import 的时候在后台跑 I/O，两者互不干扰。

### Keychain 预取的细节

```tsx
// 📜 src/utils/secureStorage/keychainPrefetch.ts

export function startKeychainPrefetch(): void {
  // 同时发起两个 keychain 读取（OAuth Token + 旧版 API Key）
  // 不等结果，立刻返回
  oauthPromise = readKeychainEntry('claude-oauth-token')
  apiKeyPromise = readKeychainEntry('claude-api-key')
}

export async function ensureKeychainPrefetchCompleted(): Promise<void> {
  // 在需要时 await——大概率已经完成了
  await Promise.all([oauthPromise, apiKeyPromise])
}
```

没有预取的话，这两个 keychain 读取是串行的（每个约 30ms），总共约 65ms。预取后这 65ms 被完全藏在 import 时间里。

---

## 1.2.2 信任检查 — 先确认安全，才敢跑 git

### 问题

启动时需要收集 git 信息（当前分支、最近 commit、文件状态），用于构建 System Prompt 的上下文。但 `git status` 不是无害的——它可以执行任意代码：

```
Git 的"隐藏执行"机制：
  ├── .git/hooks/     → post-checkout 等 hook 可以跑任意脚本
  ├── core.fsmonitor  → 配置一个文件监控程序（任意可执行文件）
  └── diff.external   → 配置一个外部 diff 工具（任意可执行文件）
```

如果用户 clone 了一个恶意仓库，仓库里的 `.git/config` 可能包含：

```ini
[core]
    fsmonitor = /tmp/evil.sh   # git status 会自动执行这个脚本
```

此时 Claude Code 一启动就跑 `git status`，就等于**在没有用户同意的情况下执行了恶意代码**。

### Claude Code 的做法：信任检查前不碰 git

```tsx
// 📜 src/main.tsx（第 354-380 行）

function prefetchSystemContextIfSafe(): void {
  const isNonInteractiveSession = getIsNonInteractiveSession()

  // -p 模式（无交互）：信任是隐含的（文档里写了：跳过信任对话框）
  if (isNonInteractiveSession) {
    void getSystemContext()  // 可以安全地跑 git 命令
    return
  }

  // 交互模式：只有用户之前已经确认信任此目录，才预取
  const hasTrust = checkHasTrustDialogAccepted()
  if (hasTrust) {
    void getSystemContext()  // 用户已确认信任，可以跑 git
  } else {
    // ❌ 不预取！等信任对话框确认后再说
    // git 命令可能通过 hooks/config 执行任意代码！
  }
}
```

信任对话框长这样：

```
┌──────────────────────────────────────────┐
│  Do you trust the files in this folder?  │
│  /Users/you/some-cloned-repo             │
│                                          │
│  [Yes, I trust this folder]  [No]        │
└──────────────────────────────────────────┘
```

用户点"Yes"后信任状态被记录，之后的启动就不再弹了。

> **为什么其他 CLI 工具不做这个？** 因为大部分 CLI 工具不会在启动时自动执行 `git` 命令。但 Claude Code 需要 git 信息来构建 AI 的上下文（"你在 main 分支，最近改了 auth.ts"），所以它必须在启动时就跑 git——这让信任检查变成了刚需。VSCode 也有类似的"Workspace Trust"机制，原因完全一样。

---

## 1.2.3 启动性能分析 — 0.5% 采样率的无感监控

```tsx
// 📜 src/utils/startupProfiler.ts

// 采样率：Anthropic 内部 100%，外部用户 0.5%
const SHOULD_PROFILE =
  process.env.USER_TYPE === 'ant' || Math.random() < 0.005

export function profileCheckpoint(name: string): void {
  if (!SHOULD_PROFILE) return  // 99.5% 的用户直接跳过，零开销
  performance.mark(name)
  if (DETAILED_PROFILING) {
    console.log(`[${name}] ${performance.now().toFixed(1)}ms  mem=${process.memoryUsage().heapUsed}`)
  }
}
```

整个启动流程埋了约 20 个 checkpoint：

```
cli_entry → main_tsx_entry → main_tsx_imports_loaded →
init_function_start → preAction_start → preAction_after_mdm →
preAction_after_settings → ... → repl_ready
```

被采样到的用户的 checkpoint 数据会上报到 Statsig，Anthropic 可以监控每个阶段的耗时趋势——如果某次发版让 `init_function_start → preAction_start` 变慢了 50ms，他们能立刻发现。

> **为什么不全量采集？** 因为 `performance.mark()` 虽然很快，但在每秒可能跑几千次的 CI 环境里，0.5% 采样可以把监控开销从"微小"降到"几乎为零"，同时仍然有足够的统计样本。

---

## 1.2.4 Setup/Commands/Agents 三路并行

注册工具和命令需要扫描文件系统（加载用户自定义的 Skills 和 Agents），这是 I/O 密集的。同时 `setup()` 需要绑定 socket。Claude Code 把它们并行化：

```tsx
// 📜 src/main.tsx（第 1913-1936 行）

// 三件事同时开始！
const setupPromise = setup(preSetupCwd, ...)           // ~28ms（socket 绑定）
const commandsPromise = getCommands(preSetupCwd)       // 扫描 .claude/skills/
const agentDefsPromise = getAgentDefinitions(preSetupCwd) // 扫描 .claude/agents/

// 先等 setup（最快完成）
await setupPromise

// 然后收割 commands 和 agents（大概率已经完成了）
const [commands, agentDefs] = await Promise.all([
  commandsPromise,
  agentDefsPromise
])
```

但有一个特殊情况——如果启用了 `--worktree`，`setup()` 会 `process.chdir()` 切换目录，这会影响 commands/agents 的加载路径。所以这种情况下不能并行：

```tsx
// 如果有 worktree，不能提前 kick commands（路径会变）
const commandsPromise = worktreeEnabled ? null : getCommands(preSetupCwd)
const agentDefsPromise = worktreeEnabled ? null : getAgentDefinitions(preSetupCwd)

await setupPromise  // setup 可能 chdir

// worktree 模式下用 chdir 后的路径重新加载
const [commands, agentDefs] = await Promise.all([
  commandsPromise ?? getCommands(currentCwd),   // 用新的 cwd
  agentDefsPromise ?? getAgentDefinitions(currentCwd)
])
```

---

## 启动流程总结

```
时间线（约 135ms 完成）：

0ms    cli.tsx: Fast Path 检查
       │
1ms    cli.tsx: earlyInput 开始捕获键盘输入（用户可以边等边打字）
       │
2ms    main.tsx 顶部: spawn keychain + MDM 子进程（并行预取）
       │        ┌── keychain 子进程在后台跑
       │        ├── MDM 子进程在后台跑
       ▼        │
2-135ms import 阶段（CPU 密集，解析 JS）
       │        │
       │        └── keychain/MDM 子进程跑完（被隐藏在 import 时间内）
       ▼
135ms  import 完成
       │
       ├── 迁移检查（11 个脚本，跑过一次就不再跑）
       ├── 信任检查 → 通过后才预取 git 上下文
       ├── 并行：setup() + getCommands() + getAgentDefs()
       ├── 连接 MCP Servers
       └── React.render(<App/>)
       │
~150ms REPL 就绪，等待用户输入
       └── earlyInput 的缓存内容被取出，用户之前打的字直接显示
```
