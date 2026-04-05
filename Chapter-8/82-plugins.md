# 8.2 第二层：Plugins 体系 — 冷酷的集装箱货轮

> 不同于 Skills 只是散落的 Markdown 片段，一个标准的 Plugin 拥有类似 NPM 包的硬核物理目录。它不仅能挂载代码沙盒，还能封装备有独立推理循环的本地微智能体。

**核心定位**：由第三方分发、支持多生境网络下载（NPM、Git）、在严苛生态雷达监控下运行的重量级装载器。

## 标准集装箱的目录结构

```
my-plugin/               # (物理坐标: ~/.claude/plugins/cache/...)
├── plugin.json          # [神经中枢] 生命周期配置单，掌控版本、依赖甚至安全认证
├── commands/            # [执行舱] 独家 Slash 扩展指令
├── agents/              # [中枢脑] 自定义微 Agent（含提示词和上下文循环）
├── hooks/               # [拦截网] hooks.json，在核心生命周期强行打入钩子
└── lsp/                 # [感官层] 独立 LSP 服务，实现代码生态的深空扫描
```

### plugin.json — 深不可测的枢纽引擎

```json
{
  "name": "enterprise-compliance",
  "version": "1.2.0",
  "dependencies": ["@anthropic/core-utils"],
  "userConfig": {
    "API_TOKEN": {
      "type": "string",
      "sensitive": true,    // 密码输入全遮挡，存入 macOS Keychain / Windows Credentials
      "description": "内网合规系统存取密匙"
    }
  },
  "mcpServers": {           // Plugin 可以自带 MCP 任意门钥匙！
    "compliance-db": { "type": "stdio", "command": "python", "args": ["..."] }
  }
}
```

## 8.2.1 架构分离：引擎室与预装货物

| 层级 | 路径 | 职责 |
|------|------|------|
| **引擎室** | `src/utils/plugins/` (~40 文件) | 底层调度：加载器、阻止列表、市场管理器。无业务逻辑，只有冰冷的上下架枢纽 |
| **预装货物** | `src/plugins/` | 实际可用的插件实例（`builtinPlugins.ts`、`bundled/` 下的官方模块） |

所有插件——无论官方内置还是外部下载——最终都要通过同一个标准装载器解析并接受生态雷达监督。

## 8.2.2 跨生态多协议装载引擎

`pluginLoader.ts` 抹平了 NPM 与 Git 的异构性鸿沟：

```tsx
// NPM 生态接驳
export async function installFromNpm(packageName, targetPath, options) {
  const npmCachePath = join(getPluginsDirectory(), 'npm-cache')  // 专属沙盒缓存区
  const args = ['install', packageSpec, '--prefix', npmCachePath]
  await execFileNoThrow('npm', args, { useCwd: false })  // 直接唤醒宿主机 npm
}

// Git 源码直接渗透
export async function gitClone(gitUrl, targetPath, ref?, sha?) {
  const args = ['clone', '--depth', '1', '--recurse-submodules', '--shallow-submodules']
  // 极致浅克隆，斩断庞大历史树，只保留当前实物
}

// 冰封 ZIP 存储
if (zipCacheMode) {
  await convertDirectoryToZipInPlace(cachePath, zipPath)  // 防篡改 + 极致性能
}
```

调度者只需提出 `plugin@version` 标志，浅层克隆、缓存生成、ZIP 动态解压等脏活累活由底层容器自动处理。

## 8.2.3 治理防线：无情的生态雷达与连坐卸载

一旦放开拉取第三方代码的端口，供应链投毒的风险随之而来。Plugin 体系设立了**全生命周期肃清雷达**：

```tsx
// pluginBlocklist.ts
export async function detectAndUninstallDelistedPlugins(): Promise<string[]> {
  const marketplace = await getMarketplace(marketplaceName)
  const delisted = detectDelistedPlugins(installedPlugins, marketplace, marketplaceName)

  // 无情拔除：在市场中被除名的插件，宿主机上的副本立刻触发物理级强制卸载
  for (const pluginId of delisted) {
    try {
      await uninstallPluginOp(pluginId, scope)
    } catch (error) {
      logForDebugging(`Failed to auto-uninstall delisted plugin...`)
    }
    // 黑名单耻辱柱：即使无法卸载，也被打上系统级标记，调度前直接封杀
    await addFlaggedPlugin(pluginId)
  }
}
```

> 在第一层 Skills 的自由阶段，系统没有精力约束散文文本的去留。但在第二层，系统具备了对 Marketplace 强心跳监听的能力。一旦感知远端插件被安全委员会除名，肃清机制立刻跨越隔离墙硬核拔除依赖。

> **下一节**：[8.3 MCP Servers](./83-mcp.md)
