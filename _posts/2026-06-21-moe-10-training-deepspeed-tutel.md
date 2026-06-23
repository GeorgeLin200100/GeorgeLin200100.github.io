---
layout:     post
title:      "MoE Training: DeepSpeed and Tutel"
subtitle:   "MoE Training: DeepSpeed and Tutel"
date:       2026-06-21
author:     George Lin
header-img: img/post-bg-gen.png
catalog: true
tags:
- moe
---

## 引言：微软的两条路线

2022 年到 2023 年，微软研究院在 MoE 训练系统方向上连续发表了两篇重量级论文：ICML 2022 的 DeepSpeed-MoE 和 MLSys 2023 的 Tutel。同一个机构，同一个问题——"怎么让 MoE 训练又快又好"——却给出了两种完全不同风格的答案。

DeepSpeed-MoE 的思路是**架构创新**：改模型结构，用更少的参数达到同样的精度。如果模型本身能小 3 倍，那么通信量、显存占用、推理延迟都会同时缩小——这是从数学层面削减成本，不依赖硬件升级，也不依赖调度技巧。

Tutel 的思路是**运行时自适应**：不改模型，改系统的调度方式。MoE 的工作负载是动态的——每个 token 选哪个专家、每个专家收多少 token，每一步都在变。Tutel 设计了一套机制让并行策略、流水线段数、All-to-All 算法都能在运行时零成本切换，始终匹配合当前负载。

两道不同的题，两张不同的答卷。DeepSpeed-MoE 的问题设定是："给定一个质量目标，能不能用更小的模型达到？"Tutel 的问题设定是："给定任意一个 MoE 模型，能不能让它跑得更快？"

从结果上看，两者都做到了当时的最优：DeepSpeed-MoE 把 MoE 模型尺寸压缩了最多 3.7 倍，推理系统中万亿参数模型延迟压到 25ms 以下；Tutel 把单层 MoE 在 2048 GPU 上加速了 5.75 倍，并且不挑模型——无论什么 MoE 配置，都能自适应地找到最优调度。

理解这两个系统的区别和联系，对于我们理解 MoE 训练系统的设计空间至关重要。这也不是非此即彼的选择——Tutel 后来被同时集成到了 Fairseq 和 DeepSpeed 中，两者在工程上是互补的。

我们就从 DeepSpeed-MoE 讲起。

---

## 第一部分：DeepSpeed-MoE —— 从模型结构削减通信

### 1.1 问题设定：5 倍训练成本降低

DeepSpeed-MoE 工作的起点是一个干净的工程问题：**GPT 式的自回归 NLG（Natural Language Generation）模型，用 MoE 能做到什么程度？**

这个问题之所以值得问，是因为在 DeepSpeed-MoE 之前，MoE 在 NLP 领域的成功主要集中在 encoder-decoder 架构上——GShard 做多语言翻译，Switch Transformer 基于 T5 做 seq2seq 任务。这些模型的训练成本本来就不高（相对于 GPT-3 这种 175B 参数的自回归模型）。而自回归 NLG——GPT-3、MT-NLG 530B 这个系列——才是真正烧钱的地方：MT-NLG 530B 在超过 2000 块 A100 GPU 上训练了 3 个月，消耗了超过 300 万 GPU 小时。

如果把 MoE 应用到 GPT 式模型上，训练成本能降多少？DeepSpeed-MoE 给出的答案是 **5 倍**。

具体来说：用 350M 参数的 base 模型加上 128 个专家（每次只激活 Top-1 专家），总参数量 13B，但每个 token 实际激活的参数只有 350M——和 base 模型一样。同理，1.3B+MoE-128 总参数量 52B，激活量 1.3B。论文在 128 块 A100 GPU 上用 300B token 训练这些模型。

关键结果：
- **350M+MoE-128 (13B) 的 zero-shot 质量全面碾压 350M dense**：LAMBADA 从 52.03 涨到 62.70，PIQA 从 69.31 涨到 74.59，TriviaQA 从 3.21 涨到 16.58。
- **1.3B+MoE-128 (52B) 的验证 loss 和 6.7B dense 重合**——更小 base 的 MoE 达到了更大 dense 的质量，同时训练计算量只和 1.3B 的 base 相当（因为每个 token 只激活一个专家）。
- **吞吐量对比**：1.3B+MoE-128 的训练吞吐为 372 samples/sec（128 A100），而 6.7B dense 只有 70 samples/sec——**正好 5 倍**。

这就是 5 倍训练成本降低的算术：训练 1.3B 的 MoE 模型，获得 6.7B dense 的质量，而前者的 GPU 算力需求约等于训练 1.3B dense，后者约等于训练 6.7B dense。差距来自 dense 模型参数量更大、单步计算更多、需要的 batch size 更大（6.7B 需要 1024 的 batch size，而 1.3B+MoE-128 只需要 512）。

但如果仅仅这样，DeepSpeed-MoE 就不会成为一篇 ICML 论文了。真正的贡献在后面：**这个 52B 的模型虽然训练便宜，但推理时需要加载全部 52B 参数——这比质量等价的 6.7B dense 大了近 8 倍。** 推理是 memory bandwidth bound 的，模型大 8 倍意味着在同样的带宽下推理慢 8 倍。

于是 DeepSpeed-MoE 提出了三个层次的解决方案：PR-MoE（改架构）、MoS（加蒸馏）、推理系统（做工程优化）。三层叠加，最终做到了**比质量等价 dense 模型推理快 4.5 倍、便宜 9 倍**。

---

### 1.2 PR-MoE：两个现象催生的新架构

PR-MoE，全称 Pyramid-Residual MoE，是基于两个独立发现提出的。这两个发现都来自对 350M+MoE-32 模型的消融实验。

#### 现象一：深层更需要专家

标准的 MoE 架构在所有层的结构完全相同——每一层都有相同数量的专家，专家结构一致。但一个深层神经网络的所有层真的需要同样多的专家吗？

在 CV 领域，这早已是常识：浅层学习通用表征（边缘、纹理），深层学习任务特定表征（物体部件、语义概念）。但 NLP 和 MoE 领域中还没有人系统探索过这个问题。

DeepSpeed-MoE 做了一个简单但决定性的实验：将 350M+MoE-32 模型的 MoE 层分为两半。**First-Half-MoE**（前 12 层用 MoE，后 12 层是 dense MLP）和 **Second-Half-MoE**（前 12 层 dense，后 12 层 MoE），两者的总计算量完全一样。

