---
root: "我想深入理解 Transformer 的 Attention 机制是怎么工作的"
explored: 2026-05-12
---

# Transformer Attention 机制：从原理到架构设计

## 加权信息聚合：Attention 的本质

Transformer 的 Attention 机制解决的核心问题是：在处理一个序列时，如何让每个位置的表示动态地「关注」序列中其他位置的信息，并根据语义相关性调整聚合权重。这一能力是 Transformer 超越 RNN 的关键所在。

整个机制围绕三组向量展开：Query（Q）、Key（K）、Value（V）。对每个 token 的向量表示，通过三组独立的线性变换分别投影到 Q、K、V 空间。计算时，用当前 token 的 Q 与所有 token 的 K 做点积，得到注意力分数，再除以 sqrt(d_k) 缩放（防止高维空间中点积值过大导致 softmax 梯度消失），经 softmax 归一化后作为权重，对 V 做加权求和，得到该 token 融合了全局上下文信息的新表示。

公式为：Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) * V

一个直观的类比是「图书馆检索」：Q 是你的检索词，K 是每本书的索引标签，V 是书的正文。检索的过程就是用 Q 匹配所有 K，再按匹配分数从 V 中按比例提取信息。Q、K、V 的分离设计使「应该关注什么」和「信息本身是什么」解耦，赋予了模型更高的表达灵活性。相比之下，RNN 中远距离依赖需要逐步传递且会衰减，而 Attention 让任意两个位置直接建立连接，路径长度恒为 O(1)，天然适合捕捉长距离依赖。

## 多头并行：为什么不只用一个头？

在单头 Attention 的基础上，原始 Transformer 论文引入了 Multi-Head Attention。其做法是将 Q、K、V 分别通过 h 组独立的线性投影，映射到 h 个低维子空间（每个维度为 d_model/h），在每个子空间中独立做 Attention，最后将 h 个输出拼接并经线性变换融合。

一个自然的疑问是：为何不直接扩大单头的维度？答案在于 softmax 的竞争性限制。softmax 是归一化操作，所有位置权重之和为 1，这意味着一个头在同一时刻只能维持一种注意力分布。如果 token 需要同时关注主语（句首）和局部上下文（前一个词），两种需求会在单头中相互竞争。多头机制将不同类型的关系分配给不同的头，各自在独立子空间中竞争，互不干扰。

本质上，扩大单头维度提升的是「精度」（区分更细微的相似度），增加头数提升的是「多样性」（同时维持多种关注模式）——二者是正交的能力维度。

## Head 的分工：涌现的专业化

许多人直觉上以为不同 head 是被「设计」成各自关注不同语言现象的——就像流水线上分工明确的工人。但事实恰恰相反：网络中没有任何显式约束要求不同 head 必须关注不同的东西。每个 head 只是拥有一组独立的参数矩阵，将输入投影到一个低维子空间并独立计算注意力。**分工是涌现的，不是设计进去的。** 梯度下降自然地发现：如果多个 head 都只学同一种关注模式，参数就是浪费；于是网络在训练中倾向于让不同 head 走向专业化。

多项实证研究系统地验证了这种功能分化：

**Voita et al. (2019)** 对 Transformer 翻译模型（英→俄）每个 head 进行了功能分析和剪枝实验，发现了三类稳定出现的专业化 head：
- **位置敏感 head**：几乎每个句子中都只关注固定相对位置的 token（如「前一个词」），行为高度一致。
- **句法 head**：捕捉语法依存关系，动词倾向于关注主语，形容词倾向于关注它修饰的名词。
- **稀有词/特殊 token head**：注意力高度集中在低频词（信息密度更高）或 `[SEP]` 等特殊 token 上。

更重要的是，大多数 head 是冗余的——剪掉这些 head 之后翻译质量几乎不受影响，真正在做有效工作的 head 只有少数几个。

**Vig & Belinkov (2019)** 在 GPT-2 上发现了明显的层位规律：浅层 head 关注局部邻近词（局部语法线索），中间层 head 捕捉更长距离的依存关系，深层 head 则更多体现句子级的语义关联，呈现出从「句法」到「语义」的渐进分层。

**Clark et al. (2019)** 在 BERT 上做了细粒度句法映射，发现第 8-9 层的某些 head 对「直接宾语」关系的捕捉准确率超过 86%，第 2 层的 head 则对指代关系有专门化表现。他们还观察到一个有趣现象：许多 head 的注意力大量流向 `[CLS]` 和 `[SEP]`——这被解释为「注意力蓄水池」效应，当 head 没有有用的语言学模式可以聚焦时，注意力权重就流向这些特殊 token 以维持归一化约束。

综合现有研究，head 分工在层位上有大致规律：

