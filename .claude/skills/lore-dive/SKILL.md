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

## Index 与 Log

`explore/index.yaml` 是唯一的跨会话状态来源。所有 session 的状态、根问题、日期、续集关系均记录于此，避免每次启动时逐个扫描 session 文件。`explore/log.md` 是 append-only 操作记录，用于审计。

### `explore/index.yaml` 格式

```yaml
- slug: llm-layering
  root: "如果让你来做整个大语言模型的分层，你会怎么分？"
  status: done
  started: 2026-05-12
  explored: 2026-05-12

- slug: llm-layering-2
  root: "残差连接和 FFN 主要解决什么问题？"
  status: done
  started: 2026-05-13
  explored: 2026-05-13
  continues: llm-layering
  merged: true

- slug: transformer-attention
  root: "Transformer 的 Attention 机制是怎么工作的"
  status: paused
  started: 2026-05-10

- slug: some-topic
  root: "某个偏题的探索"
  status: abandoned
  started: 2026-05-11
```

**字段说明：**

| 字段 | 必填 | 说明 |
|------|------|------|
| `slug` | 是 | 唯一标识，对应文件名（不含 `-session` 后缀） |
| `root` | 是 | 根问题原文 |
| `status` | 是 | `active` / `paused` / `done` / `abandoned` |
| `started` | 是 | session 创建日期（YYYY-MM-DD） |
| `explored` | done 时 | 文档生成日期 |
| `continues` | 续集时 | 主 slug |
| `merged` | 续集合并后 | `true` |

**状态语义：**

| 状态 | 含义 | Session Recovery 展示 |
|------|------|-----------------------|
| `active` | 进行中，可恢复 | 是（「进行中」分组） |
| `paused` | 话题漂移暂停，可恢复 | 是（「暂停中」分组） |
| `done` | 正常完成，已生成文档 | 否 |
| `abandoned` | 用户明确放弃 | 否 |

### `explore/log.md` 格式

每行一条记录，前缀固定，append-only：

```
## [2026-05-12] create  | llm-layering | 如果让你来做整个大语言模型的分层，你会怎么分？
## [2026-05-13] done    | llm-layering
## [2026-05-13] create  | llm-layering-2 | 残差连接和 FFN 主要解决什么问题？
## [2026-05-13] merge   | llm-layering-2 → llm-layering
## [2026-05-13] pause   | transformer-attention
```

### index 更新时机

以下操作后必须同步更新 index.yaml 和 log.md：创建 session、状态变更（active → done / paused / abandoned）、合并完成。

## Workflow

### 阶段 0：Session Recovery（每次启动必须先执行）

1. 用 Read 工具读取 `explore/index.yaml`
   - 文件不存在：用 Write 工具创建 `explore/index.yaml`（内容：`[]`），并创建 `explore/log.md`（内容：空）

2. **检查显式管理动作（优先级最高，匹配即执行，不继续后续步骤）：**
   - 用户输入含「list」或「列表」→ 执行 **list 命令**（见下方），结束流程
   - 用户输入含「合并」或「merge」且包含 slug 名称 → 执行**手动合并流程**（见下方），结束流程

3. **Session Recovery**：从 index.yaml 中筛选 `status: active` 和 `status: paused` 的条目
   - **有活跃/暂停 session：** 分组展示：
     ```
     进行中：
       1. [root] (slug, started: YYYY-MM-DD)
     暂停中：
       2. [root] (slug, started: YYYY-MM-DD)
     ```
     询问：「选择编号继续/恢复，或输入 0 开启新探索：」
     - 用户选编号 → Read 对应 `explore/<slug>-session.md` 全文，跳至阶段 2
     - 用户选 0 → 进入步骤 4
   - **无活跃/暂停 session：** 直接进入步骤 4

4. **续集检测**：检查用户输入是否含有以下关键词：「继续」「在...基础上」「之前探索过」「接着」「继续探索」「continue」「follow up」「build on」「revisit」
   - **含关键词**：从 index.yaml 中筛选 `status: done` 且无 `continues` 字段的条目，列出：
     ```
     1. [root 问题] (slug, explored: YYYY-MM-DD)
     2. [root 问题] (slug, explored: YYYY-MM-DD)
     ```
     询问用户：「选择要继续的探索编号，或输入 0 开启全新探索：」
     - 用户选编号 → Read `explore/<slug>.md` 全文作为背景上下文，进入阶段 1（**续集模式**）
     - 用户选 0 → 进入阶段 1（新建模式）
   - **不含关键词**：直接进入阶段 1（新建模式）

