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
   - 读取完整内容（恢复上下文）
   - 告知用户：「发现进行中的探索：『[root]』，继续这个 session 吗？还是开启新探索？」
   - 用户选「继续」→ 跳至阶段 2，从上次中断处继续
   - 用户选「新开」→ 进入阶段 1
4. **未找到活跃 session：** 直接进入阶段 1

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
started: <YYYY-MM-DDThh:mm:ss>
status: active
---
```

3. 立即进入阶段 2，回答根问题。

---

### 阶段 2：探索循环（每轮重复）

每轮：

1. **回答用户的问题**（充分、深入）

2. **立即用 Edit 工具追加写入 session 文件**（回答后立即执行，不可积攒）：

```markdown
## Q<n> <简短标题>

**问：** <用户原文>

**答：** <完整回答>
```

3. **判断是否需要分支提示：** 如果本轮回答引出了值得独立深挖的子问题，且用户尚未问到，在答案末尾追加一行，同时写入文件：
   > 💡 **未探索分支：** <简短描述>

   不强制用户跟进，用户可直接忽略继续提问。

4. **检测话题漂移：** 如果用户新问题与根问题语义距离较远，在回答前先提示：
   > 「这个问题和当前 session『[root]』关联不大，要新开一个 session 吗？还是继续归入当前？」

   用户选「新开」→ 用 Edit 将当前 session `status` 改为 `paused`，回到阶段 1。

5. **检测完成意图（每轮检查）：** 用户说「够了」「结束」「可以了」「生成文档」「好了」「done」→ 立即进入阶段 3。

---

### 阶段 3：生成双层文档

1. 用 Read 工具读取完整 session 文件
2. 将所有问答合成为连贯文章（去掉「问：/答：」格式，直接可读，保留核心内容）
3. 用 Write 工具生成 `explore/<slug>.md`：

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

---

## Constraints

- 每轮问答后**立即写入** session 文件，不可积攒到对话结束
- 只写入 `explore/` 目录，不修改其他文件
- 合成文章使用**中文**，连贯可读，不少于 300 字
- 每次启动**必须先执行 Session Recovery**，不可跳过
- 分支提示只在 AI 主动判断有必要时出现，不强制每轮都加
- 同一领域的深入问题**不算**话题漂移
