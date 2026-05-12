# lore-dive Session Fidelity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 修改 lore-dive skill 的阶段 2 写入协议，从「先回答后写入」改为「先写入后输出」，确保 session 文件与对话内容字符级别一致，保留所有 ASCII 图、表格、代码块。

**Architecture:** 只修改一个文件 `.claude/skills/lore-dive/SKILL.md`，将阶段 2 的步骤 1-2 替换为 Write-First Protocol。阶段 0、1、3 保持不变。

**Tech Stack:** Markdown skill 文件，无代码依赖。

---

### Task 1: 替换阶段 2 的写入协议

**Files:**
- Modify: `.claude/skills/lore-dive/SKILL.md`（阶段 2「探索循环」章节，原步骤 1-2）

- [ ] **Step 1: 读取当前 SKILL.md，定位阶段 2 的步骤 1-2**

  读取 `.claude/skills/lore-dive/SKILL.md`，找到以下原文（约第 88-100 行）：

  ```markdown
  每轮：

  1. **回答用户的问题**（充分、深入）

  2. **立即将本轮内容写入 session 文件**（回答后立即执行，不可积攒）：
     用 Read 工具读取当前 session 文件完整内容，在末尾追加以下内容，再用 Write 工具覆盖写入：

  ```markdown
  ## Q<n> <简短标题>

  **问：** <用户原文>

  **答：** <完整回答>
  ```
  ```

- [ ] **Step 2: 用 Edit 工具替换步骤 1-2 为新协议**

  将上述原文替换为：

  ```markdown
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
  ```

- [ ] **Step 3: 确认步骤 3（分支提示）也遵循同样协议**

  找到步骤 3（分支提示），确认其说明包含「先追加到 session，再输出」的顺序约束。当前原文：

  ```markdown
  3. **判断是否需要分支提示：** 如果本轮回答引出了值得独立深挖的子问题，且用户尚未问到，在答案末尾追加一行，同时写入文件：
  ```

  「同时写入文件」的表述顺序有歧义，用 Edit 工具将其改为：

  ```markdown
  3. **判断是否需要分支提示：** 如果本轮回答引出了值得独立深挖的子问题，且用户尚未问到，先将以下内容追加写入 session 文件，再输出给用户：
  ```

- [ ] **Step 4: 通读修改后的阶段 2 章节，确认无歧义**

  重新 Read SKILL.md 阶段 2 全文，检查：
  - 步骤顺序是否清晰（先 Read session → Write session → 输出给用户）
  - 没有任何「先回答」再「写入」的残余表述
  - 分支提示步骤与主步骤协议一致

- [ ] **Step 5: Commit**

  ```bash
  git add .claude/skills/lore-dive/SKILL.md
  git commit -m "feat: implement write-first protocol in lore-dive phase 2"
  ```

---

### Task 2: 验证（手动）

- [ ] **Step 1: 发起一个新的 lore-dive 会话，提出一个需要图表的问题**

  例如：「解释 TCP 三次握手的过程」。确认 AI 在输出给用户之前先执行了 Write 操作（在工具调用序列中，Write 出现在文字输出之前）。

- [ ] **Step 2: 查看生成的 session 文件**

  打开 `explore/*-session.md`，确认：
  - `**答：**` 部分包含完整的图表/表格，与对话中显示的内容一致
  - 没有被压缩为纯文字摘要

- [ ] **Step 3: 触发「结束」生成文档**

  说「结束」，确认生成的 `explore/<slug>.md` 正文中图表内容被完整保留（或合理转换），而非丢失。