| 分工类型 | 典型行为 | 常见层位 |
|---------|---------|---------|
| 位置敏感 | 总是关注前 1/后 1 个 token | 浅层 |
| 局部依存 | 关注句法直接相邻关系（修饰、并列） | 浅-中层 |
| 长距离依存 | 主谓一致、跨从句关系 | 中层 |
| 指代解析 | 代词 → 先行词 | 中层 |
| 语义/全局 | 软性主题相关性，非结构化 | 深层 |
| 标记偏移 | 关注 [SEP]/[CLS] 作为注意力蓄水池 | 各层均有 |

值得注意的是，head 的行为特征不只依赖「是哪个 head」，还高度依赖「在第几层」——层位和 head 编号共同决定一个 head 的功能特征。此外，这些研究大多基于注意力权重的可视化和统计分析，而 Jain & Wallace (2019) 等工作指出：高注意力权重并不一定等于「对预测结果真正重要」——注意力的可解释性至今仍有争议。

## 自注意力与交叉注意力：谁来提问，谁来被检索？

理解了 Attention 的基本结构后，一个关键的架构问题浮现：Q、K、V 是否必须来自同一个序列？这正是 Self-Attention 与 Cross-Attention 的本质区分。

Self-Attention（自注意力）中，Q、K、V 全部来自同一序列——序列在「关注自身」，每个 token 同时扮演提问者和被检索者。Cross-Attention（交叉注意力）中，Q 来自一个序列，K 和 V 来自另一个序列，两个不同序列之间建立信息桥梁。

在标准的 Encoder-Decoder Transformer 中，这两种 Attention 各司其职。Encoder 中使用无掩码的 Self-Attention，让输入序列中每个 token 充分融合全局上下文；Decoder 中首先使用带 Causal Mask 的 Self-Attention，确保生成第 t 个 token 时看不到未来位置（自回归生成的基础）；随后 Decoder 再使用 Cross-Attention，以自身状态作为 Q、以 Encoder 输出作为 K 和 V，实现对输入序列的「查询」，这正是翻译模型学习源语言与目标语言词对齐的核心机制。

而在仅有 Decoder 的模型（如 GPT）中，Cross-Attention 不复存在，所有层都是 Masked Self-Attention，模型在单一序列内部做因果推理。Cross-Attention 的价值不限于文本翻译——在多模态模型中，它可以让文本 token 查询图像 patch 的特征，实现跨模态融合，是 Flamingo 等模型的核心操作。

---

## 附录：探索路径

### Q1 — Attention 机制的基本原理

**问：** 我想深入理解 Transformer 的 Attention 机制是怎么工作的，帮我探索一下。

**答：**

Transformer 的 Attention 机制，本质上是一种「加权信息聚合」的方式。它解决的核心问题是：在处理一个序列时，如何让每个位置的表示都能「关注」到序列中其他位置的信息，并根据相关性动态调整权重。

整个 Attention 机制围绕三个矩阵展开：Query（Q）、Key（K）、Value（V）。对每个 token 的向量表示，通过三组独立的线性变换（权重矩阵 W^Q、W^K、W^V）分别得到 Q、K、V 向量。用 Q 与所有 K 做点积得到注意力分数，除以 sqrt(d_k) 缩放后经 softmax 归一化，再对 V 做加权求和，得到融合上下文的新表示。

公式：Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) * V

「图书馆检索」类比：Q 是检索词，K 是索引标签，V 是正文内容。相比 RNN，Attention 允许任意两个位置直接建立 O(1) 路径的连接，Q、K、V 分离设计使「应该关注什么」和「信息是什么」解耦。

> 💡 **未探索分支：** Multi-Head Attention 为何要用多个头并行而不是单头放大维度？

### Q2 — Multi-Head Attention 为什么要用多个头？

**问：** Multi-Head Attention 是什么？为什么要多个头，直接用更大的单头不行吗？

**答：**

Multi-Head Attention 将 Q、K、V 分别通过 h 组线性投影映射到 h 个低维子空间（维度 d_model/h），在每个子空间独立做 Attention，拼接后线性融合。

公式：MultiHead(Q,K,V) = Concat(head_1, ..., head_h) * W^O

放大单头维度不等价于多头，核心原因是 softmax 的竞争性：一个头只能维持一种注意力分布，无法同时满足对不同类型关系（主谓、指代、局部搭配）的关注需求。多头将这些需求分配给不同的头独立处理。扩大维度提升「精度」，增加头数提升「多样性」，二者正交。Voita et al. (2019) 的实验证实了不同头确实产生功能分化。

> 💡 **未探索分支：** 自注意力（Self-Attention）与交叉注意力（Cross-Attention）的区别，以及它们在 Encoder-Decoder 结构中的不同角色。

### Q3 — Self-Attention 与 Cross-Attention 的区别

**问：** 刚才提到了 Self-Attention 和 Cross-Attention，它们有什么本质区别？在 Transformer 里各自在哪里用？

