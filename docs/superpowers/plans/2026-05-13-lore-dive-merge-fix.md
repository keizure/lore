# lore-dive Merge Flow Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 修复 `lore-dive` skill 的合并流程与附录规则——提取独立 Merge Procedure、Bash 原文追加 session、补充附录精简格式说明。

**Architecture:** 对单个文件 `.claude/skills/lore-dive/SKILL.md` 进行四处有序编辑：(1) 阶段 3 附录格式规则补充；(2) 新增独立 `## Merge Procedure` 段落（含 Bash 原文追加）；(3) 替换阶段 3 内联合并步骤为引用；(4) 替换阶段 0 手动合并跳转引用。

**Tech Stack:** Markdown 文件编辑；Bash `printf` / `grep` / `awk` / `tail` 命令（写入 skill 指令正文）

---

## File Map

| 文件 | 操作 |
|------|------|
| `.claude/skills/lore-dive/SKILL.md` | 修改（4 处 Edit，顺序执行） |

---

### Task 1：补充阶段 3 的附录写作规则

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`（阶段 3 步骤 3，在生成文档的格式模板附录部分加入精简说明）

- [ ] **Step 1：确认待修改位置存在**

```bash
grep -n '附录：探索路径' .claude/skills/lore-dive/SKILL.md
```

预期输出：找到含 `附录：探索路径` 的行（在 `### Q1 — <标题>` 模板附近）

- [ ] **Step 2：用 Edit 替换附录模板块**

old_string：
```
```markdown
---

## 附录：探索路径

### Q1 — <标题>

**问：** ...

**答：** ...

### Q2 — <标题>

...
```
```

new_string：
```
```markdown
---

## 附录：探索路径

### Q1 — <标题>

**问：** <用户原文，完整保留>

**答：** <2-3 句核心结论，去掉例子、推导过程、图表>

### Q2 — <标题>

...
```

附录写作规则：「问：」完整保留原文；「答：」压缩为 2-3 句核心结论，目的是让读者扫描附录即可判断每轮探索讲了什么，细节保留在 session 文件。
```

- [ ] **Step 3：验证修改结果**

```bash
grep -n '2-3 句核心结论' .claude/skills/lore-dive/SKILL.md
```

预期输出：找到该行，确认规则说明已写入阶段 3

- [ ] **Step 4：Commit**

```bash
git add .claude/skills/lore-dive/SKILL.md
git commit -m "feat(lore-dive): add appendix condensed-format rule to phase-3"
```

---

### Task 2：插入独立 `## Merge Procedure` 段落

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`（在 `## Constraints` 前插入完整段落）

- [ ] **Step 1：确认插入锚点存在**

```bash
grep -n '^## Constraints' .claude/skills/lore-dive/SKILL.md
```

预期输出：找到 `## Constraints` 所在行号

- [ ] **Step 2：用 Edit 在 `## Constraints` 前插入 Merge Procedure**

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

3. **合成正文**：理解续集在主文档哪个位置深挖，将续集内容整合进对应位置，生成结构完整的新文章（不是简单拼接；只做润色和整体文档结构梳理；除非有重复，否则不删减细节）

4. **附录**：直接拼接主文档附录 + 续集附录（两份均已是精简版，不重新生成）

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

- [ ] **Step 3：验证段落已插入**

```bash
grep -c 'Merge Procedure' .claude/skills/lore-dive/SKILL.md
```

预期输出：`1`（此时只有段落标题，Task 3/4 完成后变为 3）

```bash
grep -n 'tail -n +\$N' .claude/skills/lore-dive/SKILL.md
```

预期输出：找到含 `tail -n +$N` 的行，确认 Bash 命令已写入

```bash
grep -n '直接拼接主文档附录' .claude/skills/lore-dive/SKILL.md
```

预期输出：找到步骤 4 的附录说明行

- [ ] **Step 4：Commit**

```bash
git add .claude/skills/lore-dive/SKILL.md
git commit -m "feat(lore-dive): add standalone Merge Procedure section"
```

---

### Task 3：替换阶段 3 内联合并步骤为 Merge Procedure 引用

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`（阶段 3 步骤 7 的用户选「合并」部分）

- [ ] **Step 1：确认待替换内容存在**

```bash
grep -n '快照（可选）' .claude/skills/lore-dive/SKILL.md
```

预期输出：找到含「快照（可选）」的行

- [ ] **Step 2：用 Edit 替换内联合并步骤**

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

预期输出：**无匹配**

```bash
grep -n 'main-slug = 当前 session' .claude/skills/lore-dive/SKILL.md
```

预期输出：找到该行

- [ ] **Step 4：Commit**

```bash
git add .claude/skills/lore-dive/SKILL.md
git commit -m "feat(lore-dive): replace phase-3 inline merge with Merge Procedure reference"
```

---

### Task 4：替换阶段 0 手动合并的跳转引用

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`（阶段 0 手动合并流程步骤 4）

- [ ] **Step 1：确认待替换内容存在**

```bash
grep -n '执行阶段 3' .claude/skills/lore-dive/SKILL.md
```

预期输出：找到含「执行阶段 3」的行

- [ ] **Step 2：用 Edit 替换步骤 4**

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

- [ ] **Step 3：最终整体验证**

```bash
grep -n '执行阶段 3' .claude/skills/lore-dive/SKILL.md
```

预期输出：**无匹配**

```bash
grep -c 'Merge Procedure' .claude/skills/lore-dive/SKILL.md
```

预期输出：`3`（段落标题 1 + 阶段 3 引用 1 + 阶段 0 引用 1）

```bash
grep -c '2-3 句核心结论' .claude/skills/lore-dive/SKILL.md
```

预期输出：`2`（阶段 3 附录模板 1 + 附录规则说明 1）

- [ ] **Step 4：Commit**

```bash
git add .claude/skills/lore-dive/SKILL.md
git commit -m "feat(lore-dive): replace manual merge reference with Merge Procedure"
```
