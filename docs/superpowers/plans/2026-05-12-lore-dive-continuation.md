# lore-dive 续集探索 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 lore-dive skill 增加续集探索能力：检测用户续集意图 → 加载原文档背景 → 创建续集 session → 完成后可选 AI 综合合并。

**Architecture:** 纯 prompt skill 修改，无代码。三处靶向编辑：Phase 0 末尾加续集检测、Phase 1 加续集 frontmatter 模板、Phase 3 末尾加合并提示。遵循 writing-skills TDD：RED 基准 → GREEN 修改 → 验证。

**Tech Stack:** Markdown skill 编辑，与现有 lore-dive SKILL.md 同格式。

---

## 文件清单

| 操作 | 路径 | 说明 |
|------|------|------|
| Modify | `.claude/skills/lore-dive/SKILL.md` | 三处靶向编辑 |

---

## Task 1：RED — 基准测试

目的：确认当前 skill 在续集场景下缺少正确行为，为后续修改提供对照基线。

**Files:**
- Read: `.claude/skills/lore-dive/SKILL.md`

- [ ] **Step 1: 分派基准测试 subagent（无续集功能）**

读取当前 SKILL.md，然后模拟以下场景：

```
请读取 /Users/liangjinrun/keisure/notebook/lore/.claude/skills/lore-dive/SKILL.md
然后完全按照其中的 workflow 执行以下操作：

用户说：「我想继续之前探索过的 Attention 机制，在 Multi-head 这块还有几个问题」

工作目录：/Users/liangjinrun/keisure/notebook/lore
（explore/transformer-attention.md 已存在，是之前完结的探索）
```

- [ ] **Step 2: 记录基准失败行为**

预期会出现以下偏差（逐条确认）：

- [ ] 没有检测到「继续」关键词
- [ ] 没有列出已完成的探索文档供用户选择
- [ ] 没有加载 `transformer-attention.md` 作为背景
- [ ] 没有创建 `transformer-attention-2-session.md`

---

## Task 2：GREEN — 修改 SKILL.md（三处靶向编辑）

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`

- [ ] **Step 1: 读取当前 SKILL.md**

```bash
cat /Users/liangjinrun/keisure/notebook/lore/.claude/skills/lore-dive/SKILL.md
```

- [ ] **Step 2: 编辑一 — Phase 0 末尾：新增续集检测步骤**

找到 Phase 0 中的这段文字：

```
4. **未找到活跃 session：** 直接进入阶段 1

**关键：session 文件是唯一的跨会话记忆。每次启动必须执行，不可跳过，即使用户提供了新问题。**
```

替换为：

```
4. **未找到活跃 session：** 直接进入步骤 5

5. **续集检测**：检查用户输入是否含有以下关键词：「继续」「在...基础上」「之前探索过」「接着」「继续探索」「continue」「follow up」「build on」「revisit」
   - **含关键词**：用 Glob 扫描 `explore/*.md`（排除 `*-session.md`），列出所有已完成的探索，格式：
     ```
     1. [root 问题] (文件名)
     2. [root 问题] (文件名)
     ```
     询问用户：「选择要继续的探索编号，或输入 0 开启全新探索：」
     - 用户选编号 → 用 Read 读取对应 `explore/<slug>.md` 全文作为背景上下文，进入阶段 1（**续集模式**）
     - 用户选 0 → 进入阶段 1（新建模式）
   - **不含关键词**：直接进入阶段 1（新建模式）

**关键：session 文件是唯一的跨会话记忆。每次启动必须执行，不可跳过，即使用户提供了新问题。**
```

- [ ] **Step 3: 编辑二 — Phase 1：新增续集模式 frontmatter 模板**

找到 Phase 1 中这段文字（紧接在现有 frontmatter 模板之后，step 3 之前）：

```
3. 立即进入阶段 2，回答根问题。
```

替换为：

```
**普通模式**：使用上方 frontmatter 模板。