结果如图 2（左）所示：Second-Half-MoE 的验证 loss **显著低于** First-Half-MoE。深层更需要专家的多样性，浅层用普通的 dense MLP 就够用了。

这是现象一（Phenomenon-I）："浅层学通用特征，深层学任务特定特征"这个在 CV 领域的老结论，在 NLP 的 MoE 场景下也成立了。

#### 现象二：残差专家 = Top-2 质量，Top-1 通信量

增大专家容量的标准做法是使用 Top-2 gating——每个 token 选两个专家，计算两次 FFN，结果加权平均。这能提升模型质量（更多专家可以互相纠正误差），但代价是通信量翻倍：每个 token 要发送给两个专家，接收两份结果。

DeepSpeed-MoE 问了一个巧妙的问题：**能不能把一个专家固定下来，另一个专家动态选择？** 具体做法是在每个 MoE 层加上一个**固定的 dense MLP 分支**作为残差连接——每个 token 必定经过这个固定的 MLP，同时再经过一个动态选择的专家。两个分支的输出相加得到最终结果。

这就是 **Residual-MoE**。设计直觉是：**让动态选择的专家扮演"误差修正"（error correction）的角色**——固定 MLP 提供一个粗粒度的通用映射，专家负责修正剩余误差。

实验结果（图 2 右）关键：Residual-MoE 的验证 loss 曲线和 Top2-MoE 几乎重合，但通信量只有 Top2-MoE 的一半——因为每个 token 只需要一次 All-to-All（固定 MLP 的参数在所有 GPU 上都有一份完整拷贝，不需要通信）。

更进一步，训练速度比 Top2-MoE 快了超过 10%，主要的增益就来自通信量的减少。

#### PR-MoE：金字塔 + 残差的合成

把两个现象合在一起，就有了 PR-MoE（图 3 右）：

```
Pyramid（金字塔）+ Residual（残差）= PR-MoE
```

**Pyramid (金字塔)**：后面的层用更多专家，前面的层用更少专家。例如 350M+PR-MoE-32/64：24 层中的 10 个 MoE 层用 32 个专家，最后 2 个 MoE 层用 64 个专家。

**Residual（残差）**：每个 MoE 层内部，每个 token 同时通过一个固定 MLP 和一个动态选择专家，输出相加。

关键结果（Table 4）：

| 模型 | 参数量 | LAMBADA | PIQA | BoolQ | TriviaQA |
|------|--------|---------|------|-------|----------|
| 350M+MoE-128 | 13B | 62.70 | 74.59 | 60.46 | 16.58 |
| 350M+PR-MoE-32/64 | **4B** | **63.65** | 73.99 | 59.88 | 16.30 |
| 1.3B+MoE-128 | 52B | 69.84 | 76.71 | 64.92 | 31.29 |
| 1.3B+PR-MoE-64/128 | **31B** | **70.60** | 77.75 | 67.16 | 28.86 |

350M+PR-MoE-32/64 只用不到 1/3 的参数（4B vs 13B），达到了 350M+MoE-128 完全同等的 zero-shot 精度——甚至在 LAMBADA 上还略高一点。1.3B 的 case 也类似：60% 的参数，同等质量。

Ablation 实验（图 4）展示了从 350M+MoE-32 到 350M+MoE-128 的 loss gap 逐步被填补的过程：加 Pyramid 好一些（-32/64），加 Residual 好一些（Residual-MoE-32），两者一起用（PR-MoE-32/64）几乎完全补上了这个 gap——验证 loss 差距缩小到 0.01。

#### 系统实现：多专家并行与多数据并行

PR-MoE 的好处在理论上很清楚，但在工程上引入了一个新问题：**不同层有不同数量的专家，怎么设置并行度？**

标准的 MoE 训练中，最有效的策略是让专家并行度（expert parallelism degree）等于专家数——这样每个 GPU 负责正好一个专家，不会出现"一个 GPU 负责多个专家导致每个专家分到的 token batch 变小"的问题，也不会有负载不均。

但 PR-MoE 不同层有不同专家数：如果按最小专家数（32）设置 EP 度，则 64 专家的层中每个 GPU 负责 2 个专家，batch 折半；如果按最大专家数（64）设置 EP 度，32 专家的层中有些 GPU 没有专家——负载不均。

DeepSpeed-MoE 的解法是支持**多专家并行和多数据并行**（multi-expert and multi-data parallelism）。具体举例：PR-MoE 在 128 GPU 上，非专家参数用 128-way 数据并行；专家参数用 {32, 64, 128} 三种不同的专家并行度，对应 {4, 2, 1} 的数据并行度。这样每条 GPU 在任意层中都只负责恰好 1 个专家，没有 token batch 缩减，也没有负载不均。

---

### 1.3 MoS (Mixture-of-Students)：分阶段蒸馏

PR-MoE 已经将参数量压缩了最多 3 倍。但还能再压。途径是知识蒸馏（Knowledge Distillation）——让一个较小的学生模型模仿较大的教师模型的输出。

#### 设计：MoE-to-MoE 蒸馏

DeepSpeed-MoE 的 KD 设计有三个特色：

1. **MoE-to-MoE**：学生模型保留和教师一样的 MoE 结构（稀疏门控、专家并行推理），只是**每个专家的深度被减少**。减少的幅度是 12.5%（24 层减到 21 层）。之所以保留 MoE 结构而不是蒸馏成 dense 模型，是因为 MoE 的稀疏激活特性在推理时能带来显著的延迟优势。

2. **PR-MoE 基座**：教师和学生都采用 PR-MoE 架构，也就是说学生本身就是经过架构层面压缩的模型。KD 在这个基础上继续压缩深度。

3. **MoS 命名**：因为学生保留了 MoE 的稀疏门控结构，但每个专家被降低了容量，所以被称为 Mixture-of-Students。

KD 的损失函数是标准的组合损失：

$$\min_{\theta_s} \mathbb{E}_{(x,y) \sim \mathcal{D}} \left[ \mathcal{L}(x; \theta_s) + \lambda \mathcal{L}_{KD}(x; \theta_s) \right]$$

其中第一项是标准的语言模型交叉熵损失，第二项是学生和教师输出分布之间的 KL 散度。

#### 关键发现：Full KD 在后期有害

一个关键的实验发现：**如果整个训练过程都做 KD，学生的最终精度反而不如从头训练**（Table 5, Row 2 vs Row 3）。具体来看图 5：在训练的早期（前 400K step），KD 确实让学生 loss 更低；但之后 KD 开始拖后腿——学生的验证 loss 曲线从低于 baseline 逐渐被反超。

