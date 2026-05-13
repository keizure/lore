# lore-dive Fixes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 修复 lore-dive SKILL.md 中的 5 个问题：状态命名歧义、合成文章结构缺失、续集合并数据丢失、续集标题断裂、无手动合并入口。

**Architecture:** 单文件修改。所有改动集中在 `.claude/skills/lore-dive/SKILL.md`，按阶段独立修改，互不依赖，可逐个提交。

**Tech Stack:** Markdown（SKILL.md 是纯文本指令文件，无代码运行环境，验证方式为阅读检查）

**Spec:** `docs/superpowers/specs/2026-05-13-lore-dive-fixes-design.md`

---

## 文件索引

| 操作 | 文件 |
|------|------|
| Modify | `.claude/skills/lore-dive/SKILL.md` |

---

### Task 1：Fix 1 — `paused` → `abandoned`

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`（阶段 2 话题漂移检测段落）

- [ ] **Step 1: 定位目标行**

  用 Read 工具读取 `.claude/skills/lore-dive/SKILL.md`，找到以下原文（阶段 2 第 4 步）：

  ```
  用 Read 读取当前 session 文件，将 `status: active` 改为 `status: paused`，用 Write 覆盖写入，回到阶段 1。（`paused` 为终止状态，不会被恢复，等同于 `done`。）
  ```

- [ ] **Step 2: 执行替换**

  用 Edit 工具将上述内容替换为：

  ```
  用 Read 读取当前 session 文件，将 `status: active` 改为 `status: abandoned`，用 Write 覆盖写入，回到阶段 1。
  ```

- [ ] **Step 3: 验证**

  用 Read 工具重新读取该文件，确认：
  - 文件中不再出现 `status: paused`
  - 文件中不再出现「`paused` 为终止状态」这句注释
  - `status: abandoned` 出现在话题漂移检测段落

- [ ] **Step 4: Commit**

  ```bash
  git add .claude/skills/lore-dive/SKILL.md
  git commit -m "fix(lore-dive): rename paused to abandoned for semantic clarity"
  ```

---

### Task 2：Fix 2 — 合成文章结构化指令

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`（阶段 3 第 2 步 + 模板注释）

- [ ] **Step 1: 定位第一处目标**

  找到阶段 3 第 2 步原文：

  ```
  2. 将所有问答合成为连贯文章（去掉「问：/答：」格式，直接可读，**只做润色和整体文档结构的梳理，尽量避免修改原文、和提炼删减细节，除非有重复**）
  ```

- [ ] **Step 2: 替换第一处**

  用 Edit 工具替换为：

  ```
  2. 将所有问答合成为连贯文章，遵循以下写作指令：

     **① 先判断探索类型，选择主结构：**
     - 知识建构型（从零建立认知地图）→ 叙事流结构：按概念依赖顺序展开，先讲基础后讲进阶
     - 问题深挖型（针对具体疑问深入）→ 结论导向：先给答案，再展开论证
     - 概念厘清型（理清容易混淆的概念）→ 对比分析：以混淆点为核心展开

     **② 无论何种类型，必须包含以下两个元素**（可嵌入正文，无需单独成节）：
     - 认知路径：记录探索中发生的认知转变（「以为是 X，实际是 Y，因为 Z」）
     - 易混淆点：并列展示容易混淆的概念对

     **③ 只做润色和整体结构梳理；不提炼删减细节；不简单缩写；不用通用知识结构替代探索所得**
  ```

- [ ] **Step 3: 定位第二处目标**

  找到合成文章模板中的注释行：

  ```
  <合成文章正文，连贯可读，中文，不少于 300 字>
  ```

- [ ] **Step 4: 替换第二处**

  用 Edit 工具替换为：

  ```
  <合成文章正文，中文，不少于 300 字，按上方写作指令生成>
  ```

- [ ] **Step 5: 验证**

  用 Read 工具读取文件，确认：
  - 阶段 3 第 2 步包含「判断探索类型」「认知路径」「易混淆点」三部分指令
  - 模板注释已更新
  - 原有「只做润色…」的说明保留在 ③ 中

- [ ] **Step 6: Commit**

  ```bash
  git add .claude/skills/lore-dive/SKILL.md
  git commit -m "fix(lore-dive): add structured synthesis instructions for phase 3"
  ```

---

