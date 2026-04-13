# 12.3 跨系统对比：Hermes Agent Skill vs Memento-S Evolve

> 两个系统都声称"自我改进"，但机制截然不同：一个靠匠人手感，一个靠工厂质检。这不是好坏之分，而是两种适用场景完全不同的范式。

## 12.3.1 范式差异：在线 vs 离线

```
Hermes Agent — "匠人模式"              Memento-S — "工厂模式"
──────────────────────                ──────────────────────

触发：用户交互中                       触发：Benchmark 批量评测
     Agent 使用 skill                       系统跑 500 道题
     主观发现问题                           客观判分
     当场修补                               归因→重写→验证
     直接生效                               通过才接受

节奏：偶发、不可预测                   节奏：系统化、每轮全量
质保：无                               质保：Unit Test Gate
回滚：无                               回滚：自动备份→失败回滚
```

**核心隐喻**：
- Hermes 像一个手工匠——在工作中发现刨子不好用，当场磨一磨。质量取决于手感。
- Memento-S 像一个 QA 工厂——产品（skill）上生产线，不合格品自动返工，返工后再过质检，不合格就回退。

## 12.3.2 Skill 定义

| 维度 | Hermes | Memento-S |
|------|--------|-----------|
| **本质** | 给 Agent 读的操作手册 | 可编程执行的代码/流程 |
| **格式** | 单一：YAML frontmatter + Markdown | 三种 Track：Code / Knowledge / Playbook |
| **执行方式** | 注入 system prompt → Agent 人肉执行 | 沙箱内自动执行（确定性 + LLM fallback） |
| **参数** | 非结构化（Agent 从 Markdown 理解意图） | JSON Schema（函数签名自动提取） |
| **存储** | `.md` 文件 + 支撑目录 | `.md` 文件 + Python 代码 + 脚本目录 |
| **大小限制** | 100K chars | 无硬限（由沙箱资源决定） |

### Skill 类型对比

```
Hermes（一种）:                    Memento-S（三种）:

  SKILL.md                          CODE Track
  ├── When to Use                   ├── 纯 Python
  ├── Procedure                     ├── 确定性执行 / LLM fallback
  ├── Pitfalls                      └── 数值计算、数据处理
  └── Verification
                                    KNOWLEDGE Track
  "读这个文档，                     ├── Markdown 指令 + 代码块
   然后按步骤做"                    ├── 解析命令 → ops / LLM 生成 ops
                                    └── 研究、检索、综合

                                    PLAYBOOK Track
                                    ├── scripts/ 多入口
                                    ├── CLI 参数 + artifact 收集
                                    └── 复杂多阶段 pipeline
```

Hermes 的设计更简单（一种格式覆盖所有），但**执行精度低**——Agent 可能误解操作手册。Memento-S 更复杂但**执行精度高**——Code Track 可以确定性运行，不依赖 Agent 的理解。

## 12.3.3 改进机制

### 改进触发

| 维度 | Hermes | Memento-S |
|------|--------|-----------|
| 触发源 | Agent 在使用中主观发现 | 自动化评测 pipeline |
| 频率 | 偶发 | 系统化（每轮 evolve 全量） |
| 覆盖面 | 只改 Agent 恰好用到且注意到的 | 所有失败 case 涉及的 skill |
| 成本 | 低（一次 text patch） | 高（Judge + Selector + Optimizer + Tests） |

### 改进过程

```
Hermes — 单步修补:

  Agent 发现问题
    → skill_manage(action='patch', old_string=..., new_string=...)
    → 模糊匹配定位 → 替换 → 完成
  
  总计：1 次 LLM 调用（Agent 自身的推理）


Memento-S — 五步 pipeline:

  ① judge_answer()                → 1 次 LLM 调用
  ② select_target_skill()         → 1 次 LLM 调用
  ③ build_failure_feedback_blob() → 纯计算
  ④ rewrite_skill_with_feedback() → 1 次 LLM 调用
  ⑤ unit_test_gate()              → N 次 LLM 调用（生成测试 + 判分）

  总计：4 + N 次 LLM 调用
```

### 改进质量保障

这是两者**最核心的差异**：

