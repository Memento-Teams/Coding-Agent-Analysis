# 11.7 Hermes Agent — 上下文管理

> Hermes Agent 的上下文管理和 Claude Code 解决的是同一个问题——有限的窗口，无限的对话。但两者的架构哲学截然不同：Claude Code 用五层管线"从轻到重"逐级压缩，Hermes 用三层防线"先截后存再压"。

## 11.7.1 整体架构：可插拔的 ContextEngine

Hermes 的上下文管理围绕一个**抽象基类** `ContextEngine` 构建（`agent/context_engine.py`），核心生命周期：

```
session_start → [每轮: update_from_response → should_compress? → compress] → session_end
```

这是一个**插件化设计**——通过 `context.engine` 配置切换实现。默认实现是 `ContextCompressor`（有损摘要压缩），但可以替换为 LCM（Large Context Model）等其他引擎。引擎甚至可以暴露自己的工具给 agent 调用（如 `lcm_grep`、`lcm_expand`）。

```python
# agent/context_engine.py
class ContextEngine(ABC):
    threshold_percent: float = 0.75    # 触发压缩的阈值
    protect_first_n: int = 3           # 保护头部消息数
    protect_last_n: int = 6            # 保护尾部消息数

    @abstractmethod
    def should_compress(self, prompt_tokens: int = None) -> bool: ...
    @abstractmethod
    def compress(self, messages, current_tokens=None) -> list: ...

    # 可选：引擎可以暴露工具给 agent
    def get_tool_schemas(self) -> list: return []
    def handle_tool_call(self, name, args, **kwargs) -> str: ...
```

**与 Claude Code 的关键差异**：Claude Code 的五层压缩管线是硬编码在 `query.ts` 中的固定流程，没有抽象层。Hermes 的 ContextEngine 允许整套上下文管理策略被替换——这是一个更加面向插件的设计。

## 11.7.2 三层工具结果防线（Tool Result Storage）

在 Hermes 中，工具结果的处理不是压缩管线的一部分，而是一套**独立的三层防溢出系统**（`tools/tool_result_storage.py` + `tools/budget_config.py`）：

```
工具返回结果
  │
  │  Layer 1: 工具内部预截断
  │  ─────────────────────
  │  各工具自己控制输出大小
  │  （如 search_files 限制结果条数）
  │
  ▼
  │  Layer 2: 单结果持久化 (maybe_persist_tool_result)
  │  ─────────────────────
  │  单个结果 > 100K chars？
  │    → 写入沙箱 /tmp/hermes-results/{tool_use_id}.txt
  │    → 上下文替换为 <persisted-output> 预览 + 文件路径
  │    → 模型可用 read_file 按需回看
  │
  ▼
  │  Layer 3: 单轮聚合预算 (enforce_turn_budget)
  │  ─────────────────────
  │  一轮所有结果合计 > 200K chars？
  │    → 从最大的开始溢出到文件
  │    → 直到总量降到 200K 以下
  │
  ▼
  结果进入对话上下文
```

### Layer 2 的详细流程

当工具返回超大结果时，替换为结构化的提示：

```
<persisted-output>
This tool result was too large (150,000 characters, 146.5 KB).
Full output saved to: /tmp/hermes-results/call_abc123.txt
Use the read_file tool with offset and limit to access specific sections.

Preview (first 1500 chars):
... 前 1500 字符 ...
</persisted-output>
```

几个精巧的设计：

**沙箱写入**：通过 `env.execute()` 写入，不是直接 `open()` 文件。这意味着无论后端是本地、Docker、SSH、Modal 还是 Daytona，都能工作：

```python
# 通过 heredoc 写入沙箱
cmd = f"mkdir -p {storage_dir} && cat > {remote_path} << '{marker}'\n{content}\n{marker}"
result = env.execute(cmd, timeout=30)
```

**read_file 豁免**：防止死循环

```python
# budget_config.py
PINNED_THRESHOLDS = {
    "read_file": float("inf"),  # 永远不持久化 read_file 结果
}
```

如果不做这个豁免：模型读持久化文件 → 结果又被持久化 → 模型再读 → 无限循环。

**阈值优先级**：`pinned（硬编码）> tool_overrides（配置）> registry per-tool > default（100K）`

### 与 Claude Code Layer 1 的对比

