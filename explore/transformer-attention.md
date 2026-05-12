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

本质上，扩大单头维度提升的是「精度」（区分更细微的相似度），增加头数提升的是「多样性」（同时维持多种关注模式）——二者是正交的能力维度。实验研究（Voita et al., 2019）也证实了这种功能分化：不同头确实专注于不同的语言学功能，如局部依存关系、长距离指代解析、位置敏感性等。

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