```
Hermes:                              Memento-S:
  patch → 直接生效                     rewrite → 备份 → Unit Test Gate
  无测试                                         ├── 生成 3-5 多样化测试
  无回归验证                                     ├── 回归测试（历史通过的）
  无回滚                                         ├── 加权评分（正确性 70%
  可能改好，可能改坏                             │   + skill 使用 20%
  完全依赖 Agent 判断                            │   + 执行无异常 10%）
                                                 ├── 通过率 ≥ 50% → 接受
                                                 └── 否则 → 自动回滚
```

Hermes 的风险：Agent patch 错了（比如改了一个步骤的同时破坏了另一个步骤），没有任何机制发现问题。用户下次使用这个 skill 时才会踩坑。

Memento-S 的保障：即使 LLM Optimizer 重写出了问题，Unit Test Gate 会拦住它。回归测试还确保之前能通过的 case 不会被新改动破坏。

## 12.3.4 失败归因

| 维度 | Hermes | Memento-S |
|------|--------|-----------|
| 归因者 | Agent 自己（使用者即归因者） | 专门的 LLM Selector |
| 输入 | Agent 的即时观察 | 完整 trace + judge rationale |
| 输出 | 无结构化归因（直接改） | {skill_name, failure_type, reason, confidence} |
| 置信度 | 无 | 0-1（低置信度时可选择不优化） |
| failure_type | 无分类 | routing / execution / tool / unknown |

Memento-S 的归因更精准，因为它有**完整的执行 trace** 和 **judge 的评分理由**作为证据。但也更昂贵（额外一次 LLM 调用）。

### Utility Tracking：放弃低效 skill

```
Memento-S 独有:
  每个 skill 记录 success/failure 比率
  
  utility < 0.2 AND n_samples >= 3:
    → 不再修补这个 skill
    → 尝试发现替代实现（discover_alternative_skill）
    
  Hermes 没有这个机制:
    一个 skill 被 patch 了 10 次还是不好用
    → Agent 继续修补（没有"放弃"的阈值）
```

## 12.3.5 知识泛化

| 维度 | Hermes | Memento-S |
|------|--------|-----------|
| 泛化机制 | 无 | TIP.md 跨 skill 通用规则 |
| 泛化内容 | — | "数值答案注意精度"、"格式问题保持输出稳定"等 |
| 注入方式 | — | 注入到后续所有任务的 user_text |
| 积累方式 | — | 自动提取 + 超长时自动压缩 |

Hermes 的改进**绑定具体 skill**——改了 docker-setup 的 Step 3，对 k8s-deploy 没有任何帮助。

Memento-S 的 TIP.md 是**跨 skill 的元知识**——从 math_solver 的失败中学到"注意精度"，这个 tip 会影响所有后续任务（包括使用其他 skill 的任务）。

## 12.3.6 检索

| 维度 | Hermes | Memento-S |
|------|--------|-----------|
| 方式 | 列表浏览 + 名称查看 | BM25 + Embedding 混合 + Reranker |
| 语义理解 | 无（Agent 看名字和描述判断） | 有（向量搜索） |
| 动态权重 | — | 按 query 长度自动调 BM25/Embedding 配比 |
| 成本 | 零（纯列表） | Embedding 计算 + 可选 Reranker |
| 来源 | 本地 + Hub | 本地 + 云端目录 |

## 12.3.7 安全

| 维度 | Hermes | Memento-S |
|------|--------|-----------|
| 扫描时机 | 创建/修改时 | 创建时（auditor） |
| 扫描范围 | 80+ 正则威胁模式 | security.py + compliance.py |
| 信任分级 | 4 级（builtin/trusted/community/agent-created） | 无显式分级（统一审计） |
| 路径穿越防护 | 有 | 有（Path.resolve().relative_to()） |
| 执行隔离 | 无（Agent 在宿主环境执行） | 沙箱（E2B 云沙箱 / 本地隔离） |

Memento-S 的**沙箱执行**是重要的安全差异——skill 代码在隔离环境中运行，即使有问题也不会影响宿主系统。Hermes 的 skill 由 Agent 在用户环境中执行，一条错误的 `rm` 命令就可能造成伤害。

## 12.3.8 共享