**续集模式**（从续集检测进入时）：slug 在原 slug 末尾追加 `-2`（再续集加 `-3`），frontmatter 增加 `continues` 字段：

```markdown
---
root: "<新问题或继续探索的方向>"
slug: <slug>-2
continues: <slug>
started: <YYYY-MM-DDThh:mm:ss>
status: active
---
```

创建续集 session 文件后，告知用户：「已加载『[原 root]』的探索背景，续集 session 已创建（`explore/<slug>-2-session.md`）。请继续提问。」

3. 立即进入阶段 2，回答根问题。
```

- [ ] **Step 4: 编辑三 — Phase 3 末尾：新增合并提示**

找到 Phase 3 末尾这段文字：

```
5. 告知用户：「探索文档已生成：`explore/<slug>.md`」
```

替换为：

```
5. 告知用户：「探索文档已生成：`explore/<slug>.md`」

6. **续集合并提示**（仅当 session frontmatter 含 `continues` 字段时）：询问用户：
   > 「续集探索已生成 `explore/<slug>-2.md`。要把它合并进原文档 `explore/<slug>.md` 吗？合并后只保留一个文档，续集文件会被删除。」

   **用户选「合并」：**
   1. 先 commit 当前状态（安全快照）：
      ```bash
      git add explore/<slug>.md explore/<slug>-2.md explore/<slug>-2-session.md
      git commit -m "chore: snapshot before merging <slug>-2 into <slug>"
      ```
   2. Read `explore/<slug>.md` 和 `explore/<slug>-2.md`
   3. **合成文章**：AI 重新综合正文——理解 v2 在 v1 哪个位置深挖，将 v2 内容整合进对应位置，生成结构完整的新文章（**不是简单拼接**）
   4. **附录**：v1 附录原文 + v2 附录原文 顺序拼接
   5. Write 覆盖 `explore/<slug>.md`（frontmatter `explored` 字段更新为当日日期）
   6. 删除 `explore/<slug>-2.md` 和 `explore/<slug>-2-session.md`
   7. 告知用户：「已合并至 `explore/<slug>.md`。如需回滚：`git checkout HEAD~1 -- explore/<slug>.md`」

   **用户选「保留分开」：** 不执行任何操作，两个文档都保留。
```

- [ ] **Step 5: 验证文件行数和结构**

```bash
wc -l /Users/liangjinrun/keisure/notebook/lore/.claude/skills/lore-dive/SKILL.md
grep -n "续集检测\|continues\|续集合并" /Users/liangjinrun/keisure/notebook/lore/.claude/skills/lore-dive/SKILL.md
```

Expected: 行数比原来（132行）多约 40 行；grep 能找到三处新增关键词。

- [ ] **Step 6: Commit**

```bash
cd /Users/liangjinrun/keisure/notebook/lore
git add .claude/skills/lore-dive/SKILL.md
git commit -m "feat: add continuation and merge support to lore-dive skill"
```

---

## Task 3：GREEN — 验证续集检测和 session 创建

目的：验证 skill 能正确检测续集意图、列出已完成探索、创建续集 session 文件。

- [ ] **Step 1: 确认测试前提文件存在**

```bash
ls /Users/liangjinrun/keisure/notebook/lore/explore/
```

Expected: 看到 `transformer-attention.md`（已完成的探索文档）。

- [ ] **Step 2: 分派验证 subagent**

```
请读取 /Users/liangjinrun/keisure/notebook/lore/.claude/skills/lore-dive/SKILL.md
然后完全按照其中的 workflow 执行以下操作：

用户说：「我想继续之前探索过的 Attention 机制，在 Multi-head 这块还有几个问题没搞清楚」

执行到「续集 session 已创建」即可，不需要继续探索对话。