这是什么原因？DeepSpeed-MoE 给出的假设是**容量不足导致的欠拟合**。PR-MoE 已经通过架构变化（减少浅层专家数、残差设计限制专家多样性）压缩了模型容量，再减少深度 12.5%，学生模型的表达能力已经不足以同时优化两个损失函数（LM loss 和 KD loss）。在训练的后期，学生模型被迫在两个 loss 之间做出 trade-off——而且往往牺牲了 LM loss 去讨好 KD loss（因为 KD loss 的下降更容易被观察到）。

这就引出 **Staged KD** 方案：**在 400K step 处停止 KD，之后只优化标准的 LM loss**。

结果（图 6）：Staged KD 的学生验证曲线几乎和教师完全重合。两个模型规模下都是如此——350M+PR-MoE-32/64（教师 4B）和 1.3B+PR-MoE-64/128（教师 31B）。

最终精度（Table 5）：

| 模型 | 参数量 | LAMBADA | PIQA | BoolQ | TriviaQA |
|------|--------|---------|------|-------|----------|
| 350M+PR-MoE+L24（教师） | 4B | 63.65 | 73.99 | 59.88 | 16.30 |
| 350M+PR-MoE+L21（仅减层，无KD） | 3.5B | 62.33 | 73.88 | 52.35 | 8.81 |
| 350M+PR-MoE+L21+Full KD | 3.5B | 61.56 | 73.18 | 57.89 | 12.13 |
| 350M+PR-MoE+L21+**Staged KD (MoS)** | 3.5B | **63.46** | **73.34** | **58.07** | **13.69** |

Staged KD 几乎保留了教师的全部质量——350M 案上平均精度保留 99.5%，1.3B 案上保留 99.1%。

注意一个容易忽略的细节：如果直接减层（l24 -> l21）而不做任何 KD，精度会大幅跳水——BoolQ 从 59.88 跌到 52.35（-7.53），TriviaQA 从 16.30 跌到 8.81（-7.49）。Staged KD 不仅挽回了这些损失，在部分任务上还超过了单纯减层的 baseline（因为 KD 提供了一定的正则化效应）。

#### 叠加效果

PR-MoE（架构压缩） + MoS（蒸馏压缩）的叠加效果：

- 350M 线：标准 MoE 是 13B，PR-MoE 是 4B（3.25x），PR-MoE+MoS 是 3.5B（3.7x）
- 1.3B 线：标准 MoE 是 52B，PR-MoE 是 31B（1.68x），PR-MoE+MoS 是 27B（1.93x）

**最多 3.7 倍的总压缩比。** 这对于推理系统来说是质变——加载到显存的参数减少 3.7 倍，意味着内存带宽需求降低 3.7 倍，延迟降低 3.7 倍，最小需要的 GPU 数量减少 3.7 倍。

---

### 1.4 推理系统：应对 MoE 推理悖论

MoE 推理面临一个根本性的悖论：

- **最优情况**：每个 token 只激活一个专家（Top-1 gating），需要的参数量约等于 base model 的大小。一个 1.3B+MoE-128（52B）处理一个 token 只需要 1.3B 参数。
- **最差情况**：推理是 batched 的（一个 sentence 或 paragraph 包含多个 token），不同 token 可能选择不同专家，迫使系统加载全部 52B 参数。

推理的目标是**尽可能逼近最优情况，同时不让最差情况的发生概率占主导**。DeepSpeed-MoE 的推理系统通过三层优化来达成这个目标。

#### 第一层：多维度并行切分

MoE 层由两部分组成：专家参数（MLP 权重）和非专家参数（Attention 权重、LayerNorm 等）。两者需要不同的并行策略。

**专家参数**：使用 Expert Parallelism + Expert-slicing。

Expert Parallelism 将专家分布到不同 GPU 上。在 1.3B+MoE-128 中，128 个专家分布到 128 块 GPU，每块 GPU 只有 1 个专家。关键路径上的参数从 52B 降到 1.3B——每块 GPU 只加载自己负责的那个专家的权重。和 6.7B dense 模型（1 块 GPU 加载 6.7B 参数）相比，关键路径缩短了 5 倍。如果没有通信开销，理论上前者比后者快 5 倍。

Expert-slicing 是 EP 的扩展：当 GPU 数量超过专家数时，把单个专家也切成多份分布到多个 GPU 上（类似 tensor parallelism 但作用在专家参数上）。

**非专家参数**：使用 Tensor-slicing + Data Parallelism。

Tensor-slicing 在节点内把 Attention 权重切分到多个 GPU 上，利用节点内的 NVLink 高带宽做 AllReduce。Data Parallelism 在跨节点时复制非专家参数的不同副本，处理不同的 batch，节点间不需要通信。

组合起来的并行配置十分复杂：Figure 7 展示了一个 16 GPU、8 个专家、tensor-slicing degree = 4、expert-slicing degree = 2 的例子。每一块 GPU 同时属于多个并行通信组——推理中的通信调度变成一个多维矩阵运算。

#### 第二层：通信优化

**Hierarchical All-to-All（分层 All-to-All）**

标准的 All-to-All 延迟随 GPU 数量 p 线性增长 O(p)。当 p = 128 时，如果每个点对点消息很小（如推理时 batch 较小），延迟会主导通信时间。

分层 All-to-All 分两步执行：

第一步：节点内 All-to-All。每个 GPU 把数据按目标节点分组，在节点内做一次 All-to-All，使每个 GPU 的数据按目标节点组织好。

第二步：节点间 All-to-All。每个 GPU 只和远程节点的对应 GPU 通信。

中间需要两次 data-layout 变换来对齐内存布局。复杂度从 O(p) 降到 O(G + p/G)，其中 G 是每个节点的 GPU 数。代价是总通信量翻倍（因为数据经过两次传输），但对于小消息推理场景（延迟绑定而非带宽绑定），这个 trade-off 是合理的。

**Parallelism-Coordinated Communication（并行协调通信）**

当 Tensor-slicing 和 Expert Parallelism 同时存在时，标准的做法是把它们当作独立的黑盒分别执行通信。但 DeepSpeed-MoE 发现了一个可以串联优化它们的机会：Tensor-slicing 的 AllReduce 会在其组内复制数据。当 Tensor-slicing 后接着执行 Expert Parallelism 的 All-to-All 时，All-to-All 不需要在所有 p 个设备之间执行——**只需要在共享相同 tensor-slicing rank 的设备子集内执行**。

