# lore-dive Merge Flow Fix — Design Spec

**Date:** 2026-05-13
**Scope:** `lore-dive` skill (`SKILL.md`) 的三处改动

---

## 问题描述

### 问题 1：session 合并时内容被总结而非原文复制

当前步骤 8 给出了示例格式 `### Q1 — <标题>`，与 session 文件里实际的 `## Q1 标题` 格式不同。AI 看到格式差异后判断需要「重新整理」，导致续集 session 内容被总结压缩而非逐字复制。

### 问题 2：手动合并与阶段 3 合并逻辑分叉

- 阶段 0 的手动合并流程通过引用跳转到阶段 3（「执行阶段 3『用户选合并』的完整流程」）
- 两条路径的上下文不同（手动：两文档已存在；阶段 3：刚生成续集文档），容易导致 AI 跳过 git 快照或混淆变量
- 合并逻辑散落在阶段 3 深处，不是独立可复用的单元

### 问题 3：附录写作规则缺失

阶段 3 生成文档时，附录只有格式模板（`### Q1 — <标题>` + 完整问答），没有说明附录的精简性质。Merge Procedure 里同样没有说明。

附录的设计意图是**精简版问答索引**，类似目录——读者扫一眼即可了解每轮探索的核心内容，完整原始问答保留在 session 文件里。

---

### Fix 1：Verbatim Session Import（Bash 原文追加）

**替换阶段 3 步骤 7-8（以及 Merge Procedure 中的对应步骤）为：**

```
7. 用 Bash 向主 session 追加 header 块：
   printf '\n## Imported Session: <sequel-slug>\n**Continues:** <main-slug>\n**Imported at:** <YYYY-MM-DD>\n\n' \
     >> explore/<main-slug>-session.md

8. 找到续集 session frontmatter 结束后的第一行行号 N
   （即第二个 `---` 所在行 + 1），然后用 Bash 原文追加：
   tail -n +N explore/<sequel-slug>-session.md >> explore/<main-slug>-session.md
```

**原则：**
- AI 只负责生成 header 块的 3 个字段值（sequel-slug、main-slug、日期）
- session 正文通过 Bash 命令原文追加，完全绕过 AI 处理
- 移除原有的示例格式 `### Q1 — <标题>`，消除「格式转换」暗示

---

### Fix 2：提取独立 `## Merge Procedure` 段落

在 `SKILL.md` 末尾（Constraints 之前）新增独立段落，接收两个输入参数：
- `<main-slug>`：主文档的 slug
- `<sequel-slug>`：续集文档的 slug

**Merge Procedure 步骤：**

1. **git 快照**（若环境支持）：
   ```bash
   git add explore/<main-slug>.md explore/<sequel-slug>.md explore/<sequel-slug>-session.md
   git commit -m "chore: snapshot before merging <sequel-slug> into <main-slug>"
   ```
   若不支持，告知用户「未创建 git 快照，若需回滚请手动备份。」

2. Read `explore/<main-slug>.md` 和 `explore/<sequel-slug>.md`

3. **合成正文**：理解续集在主文档哪个位置深挖，将续集内容整合进对应位置，生成结构完整的新文章（不是简单拼接；只做润色和整体文档结构梳理；除非有重复，否则不删减细节）

4. **附录**：主文档附录原文 + 续集附录原文顺序拼接

5. Write 覆盖 `explore/<main-slug>.md`（frontmatter `explored` 字段更新为当日日期）

6. 删除 `explore/<sequel-slug>.md`

7. 用 Bash 向 `explore/<main-slug>-session.md` 追加 header 块：
   ```bash
   printf '\n## Imported Session: <sequel-slug>\n**Continues:** <main-slug>\n**Imported at:** <YYYY-MM-DD>\n\n' \
     >> explore/<main-slug>-session.md
   ```

8. 找到 `explore/<sequel-slug>-session.md` frontmatter 结束后的第一行行号 N，用 Bash 原文追加：
   ```bash
   tail -n +N explore/<sequel-slug>-session.md >> explore/<main-slug>-session.md
   ```

9. 删除 `explore/<sequel-slug>-session.md`

10. 更新 index.yaml：将续集条目的 `merged` 字段设为 `true`

11. 更新 log.md：追加 `## [<YYYY-MM-DD>] merge | <sequel-slug> → <main-slug>`

12. 告知用户：「已合并至 `explore/<main-slug>.md`，原始问答数据已合入 `explore/<main-slug>-session.md`。」

---

### Fix 3：附录写作规则（阶段 3 + Merge Procedure 共用）

附录每条的格式：

```markdown
### Q<n> — <简短标题>

**问：** <用户原文，完整保留>

**答：** <2-3 句核心结论，不展开论证，不含代码块和 ASCII 图>
```

**规则：**
- 「问：」原文完整保留，不删减
- 「答：」压缩为 2-3 句核心结论，去掉例子、推导过程、图表
- 目的是让读者扫描附录即可判断「这条探索讲了什么」，需要细节时去 session 文件查

**适用位置：**
- 阶段 3 生成文档时：附录由当前 session 的所有 Q&A 精简生成
- Merge Procedure 步骤 4：直接拼接主文档附录 + 续集附录（两份均已是精简版，不重新生成）

---

### Fix 4：更新两处触发点

**阶段 0 手动合并流程步骤 4（替换原内容）：**
```
4. 用户确认 → 执行 Merge Procedure
   - main-slug = <从用户输入解析到的主 slug>
   - sequel-slug = <推断/选择到的续集 slug>
5. 用户取消 → 不执行任何操作，结束流程
```

**阶段 3 步骤 7 续集合并提示，用户选「合并」（替换原步骤 1-12）：**
```
用户选「合并」→ 执行 Merge Procedure
   - main-slug = <当前 session frontmatter 的 continues 字段值>
   - sequel-slug = <当前 session frontmatter 的 slug 字段值>
```

---

## 改动范围

| 位置 | 改动 |
|------|------|
| 阶段 0 手动合并流程 步骤 4 | 替换为「执行 Merge Procedure，传入 main/sequel slug」 |
| 阶段 3 步骤 7 用户选「合并」的步骤 1-12 | 替换为「执行 Merge Procedure，传入 main/sequel slug」 |
| 阶段 3 附录写作规则 | 新增精简格式说明（问：原文 + 答：2-3 句） |
| Merge Procedure 步骤 4 | 明确「直接拼接两份精简附录，不重新生成」 |

---

## 不在本次范围内

- 阶段 0/1/2 其他逻辑不变
- 合成文章的正文写作指令不变
- index.yaml / log.md 格式不变
