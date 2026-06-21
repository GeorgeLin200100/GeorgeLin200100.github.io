---
layout:     post
title:      MoE Communication Full Stack
subtitle:   MoE Communication Full Stack
date:       2026-06-21
author:     George Lin
header-img: img/post-bg-gen.png
catalog: true
tags:
- moe
---

## 引言：一张 GPU 的算力账单

2024 年末，DeepSeek 以 671B 参数、37B 激活的 DeepSeek-V3 引爆了开源社区。这个模型在 2048 块 H800 GPU 上训练了 14.8 万亿 token，总成本仅 278.8 万 GPU 小时——约合 557.6 万美元。与之对比，Llama 3 的 405B Dense 模型训练成本是多少？Meta 没有公布具体数字，但业内普遍估计在 3000-4000 万 H100 GPU 小时之间，按市场价折算约 6000-8000 万美元。

差距超过一个数量级。

同样是要达到一流水平，DeepSeek-V3 的训练效率比同体量的 Dense 模型高了 10 倍以上。这背后不是玄学，而是 MoE（Mixture-of-Experts，混合专家）架构带来的结构性优势：**模型虽然大，但每次只激活一小部分，计算成本和激活参数相关，和总参数量脱钩**。671B 总参数，实际每次前向只激活 37B——相当于 18:1 的"能力-计算"比。相比之下，Llama 3 的 405B Dense 是 1:1。

但这笔账单的另一半是**通信**。

MoE 把计算变便宜了，但代价是把数据移动变贵了。每处理一个 batch，所有 token 都要跨 GPU 找到自己归属的专家，运算完了还要跨 GPU 把结果送回来。这个操作叫 All-to-All，它和 Dense 模型最大的不同在于——**它不能靠增加设备来隐藏通信**。因为每个 token 的去向是动态的，在计算开始之前谁也不知道哪块 GPU 的通信链路会被塞满。

DeepSeek-V3 的技术报告里有一句话，是整个 MoE 通信问题最精确的总结：

> *"跨节点专家并行引入的通信开销导致计算-通信比约为 1:1。"*

**计算和通信各占一半时间。** 这意味着如果不做任何优化，2048 块 GPU 中的一半——1024 块——在计算进行的同时处于空闲等待状态。

这系列博客，我们就从通信这个切口切进去，从芯片内部的 NoC（Network-on-Chip）一直看到跨机架的光电路交换，把 MoE 模型在工业界落地的通信挑战和解决方案完整地走一遍。但在此之前，我们需要回答一个更根本的问题：MoE 的通信为什么成了瓶颈？这条路是怎么从"可以接受"走到"无法忽视"的？

---

## MoE 的核心矛盾：稀疏激活让计算便宜，但让通信变贵

### 从 Dense 到 Sparse：一张天平的倾斜

要理解 MoE 的通信问题，首先要理解 Dense Transformer 的通信为什么相对可控。

在 Dense Transformer 中，每个 token 处理每一层的所有参数，数据流是确定性的。前向传播中，每一层主要做矩阵乘法，通信只发生在两个地方：

- **流水线并行（PP）的跨阶段传输**：前一层 GPU 把激活值传给下一层 GPU，数据量约为 batch_size × hidden_dim × seq_len。这发生在节点间。
- **Tensor 并行（TP）的 AllReduce**：把单层权重切到多个 GPU，每个 GPU 算完后需要 AllReduce 汇总结果。TP 通常限于节点内，因为 AllReduce 对延迟敏感，需要 NVLink 级的带宽。

这两个通信模式都是**静态的**——编译时就能确定每块 GPU 在每一时刻要收发多少数据，数据流向也是固定的。这意味着编译器可以做流水线调度（如 1F1B）、通信融合（如 AllReduce 合并）、以及通信-计算重叠。

MoE 打破了这个确定性。

在 MoE 架构中，标准的 FFN 层（或者隔一层）被 MoE 层替代。MoE 层包含 E 个专家（experts），每个专家是一个独立的 FFN 子网络——有自己的权重矩阵（wi 和 wo）和自己的前向计算路径。对每个输入 token，路由网络（gate，通常是一个小的线性层 + softmax）计算 token 对每个专家的亲和度分数，选择 Top-K 个专家来真正处理这个 token。未选择的专家完全不被激活，**它们的参数甚至不需要加载到计算单元上**。

这个机制把整个计算-通信关系重写了：