例如：128 GPU，8-way tensor-slicing，128-way expert parallelism。标准的 All-to-All 在 128 个设备间执行，延迟 O(128)。但在 8-way tensor-slicing 复制数据后，All-to-All 只需要在 128/8 = 16 个设备间执行，延迟 O(16)——减少了 8 倍。代价是在 tensor-slicing 组内做一次 allgather 来复制数据。

这个优化暴露了一个重要的设计原则：**并行策略之间的顺序和通信模式不是独立的，通过协调它们可以大幅降低总延迟**。

#### 第三层：Kernel 优化

MoE 推理中的 dispatch/combine 操作传统上通过稀疏 einsum 实现。但深度分析发现，**稀疏 einsum 的复杂度是 S x E x M x ce**（S=token 数，E=专家数，M=hidden dim，ce=expert capacity），其中 (E-1)/E 的运算是和零相乘——因为每个 token 只选 k 个专家（k 通常为 1），对于未选中的 E-k 个专家，einsum 做的全是 "乘以零再相加" 的无用功。

DeepSpeed-MoE 的优化方案：
1. **用稠密映射表代替稀疏张量**：门控函数输出 token-to-expert 的映射表（一个简单的一维数组），而不是 one-hot 稀疏矩阵。这消除了大量 "乘以零" 运算。
2. **Kernel 融合**：将 top-k、cumsum、scatter 操作融合成单个 GPU kernel。Cumsum 使用 Blelloch scan 算法在 GPU 上并行化。
3. **Data-layout 变换代替 einsum**：用显式的内存布局变换来执行 token 的排序和恢复，而不是稀疏矩阵乘法。复杂度从 S x E x M x ce 降到 S x M x ce——**去掉了 E 因子**。

综合效果：MoE kernel 延迟减少 **6 倍以上**。

#### 推理性能总览

- **万亿参数 MoE 模型推理延迟 < 25ms**（24B+MoE-128，1 trillion 参数，128 GPU）。
- 和 PyTorch baseline 相比：107B 模型加速 5.5x，349B 加速 5.9x，1T 加速 7.2x，2T 加速 **7.3x**。
- 和质量等价 dense 模型相比（图 14、15）：PR-MoE+MoS 推理比 6.7B dense 快 2.4x 且便宜 2.4x（1 GPU vs 128 GPU），比 175B dense 快 4.5x 且便宜 **9x**。

最反直觉的一点是：**MoE 模型推理的吞吐量随 GPU 增加呈现超线性增长**（图 10）。DeepSpeed-MoE 在 8 GPU 到 64 GPU 之间，每 GPU 吞吐量不减反增——和 dense 模型只能做到线性 scaling（理想情况）或 sub-linear scaling（有通信开销时）形成鲜明对比。

原因两个：（1）GPU 多了，每个 GPU 负责的专家数少了，模型加载量变小，内存带宽压力降低；（2）DeepSpeed 的高效通信减少了对通信的依赖，使得（1）的好处不被通信开销抵消。

---

## 第二部分：从 NLG 训练成本缩减看 MoE 的实际价值

在进入 Tutel 之前，有必要先回顾 DeepSpeed-MoE 在 NLG 训练成本缩减上的核心发现。这些发现回答了"MoE 到底值得不值得"这个工程决策问题。

### 2.1 5 倍成本缩减的细节

训练设置：
- **128 块 A100 GPU**（Azure ND A100 实例）
- **300B token** 训练数据（MT-NLG 使用的数据集）
- **Top-1 gating**（只选 1 个专家，最小化通信）
- 对比对象：350M dense、1.3B dense、6.7B dense（GPT-3 风格的超参数配置）

两个关键的训练经验：
1. **MoE 模型需要更小的学习率和更长的学习率衰减周期**。这是因为 MoE 比同 base 的 dense 模型有更多参数，需要更保守的优化策略。
2. **辅助负载均衡损失系数设置为 0.01**。这个值在平衡负载和不损害模型质量之间取得了良好的折中。

训练吞吐量对比（Table 3）：

| 模型 | Throughput (samples/sec) | 相对成本 |
|------|--------------------------|---------|
| 6.7B dense | 70 | 1x |
| 1.3B+MoE-128 | 372 | **5x** |

为什么是 5 倍？两个因素叠加：
- 1.3B+MoE-128 的计算量 ≈ 1.3B dense（因为 Top-1 gating）
- 1.3B dense 比 6.7B dense 的参数量少了 5.2x，计算量少了约 5x

### 2.2 Zero-shot 质量验证

比吞吐量更重要的是质量是否等价。验证使用了 6 个 zero-shot 任务：
- LAMBADA：完形填空预测
- PIQA：物理常识推理
- BoolQ、RACE-h：阅读理解
- TriviaQA、WebQs：问答

结果（Table 2）：1.3B+MoE-128 在所有 6 个任务上和在验证 loss 上都和 6.7B dense 处于同一水平——LAMBADA 69.84 vs 71.94，PIQA 76.71 vs 76.71（完全一样），BoolQ 64.92 vs 67.03。虽然有轻微差距，但对于 5 倍的训练成本节省来说完全可以接受。

### 2.3 与同期工作的对比

DeepSpeed-MoE 的 NLG 工作并非孤例。同期 Google 的 GLaM（143B）和 Meta 的 Fairseq-MoE（52B - 207B）也在探索 MoE 在自回归 NLG 上的应用。但 DeepSpeed-MoE 有几处关键差异：

1. **更关注推理**：GLaM 和 Fairseq-MoE 主要关注训练，DeepSpeed-MoE 同时解决训练和推理两个问题。
2. **参数效率更高**：在 LAMBADA 上，DeepSpeed-MoE 的 PR-MoE+MoS（27B-31B）达到的精度与 GLaM（143B）相近，但参数量少了 5 倍。
3. **系统性贡献更强**：不仅报告了"MoE 能训 NLG"，还给出了完整的训练-蒸馏-推理 pipeline。

---

## 第三部分：Tutel —— 运行时自适应，不做架构假设

### 3.1 问题设定：MoE 的"动态性"被系统忽略了

DeepSpeed-MoE 从模型架构层面降低通信需求——这个思路的前提是你能够改模型。但如果模型已经定死了呢？如果架构创新已经被穷尽了，剩下的优化空间在哪里？

