# lore-dive Round 2 Fix 设计文档

**日期：** 2026-05-13
**来源：** fix.md 第二轮审查 + index/logging 架构新增
**目标文件：** `.claude/skills/lore-dive/SKILL.md`

---

## 背景

本文档记录对 lore-dive SKILL.md 的第二轮修改，基于对已落地版本的实际使用反馈。共涉及 6 个问题修复和 1 个新架构引入（index/log 系统）。

---

## 架构新增：Index / Log 系统

### 动机

原方案在 Session Recovery 时逐个扫描并读取所有 `*-session.md` 文件，每次会话启动的 token 消耗随文件数量线性增长，且难以维护续集树形关系。

### 方案：index 为主，session 文件为数据

- `explore/index.md` 是唯一的状态来源
- `explore/log.md` 是 append-only 操作记录
- Session Recovery、list 命令、合并流程全部只读 index，按需再读具体文件
- session 文件 frontmatter 仍保留（便于直接阅读），但不作为权威状态

### `explore/index.md` 格式

```markdown
# Explore Index

## Active
- llm-layering-session | 如果让你来做整个大语言模型的分层，你会怎么分？ | started: 2026-05-12

## Paused
- transformer-attention-session | Transformer 的 Attention 机制是怎么工作的 | started: 2026-05-10

## Done
- llm-layering | 如果让你来做整个大语言模型的分层，你会怎么分？ | explored: 2026-05-12
  └─ llm-layering-2 | 残差连接和 FFN 主要解决什么问题？ | explored: 2026-05-13 | merged: true

## Abandoned
- some-topic-session | 某个偏题的探索 | started: 2026-05-11
```

### `explore/log.md` 格式（append-only）

```markdown
## [2026-05-12] create  | llm-layering | 如果让你来做整个大语言模型的分层，你会怎么分？
## [2026-05-13] done    | llm-layering
## [2026-05-13] create  | llm-layering-2 | 残差连接和 FFN 主要解决什么问题？
## [2026-05-13] merge   | llm-layering-2 → llm-layering
## [2026-05-13] pause   | transformer-attention
```

### Index 更新时机

每次以下操作后必须同步更新 index 和 log：
- 创建 session（新建/续集）
- 状态变更（active → done / paused / abandoned）
- 合并完成

---

## Fix 1：阶段 0 顺序重构（管理动作优先）

**问题：** 合并触发检测在第 6 步，若存在 active session，用户会先被询问是否恢复，打断合并意图。

**新的阶段 0 流程：**

```
1. 读取 explore/index.md（不存在则创建空文件，同时创建 explore/log.md）

2. 检查是否为显式管理动作：
   a. 包含「list」「列表」→ 执行 list 命令，返回，不进入后续步骤
   b. 包含「合并」「merge」且包含 slug → 执行手动合并流程，返回，不进入后续步骤

3. 检查 index 中是否有 active 或 paused 的 session：
   - 有：分组展示
       进行中：
         1. [root 问题] (slug-session)
       暂停中：
         2. [root 问题] (slug-session)
     询问：「选择编号继续/恢复，或输入 0 开启新探索：」
     → 用户选编号：读取对应 session 文件全文，跳至阶段 2
     → 用户选 0：进入步骤 4
   - 无：直接进入步骤 4

4. 续集检测（关键词匹配）→ 续集模式或新建模式，进入阶段 1
```

**list 命令输出格式：**

```
已完成的探索（N）：

1. [root 问题] (slug, 日期)
   └─ [续集 root 问题] (slug-2, 日期) [已合并]

2. [root 问题] (slug, 日期)
```

数据来源：index.md 的 Done 区段。

---

## Fix 2：手动合并候选推断改为三级优先

**问题：** 原逻辑自动推断续集为 `<slug>-2`，多续集场景下可能合并错对象。

**新逻辑（候选范围限定为 status: done 的续集）：**