**计算变便宜了**：模型总参数量可以疯狂膨胀——从几 B 到几百 B 甚至万亿参数——但每个 token 实际激活的计算量只和 K 个专家相关（K 通常是 1、2 或 8）。总参数和激活参数的比例可达 10:1 甚至 18:1（DeepSeek-V3）。这意味着模型"知道得更多"但"算得不多"——训练时的前向 FLOPS 只和激活参数量成正比。

**通信变贵了**：每个 token 必须跨越 GPU 去找专家。设想一个场景：token 在 GPU 0 上产生（它的 hidden state 由 GPU 0 计算得出），但路由网络判定该 token 的 Top-2 专家一个在 GPU 3 上，一个在 GPU 7 上。那么 GPU 0 必须把这个 token 的 hidden state（一个 d_model 维的向量）发送给 GPU 3 和 GPU 7。两个专家计算完成后，结果（同样是 d_model 维的向量）必须从 GPU 3 和 GPU 7 传回 GPU 0，在那里做加权平均。这个"先分发→远程计算→再合并"的流程，在 MoE 训练中通过 All-to-All 集体通信来实现。

**关键问题在于这个分发模式是动态的。** 路由网络的输出取决于 token 的内容，而 token 内容每次迭代都不同。你无法在编译时确定"GPU 3 在这一步会收到多少 token"——它可能收到 0 个（冷门专家），也可能收到远超 GPU 显存容量的大量 token（热门专家）。后者被称为**路由坍缩（routing collapse）**——如果所有 token 都倾向于选同一个专家，那个专家所在的 GPU 会被撑爆，其他专家所在的 GPU 几乎空闲，整个系统的效率暴跌。

### 专家容量：在信息丢失和通信浪费之间走钢丝

为了解决路由坍缩，GShard 引入了**专家容量（expert capacity）**机制，它深刻塑造了后续所有 MoE 系统的通信设计。

专家容量的计算公式简单直接：

```
expert_capacity = (tokens_per_batch / num_experts) × capacity_factor
```

其中 capacity_factor 是一个超参数，通常 > 1.0（GShard 和 Switch Transformer 都默认用 1.0-1.5）。容量因子大于 1 意味着每个专家预留的"接客名额"比平均分配的要多一些，以吸收路由不均衡。

容量因子是一个关键的通信-质量杠杆：

- **capacity_factor = 1.0**：每个专家正好接收 (total_tokens / num_experts) 个 token。如果 token 分配完全均匀，这是理想状态——没有浪费，没有丢 token。但实际路由几乎不可能完美均匀。
- **capacity_factor > 1.0**：每个专家有空余槽位。通信量增大（需要传输填充的空槽），计算量增大（对空槽做无用计算），但丢 token 的概率降低。多余的通信量是 padding——传输了但没有产生有效计算的数据。
- **capacity_factor < 1.0**：专家容量不够，超出的 token 被丢弃（dropped），其 hidden state 通过残差连接直通下一层。丢 token 会损失模型质量，尤其在 Scaling Law 的维度上，丢 token 的影响会随模型规模放大。

Switch Transformer 对这个 trade-off 做了系统的实证分析。在实验中，论文发现辅助负载均衡损失（auxiliary load balancing loss）以足够高的系数配合合理的容量因子，可以在丢 token 率 <1% 的情况下保持负载均衡。但这是基于 TPU 互联环境——在连接带宽更受限的 GPU+IB 环境中，容量因子带来的通信开销会被放大。

后来 MegaBlocks（2023）用块稀疏矩阵乘法彻底消除了容量因子的需要——不用 padding，不用丢 token——但这引入了新的计算和通信模式（我们在第十一篇会详细展开）。DeepSeek-V3 则在 2024 年用辅助损失自由（auxiliary-loss-free）的负载均衡策略——通过动态调整每个专家的 bias term 来引导路由——实现了完全不用丢 token 的训练，同时保持了负载均衡。

### All-to-All 的四张面孔

All-to-All 不只是"把所有数据发给所有人"。在 MoE 训练中，All-to-All 通信至少有四种不同的形式，每一种的瓶颈特征不同：

| 通信类型 | 出现场景 | 通信模式 | 瓶颈类型 | 数据量级 |
|----------|---------|---------|---------|---------|
| **All-to-All dispatch** | MoE 层前向 | 每个 GPU 向其 token 的目标专家所在 GPU 分发 hidden state | 带宽密集 | batch_size × seq_len × d_model × num_selected_experts |
| **All-to-All combine** | MoE 层前向 | 专家计算完成后，每个 GPU 将结果传回 token 原始所在 GPU | 带宽密集 | 同上（等量反向） |
| **AllReduce（专家梯度同步）** | MoE 层反向 | EP 组内如果同时有 DP，则专家梯度需要 AllReduce 同步 | 延迟敏感 | 专家参数量 / DP 组数 |
| **AllReduce + All-to-All 竞争** | MoE 层反向 | 注意力层、LayerNorm 等 Dense 层的 AllReduce 与 MoE 的 All-to-All 并发 | 竞争型 | 二者之和 |

