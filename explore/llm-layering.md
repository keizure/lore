---
root: "如果让你来做整个大语言模型的分层，你会怎么分？"
explored: 2026-05-12
---

# 大语言模型的推理流程：从提示词到输出的完整分层解析

理解一个大语言模型（LLM，Large Language Model）最直观的方式，是跟着一次推理的数据流走一遍——从你输入的一段文字，到模型吐出第一个字，中间究竟经过了什么。整个过程可以分为五个站点，核心计算集中在第三站，它本身又是一个反复执行的循环结构。

## 第 1 站：文本切分（Tokenization）

模型读不懂文字，只认识整数。第一步是把文本转换成一串 token id。

**Tokenizer**（分词器）使用 **BPE**（Byte Pair Encoding，字节对编码）算法，把文本切成子词单元，每个子词在**词表**（vocab，通常 32k～128k 个条目）里对应一个整数 id。比如「世界」可能被切成一个 token，也可能是两个，取决于训练时的词表构建方式。

除了普通文字，还有一类**特殊 token**，如 `<s>`（序列开始）、`</s>`（序列结束）、`<|im_start|>`（对话格式标记），它们是模型理解「这是一段对话」「这是系统提示」的信号。

经过这一站，你的提示词变成了一串整数序列，长度记为 **seq_len**（序列长度）。

## 第 2 站：向量化（Embedding）

整数本身没有几何意义，无法做数学运算。第二步是把每个 token id 映射成一个高维浮点向量。

模型维护一个 **Embedding 矩阵**，形状为 `[vocab_size, d_model]`。每个 token id 直接查表，取出对应的一行，得到一个 **d_model** 维的向量。**d_model** 是模型的「宽度」，Llama-3 8B 中为 4096。

整个输入序列此时变成形状为 `[seq_len, d_model]` 的矩阵，这是**残差流**（Residual Stream）的起点——它将贯穿接下来所有层的计算，被每一层读写，最终承载模型对输入的全部理解。

值得注意：位置编码（**RoPE**，Rotary Position Embedding，旋转位置编码）并不在这里注入，而是在每一层 Attention 计算 Q/K 时动态施加，这是现代 LLM 区别于早期 Transformer 的重要设计。

## 第 3 站：Transformer Block 循环（核心计算）

这是模型真正「思考」的地方，也是参数量最集中的部分。残差流依次经过 **L** 层 Transformer Block（Llama-3 8B 为 32 层），每一层结构相同，顺序执行六步：

```
① RMSNorm（归一化）
② Multi-Head Attention（多头注意力，token 间信息交换）
③ 残差连接：x = x + attn_out
④ RMSNorm
⑤ FFN（前馈网络，每个 token 独立变换）
⑥ 残差连接：x = x + ffn_out
```

整个过程中，矩阵形状始终保持 `[seq_len, d_model]` 不变——残差流只是内容在变，维度从不改变。

### 残差连接：让深度网络可以被训练

**残差连接**（Residual Connection）的公式极其简单：`output = F(x) + x`。将子层（Attention 或 FFN）的输入 x 通过一条直通路（shortcut connection）绕过子层，直接加到输出上。

这个设计解决了深层神经网络最核心的训练困难：**梯度消失**。反向传播时，梯度沿直通路传回的分量包含 `+1` 项，无论网络多深，梯度都有一条保持量级的通道。此外，子层不必学习「从零到输出」的完整映射，只需学习「在当前基础上做多少调整」——学习残差比学习全量变换容易得多。

Transformer Block 里有两处残差，分别在 Attention 和 FFN 之后。贯穿所有层的那条持续被读写的向量，就是**残差流**。

**RMSNorm**（Root Mean Square Norm，均方根归一化）在每次送入子层前执行，将向量的均方根归一，防止数值尺度失控。它是更早期 **LayerNorm**（层归一化）的简化版，去掉了均值归零的步骤，计算更高效，现代主流模型普遍采用。

### FFN：每个 Token 的「独立思考」

**FFN**（Feed-Forward Network，前馈网络）是每个 Transformer Block 里的两层 **MLP**（Multi-Layer Perceptron，多层感知机）。标准结构：

```
Linear(d_model → d_ff) → 激活函数 → Linear(d_ff → d_model)
```

**d_ff** 是中间层维度，通常是 d_model 的 2.7～4 倍（如 4096 → 11008）。

FFN 最关键的特性是：**每个 token 完全独立处理，互不影响**。这与 Attention 形成鲜明对比——Attention 负责 token 之间的信息交换，FFN 则负责每个 token 自身的深度加工。直觉上，Attention 像「开会」，FFN 像「独自思考」。

**FFN 作为知识记忆库**：2021 年一篇重要论文（《Transformer Feed-Forward Layers Are Key-Value Memories》）提出，FFN 本质上是一个巨大的键值记忆库：第一层矩阵的每一行是「键」（匹配某种输入模式），第二层矩阵对应的列是「值」（输出内容），激活函数决定哪些键被激活。事实性知识（如「法国首都是巴黎」）被编码在特定神经元的权重里，早期层存语法/词法信息，深层存语义/事实信息。这一假说催生了**知识编辑**（Knowledge Editing）技术——通过定位并修改特定 FFN 权重，可以改变模型「记住」的事实。