**list 命令：**

从 index.yaml 中读取所有 `status: done` 的条目，按 `continues` 字段构建树形关系，输出：

```
已完成的探索（N）：

1. [root] (slug, explored: YYYY-MM-DD)
   └─ [续集 root] (slug-2, explored: YYYY-MM-DD) [已合并]

2. [root] (slug, explored: YYYY-MM-DD)
```

**手动合并流程：**

1. 从用户输入解析主 slug
2. **候选续集推断（按优先级，仅考虑 `status: done` 且 `merged` 不为 `true` 的 `continues` 条目）：**
   - 用户明确指定续集 slug → 直接用
   - 符合条件的续集只有一个 → 自动用
   - 符合条件的续集有多个 → 列出让用户选：
     ```
     发现多个可合并的续集，请选择：
       1. [root] (slug-2)
       2. [root] (slug-3)
     ```
   - 无符合条件的续集 → 告知「未找到可合并的续集（需 status: done 且未合并），请检查 slug 名称」，结束流程
3. 提示用户确认：「要把『[续集 slug]』的探索合并进『[主 slug]』吗？」
4. 用户确认 → 执行阶段 3「用户选合并」的完整流程
5. 用户取消 → 不执行任何操作，结束流程

**关键：`explore/index.yaml` 是唯一的跨会话状态记忆。每次启动必须读取，不可跳过，即使用户提供了新问题。**

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

**续集模式**（从续集检测进入时）：slug 在原 slug 末尾追加 `-2`（再续集加 `-3`），frontmatter 增加 `continues` 和 `thread_title` 字段：

```markdown
---
root: "<新问题或继续探索的方向>"
slug: <slug>-2
continues: <original-slug>
thread_title: "<原探索 session 文件的 root 字段值>"
started: <YYYY-MM-DDThh:mm:ss>
status: active
---
```

创建续集 session 文件后（`thread_title` 字段取自原探索 session 文件的 `root` 值），告知用户：「已加载『[原 root]』的探索背景，续集 session 已创建（`explore/<slug>-2-session.md`）。请继续提问。」

3. **更新 index.yaml 和 log.md：**
   - Read `explore/index.yaml`，在末尾追加新条目，用 Write 覆盖写入：

   **普通模式：**
   ```yaml
   - slug: <slug>
     root: "<用户的根问题>"
     status: active
     started: <YYYY-MM-DD>
   ```

   **续集模式：**
   ```yaml
   - slug: <slug>-2
     root: "<新问题或继续探索的方向>"
     status: active
     started: <YYYY-MM-DD>
     continues: <original-slug>
   ```

   - Read `explore/log.md`，在末尾追加，用 Write 覆盖写入：
   ```
   ## [<YYYY-MM-DD>] create | <slug> | <root>
   ```

4. 立即进入阶段 2，回答根问题。

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

   用户选「新开」→
   - 用 Read 读取当前 session 文件，将 `status: active` 改为 `status: paused`，用 Write 覆盖写入
   - Read `explore/index.yaml`，找到当前 slug 的条目，将 `status: active` 改为 `status: paused`，用 Write 覆盖写入
   - Read `explore/log.md`，在末尾追加 `## [<YYYY-MM-DD>] pause | <slug>`，用 Write 覆盖写入
   - 回到阶段 1

   用户选「继续归入」→ 正常进入阶段 2 回答，归入当前 session

5. **检测完成意图（每轮检查）：** 用户说「完成」「够了」「结束」「可以了」「生成文档」「好了」「done」→ 立即进入阶段 3。

---

### 阶段 3：生成双层文档

1. 用 Read 工具读取完整 session 文件
2. 将所有问答合成为连贯文章，遵循以下写作指令：

     **① 先判断探索类型，选择主结构：**
     - 知识建构型（从零建立认知地图）→ 叙事流结构：按概念依赖顺序展开，先讲基础后讲进阶
     - 问题深挖型（针对具体疑问深入）→ 结论导向：先给答案，再展开论证
     - 概念厘清型（理清容易混淆的概念）→ 对比分析：以混淆点为核心展开

     **② 无论何种类型，必须包含以下两个元素**（可嵌入正文，无需单独成节）：
     - 认知路径：记录探索中发生的认知转变（「以为是 X，实际是 Y，因为 Z」）
     - 易混淆点：并列展示容易混淆的概念对

     **③ 正文应保留探索所得的关键细节、关键例子、关键修正过程；可删去重复解释、礼貌性过渡和显然冗余的表述；不用通用知识结构替代探索所得。附录负责完整保真，不做任何删减。**
