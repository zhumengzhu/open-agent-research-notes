# 为什么主流 LLM 全是 Decoder-only？——程序员的线性代数视角

> **读者定位：** 有编程经验、没读过 ML 论文、没碰过 Transformer。读完你能理解：(1) 三种 Transformer 架构的区别——用数据库查询做类比；(2) 苏神（苏剑林）的核心洞察——为什么「满秩」这个词是关键；(3) 这个结论的边界条件在哪里。
>
> **基于：** 苏剑林，《[为什么现在的LLM都是Decoder-only的架构？](https://kexue.fm/archives/9529)》，2023-03-17。本文不是翻译，而是用程序员熟悉的语言重述其核心论证。

---

## §0 三种架构——用 SQL 做类比

Transformer 是当前所有大语言模型的骨架。它处理文本靠一个叫 **Self-Attention（自注意力）** 的机制——让每个 token（词/子词）决定应该「关注」序列里的哪些其他 token。

根据「能看到哪些 token」，Transformer 有三种用法。用数据库查询来类比最容易理解：

```text
假设序列是: A → B → C → D → E

┌─────────────────────────────────────────────────────────────┐
│ Encoder-only（如 BERT）                                      │
│                                                             │
│ 每个 token 能看到「所有」token:                               │
│   A: [A B C D E]                                            │
│   B: [A B C D E]  ← 和 A 完全一样的视野                        │
│   C: [A B C D E]                                            │
│                                                             │
│ SQL 类比: SELECT * FROM table  — 全表扫描，无 WHERE 条件       │
│ 适合: 填空、分类、理解任务                                     │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Decoder-only（如 GPT、LLaMA、DeepSeek）                       │
│                                                             │
│ 每个 token 只能看到「自己及之前」的 token（因果注意力）:         │
│   A: [A]                                                    │
│   B: [A B]                                                  │
│   C: [A B C]                                                │
│   D: [A B C D]                                              │
│   E: [A B C D E]                                            │
│                                                             │
│ SQL 类比: SELECT * FROM table WHERE position <= current      │
│          — 带 WHERE 条件的累积查询                             │
│ 适合: 文本生成、对话、代码补全                                  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Encoder-Decoder（如 T5、原始 Transformer 翻译模型）            │
│                                                             │
│ Encoder 部分: 输入做全表扫描（双向注意力）                      │
│ Decoder 部分: 输出做累积查询（单向注意力）+ 还能关注 Encoder 结果 │
│                                                             │
│ SQL 类比: 先 SELECT * FROM input_table，                     │
│          再根据结果逐行生成 output_table                       │
│ 适合: 翻译、摘要——有明确「输入→输出」边界                       │
└─────────────────────────────────────────────────────────────┘
```

现在所有主流大模型（ChatGPT、Claude、DeepSeek、通义千问）都是第二种——Decoder-only。问题来了：**为什么？**

直觉上 Encoder-Decoder 应该更强——毕竟 Encoder 能看到完整输入（全表扫描），信息量更大。Google 在 [T5](https://papers.cool/arxiv/1910.10683) 和 [UL2](https://papers.cool/arxiv/2205.05131) 两篇论文里也确实发现 Encoder-Decoder 效果更好。那为什么工业界全选了 Decoder-only？

苏神的回答是：**T5 的优势来自参数翻倍，不是双向注意力本身。** 他的核心论据是一个简洁的线性代数事实——下三角矩阵的秩。

---

## §1 实验——控制变量法

比较 T5（Encoder-Decoder）和 GPT（Decoder-only）有个麻烦：**两个变量同时变了**。

| 架构 | 输入部分注意力 | 参数量 |
|------|:---:|:---:|
| GPT（Decoder-only） | 单向 | 1× |
| T5（Encoder-Decoder） | 双向 | ∼2× |

T5 效果更好，到底是因为「双向注意力」好，还是因为「参数多一倍」好？分不清。

苏神搞了一个公平对比：**GPT vs UniLM**。

| 架构 | 输入注意力 | 输出注意力 | 参数量 |
|------|:---:|:---:|:---:|
| GPT | 单向 | 单向 | 1×（共享） |
| UniLM | **双向** | 单向 | 1×（共享） |

唯一的变量就是输入部分的注意力模式。在 10 亿参数规模上从零训练，损失函数只算输出部分（公平），结果：

> **UniLM（输入双向）相比 GPT 没有任何优势，甚至某些任务更差。**

如果这个结论有代表性，那 T5 比 GPT 好，纯粹是因为参数多了一倍，而不是架构本身。

---

## §2 核心洞察——为什么双向注意力反而更差？

这是苏神这篇文章里最精彩的部分。他用了一个程序员能秒懂的线性代数事实来解释。

### 2.1 Attention 矩阵是怎么算出来的

每个 [Attention](https://arxiv.org/abs/1706.03762) 层内部，模型做这件事：

```python
# 伪代码
Q = input @ W_q    # shape: (n, d)  — n=序列长度, d=head维度
K = input @ W_k    # shape: (n, d)
# Attention 矩阵 = softmax 作用在 Q·Kᵀ 上
scores = Q @ K.T   # shape: (n, n)
attention = softmax(scores, dim=-1)
```

关键在于：`Q @ K.T` 是两个 `(n, d)` 矩阵的乘积。注意 **`n ≫ d`**——序列长度通常 512~4096，而 head 维度通常只有 64~128。

**这就是一个低秩分解。** 像这样：

```python
# 一个 4096×4096 的矩阵，但秩最多只有 64
A = Q @ K.T  # 4096×64 乘 64×4096 → 4096×4096，rank ≤ 64
```

[已有研究](https://papers.cool/arxiv/2103.03404)证明：这种低秩 Attention 矩阵的表达能力会随层数加深**双指数级退化**。数学上，秩越低，矩阵能承载的信息量越少。

### 2.2 但 Decoder-only 不受影响——为什么？

Decoder-only 的因果 Attention 加了一个 mask——只看当前及之前的 token，不看未来的：

```python
# 因果 mask = 下三角矩阵
scores = Q @ K.T                       # shape: (n, n)
mask = torch.tril(torch.ones(n, n))     # 下三角为 1，上三角为 0
masked = scores.masked_fill(mask == 0, float('-inf'))  # 上三角填 -inf
attention = softmax(masked, dim=-1)     # 上三角 → 0，严格下三角
```

关键来了：**加上 mask 后的 Attention 矩阵是一个下三角矩阵。**

下三角矩阵有一个程序员友好的性质：

```text
行列式 = 对角线上所有元素的乘积

如果对角线上每个元素都不为 0 → 行列式 ≠ 0 → 矩阵满秩
```

因为 softmax 输出恒为正，对角线上每个元素都是正数。所以 **Decoder-only 的每个 Attention 矩阵自动满秩**——不管 n 比 d 大多少。

用代码理解：

```python
import torch

Q = torch.randn(4096, 64)
K = torch.randn(4096, 64)

# 双向 Attention：n >> d，矩阵自然低秩
bi_attn = torch.softmax(Q @ K.T, dim=-1)
print(torch.linalg.matrix_rank(bi_attn))  # 大概 ≤ 64

# Decoder-only Attention：下三角结构强制满秩
mask = torch.tril(torch.ones(4096, 4096))
causal_scores = Q @ K.T
causal_scores = causal_scores.masked_fill(mask == 0, float('-inf'))
causal_attn = torch.softmax(causal_scores, dim=-1)
print(torch.linalg.matrix_rank(causal_attn))  # = 4096（满秩）
```

### 2.3 这意味着什么

| | 注意力矩阵 | 秩 | 表达能力 |
|---|---|---|---|
| Encoder（双向） | 密集矩阵 | ≤ d（受低秩分解限制） | 受限 |
| Decoder（因果） | 下三角矩阵 | = n（满秩） | 更强 |

**Encoder 的「全表扫描」看似信息量大，实际上因为低秩约束，矩阵本身已经「压缩」了。Decoder 的「限制视野」看似吃亏，但因为下三角结构保住了满秩，每一层都在满载运转。**

这就是「少即是多」在数学层面的体现。

---

## §3 不正交就是浪费——一个程序员的直觉

如果你还觉得反直觉，换个角度：

```text
Encoder（双向）的 Attention 矩阵:
  n×n 的矩阵，但由两个 n×d 的矩阵乘出来。
  这就好比你定义了一个 4096×4096 的二维数组，但实际只用了 4096×64 的
  信息来填充。不管你怎么调参，有大量「自由度」是冗余的。
  大部分行与行之间是线性相关的——不正交。

Decoder（因果）的 Attention 矩阵:
  同样是 n×n，但下三角约束保证了每行都和其他行独立。
  没有浪费的维度。每一行都在干不同的事。
```

在程序里，一个秩为 k 的矩阵本质上只需要 k 个基向量就能表达。双向 Attention 的基向量数量被 head 维度锁死在 64 左右。而 Decoder 的因果 mask 像是一个结构约束，强制基向量展开到 n 维——等价于「信息不压缩」。

---

## §4 边界条件——这个结论不适用什么场景

苏神在文章和后续 [FAQ](https://kexue.fm/archives/9547) 中明确标注了前提：

### 4.1 限定在「生成任务」

这个结论讨论的是 **LLM 以文本生成为目标的场景**。对于纯理解任务（分类、检索、embedding），Encoder-only（[BERT](https://arxiv.org/abs/1810.04805)）仍然是更自然的选择——你不需要生成新的文本，只需要对已有文本做表示。

现实中的 RAG（检索增强生成）系统通常是 Encoder（做检索）+ Decoder（做生成）的组合。

### 4.2 实验在 10 亿参数级别

控制变量实验的规模不大。更大规模下这个结论是否持续成立，苏神自己说「留给有兴趣的读者」。不过从后来 GPT-3/4、LLaMA 等大规模模型的表现来看，方向是对的。

### 4.3 不否认参数翻倍带来的增益

文章不是要证明 Encoder-Decoder 架构「差」，而是说：如果你给 Decoder-only 也翻一倍参数（或者更深），它应该能达到甚至超过 Encoder-Decoder 的效果。T5 做得好，是因为参数量大，不是因为它用了双向注意力。

### 4.4 提出的改进方向：正反向混合注意力

苏神在文中提出了一个设想——是否可以用一半的 attention head 做正向（左到右）、一半做反向（右到左），这样每个 head 都保持满秩（下三角/上三角），但整体仍然获得双向交互。初步实验验证了有效性。这暗示了一个方向：**未来不一定是纯 Decoder-only，而是每个 head 保持满秩的混合架构。**

---

## §5 一句话总结（给你跟同事解释用）

> Decoder-only 不是「阉割版」Transformer。它的因果 mask 恰恰是一个结构约束，强制 Attention 矩阵满秩——就像数据库里加了 WHERE 条件的查询反而让索引能精确匹配。Encoder 的双向注意力是全表扫描，看着信息全，实际上因为低秩分解，矩阵里大部分行是线性相关的，浪费了大量表达空间。

---

## §6 进一步阅读

- [苏神原文](https://kexue.fm/archives/9529) — 本文的核心来源
- [原文 FAQ](https://kexue.fm/archives/9547) — 读者提问和作者详细回复，澄清了很多边界条件
- [*Attention is Not All You Need: Pure Attention Loses Rank Doubly Exponentially with Depth*](https://papers.cool/arxiv/2103.03404) — 双向 Attention 低秩退化的原始论文
- [T5 论文](https://papers.cool/arxiv/1910.10683) — Encoder-Decoder 对比实验的经典工作