**答：**

本质区别在于 Q、K、V 的来源：Self-Attention 三者来自同一序列；Cross-Attention 的 Q 来自一个序列，K 和 V 来自另一个序列。

在标准 Encoder-Decoder Transformer 中：Encoder 使用无掩码的 Self-Attention（全局上下文融合）；Decoder 先使用 Masked Self-Attention（Causal Mask 保证自回归生成不看未来），再使用 Cross-Attention（Decoder 状态作 Q，Encoder 输出作 K/V，实现源-目标语言对齐）。

仅有 Decoder 的模型（如 GPT）中没有 Cross-Attention，全部为 Masked Self-Attention。Cross-Attention 在多模态模型中同样关键，可让文本 token 查询图像特征，实现跨模态融合（如 Flamingo）。

### Q4 — Multi-Head Attention 的 Head 分工与实证研究

**问：** Multi-head Attention 中每个 head 是怎么分工的？有没有研究表明不同 head 关注不同语言现象？

**答：**

在 Multi-Head Attention 中，每个 head 拥有自己独立的一组参数矩阵（W^Q_i、W^K_i、W^V_i），将输入投影到一个维度更低的子空间（d_head = d_model / h）。每个 head 在这个子空间里独立计算注意力，最终 h 个 head 的输出被拼接后再经 W^O 线性融合。

关键在于：这些子空间是各自独立学习的——没有任何显式约束迫使不同 head 关注不同的东西。**分工是涌现的，不是设计进去的。** 梯度下降自动发现：如果多个 head 都只学一种关注模式，参数就是浪费；于是网络在训练中倾向于让不同 head 专业化，形成功能分化。

**1. Voita et al. (2019) — "Analyzing Multi-Head Self-Attention: Specialized Heads Do the Heavy Lifting, the Rest Can Be Pruned"**

对 Transformer 翻译模型（英→俄）的每个 head 进行了功能分析和剪枝实验，发现：
- **位置敏感 head（Positional heads）**：关注的总是固定相对位置的 token，例如「前一个词」或「后一个词」，在几乎每个句子中都稳定表现。
- **句法 head（Syntactic heads）**：关注语法依存关系，如动词注意其主语、形容词注意它修饰的名词。
- **稀有词/终止符 head（Rare words heads）**：有些 head 习惯性地将注意力集中到句子中频率低的词（信息量更大的词）或 [SEP] 这样的特殊 token 上。

大部分 head 是「冗余的」——只有少数几个 head 对最终翻译质量真正关键，大多数 head 被剪掉之后性能几乎不下降。

**2. Vig & Belinkov (2019) — "Analyzing the Structure of Attention in a Transformer Language Model"**

使用 GPT-2，系统分析了不同 head 和不同层的注意力模式，发现：
- 浅层 head 更多关注「局部窗口」内的邻近词（局部语法结构）
- 中间层 head 开始捕捉更长距离的依存关系
- 深层 head 则与句子级别的语义更相关

**3. Clark et al. (2019) — "What Does BERT Look At?"**

在 BERT 上做了细粒度的句法分析，发现：
- Head 在第 8-9 层对直接宾语（direct object）关系的捕捉准确率超过 86%（远超随机基线）
- Head 在第 2 层对 coreference（指代）有专门化表现
- 许多 head 表现出对 [CLS]、[SEP] 的强偏好（垃圾桶效应——注意力需要归一化，没有用处的权重会流入这些特殊 token）

**典型的分工模式归纳：**

| 分工类型 | 典型行为 | 常见层位 |
|---------|---------|---------|
| 位置敏感 | 总是关注前 1/后 1 个 token | 浅层 |
| 局部依存 | 关注句法直接相邻关系（修饰、并列） | 浅-中层 |
| 长距离依存 | 主谓一致、跨从句关系 | 中层 |
| 指代解析 | 代词 → 先行词 | 中层 |
| 语义/全局 | 软性主题相关性，非结构化 | 深层 |
| 标记偏移 | 关注 [SEP]/[CLS] 作为注意力"蓄水池" | 各层均有 |

**重要注意事项：**
1. **因果性 vs 相关性**：上述研究大多是相关性观察（attention 权重高的位置 ≈ 该 head 关注什么），但 Jain & Wallace (2019) 等工作指出：高注意力权重并不一定等于「对预测结果真正重要」。
2. **层位重要性**：head 的分工高度依赖所在层位——同样是位置 3 的 head，在第 1 层和第 10 层的行为可能完全不同。
3. **模型依赖性**：这些发现在不同模型（BERT、GPT、翻译模型）之间不完全通用。

> 💡 **未探索分支：** 注意力权重是否真的等于「重要性」？Attention 的可解释性争议（Jain & Wallace vs Wiegreffe & Pinter）是一个非常有意思的方向。