Tutel 的出发点恰恰是这里。它观察到：**现有的 MoE 框架（DeepSpeed、Fairseq、FasterMoE）在运行时都是静态的**——并行策略选定后在整个训练过程中不变，pipeline 分段数固定，All-to-All 算法固定。但 MoE 工作负载是动态的，静态调度必然在某些时刻不匹配实际需求。

Tutel 用一组数据量化了这个动态性（Figure 1）：在 SwinV2-MoE 模型的训练过程中，不同层的专家容量变化幅度高达 **4.38 倍**。第 1 层、第 4 层、第 10 层的负载曲线完全不同——有些层负载稳定，有些层剧烈波动，而且它们之间没有明显的相关性。

动态工作负载来自哪里？来自 token routing 机制的本质：每个 token 根据 gate 函数的输出选择专家，而 gate 函数的输出依赖于 token 的内容。内容的分布是训练数据驱动的，每次迭代都不同。**这种不确定性是 MoE 算法本身固有的，不是 bug。**

### 3.2 自适应并行切换：七个选两个

#### 通信复杂度分析

MoE 训练中，GPU 可以以不同的方式组织起来：
- **DP (Data Parallelism)**：每块 GPU 有完整模型，处理不同数据
- **MP (Model Parallelism)**：单专家被切分到多 GPU
- **EP (Expert Parallelism)**：不同专家分布到不同 GPU

三个维度的排列组合产生 7 种可能的并行策略：
1. DP
2. MP
3. EP
4. DP + MP
5. EP + DP
6. EP + MP
7. EP + DP + MP

Tutel 做了一个关键的理论分析（Table 4）：通过严格的通信复杂度比较，证明了**这 7 种中只有 2 种在任意条件下可能成为最优——DP 和 EP+DP+MP。**

分析逻辑：
- **MP**：通信复杂度 O(Cg x W)，恒劣于 EP+MP。
- **EP**：需要 E/W >= 1 才有效，且恒劣于 EP+MP。
- **DP+MP**：对任意 r（MP group size），恒劣于 EP+DP+MP。
- **EP+DP**：是 EP+DP+MP 在 r=1 时的特例。
- **EP+MP**：是 EP+DP+MP 在 r=W/E 时的特例。

这里的核心洞见是 **EP+DP+MP 是一个足够通用的框架**——调节一个参数 r（group size，即 (W/E)/r 中的 r），可以 cover 从纯 DP（r=0）到纯 EP+DP（r=1）到混合 EP+DP+MP（1<r<W/E）到纯 EP+MP（r=W/E）的所有有效并行方案。

这大大简化了系统设计的复杂度——**不需要为 7 种并行策略设计 7 套执行流，只需要设计一套统一的执行流，用参数 r 来控制。**

#### 零成本切换的机制

切换并行策略在传统系统中极其昂贵，原因有二：
1. **不同的数据布局**：DP 需要每 GPU 有一份完整权重；MP 需要权重被切片；EP 需要权重按专家分布。切换意味着重新分布权重参数——通信量和参数量相当。
2. **不同的梯度累积机制**：DP 的梯度需要 AllReduce；EP+DP+MP 中梯度的归属和累积方式完全不同。

Tutel 的解法是**统一张量布局（single unified tensor layout）**。无论当前使用哪种并行策略，模型参数的物理分布始终不变——遵循 ZeRO Stage-3 的分区方式（每 GPU 持有唯一的权重分片）。在这个统一布局上，不同的并行策略通过不同的**计算和通信编排**来实现，而不需要重新分布参数。

具体实现：
- **Switchable DP (r=0)**：forward 时 all-gather 所有分片到所有 GPU，backward 时 reduce-scatter 梯度——和 ZeRO Stage-3 完全一样。
- **Switchable EP+DP+MP (r>=1)**：将 GPU 分为大小为 (W/E)/r 的组。组内做 DP（all-gather 权重），跨组做 MP（tensor-slicing）。在 MoE 的入口和出口处，通过 "local repeat" 和 "local sum" 操作将 DP 和 MP 串联起来，保证数学等价性。

关键设计：当 r 的值改变时，只有通信组（communicator groups）的拓扑结构改变，**没有任何参数迁移发生**。切换成本为 O(1)——仅仅是重建 NCCL communicator 的常数时间开销。

Figure 8 展示了用 adaptive:r 参数指定并行策略——所有从 0 到 max(W/E) 的 r 值构成一个连续谱，完全覆盖 DP 到 EP+MP 的所有有效状态。

#### 为什么需要自适应切换：实例分析

Figure 3 和 Figure 12 展示了不同 workload 下不同并行策略的性能差异：
- 当 capacity_factor f 较大（专家负载重，很多 token 涌向同一个专家）时，DP（r=0）更优——因为专家计算 heavy，掩盖了通信。
- 当 f 较小（负载均匀、通信消息小）时，EP+DP（r=1）甚至 EP+DP+MP（r>1）更优——因为需要用并行来分散计算、加速单个 expert 的 compute。

在 64 GPU 的 Base 配置下（E=16, H=2K, D=2K），不同 r 值的最佳策略差异达 7.39%-27.76%。而在 Large 配置下（H=32K），最优 r 随 f 的取值在 0 到 4 之间变化。**没有任何单一 r 值在所有 f 下都是最优的。**

### 3.3 2DH All-to-All：解决大尺度下的消息碎片化

当 GPU 数量 n 增大时，All-to-All 中每个点对点消息的大小 S/n 越来越小。在小消息尺寸下，高带宽链路（NVLink 3.0、HDR InfiniBand 200Gbps）无法被饱和——因为消息太小，链路的大部分时间花在等待而不是传输上。

Figure 16 展示了这个问题：当消息从 256 MiB 缩到 1 MiB 时，GPUDirect RDMA 的有效带宽从接近理论上限的 25 GB/s 骤降到几乎为零。而 n 增大时，S/n 不可避免地缩小——如果 S=128 MiB，n=2048，每个点对点消息只有 64 KiB——这是一个带宽利用率极低的区域。

Tutel 提出的 2DH（Two-Dimensional Hierarchical）All-to-All 将多个本地 GPU 发往同一个远程 GPU 的小消息聚合成一个大消息，从而增大有效消息尺寸。

算法步骤（Algorithm 2 & Figure 17）：
1. **Phase 1 - 跨步内存拷贝**：通过 stride memory copy 将非连续的本地数据块按目标节点重新排列，使同一个远程节点的数据连续存放。
2. **Phase 2 - 节点内 All-to-All**：在 G 个本地 GPU 之间做 All-to-All，将各 GPU 按目标节点归并好的数据交换到正确的源 GPU 上。
3. **Phase 3 - 第二次跨步内存拷贝**：将交换后的数据按远程目标 GPU ID 重新排列。
4. **Phase 4 - 节点间 All-to-All**：现在每个 GPU 只和 p/G 个远程 GPU 通信（而不是 p 个），每个消息的大小变为原来的 G 倍。