### Task 3：Fix 3 — 续集合并时保留 session2 原始数据

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`（阶段 3 续集合并流程）

- [ ] **Step 1: 定位目标**

  找到阶段 3 「用户选合并」流程中的第 6、7 步原文：

  ```
     6. 删除 `explore/<slug>-2.md`
     7. 告知用户：「已合并至 `explore/<slug>.md`。如需回滚：`git checkout HEAD~1 -- explore/<slug>.md`」
  ```

- [ ] **Step 2: 替换**

  用 Edit 工具替换为：

  ```
     6. 删除 `explore/<slug>-2.md`
     7. 读取 `explore/<slug>-2-session.md` 的完整内容（frontmatter 之后的所有问答）
     8. 在 `explore/<slug>-session.md` 末尾追加以下内容，用 Write 覆盖写入：

        ```
        --- 续集 <slug>-2 ---

        （<slug>-2-session.md 中 frontmatter 之后的全部问答内容）
        ```

     9. 删除 `explore/<slug>-2-session.md`
     10. 告知用户：「已合并至 `explore/<slug>.md`，原始问答数据已合入 `explore/<slug>-session.md`。如需回滚：`git checkout HEAD~1 -- explore/<slug>.md`」
  ```

- [ ] **Step 3: 验证**

  用 Read 工具读取文件，确认：
  - 合并流程共 10 步（原 7 步 + 新增 3 步）
  - 步骤 7-9 包含读取 session2、追加到主 session、删除 session2 三个动作
  - 步骤 10 的告知内容提及 session.md

- [ ] **Step 4: Commit**

  ```bash
  git add .claude/skills/lore-dive/SKILL.md
  git commit -m "fix(lore-dive): preserve session2 raw data when merging sequel"
  ```

---

### Task 4：Fix 4 — 续集 frontmatter 增加 `thread_title`

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`（阶段 1 续集 frontmatter 模板 + Session Recovery 显示格式）

- [ ] **Step 1: 定位续集 frontmatter 模板**

  找到阶段 1 续集模式的 frontmatter 模板原文：

  ```markdown
  ---
  root: "<新问题或继续探索的方向>"
  slug: <slug>-2
  continues: <slug>
  started: <YYYY-MM-DDThh:mm:ss>
  status: active
  ---
  ```

- [ ] **Step 2: 替换 frontmatter 模板**

  用 Edit 工具替换为：

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

- [ ] **Step 3: 定位创建续集后的告知文本**

  找到原文：

  ```
  创建续集 session 文件后，告知用户：「已加载『[原 root]』的探索背景，续集 session 已创建（`explore/<slug>-2-session.md`）。请继续提问。」
  ```

- [ ] **Step 4: 替换告知文本**

  `thread_title` 取自哪里需要明确，用 Edit 替换为：

  ```
  创建续集 session 文件后（`thread_title` 字段取自原探索 session 文件的 `root` 值），告知用户：「已加载『[原 root]』的探索背景，续集 session 已创建（`explore/<slug>-2-session.md`）。请继续提问。」
  ```

- [ ] **Step 5: 定位 Session Recovery 的多 session 展示格式**

  找到阶段 0 中「如果有多个」的处理原文：

  ```
  - 如果有多个：列出所有找到的 session（标题 + 文件名），让用户选择一个继续，或选择「全部忽略，开启新探索」
  ```

- [ ] **Step 6: 替换展示格式说明**

  用 Edit 替换为：

  ```
  - 如果有多个：列出所有找到的 session，让用户选择一个继续，或选择「全部忽略，开启新探索」。展示格式：
    - 普通 session：`[root]（文件名）`
    - 续集 session：`「[thread_title]」的续集 → [root]（文件名）`
  ```

- [ ] **Step 7: 验证**

  用 Read 工具读取文件，确认：
  - 续集 frontmatter 模板包含 `thread_title` 字段
  - 告知文本说明了 `thread_title` 的取值来源
  - Session Recovery 多 session 展示格式区分了普通 session 和续集 session

- [ ] **Step 8: Commit**

  ```bash
  git add .claude/skills/lore-dive/SKILL.md
  git commit -m "fix(lore-dive): add thread_title to sequel session for exploration context"
  ```

---

### Task 5：Fix 5 — 手动触发续集合并

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`（阶段 0 末尾，新增合并触发分支）

- [ ] **Step 1: 定位插入位置**

  找到阶段 0 末尾的「关键」说明原文：

  ```
  **关键：session 文件是唯一的跨会话记忆。每次启动必须执行，不可跳过，即使用户提供了新问题。**
  ```

- [ ] **Step 2: 在该说明之前插入合并触发分支**

  用 Edit 工具，在「关键」说明之前插入以下内容：

  ```
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

  ```

- [ ] **Step 3: 验证**

  用 Read 工具读取文件，确认：
  - 阶段 0 包含第 6 步「合并触发检测」
  - 手动合并流程包含 slug 解析、文件检查、用户确认、执行合并四个环节
  - 「关键」说明仍在最后，未被删除

- [ ] **Step 4: Commit**

  ```bash
  git add .claude/skills/lore-dive/SKILL.md
  git commit -m "fix(lore-dive): add manual merge trigger in session recovery"
  ```

---

## Self-Review 检查记录

**Spec 覆盖：**
- Fix 1（paused → abandoned）→ Task 1 ✓
- Fix 2（合成文章结构）→ Task 2 ✓
- Fix 3（session2 数据保留）→ Task 3 ✓
- Fix 4（thread_title）→ Task 4 ✓
- Fix 5（手动合并入口）→ Task 5 ✓

**Placeholder 扫描：** 无 TBD/TODO，每步均含完整替换内容。

**一致性检查：**
- Task 3 的合并流程（步骤 1-10）与 Task 5 Step 5 引用的「阶段 3 步骤 1-10」一致
- Task 4 的 `thread_title` 取值说明在 frontmatter 模板和告知文本中均有体现