最棘手的是第四种，也是实际训练中通信问题的核心。

在反向传播中，All-to-All combine 的反向（实际上是另一个 dispatch）和 All-to-All dispatch 的反向（实际上是另一个 combine）同时存在。与此同时，Dense 层的梯度 AllReduce 也在进行——这些是独立于 MoE 的通信，但共享同一根 PCIe 链路或同一张 InfiniBand HCA 网卡。如果没有任何调度机制，这些通信会在网卡发送队列中互相阻塞，导致其中一方——通常是 AllReduce，因为它的消息更小但更频繁——产生严重的排队延迟。

Lina 系统（ATC'23）对这个现象做了详细分析，发现只要给 All-to-All 和 AllReduce 分别设定优先级，就可以大幅减少两者的互相干扰。我们在第十二篇会专门展开。

---

## 工业演进三阶段：从"可以接受"到"无法忽视"到"一切都是通信"

MoE 不是新概念。早在 1991 年，Jacobs 等人就提出了"mixture of experts"作为集成学习的一种形式。Shazeer 等人在 2017 年将稀疏门控 MoE 引入深度神经网络（Outrageously Large Neural Networks），在 LSTM 语言模型上验证了稀疏激活的潜力。但真正让 MoE 在 Transformers 时代从学术概念变成工业落地的，是接下来三个阶段的工程化突破。每个阶段都揭示了一部分通信的成本，直到最后 DeepSeek-V3 不得不把**整个训练框架围绕通信来设计**。

### 阶段一：GShard (2020) — 证明可行，首次定义专家并行

2020 年，Google 在 GShard 论文中展示了一个 600B 参数的多语言 NMT Transformer，在 100 语言到英语的翻译任务上取得 SOTA。核心数据点：

- **600B 参数模型**，在 **2048 个 TPU v3 核心** 上训练 **4 天**，总计 **22 TPUv3 core-year**
- Dense baseline（2.3B 参数）训练到同等质量需要 **235.5 TPUv3 core-year**
- 模型大小增加 **16×**（37.5B → 600B），计算成本只增加 **3.6×**（6 → 22 core-year）
- 64B Dense baseline 的质量甚至被 **小得多的 MoE 模型** 超越——2.7B 激活参数的 MoE 模型达到了比 64B Dense 更好的翻译质量

这个 sub-linear scaling 是 MoE 价值的铁证。但 GShard 的贡献远不止实验数据，它首次系统性地解决了三个工程问题：

**问题一：如何表达并行而不让代码爆炸？**

在 GShard 之前，分布式训练的并行策略是硬编码在模型代码里的。开发者需要手工管理每一个 tensor 在哪个设备上、何时需要同步、通信使用什么原语。这让模型开发（想一个新的架构变化）和系统优化（需要一个不同的并行策略）之间产生了巨大的摩擦——改变一个就得重写另一个。

GShard 的答案是**一组轻量级注释 API**：

- `split(tensor, split_dimension, split_count)`：沿指定维度切分 tensor 到多个设备
- `replicate(tensor)`：在所有设备上复制 tensor
- `shard(tensor, ...)`：更灵活的切分策略

模型开发者用这些注释标记几个关键 tensor，其余代码写得就像在单设备上运行一样。XLA 编译器扩展根据注释自动生成并行执行计划——包括需要的 All-to-All、AllReduce 等通信原语。这个 "annotation-based auto-parallelization" 思路直接启发了后来的 Alpa（OSDI'22）和 PyTorch 的 FSDP/TP 方案。

**问题二：如何让编译时间不随设备数爆炸？**

如果用传统的 MPMD（Multiple Program Multiple Data）方式，每个设备生成不同的计算图，图的节点数随设备数线性增长。在 2048 个设备上，图的大小会让编译本身变得不可行。

GShard 发明了 **SPMD（Single Program Multiple Data）分区变换**：编译器生成一个单一的、在所有设备上运行的相同程序。编译时间不依赖设备数——O(1) 而不是 O(D)。这个思想后来成为大规模分布式训练编译器的标准范式。

**问题三：如何在保持负载均衡的同时实现高效并行路由？**