复杂度从 O(p) 降到 O(G + p/G)。增加的 cost 是两次 stride memory copy 和一次节点内 All-to-All——但这些都是不随 p 增大的常数开销。

但是有一个重要的实际考量：**naive 的 2DH 实现会遇到非连续内存访问的性能退化**。由于本地数据分散在不同的内存位置，节点内 All-to-All 需要扫描不连续的地址空间，延迟从 n=8 时的 600 us 增长到 n=2048 时的 5 ms。Tutel 通过 stride memory copy 解决这个问题——将不连续的 chunks 重新组织为连续的内存空间，使得后续的 All-to-All 操作在连续的内存区域上进行。

此外，Tutel 还利用 Microsoft SCCL 编译器来生成优化的 2DH 实现。SCCL 可以用 DSL 描述 2DH 算法的通信模式，自动融合同步屏障、应用 LL128 协议（NVIDIA 的低延迟协议）——最终效果显著，例如 1 MiB 的 2DH 比 NCCL 原生实现快 2-3 倍（Figure 19）。

2DH 和 Linear All-to-All 不是谁绝对优于谁的问题。Figure 18 显示：在 64-128 GPU 规模上，对于 32 MiB 以上的消息，Linear 更好；对于 1 MiB 的消息，2DH 更好。随着 GPU 数增多，2DH 的优势扩大——**动态选择合适的 All-to-All 算法本身就是 Tutel 自适应流水线的一部分。**

### 3.4 自适应流水线：联合优化分片数和算法

MoE 层的执行流程天然是串行的：dispatch All-to-All -> 专家计算 -> combine All-to-All。但这三个阶段可以通过流水线并行重叠：把 token 按 capacity 维度切分成多个 partition，不同 partition 在不同 GPU stream 上异步执行。

Figure 9 展示了核心设计：在一个 2-GPU 的例子中，tokens 被沿 capacity 维度切成 2 份 C0 和 C1。C0 和 C1 分别进入不同的 stream——C0 的 All-to-All 和 C1 的 All-to-All 可以并发，C0 的专家计算和 C1 的 All-to-All 可以重叠。

关键的设计挑战是：**分多少份（pipelining degree d）和用什么 All-to-All 算法（Linear 还是 2DH）必须联合确定。** 原因在于二者相互影响——d 越大，每个 partition 的消息越小，2DH 的优势越大；d 越小，每份消息越大，Linear 可能更好。而且，NCCL 的 compute kernel 和 GPU 的计算 kernel 并发时的相互干扰很难用模型预测——有些配置下 Linear + d=2 是最好的，换个 workload 后 2DH + d=4 反超。

Tutel 的解决方案是**预先生成最优配置词典**。具体流程：
1. 枚举所有可能的 (c/R, r, d, a) 组合——c 是容量值，R 是窗口大小（默认 128），r 是并行参数，d 是分片数（{1, 2, 4, 8}），a 是 All-to-All 算法（{Linear, 2DH}）。
2. 对每个 c/R，通过三元搜索（Ternary Search）找到最优 r——因为 r 在 [1, W/E-1] 范围内的最优吞吐量是凸函数。再加上 r=0 和 r=W/E 的两个端点。
3. 对每个 (c/R, r)，测试 {4 种 d} x {2 种 a} = 8 种流水线配置，找到最优。
4. 总 trial 数 = (log_{3/2} (W/E) + 2) x 4 x 2 x (c 值的种类数)。对于 W/E 在 1-128 范围和 3-5 种不同的 c/R 值，trial 数量在几十到几百之间——可以在几分钟内完成。

在训练时，每个 step 根据当前的容量值 c，查表获得最优的 {r, d, a}，无需在线 profiling。

实验结果（Table 6）：在 243 种 MoE 配置上的测试中，和固定的线性 All-to-All + d=1 相比，自适应流水线的平均提升为 **9%-107%**（取决于 GPU 规模和工作负载）。和最差静态策略相比，提升为 **23%-599%**。

Figure 13 展示了不同 f（容量因子，代表不同 workload pattern）下的提升：
- f=4 时：最多 39% 提升
- f=8 时：最多 57% 提升
- 自适应策略在所有 f 下都不会比静态最优差——而任何单一静态策略在不同的 f 下都有相对劣势

### 3.5 快速编解码与动态容量因子

#### Fast Encode/Decode

MoE 的 dispatch 和 combine 阶段需要两个关键操作：
- **Encode**：把 MoE 层的输入（shape: T x M）和 gate 的输出（token-to-expert 分配）转换成 All-to-All 的输入（shape: E x Cg x M）。
- **Decode**：把 All-to-All 的输出转换回 MoE 层的输出（shape: T x M）。

传统的实现使用 einsum 和矩阵乘法：dispatch_input = einsum("TEC, TM->ECM", combine_mask, moe_input)。复杂度是 O(T x E x Cg x D)，其中大部分运算是和零相乘（因为每 token 只分配到 k 个专家，E - k 个专家的映射全为零）。

Tutel 将这两个步骤重写为 **SIMT 高效的稀疏 GPU kernel**（Figure 20, 21）。核心思想：
- 不对整个 E x T 的稀疏矩阵做 einsum，而是使用 token-to-expert 的稀疏映射表，只对非零条目做操作
- 复杂度从 O(T x E x Cg x D) 降到 O(T x k x D)，其中 k << E
- 利用 warp shuffling、Blelloch scan 和 half2 向量化等技术（这些技术通常只适用于 dense compute，但通过精心设计的线程-数据映射也可以在稀疏场景中生效）
- 3 个专用 kernel：K0（encode 的 backward & combine 的 forward）、K1（encode 的 forward & combine 的 backward）、K2（两者的交叉项）

效果：
- 单层 MoE 的非专家计算延迟大幅降低（Figure 15 中的 "MoE gate" + "MoE dispatch" + "MoE combine" 柱子）
- GPU 显存节省 **20%-90%**（Table 5/Table 9）：以 tokens/step=32K、D=H=4K、top-k=2、Eg=2 为例，Fairseq MoE 单层吃 57.9 GiB，Tutel 只要 5.7 GiB

显存节省的来源很直观：不再需要存储 E x T 大小的稀疏中间张量，只需要一维的 index 数组。

