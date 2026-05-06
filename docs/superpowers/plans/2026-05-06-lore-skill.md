# Lore Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create `.claude/commands/lore.md`, a project-level slash command that guides Claude through a 4-phase progressive article reading workflow (init → summary → translation → deep-dive), writing to `reading/<slug>.md` at each phase.

**Architecture:** A single markdown prompt file placed in `.claude/commands/`. Claude Code injects the file content as a system prompt when the user types `/lore <args>`, with `$ARGUMENTS` substituted for the user's input. No code, no dependencies — the entire implementation is the prompt file itself.

**Tech Stack:** Claude Code slash commands (markdown prompt files), web_fetch MCP tool (for URL input), Read/Write/Edit tools (for file I/O)

---

## File Map

| Action | Path | Responsibility |
|--------|------|----------------|
| Create | `.claude/commands/lore.md` | The complete skill prompt |
| Create | `reading/` | Output directory (created on first use by the skill) |

---

### Task 1: Create the `.claude/commands/` directory and skeleton file

**Files:**
- Create: `.claude/commands/lore.md`

- [ ] **Step 1: Create the directory**

```bash
mkdir -p .claude/commands
```

- [ ] **Step 2: Create the skeleton file with header only**

Create `.claude/commands/lore.md` with this content:

```markdown
# lore — 渐进式阅读笔记工作流

你正在运行 `lore` 阅读工作流。

**输入：** `$ARGUMENTS`
```

- [ ] **Step 3: Verify the file exists**

```bash
ls -la .claude/commands/lore.md
```

Expected output: file listed with non-zero size.

- [ ] **Step 4: Commit**

```bash
git add .claude/commands/lore.md
git commit -m "feat: add lore command skeleton"
```

---

### Task 2: Implement Phase 0 — Initialization

**Files:**
- Modify: `.claude/commands/lore.md`

This phase reads the input (URL or local file), extracts the title, derives the slug, and creates the output file header.

- [ ] **Step 1: Append Phase 0 instructions to the skill file**

Append the following to `.claude/commands/lore.md`:

````markdown

---

## 阶段 0：初始化

执行以下步骤，不要等待用户输入：

1. **判断输入类型：**
   - 如果 `$ARGUMENTS` 以 `http://` 或 `https://` 开头：使用 `web_fetch` 工具抓取内容
   - 否则：使用 `Read` 工具读取本地文件

2. **提取文章标题：** 从内容的第一个 `<h1>` 标签或 Markdown 的 `# ` 标题行提取；若无明确标题，从内容首段推断一个简短标题。

3. **推导 slug：** 将标题转为小写，将空格和特殊字符替换为连字符，只保留字母、数字和连字符。例：
   - "LLM Wiki Pattern" → `llm-wiki-pattern`
   - "What Async Promised and What it Delivered" → `what-async-promised`
   - "函数着色问题" → `function-coloring`

4. **确认 `reading/` 目录存在：** 使用 Bash 工具运行 `mkdir -p reading`

5. **创建输出文件** `reading/<slug>.md`，写入以下内容（替换占位符）：

```markdown
# <文章标题>

> 原文：<$ARGUMENTS 的值>
> 整理日期：<今天的日期，格式 YYYY-MM-DD>

---
```

完成后立即进入阶段 1，不要暂停。
````

- [ ] **Step 2: Verify syntax by reading back the file**

```bash
cat .claude/commands/lore.md
```

Expected: no truncation, markdown looks correct.

- [ ] **Step 3: Commit**

```bash
git add .claude/commands/lore.md
git commit -m "feat(lore): add phase 0 initialization"
```

---

### Task 3: Implement Phase 1 — Summary

**Files:**
- Modify: `.claude/commands/lore.md`

This phase auto-generates a structured Chinese summary and writes it to the file before asking for user input.

- [ ] **Step 1: Append Phase 1 instructions**

Append the following to `.claude/commands/lore.md`:

````markdown

---

## 阶段 1：摘要（自动执行）

不需要用户触发，直接执行：

1. **生成结构化中文摘要**，要求：
   - 一句话概括：捕捉文章最核心的洞见，不超过 40 字
   - 核心论点：3-7 条，每条一行，聚焦文章的主要观点和论证
   - 关键概念：列出文章涉及的重要术语/概念名称（3-8 个）
   - 总篇幅不超过原文的 20%

