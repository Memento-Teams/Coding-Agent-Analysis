# Chapter 12: Skill 演化 — "从经验中学习"

> Agent 不仅记住过去（Chapter 11），还能从经验中提炼出可复用的能力。本章对比两种截然不同的 Skill 演化范式：**Hermes Agent** 的"在线手工匠"模式——Agent 在使用中发现问题，当场修补；**Memento-S** 的"离线工程化"模式——自动化 pipeline 批量评测、归因、重写、验证。两者不是竞争关系，而是同一设计空间中的两个极端。

## 为什么需要 Skill 演化？

Chapter 11 讨论的记忆系统解决的是"记住事实"的问题——用户偏好 Rust、项目用 pnpm、远程端口是 2222。但**过程性知识**——"怎么配置 Docker 开发环境"、"怎么写 GRPO 训练脚本"——不是一条事实，而是一套可执行的步骤。

这类知识有三个特点：
1. **首次获取成本高**——Agent 可能试了 8 次、调了 30 个工具才搞定
2. **可复用**——下次遇到类似任务，不需要从头摸索
3. **会过时**——工具升级、环境变化，步骤可能需要修正

Skill 系统解决的就是：**把高成本获取的过程性知识持久化，让它可复用，并在复用中持续改进**。

## 两种演化范式

```
Hermes Agent — 在线手工匠                 Memento-S — 离线工程化
──────────────────────────              ──────────────────────────
  用户交互触发                              Benchmark 批量触发
       │                                        │
  Agent 执行 skill                         系统执行 task
       │                                        │
  Agent 主观发现问题                        LLM Judge 客观评分
       │                                        │
  Agent 调 patch 修补                       LLM Selector 归因到 skill
       │                                        │
  直接生效，无验证                           LLM Optimizer 重写
       │                                        │
  改好改坏全凭手感                           Unit Test Gate 验证
                                                 │
                                            通过→接受 / 失败→回滚
```

## 本章结构

| 节 | 主题 | 核心问题 |
|---|------|---------|
| [12.1 Hermes Agent — Skill 系统](./121-hermes-skill-system.md) | Online Skill Evolution | 创建→存储→调用→patch→共享的完整生命周期，模糊匹配引擎，安全门控 |
| [12.2 Memento-S — Evolve 系统](./122-memento-evolve-system.md) | Offline Skill Evolution | 评测→判分→归因→优化→验证的闭环 pipeline，三种 Skill Track，混合检索 |
| [12.3 跨系统对比](./123-cross-system-comparison.md) | Design Space Analysis | 两种范式的本质差异、各自适用场景、潜在融合方向 |

## 涉及的代码

### Hermes Agent

| 文件 | 职责 |
|------|------|
| `tools/skill_manager_tool.py` | Skill 创建/编辑/patch/删除，原子写入 |
| `tools/skills_tool.py` | Skill 加载/列表/查看，渐进式披露 |
| `tools/skills_guard.py` | 安全扫描（80+ 威胁模式），信任分级 |
| `tools/skills_hub.py` | 注册表适配器，来源追踪 |
| `tools/skills_sync.py` | 内置 skill 同步，manifest 管理 |
| `tools/fuzzy_match.py` | 9 策略模糊匹配引擎 |
| `agent/skill_commands.py` | 斜杠命令调用，skill 消息构建 |
| `agent/skill_utils.py` | Frontmatter 解析，配置变量注入 |

### Memento-S

| 文件 | 职责 |
|------|------|
| `core/evolve/main.py` | Evolve 主循环编排 |
| `core/evolve/judge.py` | LLM 判分（二值 + 0-10 软分） |
| `core/evolve/feedback.py` | 失败归因、tip 提取、skill 选择 |
| `core/evolve/optimizer.py` | LLM 驱动的 skill 重写 |
| `core/evolve/unit_test_gate.py` | 测试生成 + 回归验证 + 门控决策 |
| `core/evolve/dataset.py` | 数据集归一化、附件处理、prompt 构建 |
| `core/evolve/tracker.py` | 实验配置、准确率曲线、进度追踪 |
| `core/skills/provider/delta_skills/` | 三种 Track（Code/Knowledge/Playbook） |
| `core/skills/provider/delta_skills/retrieval/hybrid.py` | BM25 + Embedding 混合检索 |
| `core/skills/provider/delta_skills/skills/creator/` | Skill 自动创建 |
| `core/skills/provider/delta_skills/skills/auditor/` | 安全审计 + 合规检查 |

---

> **下一节**：[12.1 Hermes Agent — Skill 系统](./121-hermes-skill-system.md)