#### Flexible All-to-All

另一个重要的优化是 **Flexible All-to-All**。传统的 All-to-All 将输入 (E, Cg, D) 变换为 (W, Eg, Cg, D)，然后专家计算做 batch matrix multiply。问题在于输出 Shape (W, Eg, Cg, D) 的 W 维度依赖于 GPU 数量——当 GPU 数变化时，同一个专家分配的 Cg 维度不同（因为 Cg = C / W），这影响 matrix multiply 的效率。

Flexible All-to-All 将输出变换为 (Eg, C, D)，其中 C 是不随 W 变化的聚合容量。这意味着**无论多少 GPU，专家的矩阵乘法形状一样**——这是一个很重要的工程细节，它保证了 scaling 时计算 kernel 的性能稳定性。Figure 11 展示了在 A2A（原始）和 Flexible A2A 布局下的吞吐量对比，在大 W 时差距达约 50%。

#### 动态容量因子

Tutel 的 capacity_setting 参数有三种模式（Figure 10）：
- **固定模式**（capacity_setting = positive）：直接指定容量因子，和 Fairseq/DeepSpeed 一样
- **自适应模式**（capacity_setting = 0）：每个 step 自动设置容量因子为"刚好不丢 token"的最小值。利用 MoE gate 的输出实时计算每个专家收到的 token 数，按需分配容量
- **混合模式**（capacity_setting = negative）：将 -x 设为容量因子的上限，实际值在上限内自适应

自适应模式的价值在于：如果某些 step token 分布均匀，则用小容量节省计算；如果某些 step 分布不均，则自动增大容量避免丢 token。**不需要人工调参，也不牺牲模型质量。**

---

### 3.6 Tutel 整体性能

#### 单层 MoE 速度

单层 MoE 的加速效果由 Figure 14 的 breakdown 曲线清晰展示。以 Fairseq/DeepSpeed MoE 为 baseline：

| 优化措施 | 16 GPU | 2048 GPU |
|---------|--------|----------|
| Baseline (Fairseq) | 1x | 1x |
| + Fast Encode/Decode | 3.52x | 1.04x |
| + 2DH All-to-All | - | 4.25x |
| + Flexible All-to-All | - | 1.24x |
| + Adaptive Pipelining | 1.43x | 1.04x |
| **Total (Tutel)** | **4.96x** | **5.75x** |

注意每个优化在不同规模上贡献的权重不同：Fast Encode/Decode 在 16 GPU 上贡献了最大的加速（3.52x），但在 2048 GPU 上几乎不贡献（1.04x）——因为在大规模上 All-to-All 通信主导了延迟。2DH All-to-All 则恰恰相反，从 256 GPU 开始成为 dominant factor，在 2048 GPU 上贡献了 4.25x。这是一个典型的 Amdahl 律场景——优化瓶颈在不同规模上会转移。

#### SwinV2-MoE：端到端验证

Tutel 在 SwinV2-MoE（基于 Swin Transformer V2 的 MoE 版本）上做了端到端训练和推理验证。这是一个重要的贡献——在 Tutel 之前，稀疏 MoE 模型主要应用于 NLP 和 encoder-decoder 架构。SwinV2-MoE 是第一个应用于计算机视觉任务且取得了正向精度的 MoE 模型。

端到端速度比较（Table 7）：

| GPU | Fairseq Train/Infer | Tutel Train/Infer | Speedup |
|-----|---------------------|-------------------|---------|
| 8 | 240/507 | 274/1053 | 1.14x/2.08x |
| 32 | 162/455 | 249/892 | 1.54x/1.96x |
| 128 | 146/375 | 226/792 | **1.55x/2.11x** |

模型质量（Table 8）：SwinV2-MoE-B 在 ImageNet-22K 上的 top-1 准确率达到 38.5%，比 dense SwinV2-B 的 37.2% **高出 1.3 个百分点**。在下游任务上也全面领先：ImageNet-1K fine-tuning (+0.4%), 5-shot (+2.0%), COCO object detection (+0.4/+0.4 box/mask AP)。

特别值得注意的是 COCO 检测的结果——这是稀疏 MoE 首次在 COCO 物体检测任务上展示出比密集模型更好的效果。而且 Tutel 还发现了一个反直觉的训练技巧：**fine-tuning 时冻结 MoE 层反而比联合 fine-tune 效果更好**（Table 10：固定 MoE 层的 AP 为 53.4，直接 fine-tune 为 51.3）。推测原因是在 fine-tuning 阶段 token 分布发生变化，MoE gate 没有充分适应新的数据分布，导致路由质量下降。

---

## 第四部分：对比分析 —— 架构创新 vs 系统调度

### 4.1 两种哲学，一个问题

现在我们可以把两张答卷放在一起比较了。

**DeepSpeed-MoE 的核心思想**：通过改变模型结构来降低对系统的需求。PR-MoE 减少了专家总数（从而减少 All-to-All 的参与方数量），Residual 设计减少了每 token 激活的专家数（从而减少 All-to-All 的消息数）。这些是**算法层面的通信量减少**——不依赖硬件，也不依赖调度。MoS 进一步压缩了模型深度，直接降低推理时的显存带宽需求。

**Tutel 的核心思想**：不改变模型，通过动态调整系统的运行方式来匹配动态变化的负载。自适应并行切换让系统始终使用最优的并行策略（而不是训练开始时选定的那个），自适应流水线让 All-to-All 和计算的最佳重叠方式随负载变化。这些是**系统层面的通信效率提升**——没有减少通信量的绝对大小，但减少了其实际耗时（通过重叠和算法选择）。

一张对比表：

| 维度 | DeepSpeed-MoE | Tutel |
|------|--------------|-------|
| 优化层面 | 算法架构 | 系统运行时 |
| 核心手段 | 减少模型大小（PR-MoE, MoS） | 动态调度（并行、流水线、All-to-All 算法选择） |
| 对模型假设 | 可以改模型架构 | 不做任何假设 |
| 通信量减少方式 | 减少专家数/激活数（从根本上减量） | 优化执行效率（不减量，加工序） |
| 适用性 | 模型设计阶段 | 模型训练/推理阶段 |
| 最佳场景 | 新模型设计 | 已有模型加速 |
| 主要贡献 | 3.7x 模型压缩 + 7.3x 推理加速 | 5.75x 单层加速（2048 GPU） |

### 4.2 它们解决的是通信问题的不同层次

