# 12.1 Hermes Agent — Skill 系统

> Hermes 的 Skill 是 Agent 在工作中创建的"操作手册"——Markdown 文档，记录着完成某类任务的步骤、陷阱和模板。Agent 在后续使用中发现手册有误，当场修补。这是一个"边用边改"的在线演化模式。

## 12.1.1 创建：从复杂任务到 Skill

### 何时创建

Agent 的系统提示（`agent/prompt_builder.py`）中明确列出触发条件：

- 完成了一个 **5+ 工具调用**的复杂任务
- 修复了一个 tricky error 或发现了 workaround
- 发现了一个非显然的 workflow
- 用户纠正了 agent 的做法

这些是写进 system prompt 的**行为指令**，不是建议。Agent 在满足条件时会主动调用 `skill_manage(action='create')`。

### 创建流程

```
Agent 完成复杂任务
  │
  ├── 调用 skill_manage(action='create', name='docker-local-dev-setup', ...)
  │
  ├── 验证链：
  │   ├── 名称校验：小写、连字符/下划线、≤64 字符、文件系统安全
  │   ├── 分类校验：单层目录名，无路径穿越
  │   ├── Frontmatter 校验：YAML 语法、必填字段（name, description）
  │   ├── 内容大小：SKILL.md ≤ 100K chars，支撑文件 ≤ 1MB each
  │   └── 安全扫描：skills_guard.py 检测 80+ 威胁模式
  │
  ├── 原子写入：tempfile.mkstemp() + os.replace()
  │   （进程崩溃不会写出半截文件）
  │
  └── 清除 Skill prompt cache（下次加载时刷新）
```

## 12.1.2 存储结构

```
~/.hermes/skills/
├── category/
│   └── skill-name/
│       ├── SKILL.md              ← 主文件（必须）
│       ├── references/           ← 参考文档
│       │   └── api-guide.md
│       ├── templates/            ← 输出模板
│       ├── scripts/              ← 辅助脚本
│       └── assets/               ← 附件
├── .hub/                         ← Skills Hub 状态
│   ├── lock.json                 ← 来源追踪
│   ├── quarantine/               ← 安装前隔离
│   └── audit.log                 ← 安全审计历史
└── .bundled_manifest             ← 内置 skill 同步 manifest
```

### SKILL.md 格式

```yaml
---
name: docker-local-dev-setup
description: Set up local dev environment with Docker Compose
version: 1.2.0
author: hermes-agent
platforms: [macos, linux]           # 平台限制
required_environment_variables:     # 需要的环境变量
  - name: DOCKER_REGISTRY
    prompt: Enter Docker registry URL
metadata:
  hermes:
    tags: [docker, devops]
    requires_toolsets: [terminal]    # 条件激活
    fallback_for_toolsets: [web]     # 当 web 不可用时激活
    config:                         # 可配置参数
      - key: docker.registry
        default: "docker.io"
---

## When to Use
当需要为新项目配置 Docker 开发环境时...

## Procedure
1. 检查 Docker 是否安装...
2. 创建 docker-compose.yml...
3. 构建镜像（注意加 --no-cache）...

## Pitfalls
- macOS 上 volume mount 需要...
- 端口冲突时先检查...

## Verification
运行 `docker ps` 确认容器状态...
```

### 只有一种类型

与 Memento-S 的三种 Track（Code/Knowledge/Playbook）不同，Hermes 的 Skill **只有一种格式**——YAML frontmatter + Markdown 正文。执行方式是注入 system prompt，由 Agent 阅读后执行。Skill 本质是**给 Agent 读的操作手册**，不是可自动执行的代码。

## 12.1.3 调用：渐进式信息披露

Skill 加载采用三级渐进策略，避免一次性占满上下文：

```
Level 0: skills_list()
  → 只返回 name + description（~3K tokens 列出全部 skill）
  → Agent 看到"有什么可用的"

Level 1: skill_view(name)
  → 返回完整 SKILL.md + 关联文件列表
  → Agent 看到"怎么做"

Level 2: skill_view(name, file_path="references/api.md")
  → 返回特定参考文件
  → Agent 看到"需要的细节"
```

### 调用路径

1. **斜杠命令**：`/docker-local-dev-setup configure my project`
   - `scan_skill_commands()` 发现所有已安装 skill
   - `build_skill_invocation_message()` 加载 skill 并注入用户消息
   - 配置变量自动从 `config.yaml` 注入

2. **Session 预加载**：`hermes chat --skills skill1,skill2`
   - `build_preloaded_skills_prompt()` 在会话开始时加载
   - 整个会话期间作为 active guidance

3. **自然语言**：Agent 通过 `skill_view()` 工具按需加载

### 注入格式

```
[SYSTEM: The user has invoked the "docker-local-dev-setup" skill...]

<full SKILL.md content>

[Skill config (from ~/.hermes/config.yaml):
  docker.registry = docker.io
]

[This skill has supporting files: references/api-guide.md, templates/...]
```

## 12.1.4 自我改进：即时 Patch

### 触发

系统提示中写道：

> *When using a skill and finding it outdated, incomplete, or wrong, patch it immediately with `skill_manage(action='patch')` — don't wait to be asked. Skills that aren't maintained become liabilities.*

### Patch 机制