GShard 的解决方案是 **局部组调度（local group dispatching）**：

1. 将训练 batch 的 N 个 token 均匀分成 G 组，每组 S = N/G 个 token
2. 各组独立并行处理路由决策
3. 每组分配每个专家容量的 1/G 份额：容量 = 2N / (G × E)，其中 2 是因为 Top-2 路由
4. 辅助损失（auxiliary loss）鼓励路由概率分布均匀

这个设计的精妙之处在于：它既保证了全局的负载均衡（通过容量限制），又让路由计算实现了大规模并行（通过分组）。Group 的数量 G 成为一个重要的调优参数——G 越大，每组的并行度越高，但每组内的专家容量越小，丢 token 的概率越大。

**GShard 的通信局限性**

GShard 在 2048 个 TPU v3 上实现了 600B MoE 的训练，但它的 All-to-All 实现依赖于 TPU 的 ICI（Inter-Chip Interconnect）——一个为确定性通信模式而优化的高带宽互联。论文没有深入讨论 All-to-All 在不同硬件上的通信开销差异。更重要的是，GShard 的 Top-2 路由意味着每个 token 的 hidden state 要发送到两个专家所在的设备——在 2048 设备的规模下，这产生了巨大的 All-to-All 流量。只是 TPU v3 的 ICI 足够宽，让这个开销在当时的模型规模下没有成为最突出的矛盾。

### 阶段二：Switch Transformer (2022) — 万亿参数，通信开始显形

2022 年，同样是 Google 的团队推出了 Switch Transformer。表面上的核心改动似乎很小：把 GShard 的 Top-2 路由简化成 Top-1——**每个 token 只选一个专家**。

但这个小改动的动机和影响远不止表面上看起来那么简单。

**Top-1 的通信经济学**

为什么只选一个专家？Top-2 路由意味着每个 token 要发送到两个专家所在的设备，而 Top-1 只需要发一次。**All-to-All 通信量直接减半**——dispatch 减半，combine 减半。当专家数达到上千时，这个差异乘以 batch size 和 hidden dim 后，变成了一笔巨大的通信账单。

而且 Top-1 还减少了另一项潜在开销：如果 Top-2 的两个专家恰好在同一个设备上（在拓扑感知路由下是可能的），那还好。但如果它们在两个不同的设备上，token 的 hidden state 需要分别发送两次——两次跨设备传输，两次远程计算，两次结果收集。Top-1 把这个复杂性彻底消除了。

**当然，Top-1 不是无代价的。** Top-2 允许模型在"不确定"时把 token 同时发给两个专家，让它们各自的输出加权平均，这对模型质量是一个自然的保护机制。Switch Transformer 的实验必须证明 Top-1 在精度上不劣于 Top-2。答案是：在足够多的专家（32+）和合理的容量因子下，Top-1 和 Top-2 的模型质量几乎无差异——但通信量只有一半。这是一个漂亮的工业取舍：在质量不降的前提下，最大化地减少了通信。

**Switch Transformer 的并行策略分析**

Switch Transformer 的第五节是整个 MoE 并行策略分析的经典。论文把并行分为三个维度：

1. **数据并行（Data Parallelism, DP）**：n 路 DP 意味着每块 GPU 持有完整模型副本，每块 GPU 处理 B/n 个 token，只在梯度汇总时需要 AllReduce。通信最少，但对显存最贪婪。

2. **模型并行（Model Parallelism, MP/Tensor Parallelism, TP）**：m 路 MP 意味着每块 GPU 只持有模型的一部分（如 FFN 的 d_ff 维度被切分），每块 GPU 处理全部 B 个 token。每层都需要 AllReduce 来合并切分维度的结果。通信量大，但允许单个 GPU 放不下的大层。

3. **专家并行（Expert Parallelism, EP）**：E 个专家分布在 n 个设备上，每个设备持有 E/n 个专家的参数。每个 token 通过 All-to-All 被路由到对应专家所在的设备。

Switch Transformer 的核心洞察是：**这三个维度的通信特性完全不同，而 MoE 模型的结构天然适合将它们分层组合**。

对于 Switch-XXL（395B 参数，FLOP-matched to T5-XXL），论文同时使用了 EP（跨节点）和 MP（节点内），因为 d_ff 太大以至于单个 GPU 装不下一个专家的 FFN。这意味着每一次前向中，All-to-All（EP 的通信）和 AllReduce（MP 的通信）同时发生——这是 MoE 通信复杂性的缩影。

