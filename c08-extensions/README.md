# Chapter 8: 扩展机制 — Skills, Plugins, MCP

> 扩展机制遵循一条隐形的能量法则："**离核心越近，操作约束越少；离核心越远，通信标准越严**"。这种"同心圆式"的安全外溢设计，使得系统可以在保全核心稳定性的同时，无限向外触达未知的技术群岛。

## 本章结构

| 节 | 主题 | 核心问题 |
|---|------|---------|
| [8.1 Skills 系统](./81-skills.md) | Layer 1: Skills | 纯文本闭包如何实现零阻尼扩展？四重优先级如何防冲突？ |
| [8.2 Plugins 体系](./82-plugins.md) | Layer 2: Plugins | 集装箱式包管理如何跨生态装载？生态雷达如何防投毒？ |
| [8.3 MCP Servers](./83-mcp.md) | Layer 3: MCP | 119KB 的 client.ts 如何铸就跨次元协议网关？ |
| [8.4 三层总结](./84-summary.md) | Summary | 三层扩展机制的降维阶梯全景 |

## 前置知识

- [Chapter 3: 工具系统](../c03-tool-system/README.md) — 工具注册机制
- [Chapter 4: Prompt 工程](../c04-prompt-engineering/README.md) — Skill 的 Prompt 本质
- [Chapter 5: 服务层](../c05-services/README.md) — MCP 服务的底层实现

## 三层同心圆

```
       ┌──────────────────────┐
       │  Layer 1: Skills     │  ← 随身便签（纯 Markdown）
       │  ┌──────────────┐   │
       │  │ Layer 2:     │   │  ← 集装箱（NPM/Git 包管理）
       │  │ Plugins      │   │
       │  │ ┌────────┐  │   │
       │  │ │Layer 3:│  │   │  ← 任意门（协议级网关）
       │  │ │ MCP    │  │   │
       │  │ └────────┘  │   │
       │  └──────────────┘   │
       └──────────────────────┘
```