3. 从 session 文件 frontmatter 读取 `slug` 字段，用 Write 工具生成 `explore/<slug>.md`：

```markdown
---
root: "<根问题>"
explored: <YYYY-MM-DD>
---

# <合成标题>

<合成文章正文，中文，不少于 300 字，按上方写作指令生成>

---

## 附录：探索路径

### Q1 — <标题>

**问：** ...

**答：** ...

### Q2 — <标题>

...
```

4. 用 Edit 工具将 session 文件 `status: active` 改为 `status: done`
5. **更新 index.yaml 和 log.md：**
   - Read `explore/index.yaml`，找到当前 slug 的条目，更新 `status` 为 `done` 并追加 `explored: <YYYY-MM-DD>` 字段，用 Write 覆盖写入
   - Read `explore/log.md`，在末尾追加 `## [<YYYY-MM-DD>] done | <slug>`，用 Write 覆盖写入
6. 告知用户：「探索文档已生成：`explore/<slug>.md`」

7. **续集合并提示**（仅当 session frontmatter 含 `continues` 字段时）：询问用户：
   > 「续集探索已生成 `explore/<slug>-2.md`。要把它合并进原文档 `explore/<slug>.md` 吗？合并后只保留一个文档，续集文件会被删除。」

   **用户选「合并」：**
   1. **快照（可选）：** 若环境支持 git，执行：
      ```bash
      git add explore/<slug>.md explore/<slug>-2.md explore/<slug>-2-session.md
      git commit -m "chore: snapshot before merging <slug>-2 into <slug>"
      ```
      若不支持，跳过此步骤并告知用户：「未创建 git 快照，若需回滚请手动备份。」
   2. Read `explore/<slug>.md` 和 `explore/<slug>-2.md`
   3. **合成文章**：重新综合正文——理解 v2 在 v1 哪个位置深挖，将 v2 内容整合进对应位置，生成结构完整的新文章（不是简单拼接；只做润色和整体文档结构梳理；除非有重复，否则不删减细节）
   4. **附录**：v1 附录原文 + v2 附录原文顺序拼接
   5. Write 覆盖 `explore/<slug>.md`（frontmatter `explored` 字段更新为当日日期）
   6. 删除 `explore/<slug>-2.md`
   7. Read `explore/<slug>-2-session.md`，取 frontmatter 之后的全部内容
   8. Read `explore/<slug>-session.md`，在末尾追加以下结构化块，用 Write 覆盖写入：

      ```markdown
      ## Imported Session: <slug>-2
      **Continues:** <slug>
      **Imported at:** <YYYY-MM-DD>

      ### Q1 — <标题>

      **问：** ...

      **答：** ...
      ```
      （保留续集原始 Q 编号，不重新排序）

   9. 删除 `explore/<slug>-2-session.md`
   10. **更新 index.yaml：** Read `explore/index.yaml`，将续集条目的 `merged` 字段设为 `true`，用 Write 覆盖写入
   11. **更新 log.md：** Read `explore/log.md`，在末尾追加 `## [<YYYY-MM-DD>] merge | <slug>-2 → <slug>`，用 Write 覆盖写入
   12. 告知用户：「已合并至 `explore/<slug>.md`，原始问答数据已合入 `explore/<slug>-session.md`。」

   **用户选「保留分开」：** 不执行任何操作，两个文档都保留。

---

## Constraints

- 每轮问答后**立即写入** session 文件，不可积攒到对话结束
- 只写入 `explore/` 目录，不修改其他文件
- 合成文章使用**中文**，连贯可读，不少于 300 字
- 每次启动**必须先读取 `explore/index.yaml`**，不可跳过
- 所有状态变更必须同步更新 index.yaml 和 log.md
- 分支提示只在 AI 主动判断有必要时出现，不强制每轮都加
- 同一领域的深入问题**不算**话题漂移
- `abandoned` 只在用户明确放弃时使用；话题漂移使用 `paused`
