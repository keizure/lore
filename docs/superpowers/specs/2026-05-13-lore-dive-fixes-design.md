# lore-dive Fix 设计文档

**日期：** 2026-05-13
**来源：** fix.md — 实际使用中发现的 5 个问题
**目标文件：** `.claude/skills/lore-dive/SKILL.md`

---

## Fix 1：`paused` 语义修正 → `abandoned`

**问题：** `paused` 在直觉上表示「暂停、可恢复」，但 SKILL.md 中将其定义为终止状态（等同于 `done`），造成命名与语义矛盾，维护时容易误解。

**改动：**
- 阶段 2 话题漂移检测中，将 `status: paused` 改为 `status: abandoned`
- 删除注释「`paused` 为终止状态，不会被恢复，等同于 `done`」

**状态语义对照（改后）：**

| 状态 | 含义 |
|------|------|
| `active` | 进行中，可恢复 |
| `done` | 正常完成，已生成文档 |
| `abandoned` | 话题漂移中断，不再恢复 |

Session Recovery 只扫描 `status: active`，`done` 和 `abandoned` 均忽略。

---

## Fix 2：合成文章结构化指令

**问题：** 阶段 3 合成文章要求仅为「连贯可读，中文，不少于 300 字」，过于宽泛，容易生成平庸总结，丢失探索中的关键洞见，且不适配不同类型的探索。

**改动：** 将阶段 3 第 2 步的合成要求替换为以下指令：

```
合成文章写作指令：

1. 先判断探索类型，选择主结构：
   - 知识建构型（从零建立认知地图）
     → 叙事流结构：按概念依赖顺序展开，先讲基础后讲进阶
   - 问题深挖型（针对具体疑问深入）
     → 结论导向：先给答案，再展开论证
   - 概念厘清型（理清容易混淆的概念）
     → 对比分析：以混淆点为核心展开

2. 无论何种类型，必须包含以下两个元素
   （可嵌入正文，无需单独成节）：
   - 认知路径：记录探索中发生的认知转变
     （"以为是 X，实际是 Y，因为 Z"）
   - 易混淆点：并列展示容易混淆的概念对

3. 只做润色和整体结构梳理；不提炼删减细节；
   不简单缩写；不用通用知识结构替代探索所得

4. 中文，不少于 300 字
```

---

## Fix 3：续集合并时保留 session2 原始数据

**问题：** 阶段 3 续集合并流程在合并后删除 `<slug>-2.md`，但 `<slug>-2-session.md`（原始问答数据）未被合并进主 session，导致数据丢失。

**改动：** 在「用户选合并」流程的现有步骤之后，追加：

```
（现有步骤 1-6 不变）

7. 读取 explore/<slug>-2-session.md 的全部问答内容
8. 在 explore/<slug>-session.md 末尾追加分隔行和内容：

   --- 续集 <slug>-2 ---

   （<slug>-2-session.md 中 frontmatter 之后的全部内容）

9. 删除 explore/<slug>-2-session.md
```

合并后，`<slug>-session.md` 包含两次探索的完整原始问答数据。

---

## Fix 4：续集 session frontmatter 增加 `thread_title`

**问题：** 续集 session 文件的 `root` 字段只记录新问题（如「残差连接和 FFN 主要解决的问题是什么？」），看不出它属于哪条探索线（llm-layering），上下文断裂。

**改动：** 阶段 1 续集模式的 frontmatter 模板增加 `thread_title` 字段：

```markdown
---
root: "<新问题或继续探索的方向>"
slug: <slug>-2
continues: <original-slug>
thread_title: "<原探索的 root 问题>"
started: <YYYY-MM-DDThh:mm:ss>
status: active
---
```

`thread_title` 取自原探索 session 文件的 `root` 字段。

**附加效果：** Session Recovery 展示多个活跃 session 时，续集 session 显示格式改为：
```
「[thread_title]」的续集 → [root]（文件名）
```

---

## Fix 5：手动触发续集合并

**问题：** 续集合并提示仅在阶段 3 当次对话结束时出现；若用户关闭对话后才想合并，没有入口。自动在 Session Recovery 时检测并提示又会每次都打扰用户。

**方案：** 用户主动触发，在阶段 0 的输入检测中识别合并指令。

**触发条件：** 用户输入包含「合并」或「merge」关键词，且包含 slug 名称。

示例：
- `合并 llm-layering llm-layering-2`（指定两个 slug）
- `merge llm-layering`（只给主 slug，自动推断续集为 `llm-layering-2`）

**触发后流程：**
1. 解析出主 slug；若未指定续集 slug，自动推断为 `<slug>-2`
2. 确认 `explore/<slug>.md` 和 `explore/<slug>-2.md` 均存在
3. 确认 `explore/<slug>-2-session.md` 存在且 `status: done`
4. 提示用户确认：「要把『[slug]-2』的探索合并进『[slug]』吗？」
5. 用户确认 → 执行 Fix 3 的完整合并流程

**文件不存在时：** 提示用户检查 slug 名称，不执行任何操作。

---

## 改动范围总结

| Fix | 改动位置 | 类型 |
|-----|----------|------|
| Fix 1 | 阶段 2 话题漂移检测 | 状态值替换 |
| Fix 2 | 阶段 3 合成文章要求 | 指令替换 |
| Fix 3 | 阶段 3 续集合并流程 | 追加步骤 |
| Fix 4 | 阶段 1 续集 frontmatter 模板 | 追加字段 + Session Recovery 显示 |
| Fix 5 | 阶段 0 输入检测 | 新增分支 |

所有改动均在 `.claude/skills/lore-dive/SKILL.md` 单文件内完成。
