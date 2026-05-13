# lore-dive Merge Flow Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 修复 `lore-dive` skill 的合并流程——提取独立 Merge Procedure 段落，并用 Bash 原文追加替代 AI 重写 session 内容。

**Architecture:** 对单个文件 `SKILL.md` 进行三处编辑：(1) 在 `## Constraints` 前插入新的 `## Merge Procedure` 段落；(2) 把阶段 3 的内联合并步骤替换为对 Merge Procedure 的引用；(3) 把阶段 0 手动合并的引用也指向 Merge Procedure。

**Tech Stack:** Markdown 文件编辑，Bash grep/tail/printf 命令（写入 skill 指令）

---

## File Map

| 文件 | 操作 |
|------|------|
| `.claude/skills/lore-dive/SKILL.md` | 修改（3处 Edit） |

---

### Task 1：插入独立 `## Merge Procedure` 段落

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`（在 `## Constraints` 前插入新段落）

- [ ] **Step 1：确认插入位置存在**

```bash
grep -n '^## Constraints' .claude/skills/lore-dive/SKILL.md
```

预期输出：`352:## Constraints`（行号可能略有偏差，只要找到即可）

- [ ] **Step 2：用 Edit 工具在 `## Constraints` 前插入 Merge Procedure 段落**

old_string：
```
---

## Constraints
```

new_string：
```
---

## Merge Procedure

接收两个参数：
- `<main-slug>`：主文档的 slug
- `<sequel-slug>`：续集文档的 slug

执行以下步骤：

1. **git 快照**（若环境支持 git）：
   ```bash
   git add explore/<main-slug>.md explore/<sequel-slug>.md explore/<sequel-slug>-session.md
   git commit -m "chore: snapshot before merging <sequel-slug> into <main-slug>"
   ```
   若不支持，跳过此步骤并告知用户：「未创建 git 快照，若需回滚请手动备份。」

2. Read `explore/<main-slug>.md` 和 `explore/<sequel-slug>.md`

3. **合成文章**：理解续集在主文档哪个位置深挖，将续集内容整合进对应位置，生成结构完整的新文章（不是简单拼接；只做润色和整体文档结构梳理；除非有重复，否则不删减细节）

4. **附录**：主文档附录原文 + 续集附录原文顺序拼接

5. Write 覆盖 `explore/<main-slug>.md`（frontmatter `explored` 字段更新为当日日期）

6. 删除 `explore/<sequel-slug>.md`

7. 用 Bash 向 `explore/<main-slug>-session.md` 追加 header 块：
   ```bash
   printf '\n## Imported Session: <sequel-slug>\n**Continues:** <main-slug>\n**Imported at:** <YYYY-MM-DD>\n\n' \
     >> explore/<main-slug>-session.md
   ```

8. 找到续集 session frontmatter 结束后的第一行行号，用 Bash 原文追加正文：
   ```bash
   N=$(grep -n '^---$' explore/<sequel-slug>-session.md | awk -F: 'NR==2{print $1+1}')
   tail -n +$N explore/<sequel-slug>-session.md >> explore/<main-slug>-session.md
   ```

9. 删除 `explore/<sequel-slug>-session.md`

10. **更新 index.yaml：** Read `explore/index.yaml`，将续集条目的 `merged` 字段设为 `true`，用 Write 覆盖写入

11. **更新 log.md：** Read `explore/log.md`，在末尾追加 `## [<YYYY-MM-DD>] merge | <sequel-slug> → <main-slug>`，用 Write 覆盖写入

12. 告知用户：「已合并至 `explore/<main-slug>.md`，原始问答数据已合入 `explore/<main-slug>-session.md`。」

---

## Constraints
```

- [ ] **Step 3：验证新段落已插入**

```bash
grep -n 'Merge Procedure' .claude/skills/lore-dive/SKILL.md
```

预期输出两行：一行 `## Merge Procedure`，一行后续的「接收两个参数」引用行

```bash
grep -n 'tail -n +\$N' .claude/skills/lore-dive/SKILL.md
```

预期输出：找到包含 `tail -n +$N` 的行，确认 Bash 命令已写入

- [ ] **Step 4：Commit**

```bash
git add .claude/skills/lore-dive/SKILL.md
git commit -m "feat(lore-dive): add standalone Merge Procedure section"
```

---

