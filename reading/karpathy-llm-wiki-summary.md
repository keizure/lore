# LLM Wiki 模式：总结

> 原文：https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
> 作者：Andrej Karpathy（2026-04-04）

---

## 一句话概括

用 LLM 代替人工，持续维护一个结构化的个人 Wiki，让知识从"每次从零检索"变成"持续复利积累"。

---

## 与 RAG 的根本区别

传统 RAG 的问题：每次查询时从原始文档临时检索、临时拼凑，什么都不会沉淀下来。

LLM Wiki 的做法：每添加一篇新资料，LLM 立刻将其整合进一套持久化的 Markdown Wiki——更新相关实体页面、修订摘要、标注矛盾、补充交叉引用。知识被"编译"一次后持续维护，不再每次重新推导。

---

## 三层架构

| 层 | 内容 | 所有者 |
|---|---|---|
| 原始资料层 | 文章、论文、图片等，不可修改 | 你 |
| Wiki 层 | LLM 生成的 Markdown 文件集合 | LLM |
| Schema 层 | CLAUDE.md / AGENTS.md，定义结构与工作流 | 你 + LLM 共同迭代 |

Schema 是核心配置，它把 LLM 从泛化聊天机器人变成有纪律的 Wiki 维护者。

---

## 三个核心操作

**摄入（Ingest）**：放入新资料 → LLM 阅读、讨论、写摘要、更新索引和相关页面（一篇资料可能触及 10~15 个页面）。建议逐篇处理，全程参与。

**查询（Query）**：向 Wiki 提问 → LLM 搜索相关页面并综合作答，答案本身可以作为新页面归档回 Wiki，形成复利效应。

**检查（Lint）**：定期健康检查 → 找矛盾、找孤立页面、找遗漏的交叉引用、找可以补充的内容空白。

---

## 两个导航文件

- **index.md**：目录，每个页面一行描述，按类别组织，LLM 每次摄入后更新，查询时先读此文件定位。
- **log.md**：只追加的操作日志，记录所有摄入、查询、Lint 事件，建议使用统一前缀格式，支持 `grep` 快速解析。

---

## 为什么有效

维护知识库最累的不是阅读，而是记账式的维护工作——更新交叉引用、保持一致性、标注矛盾。人类会因此放弃 Wiki；LLM 不会厌倦，维护成本趋近于零。

分工模型：**人类负责策划资料、提出好问题、思考意义；LLM 负责其他一切。**

这在精神上延续了 Vannevar Bush 1945 年提出的 Memex 构想——私人、主动策划、文档间连接与文档本身同等重要。Bush 未能解决的"谁来维护"问题，由 LLM 解决了。

---

## 推荐工具组合

| 工具 | 用途 |
|---|---|
| Obsidian | Wiki 浏览与编辑（IDE 角色） |
| Obsidian Web Clipper | 浏览器扩展，一键将网页转为 Markdown 资料 |
| Obsidian Graph View | 可视化 Wiki 结构与链接关系 |
| Marp（Obsidian 插件） | 从 Wiki 内容直接生成 Markdown 幻灯片 |
| Dataview（Obsidian 插件） | 基于 YAML 元数据生成动态表格 |
| qmd | 本地 Markdown 搜索引擎（BM25+向量，支持 CLI 和 MCP） |
| Git | Wiki 本身即 Markdown 文件仓库，版本管理免费获得 |

---

## 适用场景

个人成长追踪、学术研究、读书笔记、企业内部知识库、竞争分析、尽职调查、旅行规划、课程笔记等——**凡是需要随时间积累知识、希望有序而非零散的场合**。

---

## 核心心智模型

> Obsidian 是 IDE；LLM 是程序员；Wiki 是代码库。

你是架构师——决定研究什么、往哪个方向深挖。LLM 是全职维护工程师——负责一切繁琐但不可或缺的基础工作。

# LLM Wiki — 逐段翻译

> 原文：https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
> 作者：Andrej Karpathy（2026-04-04）
> 译注：以"段落原文 → 中文"的形式逐段对照呈现。