```python
skill_manage(
    action='patch',
    name='docker-local-dev-setup',
    old_string='Step 3: Build the image\nRun: docker build .',
    new_string='Step 3: Build the image\nRun: docker build -t myapp:dev --no-cache .',
)
```

`patch` 是定向查找替换，不是全量重写。问题在于 LLM 生成的 `old_string` 经常和实际文本有微妙差异。

### 9 策略模糊匹配引擎

`tools/fuzzy_match.py` 从严到松依次尝试 9 种匹配策略：

```
策略 1: 精确匹配
策略 2: 行级 trim（去每行首尾空格）
策略 3: 空白归一化（多空格 → 单空格）
策略 4: 忽略缩进
策略 5: 转义符归一化（\\n → \n）
策略 6: 边界 trim（只处理首尾行）
策略 7: Unicode 归一化（智能引号 → ASCII，em-dash → --）
策略 8: 块锚定（首尾行匹配 + 中间相似度）
策略 9: 上下文感知（50% 行相似度阈值）
```

找到唯一匹配即替换。如果没找到，返回文件内容预览帮助 Agent 自我纠正。

### 无验证、无回滚

Patch 后直接生效。没有测试，没有回归验证，没有自动回滚。质量完全取决于 Agent 当时的判断。

### 其他改进方式

| Action | 用途 | 场景 |
|--------|------|------|
| `patch` | 定向查找替换 | 修正具体步骤（首选） |
| `edit` | 全量重写 SKILL.md | 大幅重构 |
| `write_file` | 添加/更新支撑文件 | 新增参考文档、模板 |
| `remove_file` | 删除支撑文件 | 清理过时内容 |

## 12.1.5 安全门控

`tools/skills_guard.py` 在每次创建和修改时扫描 **80+ 种威胁模式**：

**外泄类**：`curl ... $API_KEY`、`cat ~/.ssh/id_rsa`、DNS 外泄、/tmp 暂存后外传

**注入类**：`ignore previous instructions`、`system prompt override`

**破坏类**：`rm -rf /`、`dd if=/dev/urandom`

**持久化类**：crontab 安装、SSH 密钥注入、AWS/GCP SDK 滥用

### 信任分级策略

| 来源 | safe | caution | dangerous |
|------|------|---------|-----------|
| builtin（内置） | 允许 | 允许 | 阻止 |
| trusted（openai/anthropic） | 允许 | 允许 | 阻止 |
| community（社区） | 允许 | 阻止 | 阻止 |
| agent-created（Agent 创建） | 允许 | 询问用户 | 阻止 |

Agent 创建的 skill 如果触发 caution 级别（如包含 `subprocess.run`），会弹窗让用户确认。

## 12.1.6 共享：Skills Hub

Hermes 的 Skill 可以通过多个注册表共享：

```
来源：
├── 官方可选 skill（repo 内 optional-skills/）
├── skills.sh（Vercel 公共目录）
├── .well-known/skills/index.json（URL 发现）
├── GitHub 直连（openai/skills、anthropics/skills、自定义 tap）
├── ClawHub（第三方市场）
├── Claude marketplace
└── LobeHub（Agent 目录集成）
```

### 来源追踪（lock.json）

```json
{
  "skills": {
    "docker-local-dev-setup": {
      "source": "user/my-skills/docker-local-dev-setup",
      "installed_at": "2026-01-15T10:30:00Z",
      "content_hash": "abc123...",
      "trust_level": "community"
    }
  }
}
```

`content_hash` 用于检测上游更新和区分用户自定义修改。

### 内置 Skill 同步（skills_sync.py）

```
.bundled_manifest 记录每个内置 skill 的 origin hash：

新 skill（不在 manifest 中）→ 复制，记录 hash
已有且未被用户改过（hash 匹配）→ 安全更新
已有但被用户改过（hash 不匹配）→ 跳过，不覆盖
用户已删除（manifest 有，磁盘无）→ 尊重删除，不恢复
上游已移除（manifest 有，repo 无）→ 清理 manifest
```

## 12.1.7 完整生命周期示例

```
Day 1: 用户让 agent 配置 Docker 开发环境
  │  agent 反复试了 8 次（端口冲突、volume 权限、网络配置）
  │  → 任务完成后，agent 自动 skill_manage(action='create')
  │  → 安全扫描通过 → 写入 ~/.hermes/skills/devops/docker-local-dev-setup/

Day 3: 另一个项目也需要 Docker 环境
  │  → /docker-local-dev-setup
  │  → agent 加载 skill 按步骤执行
  │  → 发现 Step 3 少了 --no-cache，build 用了旧缓存
  │  → agent 立即 skill_manage(action='patch')
  │  → 模糊匹配定位成功 → SKILL.md 更新

Day 7: 又一个项目
  │  → 加载已修补的 skill
  │  → Step 5 的 volume mount 对 macOS 不适用
  │  → agent 再次 patch + write_file 添加 references/macos-gotchas.md

Day 30: skill 被修补了 5 次
  → 涵盖了端口冲突、缓存、macOS 兼容性、权限问题
  → 比 Day 1 初始版本完善得多
  → 其他用户可通过 Skills Hub 安装
```

---

> **下一节**：[12.2 Memento-S — Evolve 系统](./122-memento-evolve-system.md)