### Task 2：替换阶段 3 内联合并步骤为 Merge Procedure 引用

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`（阶段 3 step 7 的用户选「合并」部分）

- [ ] **Step 1：确认待替换内容存在**

```bash
grep -n '快照（可选）' .claude/skills/lore-dive/SKILL.md
```

预期输出：找到含「快照（可选）」的行

- [ ] **Step 2：用 Edit 工具替换阶段 3 的内联合并步骤**

old_string：
```
   **用户选「合并」：**
   1. **快照（可选）：** 若环境支持 git，执行：
      ```bash
      git add explore/<slug>.md explore/<sequel-slug>.md explore/<sequel-slug>-session.md
      git commit -m "chore: snapshot before merging <sequel-slug> into <slug>"
      ```
      若不支持，跳过此步骤并告知用户：「未创建 git 快照，若需回滚请手动备份。」
   2. Read `explore/<slug>.md` 和 `explore/<sequel-slug>.md`
   3. **合成文章**：重新综合正文——理解续集在主文档哪个位置深挖，将续集内容整合进对应位置，生成结构完整的新文章（不是简单拼接；只做润色和整体文档结构梳理；除非有重复，否则不删减细节）
   4. **附录**：主文档附录原文 + 续集附录原文顺序拼接
   5. Write 覆盖 `explore/<slug>.md`（frontmatter `explored` 字段更新为当日日期）
   6. 删除 `explore/<sequel-slug>.md`
   7. Read `explore/<sequel-slug>-session.md`，取 frontmatter 之后的全部内容
   8. Read `explore/<slug>-session.md`，在末尾追加以下结构化块，用 Write 覆盖写入：

      ```markdown
      ## Imported Session: <sequel-slug>
      **Continues:** <slug>
      **Imported at:** <YYYY-MM-DD>

      ### Q1 — <标题>

      **问：** ...

      **答：** ...
      ```
      （保留续集原始 Q 编号，不重新排序）

   9. 删除 `explore/<sequel-slug>-session.md`
   10. **更新 index.yaml：** Read `explore/index.yaml`，将续集条目的 `merged` 字段设为 `true`，用 Write 覆盖写入
   11. **更新 log.md：** Read `explore/log.md`，在末尾追加 `## [<YYYY-MM-DD>] merge | <sequel-slug> → <slug>`，用 Write 覆盖写入
   12. 告知用户：「已合并至 `explore/<slug>.md`，原始问答数据已合入 `explore/<slug>-session.md`。」

   **用户选「保留分开」：** 不执行任何操作，两个文档都保留。
```

new_string：
```
   **用户选「合并」：** 执行 **Merge Procedure**，其中：
   - main-slug = 当前 session frontmatter 的 `continues` 字段值
   - sequel-slug = 当前 session frontmatter 的 `slug` 字段值

   **用户选「保留分开」：** 不执行任何操作，两个文档都保留。
```

- [ ] **Step 3：验证替换结果**

```bash
grep -n '快照（可选）' .claude/skills/lore-dive/SKILL.md
```

预期输出：**无匹配**（旧内容已删除）

```bash
grep -n 'main-slug = 当前 session' .claude/skills/lore-dive/SKILL.md
```

预期输出：找到该行，确认新引用已写入

- [ ] **Step 4：Commit**

```bash
git add .claude/skills/lore-dive/SKILL.md
git commit -m "feat(lore-dive): replace phase-3 inline merge with Merge Procedure reference"
```

---

### Task 3：替换阶段 0 手动合并的跳转引用

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`（阶段 0 手动合并流程 step 4）

- [ ] **Step 1：确认待替换内容存在**

```bash
grep -n '执行阶段 3' .claude/skills/lore-dive/SKILL.md
```

预期输出：找到含「执行阶段 3」的行

- [ ] **Step 2：用 Edit 工具替换 step 4**

old_string：
```
4. 用户确认 → 执行阶段 3「用户选合并」的完整流程
5. 用户取消 → 不执行任何操作，结束流程
```

new_string：
```
4. 用户确认 → 执行 **Merge Procedure**，其中：
   - main-slug = 从用户输入解析到的主 slug
   - sequel-slug = 推断/选择到的续集 slug
5. 用户取消 → 不执行任何操作，结束流程
```

- [ ] **Step 3：验证替换结果**

```bash
grep -n '执行阶段 3' .claude/skills/lore-dive/SKILL.md
```

预期输出：**无匹配**

```bash
grep -n 'main-slug = 从用户输入' .claude/skills/lore-dive/SKILL.md
```

预期输出：找到该行，确认新引用已写入

- [ ] **Step 4：最终整体验证**

```bash
grep -c 'Merge Procedure' .claude/skills/lore-dive/SKILL.md
```

预期输出：`3`（段落标题 1 处 + 阶段 3 引用 1 处 + 阶段 0 引用 1 处）

```bash
grep -n 'tail -n' .claude/skills/lore-dive/SKILL.md
```

预期输出：找到 Bash tail 命令，确认 Merge Procedure 中的 verbatim 追加指令存在

- [ ] **Step 5：Commit**

```bash
git add .claude/skills/lore-dive/SKILL.md
git commit -m "feat(lore-dive): replace manual merge reference with Merge Procedure"
```