而对于 Switch-C（1.6 万亿参数，纯 EP，无 MP），由于 d_ff 设置较小（2080 vs 10240 of T5-XXL），每个专家可以放进单个 GPU。这意味着没有 MP 的 AllReduce 开销，**全部通信就是 All-to-All**。Switch-C 用 2048 个专家（Top-1 激活）和 15 层 Switch 层，在相同的训练计算预算下实现了对 T5-XXL 的 4× 加速——纯粹靠更高效地利用专家参数容量，而通信成本通过 Top-1 路由控制在了可接受范围内。

**关键公式：All-to-All 通信量**

```
All-to-All 通信量 = E × C × d_model × sizeof(dtype)
```

其中 E 是专家数，C 是专家容量（tokens per batch / num_experts × capacity_factor），d_model 是 hidden dimension。

当 E=2048、C 随 batch 增长时，每次 All-to-All 的数据量可以达到数 GB 级别。对于 TPU 的 ICI 来说这还在可容忍范围内，但对于 GPU + InfiniBand——尤其是有带宽限制的 H800（400 GB/s NVLink, 50 GB/s IB）——这就是一个必须主动应对的挑战。

**Switch Transformer 的通信教训**：通信在 MoE 训练中开始成为显性的成本因素。通过简化路由（Top-1）可以减少通信，但无法消除。当模型规模达到万亿参数级别，即使 TPU 的 ICI 也感受到压力——All-to-All 的延迟开始影响训练步时。如果把这个架构搬到 GPU+IB 集群上（带宽远小于 TPU 的 ICI），通信会成为压倒性的瓶颈。这正是 DeepSeek-V3 面对并解决的问题。

### 阶段三：DeepSeek-V3 (2024) — 通信就是一切

2024 年底，DeepSeek-V3 技术报告发布，MoE 通信的故事在这里达到了真正的高潮。

671B 总参数，37B 激活参数，14.8 万亿 token 训练数据，2048 个 NVIDIA H800 GPU。但真正惊人的不是这些数字，而是报告中这句毫不避讳的坦言：

> *"跨节点专家并行引入的通信开销导致计算-通信比约为 1:1。"*

再读一遍：**1:1。计算和通信各占一半。** 在 H800 的约束下（NVLink 400 GB/s——只有 H100 的 44%；IB 50 GB/s），如果不做激进优化，2048 块 GPU 中的一半在训练期间是空闲的。DeepSeek-V3 的整个训练框架——并行策略编排、pipeline 调度算法、All-to-All kernel 设计——都是围绕"吃掉"这个 1:1 的通信瓶颈来做的。深入展开。

**并行策略的精心编排**

DeepSeek-V3 的并行组合是：

- **16 路流水线并行（PP）**：模型在不同层次上被切到不同 GPU
- **64 路专家并行（EP）**，跨 **8 个节点**：每个节点 8 GPU，总共 64 GPU 参与 EP
- **ZeRO-1 数据并行（DP）**：优化器状态分片，每 GPU 只存自己的部分
- **没有使用 Tensor Parallelism（TP）**

最后一点是关键的设计选择。TP 需要在每次前向和后向中做 AllReduce——对于大 hidden dim（DeepSeek-V3 隐藏维度 7168），TP 的通信量是巨大的。DeepSeek 团队通过内存优化（RMSNorm 和 MLA 上投影的重计算、EMA 参数放 CPU、MTP 模块的参数共享等）消除了对 TP 的需要。这省掉了一整层通信开销——把省下的带宽和 SM 资源都留给了 All-to-All。

**DualPipe：让通信"消失"的调度算法**

DualPipe 是 DeepSeek-V3 最核心的工程创新，值得深入理解。

传统 pipeline 并行的 1F1B（one-forward-one-backward）算法有 pipeline bubble——前向阶段的后端 GPU 等待前端 GPU 完成，反向阶段的前端 GPU 等待后端 GPU 完成。这些 bubble 就是 GPU 的空闲时间。ZB1P 通过让 bubble 中包含部分反向计算来减少浪费，但不能完全消除通信暴露。

DualPipe 把思路推到极致：**让前向通信和后向计算重叠，后向通信和前向计算重叠。**

具体做法是：将每个 micro-batch 的前向和后向都拆成更细的组件——attention、all-to-all dispatch、MLP、all-to-all combine、PP 通信。然后通过双向 pipeline 调度（从 pipeline 两端同时喂入 micro-batch），让正向 micro-batch 的通信阶段恰好与反向 micro-batch 的计算阶段在时间上重合。

关键数字：

