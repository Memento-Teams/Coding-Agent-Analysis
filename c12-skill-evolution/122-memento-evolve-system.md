# 12.2 Memento-S — Evolve 系统

> Memento-S 的 Evolve 是一个离线自动化 pipeline：批量跑 benchmark → LLM 判分 → 归因到具体 skill → LLM 重写 → 单元测试验证 → 通过则接受，失败则回滚。它不需要人参与——每一轮 evolve 自动发现弱点、修复弱点、验证修复。

## 12.2.1 端到端流程

```
数据集（500 道题）
  │
  ▼ 逐题执行
┌─────────────────────────────────────────────────────────┐
│ execute_task_once()                                      │
│   Agent 执行任务 → 收集完整 trace（哪些 skill 被调用）    │
│   → 返回 answer + used_skills + trace                   │
└────────────┬────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────┐
│ judge_answer()                                          │
│   LLM Judge（o3-mini）对比 model_answer vs gold_answer  │
│   → is_correct? score(0-10)? rationale?                 │
└────────────┬────────────────────────────────────────────┘
             │
     ┌───────┴───────┐
     │ 正确          │ 错误 + optimize_on_error 开启
     ▼               ▼
   记录通过    ┌──────────────────────────────────────────┐
               │ select_target_skill_for_feedback()        │
               │   LLM Selector 分析 trace + rationale     │
               │   → 归因到具体 skill（confidence 0-1）     │
               └────────────┬─────────────────────────────┘
                            │
                            ▼
               ┌──────────────────────────────────────────┐
               │ optimize_skill_with_trajectory()          │
               │   构建 feedback blob → LLM 重写 skill     │
               │   → 备份原始 → 应用更新                    │
               └────────────┬─────────────────────────────┘
                            │
                            ▼
               ┌──────────────────────────────────────────┐
               │ Unit Test Gate                            │
               │   生成 3-5 测试 + 回归测试                 │
               │   加权评分 → 通过？                        │
               │     是 → 接受改进                          │
               │     否 → 从备份回滚                        │
               └────────────┬─────────────────────────────┘
                            │
                            ▼
               ┌──────────────────────────────────────────┐
               │ 可选：feedback_retry                      │
               │   用改进后的 skill 重跑同题                │
               │   修好了？→ 记录 fixed_after_feedback      │
               │   还是错？→ 下一轮 feedback（≤max_rounds） │
               └──────────────────────────────────────────┘
```

### 三种实验配置

| Profile | create_on_miss | optimize | 用途 |
|---------|---------------|----------|------|
| read-only | False | False | 纯评测（不改任何 skill） |
| read-write | True | False | 只创建缺失的 skill |
| read-write-optimize | True | True | **完整 evolve 闭环** |

## 12.2.2 三种 Skill Track

与 Hermes 只有一种 Markdown 格式不同，Memento-S 有三种可执行的 Skill 类型：

### CODE Track

纯 Python 代码，在沙箱中确定性执行。

```
execution/tracks/code_track.py

执行路径：
  确定性执行（首选）:
    解析代码 → 找到入口函数
    → 自动构建 runner（import/embed/script 三种模式）
    → 沙箱内直接运行

  LLM Fallback:
    确定性失败 → LLM 生成执行 wrapper
    → 可反射重试（修 skill 代码 vs 修执行代码）

用途：数值计算、数据变换、文件处理
参数：从函数签名自动提取
```

### KNOWLEDGE Track

Markdown 指令 + 可选的代码块。

```
execution/tracks/knowledge.py

执行路径：
  预构建命令:
    解析 SKILL.md 中的 bash/python 代码块
    → 转为 ops（run_command, read_file, write_file, ...）
    → 沙箱执行

  Direct LLM:
    LLM 读 SKILL.md → 生成 operations JSON
    → 支持动态操作：文件 I/O、目录树、子进程等

用途：研究、知识检索、信息综合
参数：request（必填）+ 可选 code override
```

### PLAYBOOK Track

多脚本编排目录。

```
execution/tracks/playbook.py

结构：
  skill-dir/
  ├── SKILL.md
  ├── scripts/
  │   ├── step1_prepare.py
  │   └── step2_analyze.py
  └── references/

执行：
  选择脚本（按名称或自动检测入口）
  → 复制 scripts/ + references/ + assets/ 到临时工作区
  → CLI 参数传递 → 执行 → 收集 artifacts

用途：复杂工作流、多阶段 pipeline
参数：script（必填）、args（CLI 参数列表）
```

### 核心差异

Hermes 的 Skill 是给 Agent 读的文档——Agent 看完后用工具手动执行。Memento-S 的 Skill 是可编程执行的代码/流程——系统自动在沙箱中运行，输出结构化结果。