工作目录：/Users/liangjinrun/keisure/notebook/lore
```

- [ ] **Step 3: 验证关键行为**

- [ ] 检测到「继续」关键词
- [ ] 扫描 `explore/*.md` 并列出已完成探索
- [ ] 列表中出现了 `transformer-attention.md`
- [ ] 读取 `transformer-attention.md` 作为背景
- [ ] 创建了 `explore/transformer-attention-2-session.md`
- [ ] session frontmatter 含 `continues: transformer-attention`
- [ ] 告知用户「已加载探索背景」

```bash
cat /Users/liangjinrun/keisure/notebook/lore/explore/transformer-attention-2-session.md
```

Expected: 包含 `continues: transformer-attention` 和 `status: active`。

---

## Task 4：GREEN — 验证合并流程

目的：验证续集完成后合并提示正确触发，合并后文档结构完整，快照 commit 存在。

- [ ] **Step 1: 分派验证 subagent（续集探索 + 合并）**

```
请读取 /Users/liangjinrun/keisure/notebook/lore/.claude/skills/lore-dive/SKILL.md
然后完全按照其中的 workflow 执行以下操作：

前提：explore/transformer-attention-2-session.md 已存在（status: active，contains: transformer-attention）。
请直接从阶段 2 继续这个续集 session。

用户问题 1：「Multi-head Attention 中每个 head 是怎么分工的？有没有研究表明不同 head 关注不同语言现象？」
用户回答后说：「好了，结束吧，合并进原文档」

完整执行阶段 2 一轮 + 阶段 3 生成文档 + 合并流程。

工作目录：/Users/liangjinrun/keisure/notebook/lore
```

- [ ] **Step 2: 验证合并行为**

- [ ] 生成了 `explore/transformer-attention-2.md`
- [ ] 触发了合并提示
- [ ] 执行了安全快照 commit（`chore: snapshot before merging...`）
- [ ] 合成文章正文被 AI 重新综合（非简单拼接）
- [ ] 附录包含 v1 + v2 两部分 Q&A
- [ ] `transformer-attention-2.md` 和 `transformer-attention-2-session.md` 已删除
- [ ] 告知了回滚命令

```bash
ls /Users/liangjinrun/keisure/notebook/lore/explore/
git -C /Users/liangjinrun/keisure/notebook/lore log --oneline -3
grep "## 附录" /Users/liangjinrun/keisure/notebook/lore/explore/transformer-attention.md
```

Expected:
- `explore/` 中只剩 `transformer-attention-session.md` 和 `transformer-attention.md`（无 `-2` 文件）
- git log 中有 `chore: snapshot before merging...`
- `transformer-attention.md` 中有 `## 附录：探索路径`

---

## Task 5：REFACTOR — 补漏洞（如有）

根据 Task 3/4 的观察，针对性修复。

**常见漏洞模式：**

| 偏差 | 修复位置 |
|---|---|
| 没检测到「继续」（句子结构复杂时） | Phase 0 步骤 5 加「只要句子中出现关键词即触发，不要求精确匹配」 |
| 列出文档时格式混乱 | Phase 0 步骤 6 精确化格式要求 |
| 合并时直接 concat 文章 | Phase 3 步骤 6.3 加「每段内容要有机融合，不保留「第一轮」「第二轮」等标记」 |
| 忘记删除 -2 文件 | Phase 3 步骤 6.6 改为更醒目的警告语 |

- [ ] **Step 1: 根据测试结果修改（如有），重新运行 Task 3/4 的验证**

- [ ] **Step 2: Commit（如有修改）**

```bash
git add .claude/skills/lore-dive/SKILL.md
git commit -m "refactor: close lore-dive continuation skill loopholes"
```

---

## Task 6：最终 Commit

- [ ] **Step 1: 检查文件状态**

```bash
git status
git log --oneline -5
```

Expected: working tree clean，log 中有 `feat: add continuation and merge support to lore-dive skill`。

- [ ] **Step 2: 如有未提交内容则 commit**

```bash
git add -A
git commit -m "feat: complete lore-dive continuation feature"
```