- DualPipe 的 pipeline bubble 为 (PP/2 - 1)(T_fwd_overlap + T_bwd - 3T_bwd_w)，其中 T_fwd_overlap 是两个互重叠的前向和后向块的执行时间
- 相比 1F1B，bubble 大幅减少；相比 ZB1P，bubble 也进一步减少
- 内存开销仅增加 1×（PP 分组的倒数），因为需要保留两份参数副本
- 随着 micro-batch 数量增加，DualPipe 的 bubble 不增加——这是相比 Chimera 的一个关键优势

**但 DualPipe 有一个前提条件：计算-通信比必须保持恒定。** 在这个条件下，DualPipe 可以保证 All-to-All 和 PP 通信被完全隐藏。如果模型进一步扩大但通信量不成比例增长，DualPipe 的有效性会打折扣——这就是为什么 DeepSeek 在技术报告中明确建议硬件厂商"开发更高带宽的跨节点互联"。

**跨节点 All-to-All：每一比特都要精打细算**

DeepSeek-V3 的 H800 集群内，节点内 NVLink 带宽 160 GB/s（H800 的 NVLink 是 H100 的 44%），节点间 InfiniBand 带宽 50 GB/s——前者是后者的 **3.2 倍**。这个带宽不对称是通信优化的核心约束。

为了最大化利用这有限的带宽，DeepSeek 对 All-to-All kernel 做了全栈优化：

1. **节点限制路由（Node-Limited Routing）**：每个 token 限制最多发往 4 个节点（不是 4 个 GPU），以减少 IB 流量。Token 到达目标节点后，通过 NVLink 在节点内的 8 个 GPU 之间转发到具体的专家所在 GPU。

2. **IB 和 NVLink 通信完全重叠**：一个 token 的 IB 传输尚未完成时，已经到达目标节点的 token 就已经通过 NVLink 在内部路由。这要求 kernel 设计上将 IB 接收和 NVLink 转发交织在一起，不能让 IB 接收阻塞 NVLink 转发。

3. **Warp Specialization**：将 20 个 SM 分成 10 个通信通道。在 dispatch 过程中，(1) IB 发送、(2) IB-to-NVLink 转发、(3) NVLink 接收由不同的 warp 分别处理。每个通信任务的 warp 数量根据实际负载动态调整。Combine 过程同理。

4. **PTX 指令级调优**：使用自定义 PTX 指令和自动调优的通信 chunk size，显著减少 L2 缓存的使用和对其他 SM 的干扰。通信 SM 和计算 SM 可以真正并行运行而不互相拖累。

5. **3.2 专家/节点的有效利用率**：虽然 DeepSeek-V3 实际只激活 8 个路由专家，但在通信策略下，它可以扩展到最多 13 个专家（4 节点 × 3.2 专家/节点），而不增加通信成本。这对未来的模型扩展留下了空间。

**给硬件厂商的需求清单**

DeepSeek-V3 技术报告第 3.5 节极为罕见地包含了"硬件设计建议"。这不是学术界常见的"我们的工作对未来硬件的启示"式展望——这是直接来自于在最前沿训练 MoE 的团队的实战需求：

1. **把通信从 SM 卸载出来**：20/132 个 SM 被用于通信，这些 SM 的 Tensor Core 完全空闲。"We aspire to see future vendors developing hardware that offloads these communication tasks from the valuable computation unit SM, serving as a GPU co-processor or a network co-processor like NVIDIA SHARP."

2. **统一 IB 和 NVLink 域**：从计算单元的角度看，应该只有一个统一的地址空间。通过简单的 read/write/multicast/reduce 原语就能操作整个 IB-NVLink 统一域。目前 SM 需要手动做 IB↔NVLink 转发的现状是巨大的效率损失。

3. **跨节点带宽仍然不够**：Even after all the optimization, the bottleneck is still inter-node bandwidth. DeepSeek explicitly recommends "developing higher bandwidth inter-node interconnects."

4. **更高的 FP8 GEMM 累加精度**、**Tile/Block 级量化支持**、**在线量化硬件支持**、**转置 GEMM 操作**——这些计算相关的建议都指向一个共同目标：减少数据移动，让 Tensor Core 以更高效率运行。

这是一个做了极致全栈优化——从 PTX 指令级到 pipeline 调度算法——把通信压榨到了物理极限的团队，然后他们对未来的硬件说：**这还不够。我们需要硬件本身发生改变。**

---

## 全栈视图：MoE 通信的六个技术层级