## 12.2.3 判分系统

### 二值判分

```python
# judge.py
judge_answer(question, model_answer, expected_answer)
→ LLM（o3-mini）对比答案
→ 输出：{
    is_correct: bool,
    extracted_final_answer: str,   # 从模型回答中提取最终答案
    reasoning: str,                # 为什么对/错
    confidence: 0-100              # 置信度
  }
```

### 软评分（0-10）

```
judge_answer_soft_score_0_to_10()

LLM 评分标准：
  10: 精确匹配
  8-9: 语义等价
  6-7: 大部分正确
  0-5: 错误

混合评分：max(llm_score, heuristic_score)
  heuristic 用 token Jaccard + 字符相似度
  → 防止 LLM 评分过严
```

### 重试与容错

```
JSON 解析失败 → 最多重试 3 次
LLM 超时 → catch + 记录异常
全失败 → 标记 skipped_exception
```

## 12.2.4 失败归因

当判分结果为错误时，系统需要确定**哪个 skill 导致了失败**。

### LLM Selector

```python
# feedback.py
select_target_skill_for_feedback(
    candidate_skills,     # trace 中使用过的 skill 列表
    execution_trace,      # 完整的执行过程
    judge_payload         # 判分结果 + 理由
)
```

输出结构：

```json
{
  "skill_name": "math_solver",
  "failure_type": "execution",     // routing / execution / tool / unknown
  "reason": "Step 3 的 skill 调用返回了 NaN，导致后续计算错误",
  "optimisation_suggestion": "在除法前添加零值检查",
  "confidence_0_to_1": 0.85
}
```

四种 failure_type：
- **routing**：选错了 skill（应该用 B 但用了 A）
- **execution**：skill 执行出错（代码异常、逻辑错误）
- **tool**：skill 调用的外部工具失败
- **unknown**：无法判断

### 置信度门控

```
confidence < threshold → 不优化（避免在归因不确定时乱改）
```

### Utility Tracking

```python
# tracker.py: UtilityTracker
per-skill 统计:
  success_count / failure_count → utility ratio

if utility < 0.2 AND n_samples >= 3:
    → should_discover(skill_name) = True
    → 不再修补这个 skill，而是尝试发现替代实现
```

低效 skill 不会被无限修补——系统会放弃它，转而寻找替代方案。

## 12.2.5 优化器

### Feedback Blob 构建

```json
// optimizer.py: build_failure_feedback_blob
{
  "task": {
    "id": "task_042",
    "question": "Compute the integral...",
    "answer_type": "numeric",
    "category": "math"
  },
  "user_text": "用户输入",
  "previous_model_answer": "模型的错误答案",
  "judge": {
    "is_correct": false,
    "score": 0.0,
    "rationale": "答案差了一个数量级..."
  },
  "trace": [
    {"step_num": 1, "skill_name": "math_solver", "status": "done", "result_preview": "..."},
    {"step_num": 2, "skill_name": "math_solver", "status": "error", "result_preview": "NaN"}
  ],
  "skill_judgement": {
    "evidence_steps": [1, 2],
    "failure_type": "execution",
    "optimisation_suggestion": "在除法前检查分母"
  }
}
```

### LLM 重写

```
LLM Optimizer 接收:
  所有 skill 文件内容（代码、SKILL.md、helpers）
  + 完整 feedback blob

LLM 输出:
  { "updates": [
      {"path": "SKILL.md", "content": "...updated..."},
      {"path": "helper.py", "content": "...updated..."}
  ]}

验证:
  path 必须相对路径，无 ".." 段
  必须在 skill 目录内
  content 非空
  去除 markdown fences
```

### 备份与应用

```
apply_skill_folder_updates():
  ① 创建备份：backups/{skill_name}/bundle.{timestamp}
  ② 逐文件应用更新
  ③ 记录 updated_files, created_files
  ④ 如果后续 Unit Test Gate 失败 → 从备份恢复
```

## 12.2.6 Unit Test Gate

这是 Memento-S 最独特的设计——用自动生成的测试来验证 skill 改进没有引入回归。

### 四阶段流程

