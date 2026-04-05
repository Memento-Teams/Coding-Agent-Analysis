# 1.1 cli.tsx — 真正的第一行代码

> `src/entrypoints/cli.tsx` 是 Bun 的构建入口点，整个应用从这里开始执行。这个文件只有一个核心思想：**能不加载就不加载**。

## 核心设计原则：按需加载

整个 cli.tsx 就是一个大的 `if-else` 链。听起来简单，但背后的设计哲学值得深思。

用一个比喻来理解：

> 假设你开了一家大型医院，有 200 个科室。一个人走进来：
>
> **场景 A**：不管来人要什么，先把 200 个科室的灯全打开、所有医生全部就位、所有设备全部预热。然后问："您要看什么？"——"哦，我就来取个体检报告。"
>
> **场景 B**：前台先问一句"您要做什么？"
> - 取报告 → 只开档案室的灯，1 秒搞定
> - 看感冒 → 只开内科，30 秒就绪
> - 做手术 → 好，那确实需要启动很多科室
>
> cli.tsx 做的就是**场景 B**。

大部分代码的写法是场景 A，因为简单：

```tsx
// 简单但浪费：一行搞定，不管用户要干什么，全部加载
import { everything } from './main.js'  // 加载 200 个模块，耗时 135ms

if (args[0] === '--version') {
  console.log(version)  // 只需要 1ms 的事，却等了 135ms
}
```

而 Claude Code 选择了更费心的写法——针对每条路径，只加载它需要的东西：

```tsx
// 麻烦但精确：每条路径只加载自己需要的
if (args[0] === '--version') {
  console.log(version)   // 0 个 import，< 1ms
  return
}
if (args[0] === 'daemon') {
  const { daemonMain } = await import('./daemon.js')  // 只加载 daemon 相关
  return
}
// ... 11 个这样的分支
// 最后才是完整加载
const { main } = await import('./main.js')  // 只有真正需要时才加载全部
```

**代价**：代码量更大、维护更复杂（11 个分支，每个要想清楚加载什么）。

**回报**：`claude --version` 从 135ms 降到 < 1ms。CI 里每天调用几千次，累积省下几分钟。

---

## 1.1.1 环境变量预设（第 1-26 行）

环境变量的设置放在文件最顶部，在所有 `import` 之前。

> **为什么必须在最顶部？** 环境变量在 Node/Bun 中是全局状态。很多模块在 `import` 时（模块求值阶段）就会读取 `process.env`。如果在 import 之后再设置，就来不及了。这体现了一个工程原则：**副作用的时序必须在依赖图之前**。

---

## 1.1.2 Fast Path 模式（第 33-279 行）

cli.tsx 定义了 11 条 Fast Path，每条都精确控制加载的模块数量：

| Fast Path | 触发条件 | 加载的模块数 | 典型延迟 |
|-----------|---------|------------|---------|
| `--version` | `args[0] === '-v'` | **0 个** | < 1ms |
| `--dump-system-prompt` | ant-only 调试 | ~5 个 | ~20ms |
| `--claude-in-chrome-mcp` | Chrome 扩展 | ~10 个 | ~30ms |
| `--daemon-worker` | 后台 worker | ~8 个 | ~25ms |
| `claude bridge/rc` | IDE 桥接 | ~15 个 | ~50ms |
| `claude daemon` | 守护进程 | ~12 个 | ~40ms |
| `claude ps/logs/attach` | 后台会话管理 | ~10 个 | ~30ms |
| `claude new/list/reply` | 模板 | ~8 个 | ~25ms |
| `--environment-runner` | 容器化运行器 | ~10 个 | ~30ms |
| `--tmux --worktree` | tmux 隔离 | ~5 个 | ~15ms |
| **完整路径（无 fast path）** | 普通交互 | **~200 个** | **~135ms** |

### `--version` 的极致优化

```tsx
if (args.length === 1 && (args[0] === '--version' || args[0] === '-v')) {
  console.log(`${MACRO.VERSION} (Claude Code)`)
  return  // 整个进程在这里结束，没有加载任何模块
}
```

`MACRO.VERSION` 是 Bun 构建时的**编译期宏替换**，不需要读文件或 require 任何模块。零 import，零 I/O，直接输出字符串然后退出。

### 桥接模式的 Fast Path

桥接模式是 Claude Code 作为 **IDE 后端** 运行的方式：

```
正常模式：   你 ──打字──> 终端 (Terminal UI) ──请求──> Claude API
桥接模式：   你 ──在 IDE 操作──> VSCode 插件 ──bridge──> Claude Code 后台 ──请求──> Claude API
```

```tsx
if (/* 命令是 remote-control/rc/bridge/sync */) {
  await enableConfigs()
  const disabledReason = await getBridgeDisabledReason()
  if (disabledReason) {
    process.stderr.write(disabledReason)
    return
  }
  await bridgeMain()  // 直接进入桥接，不加载 REPL
  return
}
```

桥接模式下，Claude Code 是个**无头后台进程**（没有用户界面），所以大量模块可以跳过：

| 模块/能力 | 正常模式（终端 UI） | Bridge 模式（无头后台） |
|-----------|:------------------:|:---------------------:|
| 终端 UI（Ink 渲染器、React 组件） | 需要 | 不需要（IDE 负责 UI） |
| Commander.js（命令行参数解析） | 需要 | 不需要（IDE 直接发指令） |
| 42 个工具注册 (tools.ts) | 需要 | 不需要（bridge 层处理） |
| 101 个 Slash Command 注册 | 需要 | 不需要 |
| Vim 模式、快捷键、自动补全 | 需要 | 不需要（IDE 自己提供） |
| 配置加载 (enableConfigs) | 需要 | 需要 |
| 认证检查 (OAuth token) | 需要 | 需要 |
| bridgeMain()（桥接核心逻辑） | 不走 | 直接进入 |

---

## 1.1.3 Early Input Capture（第 287 行）

**问题**：用户可能通过管道传入数据（`echo "fix bug" | claude`），在 `main.tsx` 加载的 135ms 期间，stdin 的数据可能丢失。

**解法**：在 import main.tsx **之前**就开始捕获 stdin，把数据缓存到队列中。等 main.tsx 加载完后再取出处理。

```tsx
const { startCapturingEarlyInput } = await import('../utils/earlyInput.js')
startCapturingEarlyInput()
profileCheckpoint('cli_before_main_import')
const { main: cliMain } = await import('../main.js')  // 这个 import 要 ~135ms
profileCheckpoint('cli_after_main_import')
await cliMain()
```

这是一个典型的**生产者-消费者**模式：earlyInput 在 main 加载期间充当缓冲区，防止数据丢失。

---

## 1.1.4 init.ts — 配置与安全的守门人

> **本节内容待完善 (WIP)**

init.ts 负责在主逻辑启动前完成所有配置和安全检查。一个值得注意的细节是它对企业网络环境的适配：

```
家庭环境：  你 ──直连──> Claude API  ✅ 简单

企业环境：  你 ──> 公司代理（需配置代理）──> 防火墙 ──> 公司 CA 证书验证（需装证书）──> Claude API
```

企业环境需要处理 TLS 证书、HTTP 代理等额外复杂度，这些都是 init.ts 的职责。

---

> 返回 [Chapter 1 目录](./README.md) | 返回 [Chapter 0 总览](../Chapter-0/README.md)
