---
root: "我想深入理解 Transformer 的 Attention 机制是怎么工作的"
slug: transformer-attention
started: 2026-05-12T10:00:00
status: done
---

## Q1 Attention 机制的基本原理

**问：** 我想深入理解 Transformer 的 Attention 机制是怎么工作的，帮我探索一下。

**答：**

Transformer 的 Attention 机制，本质上是一种「加权信息聚合」的方式。它解决的核心问题是：在处理一个序列时，如何让每个位置的表示都能「关注」到序列中其他位置的信息，并根据相关性动态调整权重。

**Scaled Dot-Product Attention 的计算过程**

整个 Attention 机制围绕三个矩阵展开：Query（Q）、Key（K）、Value（V）。

1. **输入映射：** 对每个 token 的向量表示，通过三组独立的线性变换（权重矩阵 W^Q、W^K、W^V）分别得到 Q、K、V 向量。
2. **相似度计算：** 用 Q 与所有 K 做点积，得到每对 token 之间的「注意力分数」。点积越大，代表这两个 token 的语义越相关。
3. **缩放与归一化：** 将点积除以 sqrt(d_k)（K 向量维度的平方根），防止高维空间中点积值过大导致 softmax 梯度消失。然后经过 softmax 转换为概率分布（权重之和为 1）。
4. **加权求和：** 用这组权重对 V 向量做加权求和，得到该 token 新的表示。

公式：Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) * V

**直观理解：Q、K、V 分别代表什么？**

一个常用的类比是「图书馆检索」：
- Q（Query）：你的检索词，代表当前 token 想要寻找什么信息
- K（Key）：每本书的索引标签，代表每个 token 提供了什么特征
- V（Value）：书的正文内容，代表每个 token 实际携带的信息

当你用 Q 去匹配所有 K 后，得到一组「匹配度分数」，再用这组分数从 V 中按比例提取信息。

**为什么这种机制有效？**

在 RNN 等序列模型中，远距离依赖关系需要通过多个时间步传递，信息会逐渐衰减。Attention 机制允许序列中任意两个位置直接建立连接，路径长度始终为 O(1)，因此能更好地捕捉长距离依赖。此外，Q、K、V 的分离设计让「什么样的信息应该被关注」和「信息本身是什么」解耦，增强了模型的表达灵活性。

> 💡 **未探索分支：** Multi-Head Attention 为何要用多个头并行而不是单头放大维度？

## Q2 Multi-Head Attention 为什么要用多个头？

**问：** Multi-Head Attention 是什么？为什么要多个头，直接用更大的单头不行吗？

**答：**

**Multi-Head Attention 的结构**

Multi-Head Attention 是把 Scaled Dot-Product Attention 并行执行多次的机制。具体做法是：

1. 将 Q、K、V 分别通过 h 组不同的线性投影，映射到 h 个较低维度的子空间（每个子空间维度为 d_model/h）
2. 在每个子空间中独立做 Attention 计算，得到 h 个输出
3. 将 h 个输出拼接，再经过一个线性变换融合

公式：MultiHead(Q,K,V) = Concat(head_1, ..., head_h) * W^O，其中 head_i = Attention(QW_i^Q, KW_i^K, VW_i^V)

**为什么不直接放大单头？**

这是一个关键的设计动机问题。单纯扩大单头的维度并不等价于多头，原因有以下几点：

1. **捕捉不同类型的关系：** 不同的注意力头可以在不同的子空间中学习不同类型的依赖关系。例如，一个头可能学到句法依存关系（主谓关系），另一个头可能学到指代关系（「它」指向前文的某个名词），第三个头可能学到局部搭配模式。这些不同「视角」的信息是同时有用的。单头即使维度扩大，经过 softmax 后仍只产生一个注意力分布，无法同时对同一个 token 赋予多种不同的关注模式。

2. **softmax 的竞争性限制：** softmax 是归一化操作，所有位置的权重之和为 1，这意味着注意力是「竞争性」的——关注了 A 就必然分走了对 B 的关注。单头中，如果 token 需要同时关注句子的开头（主语）和当前位置的前一个词（局部上下文），这两种需求会在 softmax 中相互竞争。多头则将这两种需求分配给不同的头，各自独立竞争，不相互干扰。

3. **表达能力的多样性 vs. 规模：** 增大单头的 d_k 提升的是「精度」（能区分更细微的相似度），而增加头数提升的是「多样性」（能同时维持多种不同的关注模式）。这两者是正交的能力维度。

**实验证据**

研究者（如 Voita et al., 2019）对 Transformer 中各个注意力头的功能做了分析，发现不同头确实专注于不同的语言学功能：有些头负责局部依存关系，有些头负责长距离指代解析，还有些头对位置信息敏感。这在一定程度上验证了多头设计确实带来了功能分化。

> 💡 **未探索分支：** 自注意力（Self-Attention）与交叉注意力（Cross-Attention）的区别，以及它们在 Encoder-Decoder 结构中的不同角色。

## Q3 Self-Attention 与 Cross-Attention 的区别

**问：** 刚才提到了 Self-Attention 和 Cross-Attention，它们有什么本质区别？在 Transformer 里各自在哪里用？

**答：**

**本质区别：Q、K、V 的来源**

Self-Attention 和 Cross-Attention 的根本区别在于 Q、K、V 三个矩阵的来源序列是否相同：

- **Self-Attention（自注意力）：** Q、K、V 全部来自同一个序列。序列中的每个 token 同时扮演「提问者」和「被检索者」的角色。序列在「关注自身」。
- **Cross-Attention（交叉注意力）：** Q 来自一个序列（通常是 Decoder 的当前状态），而 K 和 V 来自另一个序列（通常是 Encoder 的输出）。两个不同序列之间建立注意力连接。

**在标准 Transformer（Encoder-Decoder 结构）中的分布**

标准 Transformer（如最初用于机器翻译的架构）包含三种 Attention：

1. **Encoder 中的 Self-Attention：** 输入序列（如源语言句子）中的每个 token 关注同一句子的所有其他 token。目的是建立对输入序列的全面理解，让每个词的表示都融入了上下文信息。这里没有掩码（Mask），所有位置可以互相看到。

2. **Decoder 中的 Masked Self-Attention：** 目标序列（如目标语言句子）中的每个 token 关注自身序列，但只能看到当前位置及之前的 token（使用 Causal Mask 遮蔽未来位置）。这是自回归生成的关键：模型在生成第 t 个词时，不能「偷看」第 t+1 个词。

3. **Decoder 中的 Cross-Attention：** Decoder 当前层的输出作为 Q，Encoder 最终层的输出作为 K 和 V。这一层让 Decoder 能够「查询」Encoder 对输入的理解，实现了源语言和目标语言之间的对齐。这是翻译模型能够学到「哪个源语言词对应哪个目标语言词」的核心机制。

**在仅用 Decoder 的模型（如 GPT）中**

GPT 等自回归语言模型只有 Decoder，不存在 Cross-Attention。所有层都是 Masked Self-Attention。模型在一个序列内部做因果推理，每个 token 只能关注它自己和之前的 token。

**Cross-Attention 的更广泛应用**

Cross-Attention 的思路不限于文本：在多模态模型（如图文对齐）中，Cross-Attention 可以让文本 token（Q）去关注图像 patch 的特征（K、V），实现跨模态信息融合。这是 CLIP、Flamingo 等模型的核心操作之一。