现代 LLM（Llama、Mistral 等）使用 **SwiGLU** 变体替代标准 FFN：在两层 MLP 基础上增加一条门控路径（W1 经 SiLU 激活后与 W3 逐元素相乘），由三个权重矩阵（W1、W2、W3）构成，门控机制让模型更精细地控制信息流动，效果优于普通 MLP。

**SiLU**（Sigmoid Linear Unit）是现代 LLM 常用的激活函数，`f(x) = x · sigmoid(x)`，比早期的 **ReLU** 平滑，更有利于梯度传播。

## 第 4 站：输出头（LM Head）

经过 L 层 Block，残差流携带着模型对整个上下文的全部理解。但生成时，模型只需预测「下一个 token 是什么」，因此只取**最后一个 token 的向量**（前面所有 token 的信息已通过 Attention 汇聚进来）。

这个 d_model 维向量经过一个 **Linear** 投影，映射到 `[vocab_size]` 维的**logit**（未归一化得分），再经 **Softmax** 归一化，得到词表上每个 token 的概率分布。

值得一提的是，LM Head 的 Linear 权重在很多模型中与 Embedding 矩阵**共享**（称为 weight tying）——输入端把 id 变成向量，输出端把向量变回 id，两个方向用同一套参数，减少了参数量，也让语义空间更一致。

## 第 5 站：采样（Decoding）

有了概率分布，还需要从中取出一个 token，这一步叫**解码**。

**Temperature**（温度）对 logit 做缩放：Temperature < 1 让分布更尖锐（模型更「确定」），Temperature > 1 让分布更平坦（输出更随机）。Temperature = 0 在数学上会让 softmax 退化为 argmax（直接取最高概率的 token），但实现上通常直接用 argmax 代替。

**Top-k** 只保留概率最高的 k 个 token 候选，其余概率清零再归一化。**Top-p**（也叫 nucleus sampling）保留累积概率达到 p 的最小集合，比 Top-k 更自适应。两者通常结合使用。

采样得到下一个 token id 后，将其追加到输入序列，重新进入第 3 站。这就是**自回归**（autoregressive）生成：每次只生成一个 token，把它加回输入，再生成下一个，循环往复，直到生成 EOS（End of Sequence，序列结束）token 或达到最大长度。

---

## 两个视角的分层对照

以上是「数据流」视角的分层。如果从「功能抽象」视角看，同样的结构可以映射为：

| 功能层 | 对应站点 | 职责 |
|--------|---------|------|
| 符号层 | 第 1-2 站 | 文字 ↔ 向量的转换接口 |
| 计算层 | 第 3 站内部 | Attention、FFN、LayerNorm 的具体运算 |
| 表示层 | 第 3 站整体 | 残差流——信息在层间的流动与演变 |
| 语义层 | 第 3 站深层 | 知识、推理、涌现能力的载体 |
| 输出层 | 第 4-5 站 | 从表示到 token 的转换与采样 |

两种视角互补：数据流视角帮助理解「模型怎么跑」，功能抽象视角帮助理解「模型在做什么」。

---

## 附录：探索路径

### Q1 — LLM 的完整分层框架

**问：** 如果让你来做整个大语言模型的分层，你会怎么分？比如输出层、token 层、神经网络层，可以划分多级，我想在关键的位置做一些深一些的探究

**答：** 将 LLM 分为 5 个层次，从抽象到具体：L5 行为层（对齐、人格、指令跟随）、L4 语义层（知识、推理、电路）、L3 表示层（残差流、隐藏状态）、L2 计算层（Attention、FFN、LayerNorm）、L1 符号层（Token、词表、Embedding）、L0 输出层（logit、softmax、采样策略）。最值得深挖的节点：残差流、Attention Head 功能分工、FFN 作为记忆、Tokenizer 的影响、Logit Lens、Superposition。

### Q2 — 按推理流程重新分层（数据流视角）

**问：** 我希望是更接近我的使用流程的，比如提示词进入模型到一层层处理到最后输出这样分层

**答：** 按数据流动顺序划分为 5 个站点：第 1 站 Tokenization（文本→token id）、第 2 站 Embedding（token id→残差流向量矩阵）、第 3 站 N×Transformer Block（LayerNorm→Attention→残差→LayerNorm→FFN→残差，循环 L 次）、第 4 站 LM Head（最后一个 token 向量→logit→概率分布）、第 5 站 采样/Decoding（Temperature/Top-k/Top-p→下一个 token，自回归循环）。

### Q3 — 残差连接与 FFN 详解

**问：** 具体讲讲残差连接和 FFN 吧，注意必要的名词、缩写解释

**答：** 残差连接公式 `output = F(x) + x`，直通路保证梯度传播的 `+1` 分量，解决深层网络梯度消失问题，子层只学残差（调整量）而非完整映射。RMSNorm 是 LayerNorm 的简化版，去掉均值归零步骤。FFN 是两层 MLP，d_ff 约为 d_model 的 2.7~4 倍，每个 token 完全独立处理——与 Attention 的 token 间信息交换互补。FFN 作为键值记忆库：第一层行是键，第二层列是值，激活函数做匹配，早期层存语法，深层存语义事实，可通过权重编辑修改模型记忆。SwiGLU 是现代 FFN 变体，增加门控路径（W1·SiLU + W3 逐元素相乘 + W2 降维），效果更优。
