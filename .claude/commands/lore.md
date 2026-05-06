# lore — 渐进式阅读笔记工作流

你正在运行 `lore` 阅读工作流。

**输入：** `$ARGUMENTS`

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