| 维度 | Claude Code (Layer 1) | Hermes (Layer 2) |
|------|----------------------|------------------|
| 阈值 | 30K 字符 | 100K 字符 |
| 存储位置 | `~/.claude/tool-results/{hash}.txt` | `/tmp/hermes-results/{tool_use_id}.txt` |
| 写入方式 | 本地文件系统直写 | `env.execute()` 沙箱写入 |
| 跨后端 | 仅本地 | Docker/SSH/Modal/Daytona 全兼容 |
| 预览 | 一行摘要 | 1500 chars 预览 |
| 聚合预算 | 无（单条处理） | 有（Layer 3，200K 单轮预算） |
| 回退 | — | 写入失败时纯截断 |

Hermes 的 Layer 3（聚合预算）是 Claude Code 没有的——它解决的是"每个工具结果都不大，但一轮调了 20 个工具，合计炸上下文"的场景。

## 11.7.3 ContextCompressor：有损摘要压缩

默认的上下文引擎 `ContextCompressor`（`agent/context_compressor.py`）的压缩算法分四个阶段：

```
Phase 1: 廉价预处理（无 LLM 调用）
────────────────────────────────
  _prune_old_tool_results()
  旧的 tool result（>200 字符）→ "[Old tool output cleared to save context space]"
  基于 token 预算从尾部向前扫描确定保护边界

Phase 2: 确定压缩边界
────────────────────────────────
  头部保护：前 N 条消息（系统提示 + 首轮交互）
  尾部保护：基于 token 预算（~20K tokens 最近内容）
  边界对齐：不切断 tool_call/result 组

Phase 3: 结构化摘要
────────────────────────────────
  用辅助模型生成结构化摘要，模板 8 个章节：
    ## Goal
    ## Constraints & Preferences
    ## Progress (Done / In Progress / Blocked)
    ## Key Decisions
    ## Relevant Files
    ## Next Steps
    ## Critical Context
    ## Tools & Patterns

Phase 4: 组装与修复
────────────────────────────────
  组合：[头部消息] + [摘要消息] + [尾部消息]
  修复孤立的 tool_call/result 对
  调整摘要消息角色避免连续同角色冲突
```

### 迭代式摘要更新

这是 Hermes 压缩中最精巧的设计之一。第二次及之后的压缩**不是从头摘要**，而是在前一次摘要基础上**增量更新**：

```python
# 首次压缩
self._previous_summary = None
→ 生成全新摘要

# 第二次压缩
self._previous_summary = "上次的摘要内容..."
→ Prompt: "PREVIOUS SUMMARY: {old} + NEW TURNS: {new}"
→ "Update the summary... PRESERVE all existing information... ADD new progress"
```

**为什么这很重要？** 假设第一次压缩在第 50 轮触发，摘要了 1-35 轮的内容。到第 100 轮时第二次压缩触发，如果从头摘要 1-85 轮，之前的摘要会被完全重写——第一次压缩保留的细节可能在第二次中丢失。迭代更新确保了信息的**渐进累积而非重复丢失**。

**与 Claude Code 的对比**：Claude Code 的 Auto-Compact（变体 A）每次都是全量重写（"整个对话压缩为 9 节摘要"）。Session Memory 承担了跨压缩连续性的角色。Hermes 没有 Session Memory，所以靠 `_previous_summary` 在压缩层内部解决这个问题。

### 摘要预算缩放

```python
_MIN_SUMMARY_TOKENS = 2000
_SUMMARY_RATIO = 0.20          # 压缩内容的 20%
_SUMMARY_TOKENS_CEILING = 12_000

budget = max(2000, min(content_tokens * 0.20, context_length * 0.05))
```

大模型上下文（如 200K）的摘要预算更大（最高 12K tokens），小模型的摘要更精简。Claude Code 固定 9 节结构，没有按模型缩放。

### 尾部保护：Token 预算 vs 固定消息数

```python
# Hermes: 基于 token 预算的尾部保护
def _find_tail_cut_by_tokens(self, messages, head_end, token_budget=None):
    # 从尾部向前累加 token，直到超预算
    # 软上限允许 1.5x 超预算（避免切在大消息中间）
    # 硬底线：至少保护 3 条消息
```

**与 Claude Code 的对比**：

