---
name: lore-dive
description: Use when the user wants to deeply explore a question through open-ended conversation and capture the full exploration as a structured document — triggered by a question, topic, or concept to investigate. Does NOT require a URL or file — the user's question IS the input.
---

# lore-dive — 深度探索笔记

## Overview

对用户的根问题进行深度对话探索，每轮问答实时写入 session 文件，探索结束后生成双层文档：合成文章（可直接阅读）+ 探索路径附录（保留完整问答过程）。

## When to Use

- 用户提出一个想深入探索的问题或话题，需要系统性记录
- 用户想通过对话一步步挖掘某个概念
- 用户没有提供 URL 或文件路径，只有一个想探索的问题

**不适用：** 用户只是问一个简单问题，没有「把探索过程沉淀成文档」的意图。

## Workflow

### 阶段 0：Session Recovery（每次启动必须先执行）

1. 用 Glob 工具扫描 `explore/*-session.md`
2. 用 Read 工具读取找到的文件，检查 frontmatter 中 `status` 字段
3. **找到 `status: active` 的 session：**
   - 如果只有一个：读取完整内容，告知用户：「发现进行中的探索：『[root]』，继续这个 session 吗？还是开启新探索？」
   - 如果有多个：列出所有找到的 session（标题 + 文件名），让用户选择一个继续，或选择「全部忽略，开启新探索」
   - 用户选「继续」→ 读取对应 session 完整内容，跳至阶段 2，从上次中断处继续
   - 用户选「新开」→ 进入阶段 1
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

---

### 阶段 1：新建 Session

1. 从根问题推导 slug（小写英文，空格和特殊字符替换为 `-`，只保留字母/数字/连字符）
   - 「如果让你来做整个大语言模型的分层，你会怎么分？」→ `llm-layering`
   - 「Transformer 的 Attention 机制是怎么工作的」→ `transformer-attention`

2. 用 Write 工具创建 `explore/<slug>-session.md`（注意：路径相对于项目根目录）：

```markdown
---
root: "<用户的根问题>"
slug: <slug>
started: <YYYY-MM-DDThh:mm:ss>
status: active
---
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

---

### 阶段 2：探索循环（每轮重复）

每轮：

1. **构思完整回答**（含所有 ASCII 图、表格、代码块，不截断，不压缩）

2. **Write-First Protocol（先写入 session，再输出给用户）**：
   - 用 Read 工具读取当前 session 文件完整内容
   - 在末尾追加完整回答块，用 Write 工具覆盖写入
   - 将刚才写入的内容**原文**输出给用户（不重新生成，不改写）

   **禁止**：先回答用户再写入 session。输出给用户的内容必须来自刚写入 session 的部分，两者是同一份文字。

   写入格式：

```markdown
## Q<n> <简短标题>

**问：** <用户原文>

**答：** <完整回答，含所有 ASCII 图、表格、代码块，与输出给用户的内容字符级别一致>
```

3. **判断是否需要分支提示：** 如果本轮回答引出了值得独立深挖的子问题，且用户尚未问到，先将以下内容追加写入 session 文件，再输出给用户：
   > 💡 **未探索分支：** <简短描述>

   不强制用户跟进，用户可直接忽略继续提问。

4. **检测话题漂移：** 如果用户新问题与根问题语义距离较远，在回答前先提示：
   > 「这个问题和当前 session『[root]』关联不大，要新开一个 session 吗？还是继续归入当前？」

   用户选「新开」→ 用 Read 读取当前 session 文件，将 `status: active` 改为 `status: paused`，用 Write 覆盖写入，回到阶段 1。（`paused` 为终止状态，不会被恢复，等同于 `done`。）

5. **检测完成意图（每轮检查）：** 用户说「完成」「够了」「结束」「可以了」「生成文档」「好了」「done」→ 立即进入阶段 3。

---

### 阶段 3：生成双层文档

1. 用 Read 工具读取完整 session 文件
2. 将所有问答合成为连贯文章（去掉「问：/答：」格式，直接可读，保留核心内容）
3. 从 session 文件 frontmatter 读取 `slug` 字段，用 Write 工具生成 `explore/<slug>.md`：

```markdown
---
root: "<根问题>"
explored: <YYYY-MM-DD>
---

# <合成标题>

<合成文章正文，连贯可读，中文，不少于 300 字>

---

## 附录：探索路径

### Q1 — <标题>

**问：** ...

**答：** ...

### Q2 — <标题>

...
```

4. 用 Edit 工具将 session 文件 `status: active` 改为 `status: done`
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

---

## Constraints

- 每轮问答后**立即写入** session 文件，不可积攒到对话结束
- 只写入 `explore/` 目录，不修改其他文件
- 合成文章使用**中文**，连贯可读，不少于 300 字
- 每次启动**必须先执行 Session Recovery**，不可跳过
- 分支提示只在 AI 主动判断有必要时出现，不强制每轮都加
- 同一领域的深入问题**不算**话题漂移
