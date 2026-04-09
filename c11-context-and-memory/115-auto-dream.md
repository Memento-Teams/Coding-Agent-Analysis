# 11.5 autoDream — 后台记忆整合

> 人在睡眠时会"整理"白天的记忆——巩固重要的，丢弃琐碎的。Claude Code 的 autoDream 做的是同样的事：在空闲时回顾记忆文件，合并碎片，修剪过时的内容，保持记忆系统的健康。

## 11.5.1 为什么需要 Dream？

随着时间推移，MemDir 中的记忆会出现几个问题：

```
问题 1：碎片化
────────────
  feedback_testing_1.md  — "不要 mock 数据库"
  feedback_testing_2.md  — "测试时用 docker-compose 起 PostgreSQL"
  feedback_testing_3.md  — "CI 里的测试也要用真实数据库"
  
  → 这三条可以合并为一条

问题 2：过时
────────────
  project_auth_rewrite.md  — "认证重构预计下周完成"
  
  → 三个月前写的，重构早已完成

问题 3：索引膨胀
────────────
  MEMORY.md 已经 180 行了，再加几条就超 200 行上限
  
  → 需要修剪低价值的条目

问题 4：相对日期失效
────────────
  project_release.md  — "merge freeze 从本周四开始"
  
  → "本周四"是什么时候？记忆写入后就不再准确
```

extractMemories（会话结束时触发）只负责"写入"——它把新信息存进来，但不管理已有的记忆。autoDream 是专门的"管理员"——**不创建新记忆，只优化已有的**。

## 11.5.2 触发条件

autoDream 通过 **CronTool 调度**触发，在 Claude Code 空闲时运行：

```
Claude Code 处于 Proactive/Autonomous 模式
  │
  ├── CronTool 每隔一段时间触发一次 tick
  ├── tick handler 检查：有没有有用的事情可做？
  │     ├── 有待处理的任务？→ 去做任务
  │     ├── 记忆需要整合？→ 触发 autoDream
  │     └── 什么都不需要？→ Sleep
  │
  └── autoDream 启动
```

### 整合锁

```tsx
// 防止并发做梦
const lock = await acquireConsolidationLock()
if (!lock) {
  return  // 已经有另一个 dream 在进行，跳过
}
try {
  await runDream()
} finally {
  releaseLock(lock)
}
```

为什么需要锁？在 Proactive 模式下，多个 tick 可能同时触发。如果两个 dream 同时修改记忆文件，会造成数据冲突。锁确保同一时间只有一个 dream 在运行。

## 11.5.3 四个阶段

autoDream 的执行分为严格的四个阶段：

```
┌─────────────────────────────────────────────────────────┐
│                     autoDream                            │
│                                                          │
│  ① Orient ──→ ② Gather ──→ ③ Consolidate ──→ ④ Prune   │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### ① Orient — 审视现有记忆

```
读取所有记忆文件
  ├── 列出 MEMORY.md 索引
  ├── 扫描每个 .md 文件的 frontmatter
  │     name, description, type
  ├── 识别：
  │     ├── 哪些记忆可能过时了？
  │     ├── 哪些记忆有重叠？
  │     ├── 索引还有多少空间？
  │     └── 有没有"漂移"的记忆（内容和 description 不匹配）？
  └── 形成整合计划
```

### ② Gather — 收集新信号

```
从多个来源收集需要整合的信号：
  ├── 最近的会话日志
  │     有没有新的反馈或决策？
  ├── "漂移"的记忆
  │     description 说是"测试相关"，但内容已经扩展到包含 CI 配置
  └── 相对日期
       "下周四" → 需要转换为绝对日期
```

### ③ Consolidate — 合并、更新

```
执行整合操作：
  ├── 合并碎片记忆
  │     feedback_testing_1/2/3.md → feedback_testing.md（合并为一条）
  │
  ├── 更新过时内容
  │     project_auth_rewrite.md
  │       "预计下周完成" → "已于 2025-03-15 完成"
  │
  ├── 转换相对日期为绝对日期
  │     "本周四" → "2025-04-10"
  │
  └── 修复描述漂移
       更新 frontmatter 的 description 使其与内容一致
```

### ④ Prune — 修剪索引

```
确保 MEMORY.md 保持健康：
  ├── 当前行数 > 180？
  │     → 识别低价值条目
  │     → 合并或删除
  ├── 总大小 > 20KB？
  │     → 缩短过长的索引条目
  └── 最终保持：
       < 200 行
       < 25KB
```

## 11.5.4 Dream Prompt

autoDream 使用的 Prompt 是整个系统中最"诗意"的：

```
You are performing a dream — a reflective pass over your memory files.
Synthesize what you've learned recently into durable, well-organized
memories so that future sessions can orient quickly.
```

**为什么叫"做梦"而不是"记忆整合"或"后台维护"？**

这不只是命名品味——隐喻精确传达了任务的"气质"：

1. **反思性的**——不是执行命令，而是回顾和思考
2. **从容的**——不是紧急任务，有充裕的时间做好
3. **无意识的**——在后台发生，用户不需要知道
4. **整合性的**——把碎片拼成完整的图景

这个隐喻帮助 Claude 理解它应该以什么"心态"处理这个任务：不是像 debug 时那样急切地找问题，而是像整理书架一样从容地归类和修剪。

## 11.5.5 extractMemories vs autoDream

两者都和记忆相关，但职责完全不同：

```
                extractMemories              autoDream
───────────────────────────────────────────────────────
触发时机      会话结束时                    空闲时（CronTool）
职责          写入新记忆                    整合已有记忆
输入          最近的对话内容                现有记忆文件
输出          新的 .md 文件                 更新已有 .md 文件
频率          每次会话结束                  周期性
是否创建记忆  是                            否（只合并/更新/删除）
```

类比：
- **extractMemories** 像下课后写笔记——把刚学到的记下来
- **autoDream** 像考试前整理笔记——合并重复的、丢弃过时的、理清结构

## 11.5.6 实际执行示例

```
Dream 开始
│
├── ① Orient
│   读取 MEMORY.md（23 条索引）
│   读取所有记忆文件
│   发现：
│     - feedback_testing_1.md 和 feedback_testing_2.md 内容重叠
│     - project_release.md 提到"本周四"但已过去两周
│     - user_preferences.md 的 description 过于笼统
│
├── ② Gather
│   检查最近的会话日志
│   没有新的信号需要整合
│
├── ③ Consolidate
│   合并 feedback_testing_1.md + feedback_testing_2.md
│     → feedback_testing.md（内容合并，保留 Why 和 How to apply）
│   删除 feedback_testing_1.md 和 feedback_testing_2.md
│   
│   更新 project_release.md
│     "merge freeze 从本周四开始" → "merge freeze 从 2025-03-27 开始（已过期）"
│   
│   更新 user_preferences.md 的 description
│     "用户偏好" → "偏好 tabs 缩进、单引号、无分号的 TypeScript 风格"
│
├── ④ Prune
│   MEMORY.md 从 23 行减少到 21 行（合并了两条）
│   总大小 OK（< 25KB）
│
└── Dream 结束
```

---

> **下一节**：[11.6 四层协作：全景视图](./116-cooperation.md)
