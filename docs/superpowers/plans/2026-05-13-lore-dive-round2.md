# lore-dive Round 2 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 lore-dive Round 2 设计文档中的 7 项改动全部落地到 `.claude/skills/lore-dive/SKILL.md`。

**Architecture:** 所有改动集中在单一 Markdown 文件（SKILL.md）。核心架构变化是引入 `explore/index.yaml` 作为唯一状态来源，替代逐个扫描 session 文件；同时修复 6 个已知问题（阶段 0 顺序、合并候选推断、git 可选、session 追加格式、paused 状态、合成指令）。

**Tech Stack:** Markdown 文件编辑（Read + Edit + Write 工具）

**设计文档：** `docs/superpowers/specs/2026-05-13-lore-dive-round2-design.md`
**目标文件：** `.claude/skills/lore-dive/SKILL.md`

---

## 文件改动范围

| 文件 | 操作 |
|------|------|
| `.claude/skills/lore-dive/SKILL.md` | 修改（全部 8 个任务均在此文件） |

---

### Task 1：在 Overview 之后添加 Index/Log 数据模型说明

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`

- [ ] **Step 1: Read 目标文件，确认当前内容**

  运行：Read `.claude/skills/lore-dive/SKILL.md`
  预期：看到 `## When to Use` 区段，确认文件已加载

- [ ] **Step 2: 在 `## When to Use` 之后、`## Workflow` 之前插入数据模型说明**

  使用 Edit 工具，old_string：
  ```
  **不适用：** 用户只是问一个简单问题，没有「把探索过程沉淀成文档」的意图。

  ## Workflow
  ```

  new_string：
  ```
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
  ```

- [ ] **Step 3: Read 文件验证插入结果**

  运行：Read `.claude/skills/lore-dive/SKILL.md`，确认 `## Index 与 Log` 区段出现在 `## When to Use` 之后、`## Workflow` 之前

- [ ] **Step 4: Commit**

  ```bash
  git add .claude/skills/lore-dive/SKILL.md
  git commit -m "feat(lore-dive): add index/log data model documentation"
  ```

---