2. **追加写入文件** `reading/<slug>.md`（在现有内容末尾追加）：

```markdown
## 摘要

### 一句话概括
<内容>

### 核心论点
- <论点 1>
- <论点 2>
- ...

### 关键概念
- <概念 1>
- <概念 2>
- ...

---
```

3. **告知用户** 文件已更新，并在对话中展示摘要全文。

4. **询问用户：**

> 摘要已写入 `reading/<slug>.md`。
>
> 接下来你想：
> - **继续翻译**（逐段中英对照）
> - **跳过翻译，直接下钻某个概念**（告诉我概念名称）
> - **完成**，就此结束

等待用户回复后，根据回复进入对应阶段：
- 包含「翻译」、「继续」、「好」、「可以」→ 进入阶段 2
- 包含具体概念名称或「下钻」→ 跳过阶段 2，进入阶段 3
- 包含「完成」、「结束」、「不需要」→ 进入退出流程
````

- [ ] **Step 2: Commit**

```bash
git add .claude/commands/lore.md
git commit -m "feat(lore): add phase 1 summary"
```

---

### Task 4: Implement Phase 2 — Translation

**Files:**
- Modify: `.claude/commands/lore.md`

This phase translates the full article paragraph by paragraph, writing to the file before asking what's next.

- [ ] **Step 1: Append Phase 2 instructions**

Append the following to `.claude/commands/lore.md`:

````markdown

---

## 阶段 2：翻译（用户确认后执行）

1. **逐段翻译原文**，以段落为单位，每段格式如下：

```markdown
**原文**
> <原文段落，保持原文不变>

**译文**
<中文翻译>

---
```

   翻译要求：
   - 忠实原意，不意译过度
   - 保留专有名词（首次出现可附原文）
   - 保持原文的段落结构和标题层级

2. **追加写入文件** `reading/<slug>.md`：

```markdown
## 翻译

**原文**
> <第一段原文>

**译文**
<第一段译文>

---

**原文**
> <第二段原文>

**译文**
<第二段译文>

---
```

3. **写入完成后告知用户：**

> 翻译已追加到 `reading/<slug>.md`。
>
> 想对哪些概念做深度下钻？请直接告诉我概念名称（可以多个），或说「完成」退出。

等待用户回复，然后：
- 用户提供概念名称 → 进入阶段 3
- 用户说「完成」、「结束」、「不需要」→ 进入退出流程
````

- [ ] **Step 2: Commit**

```bash
git add .claude/commands/lore.md
git commit -m "feat(lore): add phase 2 translation"
```

---

### Task 5: Implement Phase 3 — Concept Deep-Dive and Exit

**Files:**
- Modify: `.claude/commands/lore.md`

This phase handles user-specified concept deep-dives (repeatable) and the exit flow.

- [ ] **Step 1: Append Phase 3 and exit instructions**

Append the following to `.claude/commands/lore.md`:

````markdown

---

## 阶段 3：概念下钻（用户指定，可重复）

用户每次指定一个或多个概念名称时执行：

1. **对每个概念生成深度展开内容**，包括：
   - 背景：这个概念从何而来、解决什么问题
   - 原理：核心机制是什么，如何运作
   - 与其他概念的关系：和本文其他概念的联系、对比
   - 示例（如适用）：具体代码、案例或类比说明

2. **写入文件** `reading/<slug>.md`：
   - 如果文件中还没有 `## 深度笔记` 节，先追加该节标题：
     ```markdown
     ## 深度笔记
     ```
   - 然后追加概念子节：
     ```markdown
     ### <概念名>

     <深度展开内容>

     ---
     ```

3. **每个概念写入后告知用户，** 然后询问：

> `<概念名>` 的深度笔记已追加到 `reading/<slug>.md`。
>
> 还有其他概念要下钻吗？（直接说概念名称，或说「完成」退出）

重复此流程直到用户退出。

---

## 退出流程

当用户说「完成」、「结束」、「不需要了」、「好了」或类似表达时：

告知用户：

> 笔记已完成，文件保存在 `reading/<slug>.md`。

然后停止，不再执行任何操作。

---

## 全局约束

- 所有生成的内容必须使用**中文**
- 只写入 `reading/` 目录，不修改其他任何文件
- 每个阶段必须先写入文件，再向用户展示内容或提问
- 摘要总篇幅不超过原文的 20%
- 翻译以段落为单位，不拆散段落
- 概念下钻只聚焦用户指定的概念，不随意扩展到其他概念
````