---

## 标题与导言

**原文**
> A pattern for building personal knowledge bases using LLMs.
>
> This is an idea file, it is designed to be copy pasted to your own LLM Agent (e.g. OpenAI Codex, Claude Code, OpenCode / Pi, or etc.). Its goal is to communicate the high level idea, but your agent will build out the specifics in collaboration with you.

**译文**
这是一种用 LLM 构建个人知识库的模式。

这是一份想法文档，设计上供你直接复制粘贴给自己的 LLM Agent（如 OpenAI Codex、Claude Code、OpenCode/Pi 等）。它的目标是传达核心思路，具体实现细节由你与 Agent 协作完成。

---

## 核心思路（The core idea）

**原文（第1段）**
> Most people's experience with LLMs and documents looks like RAG: you upload a collection of files, the LLM retrieves relevant chunks at query time, and generates an answer. This works, but the LLM is rediscovering knowledge from scratch on every question. There's no accumulation. Ask a subtle question that requires synthesizing five documents, and the LLM has to find and piece together the relevant fragments every time. Nothing is built up. NotebookLM, ChatGPT file uploads, and most RAG systems work this way.

**译文**
大多数人使用 LLM 处理文档的方式是 RAG：你上传一批文件，LLM 在查询时检索相关片段并生成答案。这种方式可用，但 LLM 每次回答都要从零开始重新发现知识，没有任何积累。如果提出一个需要综合五份文档才能回答的深层问题，LLM 每次都得重新查找、拼凑相关片段。什么东西都没有沉淀下来。NotebookLM、ChatGPT 文件上传以及大多数 RAG 系统都是这样运作的。

---

**原文（第2段）**
> The idea here is different. Instead of just retrieving from raw documents at query time, the LLM **incrementally builds and maintains a persistent wiki** — a structured, interlinked collection of markdown files that sits between you and the raw sources. When you add a new source, the LLM doesn't just index it for later retrieval. It reads it, extracts the key information, and integrates it into the existing wiki — updating entity pages, revising topic summaries, noting where new data contradicts old claims, strengthening or challenging the evolving synthesis. The knowledge is compiled once and then *kept current*, not re-derived on every query.

**译文**
这里提出的思路不同。LLM 不是在查询时才从原始文档中检索，而是**增量式地构建并维护一个持久化的 Wiki**——一套结构化、相互交叉链接的 Markdown 文件集合，置于你和原始资料之间。当你添加一篇新资料时，LLM 不只是为了之后检索而对其建立索引，而是阅读它、提取关键信息，并将其整合进现有 Wiki——更新实体页面、修订主题摘要、标注新数据与旧观点的矛盾之处，强化或挑战正在演进的综合认知。知识被一次性编译，然后*持续更新*，而不是在每次查询时重新推导。

---

**原文（第3段）**
> This is the key difference: **the wiki is a persistent, compounding artifact.** The cross-references are already there. The contradictions have already been flagged. The synthesis already reflects everything you've read. The wiki keeps getting richer with every source you add and every question you ask.

**译文**
这是关键区别：**Wiki 是一个持久化、复利式累积的制品。** 交叉引用已经在那里了，矛盾已经被标注了，综合认知已经反映了你读过的所有内容。每添加一篇资料、每问一个问题，Wiki 就更丰富一分。

---

**原文（第4段）**
> You never (or rarely) write the wiki yourself — the LLM writes and maintains all of it. You're in charge of sourcing, exploration, and asking the right questions. The LLM does all the grunt work — the summarizing, cross-referencing, filing, and bookkeeping that makes a knowledge base actually useful over time. In practice, I have the LLM agent open on one side and Obsidian open on the other. The LLM makes edits based on our conversation, and I browse the results in real time — following links, checking the graph view, reading the updated pages. Obsidian is the IDE; the LLM is the programmer; the wiki is the codebase.

