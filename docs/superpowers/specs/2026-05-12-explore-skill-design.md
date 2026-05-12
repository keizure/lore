# lore-dive Skill 设计文档

**日期：** 2026-05-12
**状态：** 已批准

---

## 概述

一个 Claude Code skill，让用户能够对一个根问题进行深度探索，同时将整个过程结构化地记录下来，最终生成一篇双层文档：合成文章（主体，可直接阅读）+ 探索路径（附录，保留完整问答过程）。

---

## Skill 形态

**这是一个 superpowers-compatible skill，不是 Claude Code command。**

- 文件位置：`.claude/skills/lore-dive/SKILL.md`
- 格式：YAML frontmatter + markdown，与 `.claude/skills/lore/SKILL.md` 完全一致
- 触发：通过 Skill tool 调用，注册为项目本地 skill

```yaml
---
name: lore-dive
description: Use when the user wants to deeply explore a question through conversation and capture the process as a structured document.
---
```

---

## 触发方式

```
/lore-dive "你的根问题"
```

Skill 以根问题为入口自动开启 session。无需显式关闭命令，AI 自然检测用户的完成意图。

---

## 工作流

1. **触发**：用户调用 `/explore "根问题"`
2. **Session Recovery（每次启动强制执行）**：扫描 `explore/` 下所有 `*-session.md`，找到 `status: active` 的文件
   - 找到：读取完整内容重建上下文，询问用户「继续当前探索『xxx』还是开启新 session？」
   - 未找到：以用户提供的根问题新建 session
   - **Session 文件是唯一的跨会话记忆，每轮必须读取**
3. **探索循环**：
   - 用户自由提问，AI 逐轮回答
   - 每轮交互实时追加到 session 文件
   - AI 按需在答案末尾提示未探索分支（`> 💡 未探索分支：xxx`）
   - AI 检测话题漂移并提示是否新开 session
4. **完成检测**：AI 识别用户的完成意图（「够了」「结束」「可以了」「生成」等自然语言）
5. **生成文档**：从 session 文件生成双层最终文档，session 标记为 `status: done`

---

## 文件结构

```
lore/
├── reading/          ← 阅读笔记（现有，不变）
├── explore/          ← 新增：探索文档
│   ├── <slug>-session.md    ← 进行中的 session 草稿
│   └── <slug>.md            ← 生成后的最终文档
└── sources/
```

文件名 slug 由 AI 从根问题自动生成（英文短横线格式）。

---

## Session 文件格式

```markdown
---
root: "如果让你来做整个大语言模型的分层，你会怎么分？"
started: 2026-05-12T10:00:00
status: active
---

## Q1 根问题
**问：** 如果让你来做整个大语言模型的分层，你会怎么分？
**答：** ...

> 💡 **未探索分支：** Attention 机制在哪一层？

## Q2 跟进
**问：** Tokenizer 和 Embedding 是同一层吗？
**答：** ...
```

---

## 最终文档格式（双层）

```markdown
---
root: "..."
explored: 2026-05-12
---

# <合成标题>

（合成文章正文，连贯可读，去掉对话噪音）

---

## 附录：探索路径

### Q1 — 根问题
**问：** ...
**答：** ...

### Q2 — ...
...
```

---

## AI 行为规则

### ① 分支提示（按需）

- **触发条件**：AI 判断当前回答自然引出了一个值得独立深挖的子问题，且用户尚未问到它
- **格式**：在当前答案末尾，单行 `> 💡 未探索分支：xxx`
- **注意**：不强制用户跟进，用户可直接忽略继续提问

### ② 话题漂移检测

- **触发条件**：用户新问题与当前 session 根问题语义距离较远（AI 自行判断）
- **行为**：在回答前提示：
  `「这个问题和当前 session『xxx』关联不大，要新开一个 session 吗？还是继续归入当前？」`
- **用户选「新开」时**：完成当前 session 并创建新的 session 文件

### ③ 完成检测 → 生成文档

- **触发条件**：用户表达完成意图（「够了」「结束」「可以了」「生成」等）
- **行为**：
  1. 将 session 文件中所有问答合成为连贯文章（主体）
  2. 原问答结构整理为附录
  3. 写入 `explore/<slug>.md`
  4. Session 草稿标记 `status: done`
  5. 告知用户文件路径

---

## 与现有 lore 系统的关系

- `explore/` 与 `reading/` 平级，独立命名空间，互不干扰
- 两者都以 Markdown 文件为真实来源，风格一致
- 未来可考虑在 `reading/` 笔记中交叉引用 `explore/` 探索文档（本期不做）

---

## 实现注意事项

**Skill 编写采用 writing-skills TDD 流程：**

1. **RED**：先跑无 skill 的基准测试，记录 AI 自然状态下的行为偏差（比如忘记写文件、忘记恢复 session）
2. **GREEN**：写最小化 skill 覆盖这些偏差
3. **REFACTOR**：发现新漏洞时补充，直到行为稳定

参考：`superpowers:writing-skills`
