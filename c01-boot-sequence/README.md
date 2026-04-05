# Chapter 1: 启动流程 — "一条命令如何跑起来"

> 当你在终端输入 `$ claude "fix this bug"` 按下回车，到屏幕上出现第一个字符，中间发生了什么？这一章带你走完整条链路。

## 本章结构

| 节 | 主题 | 核心问题 |
|---|------|---------|
| [1.0 涉及的文件与目录](./10-files-overview.md) | Files Overview | 启动流程涉及哪些源文件？ |
| [1.1 cli.tsx — 真正的第一行代码](./11-cli-entry.md) | CLI Entry Point | 程序是如何用 "按需加载" 实现极速启动的？ |
| [1.2 main.tsx — 3400 行的总指挥](./12-main-tsx.md) | Main Orchestrator | 并行预取、信任检查、启动性能监控 |

## 前置知识

阅读本章前，建议先完成 [c00: 项目概览](../c00-project-overview/README.md)，特别是：
- [0.0 全局架构总览](../c00-project-overview/00-architecture-overview.md) — 理解各层的位置
- [0.1 启动流程概览](../c00-project-overview/01-boot-sequence.md) — 本章是这张时序图的"放大版"
