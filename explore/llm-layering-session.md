---
root: "如果让你来做整个大语言模型的分层，你会怎么分？比如输出层、token 层、神经网络层，可以划分多级，我想在关键的位置做一些深一些的探究"
slug: llm-layering
started: 2026-05-12T14:00:00
status: done
---

## Q1 LLM 的完整分层框架

**问：** 如果让你来做整个大语言模型的分层，你会怎么分？比如输出层、token 层、神经网络层，可以划分多级，我想在关键的位置做一些深一些的探究

**答：**

将 LLM 分为 5 个层次，从抽象到具体：

**L5 行为层 (Behavioral Layer)**：对齐、人格、指令跟随——RLHF/DPO 塑造，safety 机制在特定层的特定方向上可被找到。

**L4 语义层 (Semantic Layer)**：知识、推理、电路——事实性知识主要存在 FFN 中；Attention Head 组合执行特定算法（归纳头、名称回指头等）；线性探针能在隐藏状态中找到概念方向。

**L3 表示层 (Representation Layer)**：残差流、隐藏状态——贯穿全网络的主干道，每个 token 在每层有一个 d_model 维向量；Superposition 假说认为模型把远多于 d_model 数量的特征叠加编码进这条流中。

**L2 计算层 (Computation Layer)**：Attention、FFN、LayerNorm——Multi-Head Attention 做信息聚合，FFN 做特征变换（被认为是记忆存储器），残差连接让信号绕过计算层传递。

**L1 符号层 (Symbol Layer)**：Token、词表、Embedding——Tokenizer（BPE/WordPiece）把文字切成 token，Embedding 矩阵映射到高维向量，位置编码（RoPE/ALiBi）注入位置信息。

**L0 输出层 (Output Layer)**：logit、softmax、采样策略——线性投影到词表大小，归一化得到概率分布，Temperature/Top-k/Top-p 决定采样方式。

最值得深挖的关键节点：残差流（L3）、Attention Head 功能分工（L2/L4）、FFN 作为记忆（L2/L4）、Tokenizer 的影响（L1）、Logit Lens（L0/L3）、Superposition（L3）。

💡 **未探索分支：** Logit Lens 技术——如何用中间层直接预测输出来理解信息在层间的演变

## Q2 按推理流程重新分层（数据流视角）

**问：** 我希望是更接近我的使用流程的，比如提示词进入模型到一层层处理到最后输出这样分层

**答：**

按数据流动顺序，一次推理经过 5 个站点：

**第 1 站 Tokenization**：文本 → token id 序列。Tokenizer 用 BPE 规则切分，每个子词映射到词表中的一个整数 id，加上特殊 token（如 `<s>`、`<|im_start|>`）。

**第 2 站 Embedding**：token id 序列 → [seq_len, d_model] 的浮点矩阵。每个 id 查 Embedding 矩阵得到一个 d_model 维向量，这是残差流的起点。（RoPE 位置编码不在这里加，而是在每层 Attention 的 Q/K 计算时注入。）

**第 3 站 N × Transformer Block（核心循环）**：残差流依次经过 L 层（Llama-3 8B 为 32 层），每层内部：① LayerNorm → ② Multi-Head Attention（token 间信息交换）→ ③ 残差连接 → ④ LayerNorm → ⑤ FFN（每个 token 独立变换）→ ⑥ 残差连接。shape 始终是 [seq_len, d_model]。

**第 4 站 LM Head**：只取最后一个 token 的向量 → Linear 投影到 [vocab_size]（logit）→ Softmax → 概率分布。前面所有 token 的信息已经通过 Attention 汇聚进这个向量。

**第 5 站 采样 / Decoding**：Temperature 缩放 → Top-k/Top-p 截断 → 从分布中采样得到下一个 token → 追加到序列，重新进入第 3 站（自回归）。

关键特征：每次只生成一个 token；残差流贯穿第 3 站被每层读写；整个过程循环直到 EOS token 或达到最大长度。

💡 **未探索分支：** KV Cache 机制——自回归时如何避免对历史 token 重复计算 Attention

## Q3 残差连接与 FFN 详解

**问：** 具体讲讲残差连接和 FFN 吧，注意必要的名词、缩写解释

**答：**

**残差连接（Residual Connection）**：将子层的输入 x 直接加到该子层输出上：`output = F(x) + x`。直通路（shortcut/skip connection）让梯度反向传播时多一条 `+1` 的通道，解决深层网络的梯度消失问题。子层只需学习「调整量」（残差）而非完整映射，训练更容易。Transformer Block 内有两处残差：Attention 之后一次，FFN 之后一次。残差流（Residual Stream）即贯穿所有层、被每层读写的 x 向量。

**LayerNorm（层归一化）**：送入 Attention 或 FFN 前，将向量均值归零、方差归一，防止数值尺度失控。现代模型（Llama 等）使用 RMSNorm（Root Mean Square Norm，均方根归一化），去掉均值归零步骤，更简单高效。

**FFN（Feed-Forward Network，前馈网络）**：每个 Transformer Block 中的两层 MLP（多层感知机）。结构：Linear(d_model → d_ff) → 激活函数(SiLU/GeLU) → Linear(d_ff → d_model)。d_ff 通常是 d_model 的 2.7~4 倍（如 4096 → 11008）。关键特性：每个 token 完全独立处理，不跨 token——与 Attention 负责 token 间信息交换的职责互补。

**FFN 作为键值记忆库**：2021 年论文提出 FFN 本质是巨大的键值记忆库：第一层矩阵的每一行是「键」（代表输入模式），第二层矩阵的每一列是「值」（代表输出内容），激活函数决定哪些键被匹配。事实性知识（如「法国首都是巴黎」）被编码在特定神经元中，可通过权重编辑来修改（知识编辑技术）。早期层存语法信息，深层存语义/事实信息。

**SwiGLU**：现代 LLM（Llama、Mistral）使用的 FFN 变体，在标准两层 MLP 基础上增加门控路径：两条并行投影（W1 和 W3），W1 经 SiLU 激活后与 W3 逐元素相乘，再经 W2 降维。门控机制决定哪些信息被放行，效果优于普通 MLP。

**分工总结**：Attention 负责「开会」（token 间信息交换），FFN 负责「独自思考」（每个 token 结合知识对信息深度加工），残差连接确保信息流动顺畅且梯度可以传播。

💡 **未探索分支：** 知识编辑（Knowledge Editing）——如何定位并修改 FFN 中存储的具体事实