**译文**
你从不（或极少）亲自写 Wiki——LLM 负责撰写和维护全部内容。你负责挑选资料来源、探索方向、提出正确的问题。LLM 则承担所有繁琐的工作——摘要、交叉引用、归档、记账，正是这些工作让知识库随时间推移真正变得有用。在我的实际操作中，一侧开着 LLM Agent，另一侧开着 Obsidian。LLM 根据对话做出修改，我实时浏览结果——跟着链接走、查看图谱视图、阅读更新后的页面。Obsidian 是 IDE；LLM 是程序员；Wiki 是代码库。

---

**原文（第5段：应用场景列表）**
> This can apply to a lot of different contexts...

**译文**
这套模式可以应用于许多不同场景，例如：

- **个人**：追踪自己的目标、健康、心理、自我提升——归档日记、文章、播客笔记，逐步构建关于自己的结构化图景。
- **研究**：在数周或数月内深入研究一个课题——阅读论文、文章、报告，增量式构建带有演进论点的综合 Wiki。
- **读书**：逐章归档，为人物、主题、情节线索构建页面，并建立相互连接。读完后你会拥有一套丰富的伴随 Wiki。可以想象像 [Tolkien Gateway](https://tolkiengateway.net/wiki/Main_Page) 这样的粉丝 Wiki——数千个相互链接的页面，由志愿者社区历经多年构建。你个人读书时就可以做出类似的东西，由 LLM 负责所有交叉引用和维护工作。
- **企业/团队**：由 LLM 维护的内部 Wiki，以 Slack 消息、会议记录、项目文档、客户通话为输入，可加入人工审核环节。Wiki 之所以能保持最新，是因为 LLM 负责了团队中没人愿意做的维护工作。
- **竞争分析、尽职调查、旅行规划、课程笔记、兴趣深潜**——任何需要随时间积累知识、希望有序而非零散的场合。

---

## 架构（Architecture）

**原文**
> There are three layers:
>
> **Raw sources** — your curated collection of source documents. Articles, papers, images, data files. These are immutable — the LLM reads from them but never modifies them. This is your source of truth.
>
> **The wiki** — a directory of LLM-generated markdown files. Summaries, entity pages, concept pages, comparisons, an overview, a synthesis. The LLM owns this layer entirely. It creates pages, updates them when new sources arrive, maintains cross-references, and keeps everything consistent. You read it; the LLM writes it.
>
> **The schema** — a document (e.g. CLAUDE.md for Claude Code or AGENTS.md for Codex) that tells the LLM how the wiki is structured, what the conventions are, and what workflows to follow when ingesting sources, answering questions, or maintaining the wiki. This is the key configuration file — it's what makes the LLM a disciplined wiki maintainer rather than a generic chatbot. You and the LLM co-evolve this over time as you figure out what works for your domain.

**译文**
架构分三层：

**原始资料层（Raw sources）**——你策划的原始文档集合：文章、论文、图片、数据文件。这一层不可变——LLM 只读，不修改。这是你的事实来源（source of truth）。

**Wiki 层**——LLM 生成的 Markdown 文件目录：摘要、实体页面、概念页面、对比、概览、综合分析。LLM 完全拥有这一层，负责创建页面、在新资料到来时更新页面、维护交叉引用、保持全局一致性。你来读，LLM 来写。

**Schema 层**——一份配置文档（如 Claude Code 用 CLAUDE.md，Codex 用 AGENTS.md），告诉 LLM Wiki 的结构是什么、规范是什么、在处理新资料/回答问题/维护 Wiki 时应遵循哪些工作流程。这是核心配置文件——正是它让 LLM 成为有纪律的 Wiki 维护者，而非泛化的聊天机器人。随着你弄清楚什么在你的领域中有效，你和 LLM 会共同迭代这份文档。

---

## 操作（Operations）

**原文（Ingest）**
> **Ingest.** You drop a new source into the raw collection and tell the LLM to process it. An example flow: the LLM reads the source, discusses key takeaways with you, writes a summary page in the wiki, updates the index, updates relevant entity and concept pages across the wiki, and appends an entry to the log. A single source might touch 10-15 wiki pages. Personally I prefer to ingest sources one at a time and stay involved — I read the summaries, check the updates, and guide the LLM on what to emphasize. But you could also batch-ingest many sources at once with less supervision. It's up to you to develop the workflow that fits your style and document it in the schema for future sessions.

**译文**
**摄入（Ingest）**。将一篇新资料放入原始资料集，让 LLM 处理它。示例流程：LLM 阅读资料，与你讨论关键要点，在 Wiki 中撰写摘要页面，更新索引，更新 Wiki 中相关的实体和概念页面，并在日志中追加一条记录。一篇资料可能会涉及 10~15 个 Wiki 页面。我个人倾向于每次处理一篇资料并全程参与——阅读摘要、检查更新、引导 LLM 重点强调什么。但你也可以一次性批量摄入多篇资料，减少监督。如何发展出适合自己风格的工作流程，由你来决定，并记录在 Schema 中供后续使用。

---

**原文（Query）**
> **Query.** You ask questions against the wiki. The LLM searches for relevant pages, reads them, and synthesizes an answer with citations. Answers can take different forms depending on the question — a markdown page, a comparison table, a slide deck (Marp), a chart (matplotlib), a canvas. The important insight: **good answers can be filed back into the wiki as new pages.** A comparison you asked for, an analysis, a connection you discovered — these are valuable and shouldn't disappear into chat history. This way your explorations compound in the knowledge base just like ingested sources do.

**译文**
**查询（Query）**。你向 Wiki 提问。LLM 搜索相关页面，阅读后综合出带引用的答案。答案可以根据问题采取不同形式——Markdown 页面、对比表格、幻灯片（Marp）、图表（matplotlib）、画布。关键洞见：**好的答案可以作为新页面归档回 Wiki。** 你请求的对比分析、你发现的关联——这些都有价值，不应消失在聊天记录中。这样，你的探索就像摄入的资料一样，在知识库中持续复利积累。

---

**原文（Lint）**
> **Lint.** Periodically, ask the LLM to health-check the wiki. Look for: contradictions between pages, stale claims that newer sources have superseded, orphan pages with no inbound links, important concepts mentioned but lacking their own page, missing cross-references, data gaps that could be filled with a web search. The LLM is good at suggesting new questions to investigate and new sources to look for. This keeps the wiki healthy as it grows.

**译文**
**检查（Lint）**。定期让 LLM 对 Wiki 做健康检查：查找页面间的矛盾、被新资料推翻的陈旧观点、没有入链的孤立页面、被提及但缺少独立页面的重要概念、缺失的交叉引用、可以通过网络搜索填补的数据空白。LLM 擅长建议新的探究问题和新的资料来源。这让 Wiki 在增长的同时保持健康。

---

## 索引与日志（Indexing and logging）

**原文**
> Two special files help the LLM (and you) navigate the wiki as it grows. They serve different purposes:
>
> **index.md** is content-oriented. It's a catalog of everything in the wiki — each page listed with a link, a one-line summary, and optionally metadata like date or source count. Organized by category (entities, concepts, sources, etc.). The LLM updates it on every ingest. When answering a query, the LLM reads the index first to find relevant pages, then drills into them. This works surprisingly well at moderate scale (~100 sources, ~hundreds of pages) and avoids the need for embedding-based RAG infrastructure.
>
> **log.md** is chronological. It's an append-only record of what happened and when — ingests, queries, lint passes. A useful tip: if each entry starts with a consistent prefix (e.g. `## [2026-04-02] ingest | Article Title`), the log becomes parseable with simple unix tools — `grep "^## \[" log.md | tail -5` gives you the last 5 entries. The log gives you a timeline of the wiki's evolution and helps the LLM understand what's been done recently.

**译文**
两个特殊文件帮助 LLM（和你）随着 Wiki 的增长进行导航，各司其职：

**index.md** 是内容导向的。它是 Wiki 的目录——每个页面附有链接、一行摘要，以及可选的元数据（如日期或资料来源数量），按类别组织（实体、概念、来源等）。LLM 在每次摄入时更新它。回答查询时，LLM 先读取索引以找到相关页面，再深入阅读。在中等规模（约 100 篇资料、数百个页面）下，这种方式效果出人意料地好，且避免了基于 Embedding 的 RAG 基础设施需求。

**log.md** 是时序导向的。它是一份只追加的记录，记录发生了什么和发生时间——摄入、查询、Lint。一个实用技巧：如果每条记录以一致的前缀开头（如 `## [2026-04-02] ingest | 文章标题`），日志就可以用简单的 Unix 工具解析——`grep "^## \[" log.md | tail -5` 给出最近 5 条记录。日志提供了 Wiki 演进的时间线，帮助 LLM 理解最近做了什么。

---

## 可选：CLI 工具（Optional: CLI tools）

**原文**
> At some point you may want to build small tools that help the LLM operate on the wiki more efficiently. A search engine over the wiki pages is the most obvious one — at small scale the index file is enough, but as the wiki grows you want proper search. [qmd](https://github.com/tobi/qmd) is a good option: it's a local search engine for markdown files with hybrid BM25/vector search and LLM re-ranking, all on-device. It has both a CLI (so the LLM can shell out to it) and an MCP server (so the LLM can use it as a native tool). You could also build something simpler yourself — the LLM can help you vibe-code a naive search script as the need arises.

**译文**
在某个时间节点，你可能会想构建一些小工具来帮助 LLM 更高效地操作 Wiki。最明显的需求是 Wiki 页面的搜索引擎——小规模时索引文件已够用，但随着 Wiki 增长，你会需要真正的搜索能力。[qmd](https://github.com/tobi/qmd) 是一个不错的选择：它是一个本地 Markdown 文件搜索引擎，支持 BM25/向量混合搜索和 LLM 重排序，全部在设备本地运行。它同时提供 CLI（LLM 可以调用 Shell）和 MCP server（LLM 可将其作为原生工具）两种接口。你也可以自己构建更简单的东西——LLM 可以帮你随用随写一个简易搜索脚本。

---

## 技巧与窍门（Tips and tricks）

**原文**
> - **Obsidian Web Clipper** is a browser extension that converts web articles to markdown. Very useful for quickly getting sources into your raw collection.
> - **Download images locally.** In Obsidian Settings → Files and links, set "Attachment folder path" to a fixed directory (e.g. `raw/assets/`). Then in Settings → Hotkeys, search for "Download" to find "Download attachments for current file" and bind it to a hotkey (e.g. Ctrl+Shift+D). After clipping an article, hit the hotkey and all images get downloaded to local disk. This is optional but useful — it lets the LLM view and reference images directly instead of relying on URLs that may break. Note that LLMs can't natively read markdown with inline images in one pass — the workaround is to have the LLM read the text first, then view some or all of the referenced images separately to gain additional context. It's a bit clunky but works well enough.
> - **Obsidian's graph view** is the best way to see the shape of your wiki — what's connected to what, which pages are hubs, which are orphans.
> - **Marp** is a markdown-based slide deck format. Obsidian has a plugin for it. Useful for generating presentations directly from wiki content.
> - **Dataview** is an Obsidian plugin that runs queries over page frontmatter. If your LLM adds YAML frontmatter to wiki pages (tags, dates, source counts), Dataview can generate dynamic tables and lists.
> - The wiki is just a git repo of markdown files. You get version history, branching, and collaboration for free.

**译文**
- **Obsidian Web Clipper** 是一个浏览器扩展，可将网页文章转为 Markdown，非常适合快速将资料来源收入原始资料集。
- **本地下载图片**。在 Obsidian 设置 → Files and links 中，将"附件文件夹路径"设为固定目录（如 `raw/assets/`）。再在设置 → 快捷键中搜索"Download"，找到"Download attachments for current file"并绑定快捷键（如 Ctrl+Shift+D）。剪辑文章后按快捷键，所有图片下载到本地磁盘。这是可选项，但很实用——它让 LLM 能直接查看和引用图片，而不依赖可能失效的 URL。注意 LLM 无法在一次阅读中原生处理含内联图片的 Markdown，变通方法是先让 LLM 读文本，再单独查看部分或全部图片以获取额外上下文，稍显笨拙但基本够用。
- **Obsidian 的图谱视图**是查看 Wiki 结构的最佳方式——什么连接什么、哪些页面是枢纽、哪些是孤立页面。
- **Marp** 是基于 Markdown 的幻灯片格式，Obsidian 有对应插件，适合直接从 Wiki 内容生成演示文稿。
- **Dataview** 是一个 Obsidian 插件，可对页面 YAML 前置元数据（frontmatter）运行查询。如果 LLM 为 Wiki 页面添加了 YAML 元数据（标签、日期、资料来源数量），Dataview 可生成动态表格和列表。
- Wiki 本质上就是一个 Markdown 文件的 Git 仓库，版本历史、分支、协作全部免费获得。

---

## 为什么有效（Why this works）

**原文**
> The tedious part of maintaining a knowledge base is not the reading or the thinking — it's the bookkeeping. Updating cross-references, keeping summaries current, noting when new data contradicts old claims, maintaining consistency across dozens of pages. Humans abandon wikis because the maintenance burden grows faster than the value. LLMs don't get bored, don't forget to update a cross-reference, and can touch 15 files in one pass. The wiki stays maintained because the cost of maintenance is near zero.
>
> The human's job is to curate sources, direct the analysis, ask good questions, and think about what it all means. The LLM's job is everything else.
>
> The idea is related in spirit to Vannevar Bush's Memex (1945) — a personal, curated knowledge store with associative trails between documents. Bush's vision was closer to this than to what the web became: private, actively curated, with the connections between documents as valuable as the documents themselves. The part he couldn't solve was who does the maintenance. The LLM handles that.

**译文**
维护知识库的繁琐之处不在于阅读或思考，而在于记账式的事务：更新交叉引用、保持摘要的时效性、记录新数据何时推翻旧观点、在数十个页面间维持一致性。人类放弃 Wiki 是因为维护负担的增长速度超过了价值增长速度。LLM 不会厌倦、不会忘记更新交叉引用，还能在一次操作中同时修改 15 个文件。Wiki 之所以能持续维护，是因为维护的成本趋近于零。

人类的工作是策划资料来源、指导分析方向、提出好问题、思考这一切意味着什么。其他一切都是 LLM 的工作。

这个想法在精神上与 Vannevar Bush 的 Memex（1945 年）相近——一个私人的、经过策划的知识存储，文档之间有关联路径。Bush 的愿景比现在的 Web 更接近于此：私密、主动策划，文档之间的连接与文档本身同等重要。他没能解决的问题是：谁来做维护？LLM 解决了这个问题。

---

## 说明（Note）

**原文**
> This document is intentionally abstract. It describes the idea, not a specific implementation. The exact directory structure, the schema conventions, the page formats, the tooling — all of that will depend on your domain, your preferences, and your LLM of choice. Everything mentioned above is optional and modular — pick what's useful, ignore what isn't. For example: your sources might be text-only, so you don't need image handling at all. Your wiki might be small enough that the index file is all you need, no search engine required. You might not care about slide decks and just want markdown pages. You might want a completely different set of output formats. The right way to use this is to share it with your LLM agent and work together to instantiate a version that fits your needs. The document's only job is to communicate the pattern. Your LLM can figure out the rest.

**译文**
本文档有意保持抽象。它描述的是思路，而非具体实现。确切的目录结构、Schema 约定、页面格式、工具选型——所有这些都取决于你的领域、你的偏好和你选择的 LLM。上面提到的一切都是可选且模块化的——取其有用的，忽略无关的。例如：你的资料可能全是纯文本，那就完全不需要图片处理；你的 Wiki 可能规模很小，只需要索引文件，不需要搜索引擎；你可能不在乎幻灯片，只想要 Markdown 页面；你可能想要完全不同的输出格式。正确的使用方式是：把这份文档发给你的 LLM Agent，共同实例化一个适合你需求的版本。本文档唯一的工作是传达这个模式，其余的，你的 LLM 自会搞定。