### Task 2：重写阶段 0（管理动作优先 + 从 index.yaml 读取状态）

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`

- [ ] **Step 1: 替换整个阶段 0 区段**

  使用 Edit 工具，old_string（从 `### 阶段 0` 到 `**关键：session 文件...`）：
  ```
  ### 阶段 0：Session Recovery（每次启动必须先执行）

  1. 用 Glob 工具扫描 `explore/*-session.md`
  2. 用 Read 工具读取找到的文件，检查 frontmatter 中 `status` 字段
  3. **找到 `status: active` 的 session：**
     - 如果只有一个：读取完整内容，告知用户：「发现进行中的探索：『[root]』，继续这个 session 吗？还是开启新探索？」
     - 如果有多个：列出所有找到的 session，让用户选择一个继续，或选择「全部忽略，开启新探索」。展示格式：
       - 普通 session：`[root]（文件名）`
       - 续集 session：`「[thread_title]」的续集 → [root]（文件名）`
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

  6. **合并触发检测**：检查用户输入是否包含「合并」或「merge」关键词，且包含 slug 名称：
     - **匹配时**：进入手动合并流程（见下方），**不进入阶段 1**
     - **不匹配时**：继续正常流程

  **手动合并流程：**
  1. 从用户输入解析主 slug；若未指定续集 slug，自动推断为 `<slug>-2`
  2. 检查以下文件是否均存在：
     - `explore/<slug>.md`
     - `explore/<slug>-2.md`
     - `explore/<slug>-2-session.md`（且 `status: done`）
  3. 若文件不完整：告知用户「未找到可合并的文件，请检查 slug 名称」，结束流程
  4. 若文件完整：提示用户确认：「要把『<slug>-2』的探索合并进『<slug>』吗？」
  5. 用户确认 → 执行阶段 3「用户选合并」的完整流程（步骤 1-10）
  6. 用户取消 → 不执行任何操作，结束流程

  **关键：session 文件是唯一的跨会话记忆。每次启动必须执行，不可跳过，即使用户提供了新问题。**
  ```

  new_string：
  ```
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
  2. **候选续集推断（按优先级）：**
     - 用户明确指定续集 slug → 直接用
     - index.yaml 中该 slug 下只有一个 `status: done` 的 `continues` 条目 → 自动用
     - 有多个 `status: done` 的 `continues` 条目 → 列出让用户选：
       ```
       发现多个可合并的续集，请选择：
         1. [root] (slug-2)
         2. [root] (slug-3)
       ```
     - 无 `status: done` 的续集 → 告知「未找到可合并的续集，请检查 slug 名称」，结束流程
  3. 提示用户确认：「要把『[续集 slug]』的探索合并进『[主 slug]』吗？」
  4. 用户确认 → 执行阶段 3「用户选合并」的完整流程
  5. 用户取消 → 不执行任何操作，结束流程

  **关键：`explore/index.yaml` 是唯一的跨会话状态记忆。每次启动必须读取，不可跳过，即使用户提供了新问题。**
  ```

- [ ] **Step 2: Read 文件验证**

  确认：阶段 0 开头是「Read `explore/index.yaml`」，步骤 2 是管理动作检查，步骤 3 是 Session Recovery，旧的 Glob 扫描已消失

- [ ] **Step 3: Commit**

  ```bash
  git add .claude/skills/lore-dive/SKILL.md
  git commit -m "feat(lore-dive): rewrite phase 0 with index.yaml and management action priority"
  ```

---

### Task 3：更新阶段 1 — 创建 session 后同步写入 index.yaml 和 log.md

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`

- [ ] **Step 1: 替换阶段 1 第 3 步**

  使用 Edit 工具，old_string：
  ```
  创建续集 session 文件后（`thread_title` 字段取自原探索 session 文件的 `root` 值），告知用户：「已加载『[原 root]』的探索背景，续集 session 已创建（`explore/<slug>-2-session.md`）。请继续提问。」

  3. 立即进入阶段 2，回答根问题。
  ```

  new_string：
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
  ```

- [ ] **Step 2: Read 文件验证**

  确认：阶段 1 第 3 步包含 index.yaml 追加和 log.md 追加，原第 3 步「立即进入阶段 2」变为第 4 步

- [ ] **Step 3: Commit**

  ```bash
  git add .claude/skills/lore-dive/SKILL.md
  git commit -m "feat(lore-dive): update phase 1 to write index.yaml and log.md on session create"
  ```

---

### Task 4：更新阶段 2 话题漂移 — abandoned → paused，同步 index.yaml/log.md

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`

- [ ] **Step 1: 替换话题漂移处理逻辑**

  使用 Edit 工具，old_string：
  ```
     用户选「新开」→ 用 Read 读取当前 session 文件，将 `status: active` 改为 `status: abandoned`，用 Write 覆盖写入，回到阶段 1。
  ```

  new_string：
  ```
     用户选「新开」→
     - 用 Read 读取当前 session 文件，将 `status: active` 改为 `status: paused`，用 Write 覆盖写入
     - Read `explore/index.yaml`，找到当前 slug 的条目，将 `status: active` 改为 `status: paused`，用 Write 覆盖写入
     - Read `explore/log.md`，在末尾追加 `## [<YYYY-MM-DD>] pause | <slug>`，用 Write 覆盖写入
     - 回到阶段 1

     用户选「继续归入」→ 正常进入阶段 2 回答，归入当前 session
  ```

- [ ] **Step 2: Read 文件验证**

  确认：「新开」分支现在写入 `paused`，并更新 index.yaml 和 log.md，且有「继续归入」分支

- [ ] **Step 3: Commit**

  ```bash
  git add .claude/skills/lore-dive/SKILL.md
  git commit -m "fix(lore-dive): topic drift now sets paused instead of abandoned, syncs index/log"
  ```

---

### Task 5：更新阶段 3 完成流程 — 同步 index.yaml 和 log.md

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`

- [ ] **Step 1: 在阶段 3 第 4 步之后插入 index/log 更新**

  使用 Edit 工具，old_string：
  ```
  4. 用 Edit 工具将 session 文件 `status: active` 改为 `status: done`
  5. 告知用户：「探索文档已生成：`explore/<slug>.md`」
  ```

  new_string：
  ```
  4. 用 Edit 工具将 session 文件 `status: active` 改为 `status: done`
  5. **更新 index.yaml 和 log.md：**
     - Read `explore/index.yaml`，找到当前 slug 的条目，更新 `status` 为 `done` 并追加 `explored: <YYYY-MM-DD>` 字段，用 Write 覆盖写入
     - Read `explore/log.md`，在末尾追加 `## [<YYYY-MM-DD>] done | <slug>`，用 Write 覆盖写入
  6. 告知用户：「探索文档已生成：`explore/<slug>.md`」
  ```

- [ ] **Step 2: Read 文件验证**

  确认：阶段 3 步骤 5 是 index/log 更新，步骤 6 是告知用户，原第 5 步已变为第 6 步

- [ ] **Step 3: Commit**

  ```bash
  git add .claude/skills/lore-dive/SKILL.md
  git commit -m "feat(lore-dive): update phase 3 done flow to sync index.yaml and log.md"
  ```

---

### Task 6：重写阶段 3 合并流程 — git 可选 + 结构化块 + index/log 更新

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`

- [ ] **Step 1: 替换整个「用户选合并」区段**

  使用 Edit 工具，old_string：
  ```
     **用户选「合并」：**
     1. 先 commit 当前状态（安全快照）：
        ```bash
        git add explore/<slug>.md explore/<slug>-2.md explore/<slug>-2-session.md
        git commit -m "chore: snapshot before merging <slug>-2 into <slug>"
        ```
     2. Read `explore/<slug>.md` 和 `explore/<slug>-2.md`
     3. **合成文章**：AI 重新综合正文——理解 v2 在 v1 哪个位置深挖，将 v2 内容整合进对应位置，生成结构完整的新文章（**不是简单拼接,只做润色和整体文档结构的梳理，尽量避免修改原文、和提炼删减细节，除非有重复**）
     4. **附录**：v1 附录原文 + v2 附录原文 顺序拼接
     5. Write 覆盖 `explore/<slug>.md`（frontmatter `explored` 字段更新为当日日期）
     6. 删除 `explore/<slug>-2.md`
     7. 读取 `explore/<slug>-2-session.md` 的完整内容（frontmatter 之后的所有问答）
     8. 在 `explore/<slug>-session.md` 末尾追加以下内容，用 Write 覆盖写入：

        ```
        --- 续集 <slug>-2 ---

        （<slug>-2-session.md 中 frontmatter 之后的全部问答内容）
        ```

     9. 删除 `explore/<slug>-2-session.md`
     10. 告知用户：「已合并至 `explore/<slug>.md`，原始问答数据已合入 `explore/<slug>-session.md`。如需回滚：`git checkout HEAD~1 -- explore/<slug>.md`」

     **用户选「保留分开」：** 不执行任何操作，两个文档都保留。
  ```

  new_string：
  ```
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
  ```

- [ ] **Step 2: Read 文件验证**

  确认：git 步骤有「可选」说明；session 追加格式为 `## Imported Session: ...` 结构化块；末尾有 index.yaml 和 log.md 更新步骤

- [ ] **Step 3: Commit**

  ```bash
  git add .claude/skills/lore-dive/SKILL.md
  git commit -m "fix(lore-dive): rewrite merge flow - optional git, structured block, index/log sync"
  ```

---

### Task 7：更新阶段 3 合成文章指令 ③

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`

- [ ] **Step 1: 替换合成指令第 ③ 条**

  使用 Edit 工具，old_string：
  ```
       **③ 只做润色和整体结构梳理；不提炼删减细节；不简单缩写；不用通用知识结构替代探索所得**
  ```

  new_string：
  ```
       **③ 正文应保留探索所得的关键细节、关键例子、关键修正过程；可删去重复解释、礼貌性过渡和显然冗余的表述；不用通用知识结构替代探索所得。附录负责完整保真，不做任何删减。**
  ```

- [ ] **Step 2: Read 文件验证**

  确认：指令 ③ 已替换，包含「可删去重复解释、礼貌性过渡」和「附录负责完整保真」

- [ ] **Step 3: Commit**

  ```bash
  git add .claude/skills/lore-dive/SKILL.md
  git commit -m "fix(lore-dive): update synthesis instruction to clarify main body vs appendix"
  ```

---

### Task 8：更新 Constraints 区段

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`

- [ ] **Step 1: 替换 Constraints 区段**

  使用 Edit 工具，old_string：
  ```
  ## Constraints

  - 每轮问答后**立即写入** session 文件，不可积攒到对话结束
  - 只写入 `explore/` 目录，不修改其他文件
  - 合成文章使用**中文**，连贯可读，不少于 300 字
  - 每次启动**必须先执行 Session Recovery**，不可跳过
  - 分支提示只在 AI 主动判断有必要时出现，不强制每轮都加
  - 同一领域的深入问题**不算**话题漂移
  ```

  new_string：
  ```
  ## Constraints

  - 每轮问答后**立即写入** session 文件，不可积攒到对话结束
  - 只写入 `explore/` 目录，不修改其他文件
  - 合成文章使用**中文**，连贯可读，不少于 300 字
  - 每次启动**必须先读取 `explore/index.yaml`**，不可跳过
  - 所有状态变更必须同步更新 index.yaml 和 log.md
  - 分支提示只在 AI 主动判断有必要时出现，不强制每轮都加
  - 同一领域的深入问题**不算**话题漂移
  - `abandoned` 只在用户明确放弃时使用；话题漂移使用 `paused`
  ```

- [ ] **Step 2: Read 文件验证，全文检查**

  Read 完整 SKILL.md，确认：
  - 无旧的「Glob 扫描 session 文件」逻辑
  - 无旧的「status: abandoned」在话题漂移处
  - 无「`--- 续集 xxx ---`」格式
  - 合并候选推断为三级优先
  - git 步骤有可选说明

- [ ] **Step 3: Commit**

  ```bash
  git add .claude/skills/lore-dive/SKILL.md
  git commit -m "fix(lore-dive): update constraints to reflect index.yaml and paused/abandoned semantics"
  ```