从 GShard 到 DeepSeek-V3，MoE 的通信挑战已经被充分暴露。解决这个挑战的努力分布在从芯片到数据中心的整个技术栈上——没有任何一个单点优化可以解决全部问题。这是整个系列博客的导航地图：

```
层级 0：芯片内部互联（On-Chip NoC）
       GPU 内部 SM 与 L2 分片之间的 Crossbar/层次化网络
       非均匀延迟（最高差 70%）、带宽分层、跨 GPC 分区代价
       代表研究：Uncovering Real GPU NoC Characteristics (MICRO'24)
       系列位置：第二篇

层级 1：Die-to-Die 互联
       多芯粒封装内的芯片间连接带宽
       NVLink-C2C（Grace↔Hopper, 900GB/s）、NV-HBI（Blackwell 双 die, 10TB/s）
       系列位置：第三篇

层级 2：节点内 GPU 间互联（Intra-Node Scale-Up）
       NVLink + NVSwitch 构成的 GPU 全互联交换网络
       8 卡 DGX H100 → 72 卡 NVL72 → 576 卡 NVL576
       网内计算（SHARP/NVLS）让交换机不只是转发
       系列位置：第三、四、五、六、七篇

层级 3：节点间集群互联（Inter-Node Scale-Out）
       InfiniBand NDR/XDR、RoCE v2、TPU ICI/Torus/OCS
       层次化 All-to-All 算法、容错路由、拓扑感知调度
       系列位置：第八、九篇

层级 4：通信调度与训练系统
       并行策略组合（TP+PP+EP+DP）、通信-计算重叠、训推分离优化
       DeepSpeed-MoE / Tutel / MegaBlocks / FlexMoE / Lina
       系列位置：第十、十一、十二篇

层级 5：全系统集成
       功耗墙、热节流、Scale-up vs Scale-out 的通信账
       冷却感知调度、并行策略的全系统 Profiling
       系列位置：第十三篇
```

每一个层级都有独立的技术挑战和解法，但关键的是它们之间的交互效应：

- 如果层级 0 的 NoC 延迟足够均匀，MoE token dispatch 的确定性就更好，层级 4 的调度算法可以做得更激进
- 如果层级 2 的 NVSwitch 拓扑是非阻塞全互联，层级 4 的层次化 All-to-All（如 Tutel 的 2DH）就可以把更多流量安全地推到节点内
- 如果层级 3 的 IB 上有网内计算（SHARP），层级 4 的调度系统就可以少操心 AllReduce 和 All-to-All 的竞争——它们在交换机上已经被处理了
- 如果层级 5 的冷却和功耗不受控制，即使层级 2-4 的通信优化完美，热节流会让部分 GPU 降频——所有 GPU 等待最慢的那个，通信优化的收益被功耗限制抹平

**这正是这个系列的价值所在：不是孤立地讲每个技术，而是把它们放在一起，让工程师看到全栈的交互效应——在哪个层级做优化会有最大收益，哪个层级已经接近物理极限。**

---

## 2024-2026 工业界 MoE 通信配置快照

| 模型/系统 | 年份 | 总参数 | 激活参数 | 专家配置 | 训练硬件 | 通信架构要点 |
|-----------|------|--------|---------|---------|---------|------------|
| GShard | 2020 | 600B | ~20B | Top-2, 2048 专家 | 2048 TPUv3 | 首次定义 EP，SPMD 分区，sub-linear scaling |
| Switch-C | 2022 | 1.6T | ~? | Top-1, 2048 专家 | TPUv3 | Top-1 路由减半通信，纯 EP 无 TP |
| GPT-4 | 2023 | ~1.8T? | ~280B? | 8× MoE（推测） | 未公开（推测 ~25K A100） | 首个顶级商用 MoE，推测 8 路 EP |
| Mixtral 8×7B | 2024 | 46.7B | 12.9B | Top-2, 8 专家 | 未公开 | 开源社区最广泛验证的小规模 MoE |
| DeepSeek-V2 | 2024 | 236B | 21B | Top-6, 160 路由+2 共享 | H800 集群 | MLA + 细粒度专家 + 辅助损失平衡 |
| DeepSeek-V3 | 2024 | 671B | 37B | Top-8+1 共享, 256 路由 | 2048 H800 | 1:1 计算-通信比，DualPipe，aux-loss-free |
| Llama 4 | 2025 | ~400B? | ~50B? | MoE（确认） | 未公开 | Meta 首次在主力模型用 MoE |
| Gemini 2.0 | 2025 | 未公开 | 未公开 | MoE（确认） | TPUv5p | Google 的 MoE + TPU + OCS 全栈优势 |