| 维度 | Hermes | Memento-S |
|------|--------|-----------|
| 机制 | Skills Hub（多源注册表） | 云端目录 + 导入适配器 |
| 来源 | skills.sh / GitHub / ClawHub / LobeHub / 自定义 | OpenSkills 格式导入 |
| 追踪 | lock.json（hash + timestamp + source） | 持久化层管理 |
| 更新检测 | content_hash 比对 | 版本管理 |
| 用户自定义保护 | hash 不匹配则跳过同步 | — |

## 12.3.9 成本对比

```
单次 Skill 改进成本:

Hermes:
  Agent patch: ~0（包含在当前对话 token 中，无额外 API 调用）

Memento-S:
  Judge:     ~1K tokens (output)
  Selector:  ~1K tokens (output)
  Optimizer: ~2K tokens (output)
  Gate tests: 3-5 × (execute + judge) ≈ 5-10K tokens
  ─────────────────────────────────
  总计：~10-15K tokens per failed case

批量 Evolve（500 题，80% 首次正确）:
  100 失败 × 15K = ~1.5M tokens
  + feedback retry rounds
  + 可能的 alternative discovery
  总计：~2-5M tokens per evolve cycle
```

## 12.3.10 适用场景

| 场景 | 推荐 | 理由 |
|------|------|------|
| 用户日常交互中的 skill 改善 | **Hermes** | 低延迟，零额外成本，即时反馈 |
| Benchmark 驱动的系统级优化 | **Memento-S** | 全量覆盖，质量保障，量化改进 |
| 安全敏感环境 | **Memento-S** | 沙箱执行，不会影响宿主 |
| 社区共享 skill | **Hermes** | 成熟的 Hub 生态 |
| Skill 从 0 到 1 创建 | 两者各有所长 | Hermes: Agent 从经验提取；Memento-S: 从 trace 自动生成 |
| Skill 从 1 到 N 精炼 | **Memento-S** | 有验证、有回归、有量化 |

## 12.3.11 潜在融合方向

两个系统不是互斥的。理论上可以构建一个两层 Skill 演化架构：

```
白天（在线）:
  Hermes 模式
  ├── Agent 使用 skill 服务用户
  ├── 发现问题即时 patch
  ├── 记录使用 trace 和失败信息
  └── 低成本、低延迟

晚上（离线）:
  Memento-S 模式
  ├── 收集白天的失败 trace
  ├── 批量跑 evolve pipeline
  ├── LLM 归因 + 重写 + Unit Test Gate
  ├── 通过的改进替换线上 skill
  └── 生成 TIP.md 通用规则

融合价值:
  ├── Hermes 的即时 patch 提供快速响应
  ├── Memento-S 的 pipeline 提供系统化质量提升
  ├── Unit Test Gate 防止白天的"手感 patch"引入回归
  └── TIP.md 让所有 skill 受益于每一次失败
```

## 12.3.12 总结

| 维度 | Hermes Agent | Memento-S Evolve |
|------|-------------|-----------------|
| **改进范式** | 在线手工匠 | 离线工程化 |
| **改进触发** | Agent 主观判断 | 评测 pipeline 客观驱动 |
| **Skill 类型** | 一种（Markdown 手册） | 三种 Track（Code/Knowledge/Playbook） |
| **执行精度** | 低（Agent 理解文档后手动做） | 高（沙箱确定性执行） |
| **质量保障** | 无 | Unit Test Gate + 回归测试 |
| **失败归因** | 隐式（Agent 自己判断） | 显式（LLM Selector + confidence） |
| **知识泛化** | 无 | TIP.md 跨 skill 元学习 |
| **检索** | 列表浏览 | 混合检索（BM25 + Embedding） |
| **安全** | 正则扫描 | 审计 + 沙箱执行 |
| **成本** | ~0 per patch | ~15K tokens per failed case |
| **定位** | 日常使用的渐进改善 | 系统级的批量优化 |

**一句话**：Hermes 让 skill 在使用中"自然生长"，Memento-S 让 skill 在评测中"工程进化"。前者快但不确定，后者慢但可靠。

---

> 返回 [Chapter 12 目录](./README.md) | 返回 [c00 总览](../c00-project-overview/README.md)