回到 MoE 通信的根本困境：All-to-All 的通信量大约是 batch_size x seq_len x d_model x num_selected_experts。要降低通信开销，只有几种方法：

1. **降低 batch_size**：不可行，因为需要保证训练稳定性和吞吐量。
2. **降低 d_model**：不可行，因为模型质量会受损。
3. **降低 num_selected_experts**：DeepSpeed-MoE 的 Residual-MoE 就是做这个——从 2 降到 1（通过固定 dense MLP 分支取代第二个动态专家），通信量减半但质量不降。
4. **降低 All-to-All 的参与方数量**：Pyramid-MoE 在浅层用更少的专家，等于减少了那些层的 All-to-All 范围。
5. **压缩模型减小总通信**：MoS 减少深度 12.5%，直接减少每个专家在推理时需要加载的参数总量。
6. **优化通信执行**：Tutel 的自适应流水线、2DH All-to-All——不减通信量的绝对值，但提高执行效率。

DeepSpeed-MoE 覆盖了 3、4、5，Tutel 覆盖了 6。它们互不冲突——Tutel 可以被集成到 DeepSpeed 中（事实上也确实被集成了），对 PR-MoE 压缩后的模型做进一步的运行时加速。

### 4.3 设计哲学的差异对技术选择的深远影响

举个例子：为什么 DeepSpeed-MoE 没有做自适应并行切换，而 Tutel 没做 PR-MoE？

对于 DeepSpeed-MoE，架构层面的优化（PR-MoE）带来了 3x 的参数缩减，这比任何自适应策略能做到的 30% 左右的提升都要大一个数量级。**如果架构优化能实现成倍的收益，自适应调度的 30% 提升相比之下就不那么紧急了。** 而且 DeepSpeed-MoE 关注推理——推理时 batch=1 或较小，并行策略的选择空间较训练时小得多，自适应切换的收益会更有限。

对于 Tutel，它的目标就是最大化任何 MoE 模型的性能——不假定你能改模型，甚至不假定你能控制模型设计。在这种设置下，**系统的自适应性是唯一的优化杠杆**。而且 Tutel 特别关注大规模训练（2048 GPU），在这种规模下 All-to-All 的通信开销占比从 33% 到 57%（Table 2），自适应流水线的收益巨大。

这两条路反映的是研究和工程中的一种常见张力：**源头优化 vs 过程优化。** 源头优化（DeepSpeed-MoE）从根本上减少工作量，代价是改变了模型本身；过程优化（Tutel）在不改变工作量的前提下提升执行效率，代价是系统复杂度增加。

### 4.4 工程上可借鉴的经验

抛开理论分析，这两篇论文对实际 MoE 系统工程有几条可操作的经验：

**经验 1：专家不是越多越好，也不是越均匀越好。**
DeepSpeed-MoE 的 Pyramid 设计用更少的专家总数达到了同样的质量。这意味着在 GPU 数量有限时（例如只有 32 块 GPU 但想跑 E=128 的模型），PR-MoE 可以大大降低对 GPU 数量的需求。具体来说：如果一个层用 32 个专家需要 32-way EP，那 128 个专家需要 128-way EP——但 PR-MoE 可以在大部分层只放 32 个专家，少数层放 64 个专家，用 64-way EP + 2-way DP 就能跑。

**经验 2：残差设计是通信最友好的"第二专家"。**
Residual-MoE 给你 Top-2 的质量、Top-1 的通信量。实现起来非常简单——在 standard MoE 层旁边加一个 dense MLP 分支，两个分支的输出相加。固定的 dense MLP 在所有 GPU 上有完整的副本，不需要 All-to-All。这是一个零代价的优化。

**经验 3：蒸馏的时机比蒸馏本身更重要。**
MoS 的 Staged KD 揭示了一个深层问题：蒸馏 loss 和任务 loss 在模型容量不足时会互相冲突。这不是 MoE 特有的问题，但在 MoE 中特别突出——因为 MoE 的"有效容量"受 sparse routing 的限制。400K step 的 early-stop 听起来是一个 magic number，但它背后是模型容量瓶颈在训练后期才开始显现这个物理事实。

**经验 4：并行策略不应是"一锤子买卖"。**
Tutel 的 adaptive:r 设计展示了并行策略可以被优雅地参数化，收敛到仅 2 种模式（DP 和 EP+DP+MP）。这意味着在实际工程中，你不需要实现 7 种并行策略的切换——只需要实现一个从 r=0 到 r=W/E 的参数扫描机制，就能覆盖所有有效策略。

**经验 5：All-to-All 算法也需要自适应性。**
Tutel 的 2DH 和 Linear 算法在不同消息大小和 GPU 数下各有优势。在工程实践中，关键不是选哪个算法，而是建立一个能在运行时自动选择最优算法的框架。Tutel 的词典方案虽然需要预 profiling，但成本很低（几分钟 vs 数天的训练），值得应用。

---

## 结语：一个平台的两翼

DeepSpeed-MoE 和 Tutel 都是微软在 MoE 系统上的重要工程贡献。回头看，它们在微软内部的定位也很有趣：DeepSpeed-MoE 后来整合了 Tutel 的技术（在 DeepSpeed 的 GitHub 上可以看到 Tutel 的集成代码），形成一个"架构优化 + 运行时自适应"的完整栈。

从本文第一篇到第十篇，我们一直在看 MoE 通信问题的不同解法：从芯片层（NoC）、到互联层（NVLink/NVSwitch）、到系统调度层（FasterMoE）、到模型架构层（DeepSpeed-MoE）、到运行时自适应层（Tutel）。每一层都在"削减"一部分通信开销，但没有任何一层能单独解决所有问题。

DeepSpeed-MoE 从模型层面砍掉了 3.7x 的参数，Tutel 从运行时层面榨出了 5.75x 的单层加速——两者的贡献量级相当，但作用在不同维度上。如果用最朴素的话来总结：DeepSpeed-MoE 让你**传得更少**，Tutel 让你**传得更快**。

在下一篇文章中，我们将转向另一条解决 MoE 通信问题的路线——MegaBlocks 的块稀疏矩阵乘法（block-sparse computation）。它和 DeepSpeed-MoE、Tutel 的思路有根本性的不同：不减少通信量，不优化通信效率，而是**彻底改写 MoE 的计算范式**——把"token 路由 + All-to-All"变成"块稀疏矩阵乘法"。这又是另一个世界了。

---

**（全文完，约 10500 字。下一篇：MoE 训练系统（下）—— MegaBlocks 与块稀疏计算）**