- [ ] **Step 2: Commit**

```bash
git add .claude/commands/lore.md
git commit -m "feat(lore): add phase 3 deep-dive and exit flow"
```

---

### Task 6: Smoke Test — URL Input

**Files:**
- Read: `.claude/commands/lore.md` (verify it loads correctly)
- Creates: `reading/what-color-is-your-function.md` (or similar)

This is a manual end-to-end test. Run it in a Claude Code session in this repo.

- [ ] **Step 1: Open Claude Code in this repo**

```bash
cd /Users/liangjinrun/keisure/notebook/dicovery
claude
```

- [ ] **Step 2: Run lore with a URL**

Type in the Claude Code session:
```
/lore https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/
```

(This is Bob Nystrom's "What Color is Your Function?" article — the origin of the function coloring concept referenced in the async article.)

- [ ] **Step 3: Verify Phase 0 output**

Check that `reading/what-color-is-your-function.md` was created with:
- Correct title
- Original URL in header
- Today's date

```bash
head -6 reading/what-color-is-your-function.md
```

Expected:
```markdown
# What Color is Your Function?

> 原文：https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/
> 整理日期：2026-05-06

---
```

- [ ] **Step 4: Verify Phase 1 output**

After the skill presents the summary, check the file:

```bash
grep -A 20 "## 摘要" reading/what-color-is-your-function.md
```

Expected: `### 一句话概括`, `### 核心论点`, `### 关键概念` sections all present and in Chinese.

- [ ] **Step 5: Test skip-translation path**

In the Claude Code session, reply: `跳过翻译，下钻"函数着色"`

Expected: Skill skips to Phase 3, writes `## 深度笔记` + `### 函数着色` section to the file.

- [ ] **Step 6: Verify deep-dive output**

```bash
grep -A 5 "## 深度笔记" reading/what-color-is-your-function.md
```

Expected: `### 函数着色` sub-section present with Chinese content.

- [ ] **Step 7: Exit**

Reply: `完成`

Expected: Skill says file is saved and stops.

---

### Task 7: Smoke Test — Local File Input

**Files:**
- Read: `reading/karpathy-llm-wiki-summary.md` (use existing file as raw input)
- Creates: `reading/<slug>.md` for local file test

- [ ] **Step 1: Run lore with a local file path**

In a Claude Code session:
```
/lore reading/async-future-directions.md
```

- [ ] **Step 2: Verify it reads the local file correctly**

The skill should NOT use web_fetch. It should use the Read tool and pick up the content from the local file. Verify in the conversation that Claude references content from `async-future-directions.md`.

- [ ] **Step 3: Verify output file is created**

```bash
ls reading/
```

Expected: a new `.md` file created (e.g., `async-future-directions-notes.md` or similar — slug derived from the content title).

- [ ] **Step 4: Test full translation path**

Reply: `继续翻译`

Expected: Skill generates translation and appends `## 翻译` section to the file.

- [ ] **Step 5: Test multi-concept deep-dive**

Reply: `下钻"虚拟线程"和"代数效应"`

Expected: Both `### 虚拟线程` and `### 代数效应` sub-sections written under `## 深度笔记`.

- [ ] **Step 6: Commit the test output files if they look good**

```bash
git add reading/
git commit -m "test: add lore smoke test output files"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Covered by |
|-----------------|-----------|
| URL input → web_fetch | Task 2 (Phase 0) |
| Local file input → Read tool | Task 2 (Phase 0) |
| Slug derivation | Task 2 (Phase 0) |
| Create file header at init | Task 2 (Phase 0) |
| Auto summary with three sub-sections | Task 3 (Phase 1) |
| Write before asking user | Task 3 (Phase 1) |
| Skip translation path | Task 3 (Phase 1) |
| Full translation with original+translation format | Task 4 (Phase 2) |
| User-specified concept deep-dive | Task 5 (Phase 3) |
| Repeatable deep-dive loop | Task 5 (Phase 3) |
| Exit on "完成" / "结束" | Task 5 (exit flow) |
| All output in Chinese | Task 5 (global constraints) |
| Only write to reading/ | Task 5 (global constraints) |
| File-first, then show user | Task 3, 4, 5 (each phase) |
| Smoke test URL | Task 6 |
| Smoke test local file | Task 7 |

All spec requirements covered. No gaps found.