| 维度 | Claude Code | Hermes |
|------|------------|--------|
| 尾部保护策略 | 不在 Compact 层做尾部保护（Context Collapse 变体 B 做部分保留） | Token 预算制（默认 `threshold * ratio` ≈ 20K tokens） |
| 边界对齐 | 消息分组（grouping.ts），tool_use + result 不可拆 | `_align_boundary_forward/backward`，同样保证 tool 组完整 |
| 保护最低限 | 无硬底线 | 至少 3 条消息 |

### Tool Pair 修复

压缩后可能出现孤立的 tool_call 或 tool_result。Hermes 的 `_sanitize_tool_pairs()` 处理两种情况：

```
情况 1：tool_result 的 call_id 没有对应的 assistant tool_call
  → 删除这些孤立的 tool_result

情况 2：assistant tool_call 的 result 被压缩掉了
  → 插入 stub: "[Result from earlier conversation — see context summary above]"
```

Claude Code 通过分组（grouping.ts）在压缩前就保证了 tool_use/result 不被拆散，所以不需要事后修复。Hermes 的方式更宽松——先压缩再修复，容错性更强但逻辑更复杂。

## 11.7.4 阈值与触发机制

```python
# Hermes
threshold_percent = 0.50          # 默认在 50% 时触发
threshold_tokens = context_length * 0.50

# Claude Code
effectiveWindow = context_length - 20K     # ~90%
threshold = effectiveWindow - 13K          # ~83.5%
```

Hermes 的默认阈值（50%）远低于 Claude Code（~83.5%）。这意味着 Hermes 更早触发压缩，但压缩后保留的尾部上下文比例更大。Claude Code 尽量不压缩（"能不压就不压"），Hermes 更倾向于提前压缩。

### 容错机制

| 维度 | Claude Code | Hermes |
|------|------------|--------|
| 压缩失败 | 连续 3 次失败后熔断，禁用自动压缩 | 进入 600 秒冷却期，期间直接丢弃中间 turns（带静态提示） |
| 413 恢复 | 先 Context Collapse，再 Reactive Compact | 依赖 `should_compress_preflight()` 预检 |
| 摘要失败 | — | 插入静态 fallback："Summary generation was unavailable. N turns were removed..." |

## 11.7.5 Prompt Caching

Hermes 对 Anthropic 模型使用 `system_and_3` 缓存策略（`agent/prompt_caching.py`）：

```python
# 4 个 cache_control breakpoint（Anthropic 上限）：
# 1. 系统提示（稳定，跨所有轮次）
# 2-4. 最近 3 条非系统消息（滑动窗口）
```

**与 Claude Code 的对比**：Claude Code 的 Compact 是 Fork 子 Agent 完成的——子 Agent 共享父会话的 Prompt Cache。Hermes 的压缩用的是独立的辅助模型（便宜/快速），不共享 Cache。

## 11.7.6 Context References：@引用语法

Hermes 提供了 `@` 引用语法（`agent/context_references.py`），在用户消息发送前展开：

```
@file:src/app.ts          → 注入文件内容（支持 :10-20 行范围）
@folder:src/components     → 注入目录树
@diff / @staged            → 注入 git diff
@git:3                     → 注入最近 3 次 commit 的 patch
@url:https://...           → 抓取网页内容
```

**安全措施**：
- 路径沙箱：限制在工作目录内
- 敏感文件硬编码拒绝（`.ssh`、`.env`、`.netrc` 等）
- Token 限制：硬上限 50% context_length，软警告 25%

Claude Code 没有 `@` 引用语法——文件和 URL 的上下文注入通过工具调用（FileRead、WebFetch）完成，不是在用户消息预处理阶段。

## 11.7.7 小结：Hermes 上下文管理的设计哲学

| 设计选择 | 解释 |
|---------|------|
| 插件化 ContextEngine | 上下文策略可替换，不绑定单一实现 |
| 工具结果独立处理 | 三层防线与压缩管线解耦，各管各的 |
| 迭代式摘要 | 不依赖 Session Memory，在压缩层内部保证跨压缩连续性 |
| 沙箱写入 | 适配多后端（Docker/SSH/Modal），不假设本地文件系统 |
| 低阈值早压缩 | 50% 就压，但保留更多尾部上下文 |
| 辅助模型做摘要 | 用便宜模型降成本，不 Fork 子 Agent |

---

> **下一节**：[11.8 Hermes Agent — 记忆系统](./118-hermes-memory-system.md)