```
阶段 1：生成多样化测试用例
────────────────────────
  5 个类别各生成 1 个：
    core_functionality  — 核心功能
    edge_case          — 边界情况
    input_variation    — 输入变体
    error_handling     — 错误处理
    complex_scenario   — 复杂场景

  LLM 生成：{"question": "...", "expected_answer": "..."}
  去重：不生成和已有用例重复的
  兜底：如果全部生成失败 → 合同烟雾测试
    "Call {skill_name} at least once and return '{skill_name}'"

阶段 2：加载回归用例
────────────────────────
  从 regression_cases.jsonl 加载最近 2 个通过的用例
  回归失败 → gate 直接失败（hard gate 模式）

阶段 3：逐用例执行 + 评分
────────────────────────
  每个用例：
    execute_task_once(agent, question)
    → judge_answer_soft_score_0_to_10(expected)
    → 加权评分：
        answer_correct:     70%（score > threshold?）
        used_target_skill:  20%（调了目标 skill?）
        trace_done:         10%（执行无异常?）
    → passed = 三项全满足

阶段 4：聚合决策
────────────────────────
  gate_passed =
    回归用例全部通过（hard gate 时）
    AND
    生成测试通过率 ≥ 50%

  通过 → 保存通过的用例到回归缓存（下次可复用）
  失败 → 从备份恢复 skill，标记优化失败
```

### 关键设计

- **多样性**：5 个类别防止测试偏向单一场景
- **回归缓存**：通过的测试自动积累，未来验证越来越严格
- **工具使用验证**：不仅检查答案对不对，还检查是否真的调了目标 skill（防止 Agent 绕过 skill 直接给答案）
- **软评分**：允许部分正确（6-7 分），不是非黑即白

## 12.2.7 Tips 系统：跨 Skill 的泛化知识

```python
# feedback.py
write_tip_for_failure(tip_path, judge_rationale, trace)
```

从失败中提取**不绑定具体 skill** 的通用规则，写入 TIP.md：

```
启发式提取规则：
  答案类型是 numeric → "注意数值精度和单位"
  Judge 提到 'format' → "保持输出结构稳定"
  Trace 步数到达上限 → "任务太复杂时拆分为子任务"
  有附件/图片 → "确保处理了所有附件内容"
```

TIP.md 注入到后续所有任务的 user_text 中：

```
Learning Tips (from previous runs):
- 数值答案注意精度和单位转换
- 如果 judge 提到格式问题，保持输出格式稳定
- [more tips...]
```

超过 3000 chars 时 LLM 自动压缩。这是一个**跨 skill 的元学习机制**——不改某个具体 skill，而是改 Agent 的行为习惯。

## 12.2.8 混合检索引擎

Memento-S 用 BM25 + Embedding 混合检索为任务匹配最相关的 skill。

### 检索管线

```
query（任务描述）
  │
  ├── BM25 召回（k*2 个候选）
  │   关键词匹配 skill name + description
  │
  ├── Embedding 召回（k*2 个候选）
  │   Chroma 向量搜索（余弦相似度）
  │   + 额外 query 变体的召回（取 max score）
  │
  ├── 动态权重（按 query 长度）
  │   1-2 tokens → BM25 0.7 / Emb 0.3
  │   3-5 tokens → BM25 0.5 / Emb 0.5
  │   6+ tokens  → BM25 0.3 / Emb 0.7
  │
  ├── Score-Aware RRF 融合
  │   归一化分数 → score factor → 自适应 K → 加权排名
  │
  ├── 可选：Cross-Encoder Reranker 二次精排
  │
  └── 过滤 min_score → 返回 top-k
```

### 为什么混合？

- 短 query（"sort array"）→ BM25 精确匹配更好
- 长 query（"how to fine-tune a language model with GRPO"）→ Embedding 语义理解更好
- 动态权重自动切换策略

## 12.2.9 Skill 创建器

当系统找不到匹配的 skill 时（`create_on_miss=True`），自动创建：

```
core/skills/provider/delta_skills/skills/creator/

extraction.py:
  从 Agent 的执行 trace 中提取：
  - 成功的工具调用序列
  - 使用的参数模式
  - 输出格式

creator.py:
  LLM 生成 SKILL.md + 可选代码文件
  → 选择 Track 类型（code/knowledge/playbook）
  → 生成参数 schema
  → 安全审计

prompts.py:
  创建 Prompt 模板
  → 注入 trace evidence
  → 要求生成可复用的 skill
```

## 12.2.10 准确率曲线

`tracker.py` 跟踪每轮 feedback 后的累积准确率：

```
feedback_round_accuracy_curve():

输出：
  Round 0: 500 tasks, 400 correct, 80.0%
  Round 1: 100 tasks reached, 85 correct, 85.0% (+50 newly fixed)
  Round 2: 50 tasks reached, 43 correct, 86.0% (+8 newly fixed)
  Round 3: 15 tasks reached, 13 correct, 86.6% (+3 newly fixed)
```

典型模式：第一轮 feedback 修复最多，后续轮次收益递减。这提供了**何时停止 evolve** 的量化依据。

---

> **下一节**：[12.3 跨系统对比](./123-cross-system-comparison.md)
