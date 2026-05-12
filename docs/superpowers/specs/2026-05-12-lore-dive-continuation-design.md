# lore-dive 续集探索设计文档

**日期：** 2026-05-12
**状态：** 已批准
**扩展对象：** `.claude/skills/lore-dive/SKILL.md`

---

## 概述

为 lore-dive skill 增加「续集探索」能力：当用户想在一个已完结的探索话题上继续深挖时，skill 能检测意图、加载原文档作为背景、创建续集 session，并在完成后提供可选的 AI 综合合并。

---

## 触发条件

**用户主动触发**（不自动检测）：用户输入中含有以下关键词时，Phase 0 进入续集检测流程：

- 中文：「继续」「在...基础上」「之前探索过」「接着」「继续探索」
- 英文：「continue」「follow up」「build on」「revisit」

---

## Phase 0 扩展：续集检测

在现有 Session Recovery 流程末尾追加：

```
5. 检测用户输入是否含有续集关键词（见上）
6. 如果是：用 Glob 扫描 explore/*.md（排除 *-session.md），列出所有已完成的探索
   - 格式：序号 + root 问题 + 文件名，例如：
     「1. Transformer 的 Attention 机制是怎么工作的 (transformer-attention.md)」
   - 询问用户：「选择要继续的探索，或输入 0 开启全新探索：」
7. 用户选择一个已完成文档 → 读取其全文作为背景上下文，进入阶段 1（续集模式）
8. 用户选 0 → 正常进入阶段 1（新建模式）
```

---

## 阶段 1 扩展：续集 Session 文件格式

续集模式下，slug 规则：在原 slug 末尾追加 `-2`（再续集加 `-3`，以此类推）。

```markdown
---
root: "<新问题或继续探索的方向>"
slug: <slug>-2
continues: <slug>
started: <YYYY-MM-DDThh:mm:ss>
status: active
---
```

`continues` 字段记录原 session 的 slug，建立血缘关系。

创建续集 session 文件后，告知用户：

> 「已加载『[原 root]』的探索背景，续集 session 已创建。请继续提问。」

---

## 阶段 3 扩展：完成后合并提示

续集 session（frontmatter 中含 `continues` 字段）完成生成 `<slug>-2.md` 后，询问用户：

> 「续集探索已生成 `explore/<slug>-2.md`。要把它合并进原文档 `explore/<slug>.md` 吗？合并后只保留一个文档，续集文件会被删除。」

### 用户选「合并」

| 文档部分 | 合并方式 |
|---|---|
| **合成文章（主体）** | AI 重新综合：读取 v1 + v2 全文，理解 v2 在 v1 哪个位置深挖，将 v2 内容整合进对应位置，生成结构完整的新文章 |
| **附录：探索路径** | 直接追加：v1 Q&A 保持不变，v2 Q&A 按时间顺序追加在后面 |

执行步骤：
1. **先 commit 当前状态（安全快照）**：
   ```bash
   git add explore/<slug>.md explore/<slug>-2.md explore/<slug>-2-session.md
   git commit -m "chore: snapshot before merging <slug>-2 into <slug>"
   ```
2. Read `explore/<slug>.md` 和 `explore/<slug>-2.md`
3. AI 综合生成新的合成文章正文（不简单拼接，需理解结构关系）
4. 附录部分：v1 附录原文 + v2 附录原文 顺序拼接
5. Write 覆盖 `explore/<slug>.md`（frontmatter 更新 `explored` 日期）
6. 删除 `explore/<slug>-2.md` 和 `explore/<slug>-2-session.md`
7. 告知用户：「已合并至 `explore/<slug>.md`，合并前状态已保存在上一个 commit，如需回滚可 `git checkout HEAD~1 -- explore/<slug>.md`」

### 用户选「保留分开」

不执行任何操作，两个文档都保留。

---

## 文件生命周期

```
续集探索中：
  explore/<slug>-session.md      (status: done)   ← 原 session，已完结
  explore/<slug>.md                               ← 原最终文档
  explore/<slug>-2-session.md    (status: active) ← 续集 session（进行中）

续集完成，不合并：
  explore/<slug>-session.md      (status: done)
  explore/<slug>.md
  explore/<slug>-2-session.md    (status: done)
  explore/<slug>-2.md                             ← 续集文档

续集完成，合并后：
  explore/<slug>-session.md      (status: done)
  explore/<slug>.md                               ← 合并后的完整文档（已更新）
  （<slug>-2-session.md 和 <slug>-2.md 已删除）
```

---

## 对现有 SKILL.md 的修改范围

| 位置 | 修改内容 |
|---|---|
| Phase 0 末尾 | 新增步骤 5-8：续集关键词检测 + 已完成文档列举 |
| 阶段 1 | 续集模式下 frontmatter 模板中增加 `continues` 字段 + 告知用户背景已加载 |
| 阶段 3 末尾 | 新增：检测 `continues` 字段 → 触发合并提示 → 合并或保留分开 |

不修改阶段 2（探索循环），对现有正常探索流程零影响。