```
优先级 1：用户明确指定续集 slug → 直接用
优先级 2：index 中该 slug 下只有一个 done 续集 → 自动用，无需询问
优先级 3：index 中有多个 done 续集 → 列出让用户选：
  「发现多个可合并的续集，请选择：
    1. [root] (slug-2)
    2. [root] (slug-3)」

无 done 续集时：提示「未找到可合并的续集，请检查 slug 名称」，结束流程
```

---

## Fix 3：git commit 标记为可选

**问题：** Constraints 规定「只写入 explore/ 目录」，但合并流程要求 git commit，环境不支持时变为不可执行规范。

**改动：** 合并流程第 1 步改为：

> 若环境支持 git，执行快照 commit：
> ```bash
> git add explore/<slug>.md explore/<slug>-2.md explore/<slug>-2-session.md
> git commit -m "chore: snapshot before merging <slug>-2 into <slug>"
> ```
> 若不支持，跳过此步骤并告知用户：「未创建 git 快照，若需回滚请手动备份。」

---

## Fix 4：合并后 session 追加改为结构化块

**问题：** `--- 续集 xxx ---` 标记不够结构化，且续集 Q 编号与主 session 编号冲突。

**改动：** 在 `explore/<slug>-session.md` 末尾追加格式改为：

```markdown
## Imported Session: <slug>-2
**Continues:** <slug>
**Imported at:** <YYYY-MM-DD>

### Q1 — <标题>

**问：** ...

**答：** ...

### Q2 — <标题>

...
```

续集原始 Q 编号保留不变（Q1、Q2…），靠块头区分来源。

---

## Fix 5：话题漂移改为 paused，重新引入 paused 状态

**问题：** 话题漂移时直接将当前 session 设为 `abandoned` 过重，用户可能只是暂时切出去，旧的还想以后继续。

**状态语义（修订后）：**

| 状态 | 含义 | Session Recovery 是否展示 |
|------|------|--------------------------|
| `active` | 进行中，可恢复 | 是（「进行中」分组） |
| `paused` | 话题漂移暂停，可恢复 | 是（「暂停中」分组） |
| `done` | 正常完成，已生成文档 | 否 |
| `abandoned` | 用户明确放弃 | 否 |

**话题漂移触发逻辑（阶段 2）：**

```
提示：「这个问题和当前 session『[root]』关联不大，
       要新开一个 session 吗？还是继续归入当前？」

用户选「新开」：
→ 将当前 session 改为 paused（更新 index + session frontmatter）
→ log 追加 pause 记录
→ 回到阶段 1 新建 session

用户选「继续归入」：
→ 正常进入阶段 2 回答，归入当前 session
```

`abandoned` 只在用户明确说「放弃」「不要这个探索」时触发。

---

## Fix 6：阶段 3 合成文章指令调整

**问题：** 「不提炼删减细节」与「连贯成文」天然冲突，正文与附录分工不清。

**改动：** 将合成写作指令第 ③ 条替换为：

> 正文应保留探索所得的关键细节、关键例子、关键修正过程；可删去重复解释、礼貌性过渡和显然冗余的表述；不用通用知识结构替代探索所得。附录负责完整保真，不做任何删减。

---

## 改动范围总结

| 改动 | 位置 | 类型 |
|------|------|------|
| Index/Log 系统 | 全局新增 + 阶段 0 | 架构新增 |
| Fix 1：阶段 0 顺序重构 | 阶段 0 | 流程重写 |
| Fix 2：合并候选三级推断 | 阶段 0 合并流程 | 逻辑替换 |
| Fix 3：git 可选 | 阶段 3 合并步骤 1 | 措辞修改 |
| Fix 4：session 追加结构化块 | 阶段 3 合并步骤 8 | 格式替换 |
| Fix 5：paused 状态重引入 | 阶段 0 + 阶段 2 + 状态定义 | 状态 + 流程修改 |
| Fix 6：合成文章指令调整 | 阶段 3 第 2 步 ③ | 措辞替换 |

所有改动均在 `.claude/skills/lore-dive/SKILL.md` 单文件内完成。
