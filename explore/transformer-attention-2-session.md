---
root: "继续探索 Attention 机制，重点深挖 Multi-Head Attention 还没搞清楚的问题"
slug: transformer-attention-2
continues: transformer-attention
started: 2026-05-12T00:00:00
status: done
---

## Q1 Multi-Head Attention 的 Head 分工与实证研究

**问：** Multi-head Attention 中每个 head 是怎么分工的？有没有研究表明不同 head 关注不同语言现象？

**答：**

### 头的分工：理论层面

在 Multi-Head Attention 中，每个 head 拥有自己独立的一组参数矩阵（W^Q_i、W^K_i、W^V_i），将输入投影到一个维度更低的子空间（d_head = d_model / h）。每个 head 在这个子空间里独立计算注意力，最终 h 个 head 的输出被拼接后再经 W^O 线性融合。

关键在于：这些子空间是各自独立学习的——没有任何显式约束迫使不同 head 关注不同的东西。**分工是涌现的，不是设计进去的。** 梯度下降自动发现：如果多个 head 都只学一种关注模式，参数就是浪费；于是网络在训练中倾向于让不同 head 专业化，形成功能分化。

### 实证研究：头真的分工吗？

**1. Voita et al. (2019) — "Analyzing Multi-Head Self-Attention: Specialized Heads Do the Heavy Lifting, the Rest Can Be Pruned"**

这是最直接的研究。他们对 Transformer 翻译模型（英→俄）的每个 head 进行了功能分析和剪枝实验，发现：
- **位置敏感 head（Positional heads）**：关注的总是固定相对位置的 token，例如「前一个词」或「后一个词」，在几乎每个句子中都稳定表现。
- **句法 head（Syntactic heads）**：关注语法依存关系，如动词注意其主语、形容词注意它修饰的名词。
- **稀有词/终止符 head（Rare words heads）**：有些 head 习惯性地将注意力集中到句子中频率低的词（信息量更大的词）或 [SEP] 这样的特殊 token 上。

更重要的是，他们发现大部分 head 是「冗余的」——只有少数几个 head 对最终翻译质量真正关键，大多数 head 被剪掉之后性能几乎不下降。

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

### 典型的分工模式归纳

| 分工类型 | 典型行为 | 常见层位 |
|---------|---------|---------|
| 位置敏感 | 总是关注前 1/后 1 个 token | 浅层 |
| 局部依存 | 关注句法直接相邻关系（修饰、并列） | 浅-中层 |
| 长距离依存 | 主谓一致、跨从句关系 | 中层 |
| 指代解析 | 代词 → 先行词 | 中层 |
| 语义/全局 | 软性主题相关性，非结构化 | 深层 |
| 标记偏移 | 关注 [SEP]/[CLS] 作为注意力"蓄水池" | 各层均有 |

### 重要的注意事项

1. **因果性 vs 相关性**：上述研究大多是相关性观察（attention 权重高的位置 ≈ 该 head 关注什么），但 Jain & Wallace (2019) 等工作指出：高注意力权重并不一定等于「对预测结果真正重要」。
2. **层位重要性**：head 的分工高度依赖所在层位——同样是位置 3 的 head，在第 1 层和第 10 层的行为可能完全不同。
3. **模型依赖性**：这些发现在不同模型（BERT、GPT、翻译模型）之间不完全通用。

> 💡 **未探索分支：** 注意力权重是否真的等于「重要性」？Attention 的可解释性争议（Jain & Wallace vs Wiegreffe & Pinter）是一个非常有意思的方向。