> 注：GPT-4 的 MoE 架构在 2023 年 6 月由 George Hotz、Soumith Chintala 等人公开透露（8×220B 专家，每次激活 2 个）。其他标注 "?" 的项目为基于公开信息的合理外推。

这个表的每一行，通信都是训练系统设计的核心考量。没有例外。

---

## 关键技术决策 Checklist

对于正在或准备部署 MoE 训练的工程师，在动手优化之前先回答这几个问题：

**1. 你的通信瓶颈在哪个层级？**

在目标集群上跑一个 profiled 训练步（哪怕是小规模），用 Nsight Systems + NCCL profiler + IB/RoCE 计数器确认各项占比。重点看：All-to-All dispatch/combine 各占 MoE 层时间的百分比，AllReduce（Dense 层）占非 MoE 层时间的百分比，以及它们是否有重叠或互相阻塞。DeepSeek-V3 的通信占比是 ~50%。如果你的比例显著更高（比如 70%+），通信是压倒性的首要矛盾，需要从通信消除（如 ExFlow）或通信隐藏（如 DualPipe）入手。如果比例更低（比如 20-30%），应该先优化计算效率。

**2. 你的计算-通信比是多少？**

这个比值直接决定了你能用 DualPipe 式的重叠策略做到什么程度。计算-通信比 ≈ 1:1 时，通过精巧调度可以完全隐藏通信；但如果通信占主导（比如推理中的 batch=1 极端场景），就需要走通信消除或调度优化的路线。对不同并行配置（EP=32 vs 64, PP=8 vs 16）都测一次。

**3. 你的节点内/节点间带宽梯度是多少？**

对于 H100：NVLink 900 GB/s vs IB 400 GB/s ≈ 2.25:1。对于 H800：NVLink 400 GB/s vs IB 200 GB/s ≈ 2:1（DeepSeek 报告的是 160 vs 50 ≈ 3.2:1，因为他们可能用的是 NDR200 的一半带宽）。这个梯度决定了层次化 All-to-All 能发挥多大作用。如果梯度很小（比如一些云端实例的 NVLink 和 IB 带宽接近），层次化 All-to-All 的收益有限。

**4. 你的专家数和激活专家数是否在通信成本可接受的范围内？**

Switch Transformer 告诉我们：增加专家数但保持激活专家数不变，计算量不增加，但通信量增加（更多设备参与 All-to-All）。DeepSeek-V3 选择 Top-8 激活 256 路由专家，是基于对通信成本的精算。如果你的模型只需要 8 个专家（如 Mixtral），通信成本天然就低得多——你需要权衡的是更少的专家是否能提供足够的模型容量。

**5. 你是否在并行策略中给通信留了专用 SM？**

DeepSeek-V3 分配了 20/132 个 SM 做通信——这些 SM 的 Tensor Core 是空闲的。如果你的通信占比高但 SM 利用率已经打满，考虑是否要像 DeepSeek 那样专门划分通信 SM，或者改用 NCCL 的 CE（Copy Engine）collective 模式（NCCL 2.28+），将通信卸载到 GPU Copy Engine 而非 SM。

---

## 下一篇预告

第二篇将深入到层级 0——GPU 内部 NoC。基于 KAIST/UBC/Microsoft 在 MICRO 2024 发表的实测研究，我们将看到：

- NVIDIA V100/A100/H100 三代 GPU 的片上网络架构实测——延迟为什么可以差 70%？
- L2 分片的物理分布如何影响 MoE token dispatch 的确定性？
- 跨 GPC 分区的通信代价是多少？MIG 和计算分区如何改变 NoC 的行为？
- 这些物理底层的非理想性如何倒逼上层的 kernel 设计和通信算法？

---

**本文引用的主要资料**：
1. Lepikhin et al., "GShard: Scaling Giant Models with Conditional Computation and Automatic Sharding", ICLR 2021. arXiv: 2006.16668
2. Fedus et al., "Switch Transformers: Scaling to Trillion Parameter Models with Simple and Efficient Sparsity", JMLR 2022. arXiv: 2101.03961
3. DeepSeek-AI, "DeepSeek-V3 Technical Report", arXiv: 2412.19437, 2024
4. Shazeer et al., "Outrageously Large Neural Networks: The Sparsely-Gated Mixture-of-Experts Layer", ICLR 2017. arXiv: 1701.06538
5. Roune, B. H., "Designing AI Chip Hardware and Software", 2026（个人技术指南）
