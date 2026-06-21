---
layout:     post
title:      SHARP In-Network Computing
subtitle:   SHARP In-Network Computing
date:       2026-06-21
author:     George Lin
header-img: img/post-bg-gen.png
catalog: true
tags:
- moe
---

## 引言：当交换机学会了"算数"

2021 年 6 月，英伟达在 MLPerf 训练基准测试的 BERT 榜单上提交了一个看似微妙的数据：用了某个"网络优化"的版本，BERT 训练吞吐比不用的情况高了 17%。在 MLPerf 的顶尖玩家圈里，1-2% 的分数差距就足以决定一场竞赛的输赢，17% 看起来像是换了一代 GPU。

但实际的硬件配置完全一样。相同的 HDR InfiniBand 交换机，相同的 A100 GPU，相同的 NCCL 库。唯一的区别是打开了一个叫 SHARP 的功能。

SHARP 是 Scalable Hierarchical Aggregation and Reduction Protocol 的缩写。名字很长，做起来却只需要一句话概括：**数据流经交换机时，就地在交换机内完成归约计算，而不是等数据到达目标 GPU 再由软件做**。

想象一个最普通的 AllReduce 操作。不用 SHARP 时，所有 GPU 互相发送自己的梯度——40 个 GPU 的 AllReduce 意味着每个 GPU 要向 39 个对等方发送数据，然后在自己的内存里对所有收到的梯度做加法。网络中流动的总数据量，是梯度本身的 39 倍。

用 SHARP 后，梯度在到达交换机时就被加起来了。假设树状拓扑，交换机内部有硬件加法器。梯度从 GPU 发出，经过第一级交换机时，所有来自这一级子树内的梯度被聚合为一份，继续向上传到父级交换机，再次聚合。最终到达根交换机时，只剩一个完整的求和结果。根交换机把这个结果沿着树广播下去，所有 GPU 拿到相同的累加梯度。网络中流动的总数据量砍掉了一半以上。

这就是网内计算（In-Network Computing）的本质：**在数据流经网络时对其进行计算，而非仅仅传输**。它把网络从被动管道变成了主动参与者。SHARP 是网内计算在工业界最重要的实现，而它的一套技术逻辑——从 InfiniBand 到 NVSwitch、从小消息到大消息、从单租户到多租户——完整展现了"把通信变计算"这条技术路线的所有关键决策。

2024 年末，DeepSeek 团队在 DeepSeek-V3 的硬件建议中明确点名了 SHARP："我们希望未来的硬件商开发类似 NVIDIA SHARP 的加速器，作为 GPU 协处理器或网络协处理器，将通信从昂贵的 SM 上卸载掉。"这个判断来自训练千亿参数 MoE 模型的第一手经验——在 14.8 万亿 token 的训练中，DeepSeek 团队被迫将 132 个 SM 中的 20 个（15%）专门用于通信处理，而这些 SM 上本可以跑 tensor core 计算。

一个功能的工业价值，不看它的宣传材料多漂亮，看那些最苛刻的用户是否会主动点名需要它。DeepSeek 点名了 SHARP。

这篇文章是网内计算的上篇，我们聚焦 SHARP 和 NVLink SHARP (NVLS) 的架构本身：它们是什么、怎么工作、每一代解决了什么问题、以及与 NCCL 和 MoE 训练的关系。下篇将讨论 DySHARP——如何把网内计算从静态集体操作推向 MoE 的动态、不规则通信。

---

## 系统剖析：SHARP 的四代演进

### 核心思想：在流经时归约，而非传递后归约

网内计算的思想起源可以追溯到 2010 年代早期学术圈对主动网络（active networks）和网内聚合（in-network aggregation）的探索。但将它们从学术原型变成工业产品的，是 SHARP 的原始论文 *Scalable Hierarchical Aggregation Protocol: A Hardware Architecture for Efficient Data Reduction*（2016 年发表于 COMHPC 研讨会）。论文提出了一套完整的硬件架构，将聚合逻辑植入 InfiniBand 交换机的 ASIC 内部。

为什么必须放进 ASIC？因为集体通信操作有两个特性让它天然适合硬件化：

1. **计算模式极其固定**：AllReduce 的求和、求积、求最大值/最小值——这些操作是 bit 级确定的，不需要分支预测、不需要指令译码、不需要访存层次。用通用处理器做这种操作，99% 的晶体管线都在干无关的事。一个简单的整数加法器或浮点加法器就够了。

2. **通信模式极其规律**：AllReduce 的参与者集合是静态的，从 task 启动到 task 结束不会变。这意味着交换机不需要查询动态路由表——所有目的地是预配置的。树的结构也是固定的。

这两个特性意味着，可以在交换机 ASIC 中以极低的面积和功耗开销（不到交换机芯片总面积的 1%）植入专用的聚合硬件，而获得按数量级计的性能提升。英伟达自己的估计是：**聚合逻辑的面积开销不到 0.1%**，换来的是某些场景下 5× 到 9× 的延迟改善。

但面积只是账本的一边。真正的问题在于：**你放在交换机的加法器应该支持什么精度？处理多大的消息？同时支持多少条流？** 每一代 SHARP 的架构演进，本质上都是这几个参数在不同时代的重新校准。

### SHARP v1：小消息归约的 EDR 时代

SHARP v1 随 EDR (100Gb/s) InfiniBand 交换机推出，目标是 HPC 领域的科学计算应用。在这一阶段，最大瓶颈是小消息 MPI 集合操作的延迟。

为什么是小消息？因为 HPC 应用的特征和 AI 训练截然不同。典型的 MPI 应用会在每个时间步的边界执行大量 Barrier 同步和 AllReduce（通常用于边界条件交换或物理量归一化），消息量小——往往只有几个整型或浮点数——但操作频率高、延迟敏感。一个 100 微秒的 Barrier 延迟在每秒迭代一次的科学计算应用中意味着 1% 的总效率损失。而 2048 节点上的 MPI Barrier 延迟可以轻松超过 1 毫秒。

SHARP v1 的硬件设计围绕这个场景展开：

- **支持的归约操作**：SUM、MIN、MAX、AND、OR、XOR。涵盖 MPI 标准定义的所有基本归约算子。
- **支持的数据类型**：Integer（16/32/64-bit）和浮点（FP16/FP32/FP64）。注意 FP64 的加入——这是 HPC 社区对精度的硬要求，AI 训练后来全切到 FP16/BF16 甚至 FP8，但气候模拟、CFD、分子动力学等 HPC 应用至今仍严重依赖 FP64。
- **最大 payload**：1KB。在 EDR 100Gb/s 时代，1KB 的消息在物理层只需要约 0.1 微秒的序列化延迟，但足以覆盖 HPC 场景中绝大多数归约消息（几个标量到几百个标量）。
- **并行度**：支持多个科学计算应用同时使用 SHARP，每个应用有自己的树结构。切换开销极低。

性能数据的说服力来自真实部署。俄亥俄州立大学的 MVAPICH2 团队（MVAPICH2 是 HPC 社区最主流的开源 MPI 实现之一）在 TACC（Texas Advanced Computing Center）的 Frontera 超级计算机上做了大规模实测，结果是 SHARP v1 在前沿应用中最重要两个操作的加速：

- **MPI AllReduce 5× 性能提升**：在数千节点的规模下（Frontera 是 TOP500 级别的超算），AllReduce 的延迟从毫秒级压缩到亚毫秒级。这 5× 的提升在小消息场景下特别显著，因为延迟在没优化的小消息 AllReduce 中由网络跳数和软件栈开销共同主导。
- **MPI Barrier 9× 性能提升**：Barrier 在传统实现中是最难优化的集体操作——它没有数据要归约，但需要所有参与者达到同步点。SHARP 通过在交换机硬件中端接 Barrier 请求来消除多轮同步往返，效果立竿见影。

这些数字背后是一个关键的架构决策：**让交换机处理控制面，而不只是数据面**。传统的 Barrier 依赖软件层的一轮轮 ACK 消息，每一轮都需要从 NIC 到 CPU 的完整中断和上下文切换。SHARP 把 Barrier 状态机放在交换机内——交换机收到所有叶子节点的到达信号后，只生成一条完成信号广播回去。控制和数据融合在一起，消除了控制面的独立开销。

MVAPICH2 的性能结果发表在 *Scalable MPI Collectives using SHARP: Large Scale Performance Evaluation on the TACC Frontera System* 中，这成为 HPC 社区广泛引用的基准。一个关键细节是：这些测试是在真实的超算生产环境中进行的，有真实的工作负载干扰，不是隔离出来的实验室数据。

### SHARP v2：大消息流式聚合的 HDR 时代

SHARP v1 为 HPC 而生。但当 AI 训练在 2018-2020 年爆炸式增长后，出现了一个 v1 完全无法处理的场景：大梯度 AllReduce。

在 ResNet-50 或 BERT 这样的模型中，每步训练的梯度聚合涉及每层权重矩阵的全部元素。BERT Large 有约 340M 参数，每个参数按 FP16 算是 2 字节，总梯度约 680MB。这个数字随着模型规模线性增长：GPT-3 有 175B 参数（梯度 350GB），一个 AllReduce 操作要移动 350GB 数据。

SHARP v1 的最大 payload 是 1KB。差了 8 个数量级。

这就是 SHARP v2 的核心创新：**Streaming Aggregation Tree (SAT)**，一种为大消息专门设计的流式聚合协议。

要理解 SAT，需要理解大消息 AllReduce 和小消息 AllReduce 的根本区别。小消息用树（tree-based），大消息用环（ring-based）或递归加倍（recursive doubling）——这是分布式计算中的"常识"。原因是：

- **小消息**：延迟主导。树的跳数少（log_P），延迟低。消息长度小到带宽利用率不是问题。
- **大消息**：带宽主导。环的带宽利用率高（链路上的所有流量都是有效数据负载），而树在根附近会产生瓶颈——非叶子节点同时接收和发送，有效带宽只有链路带宽的一半。这就是许多人熟悉的"halving-doubling 两阶段"（先递归减半归约，再递归加倍广播），也是经典 AllReduce 算法的标准套路。

但 SHARP v2 的 SAT 翻了这个常识：**它用树做带宽优化的大消息聚合，但不牺牲吞吐**。怎么做到的？

核心在于 SAT 的"流式"特性。非 SHARP 的树状 AllReduce 之所以带宽受限，是因为每个中间节点必须**等所有子节点的数据到齐**后才开始归约，然后才能向父节点发送。这叫"store-and-forward"——存完再发，引入的延迟和内存压力随消息大小线性增长。

SAT 的工作方式完全不同。收到第一个子节点的第一个数据 chunk——甚至 chunk 还可以更细地切分——交换机就把这个 chunk 的归约开始执行。它不需要等所有子节点都发完。归约完成后立即向父节点转发。整个聚合是流式的：数据在流过交换机的同时被归约，不堆积、不等待。这与 CPU 上 SIMD 运算中的"流水线"概念异曲同工。

具体流程（以 3 层树为例，叶子节点发送 FP16 梯度 tensor T）：

1. **叶子 GPU（第 0 层）**：将 T 切分为 bundle（例如每个 bundle = 32KB）。将第一个 bundle 发送给父级交换机。
2. **第 1 级交换机**：接收到两个子叶的 bundle_1。SAT 逻辑对两个 bundle_1 做 elem-wise SUM（FP16+FP16）。得到的归约结果 bundle_1' 立即向上发送给父交换机（第 2 级）。
3. **流水线叠加**：在第 1 级交换机正在发送 bundle_1' 的同时，它已经开始接收子叶的 bundle_2。接收、归约、发送三个操作在 pipeline 上同时进行。
4. **第 2 级（根）交换机**：收到两个子交换机的 bundle_1' 和 bundle_1''，归约得到最终 bundle_1_final。根交换机同时开始向叶子广播归约结果。
5. 整个过程，每个链路的双向带宽都被用满（接收 + 发送同时进行），没有"等待所有数据"的空闲期。

SAT 的效果在 MLPerf 的公开数据中得到了定量验证。英伟达在 2021 年 6 月的 MLPerf v1.0 训练基准测试中使用 SHARP v2，在 BERT 上展示了 17% 更高的训练吞吐。Michael Houston（英伟达 AI 系统副总裁兼首席架构师）在 UC Berkeley 的机器学习系统课程中公开讲解了这个数据，他展示的核心因果链是：

**AllReduce 带宽 2× 提升 → BERT 训练吞吐 17% 提升**

2× 带宽提升来自 SAT 消除了传统 AllReduce 中的"数据往返次数膨胀"——流式聚合让网络中流动的总字节量接近理论下界（接近梯度大小本身），而不是原始 MPI AllReduce 的 O(P) 倍。

SHARP v2 的其他关键支持：

- **GPUDirect RDMA 集成**：GPU 梯度的地址由 NCCL 注册到 SHARP，SHARP 交换机可以直接从 GPU HBM 读取梯度进行聚合，跳过 CPU 内存。这意味着没有 GPU→CPU→NIC 的两次拷贝，只有 GPU→NIC 一次。在大梯度场景中，这一步省下的是内存带宽和 PCIe 带宽——两者都是极度稀缺的资源。
- **单工作负载**：SHARP v2 一次只支持一个 AI 训练任务使用 SAT。这对单一训练集群完全够用——一个 8 节点、64 GPU 的 DGX SuperPOD 通常只跑一份训练作业。

### SHARP v3：多租户的 NDR 时代

到了 H100 + Quantum-2 NDR InfiniBand (400Gb/s) 时代，AI 基础设施的格局发生了关键变化：云厂商开始大规模采购 H100，将 NDR 交换机部署在共享集群中。

共享集群意味着多个 AI 训练任务可能同时跑在不同的节点组上，每个都希望使用 SHARP。如果说 SHARP v2 的"单工作负载"限制在私有 DGX 集群上不是问题，那在 Azure/AWS/GCP 的 NDR IB 集群上就是致命缺陷。

SHARP v3 的核心创新是**多租户网内计算**：在一个交换机上，多个 AI 工作负载可以并行使用 SHARP，彼此不干扰。

这个设计的挑战在于硬件隔离。每个 VTEP（Virtual Tunnel Endpoint）或 IB 分区（partition）内的 SHARP 树之间不能互相影响——既不能泄漏数据，也不能产生资源争用。交换机的 SAT 聚合逻辑需要支持多个并发的聚合树，每个有自己的缓冲区、自己的流控状态、自己的完成追踪。这需要把聚合缓冲区从"一个大的全局缓冲"改成"多个小的按租户隔离的缓冲"。每个租户的聚合流有自己的信用（credit）机制，不会因为其他租户的拥塞而饥饿。

SHARP v3 的性能数据来自 Microsoft Azure 的公开演讲。Jithin Jose（Azure 首席软件工程师）在 GTC 2024 的 *Transforming Clouds to Cloud-Native Supercomputing* 演讲中展示了 Azure 的 H100 + Quantum-2 NDR IB 集群使用 SHARP v3 的实测结果。他放出的 AllReduce 延迟图是整个 v3 最直观的性能证据：标有 "With SHARPv3" 的红色曲线，表明**AllReduce 延迟比不用 SHARP 改善了接近一个数量级**。

注意这里的关键是"延迟"而非"带宽"。在 H100+NDR 的配置下，SHARP v3 对小消息 AllReduce 的延迟改善尤其显著——因为消息越小，端到端延迟中的软件栈开销（NCCL 的 ring/channel 启动延迟、barrier 同步开销）占比越高，硬件聚合消除的正是这部分开销。Azure 的数据验证了云环境下的 SHARP 价值——即使在共享网络上运行，只要隔离设计到位，网内计算的性能提升依然显著。

### SHARP v4：XDR 与新算法

SHARP v4 随 Quantum-X800 XDR (800Gb/s) InfiniBand 交换机发布，引入了"新算法以支持更广泛种类的集体通信"。英伟达没有在公开博客中详细展开新算法的具体内容（截至 2024 年 10 月发布时这种保守是正常的——XDR 产品刚到早期部署阶段），但从他们明确提到的方向来看，v4 的核心扩展有两个：

1. **更多的集体通信原语**：传统 SHARP 支持的是 AllReduce、Broadcast、Barrier、Reduce。但 AI 训练的新趋势引入了更多样化的通信模式——如 AllGather、Reduce-Scatter、All-to-All 的变种。实际部署中，MoE 训练带来的 All-to-All 占比越来越高（在 DeepSeek-V3 中通信占 MoE 层 70.4% 的时间），传统 SHARP 无法直接加速 All-to-All，但它可以加速 MoE 之外的 AllReduce——这释放的带宽间接使 All-to-All 受益。

2. **更灵活的聚合拓扑**：NDR 时代的聚合树是静态预配置的——任务启动时建树，任务结束时销毁。XDR 时代可能会支持动态重配置聚合树——根据流量模式在运行时调整聚合策略。这对 MoE 特别有意义：专家并行组内的 AllReduce 发生在相同 EP group 内的 GPU 之间，这些 GPU 在物理拓扑中可能分布在多个交换机的不同端口上。动态调整聚合树可以根据 EP 分组来优化物理跳数。

### SHARP 世代演进一览

| 世代 | 交换机平台 | 带宽 | 关键创新 | 核心场景 | 代表性能 |
|------|----------|------|---------|---------|---------|
| v1 | EDR (Quantum) | 100Gb/s | 小消息归约、硬件 Barrier | HPC 科学计算 | 5× AllReduce, 9× Barrier |
| v2 | HDR (Quantum) | 200Gb/s | SAT 流式聚合、GPUDirect RDMA | AI 训练大消息 | 2× AllReduce 带宽, 17% BERT 加速 |
| v3 | NDR (Quantum-2) | 400Gb/s | 多租户并行、硬件隔离 | 云 AI 集群 | ~10× AllReduce 延迟改善 |
| v4 | XDR (Quantum-X800) | 800Gb/s | 更广泛集体通信、动态拓扑 | 下一代 AI 训练 | 待公开 |

---

## 关键技术决策：SHARP 的五个架构取舍

### 决策一：LLT vs SAT —— 两种协议什么时候用

SHARP 内部不是一种算法走天下。它有两套完全独立的协议引擎：LLT（Low Latency Tree，低延迟树）和 SAT（Streaming Aggregation Tree，流式聚合树）。两者的切换完全由消息大小决定。

**LLT（低延迟树）用于小消息**。它的设计哲学是：**减少跳数 > 提高带宽利用率**。

- 聚合操作在每一跳都"存储-转发"：中间节点接收完整的多数据集，执行归约，再发给父节点。对于 1KB 以下的消息，存储和完成归约的时间可以忽略，但多的每一跳都有端到端延迟惩罚。LLT 选择浅树——比如在大规模集群中用 2-3 层的树而非 4-5 层，减少跳数换取延迟。
- 控制面开销被显著减少：SHARP v1 的 Barrier 在 LLT 模式下由交换机的控制逻辑直接在硬件中完成（即前面提到的状态机），每条消息都不需要穿过软件栈。

**SAT（流式聚合树）用于大消息**。它的设计哲学是：**让网络带宽而非延迟成为瓶颈，维持链路的持续满负荷**。

- 流式处理消除了"等待所有子节点"的延迟。第一条数据到达即开始归约和转发。
- 聚合在 chunk 级而非消息级完成，树可以有更多层而不会导致根节点带宽崩塌——因为根节点永远在转发而非等待。
- 对于 MoE 训练的 AllReduce（专家梯度可能动辄几百 MB 到几 GB），SAT 是唯一可行的 SHARP 引擎。

**切换逻辑**：消息大小超过某个阈值（由 NIC 的 transport 层根据 MTU、RTT、拓扑深度等参数计算）时从 LLT 切到 SAT。在典型配置中，这个边界大概在几 KB——具体数字和交换机内部缓冲区大小也有关。

这个双协议设计说明了一个底层道理：**网内计算无法用一种算法涵盖所有通信场景**。即使是同一种集体操作（AllReduce），消息大小的变化就会改变最优策略。更不用说像 All-to-All 这种完全不同结构的操作了。

### 决策二：包格式 —— 网内计算的数据面设计

网内计算区别于传统包转发的根本不同在于：**交换机需要理解包的数据语义**，而不仅仅是包头。

传统 InfiniBand 交换机的数据面只关心两件事：路由（这个包去哪个输出端口？）和流控（发送方是否还有信用？）。包负载对交换机完全不透明——它只是按字节存储和转发。但 SHARP 需要交换机**对包负载执行算术操作**，这意味着负载格式必须被交换机的聚合逻辑所理解。

SHARP 的包格式设计有几个微妙之处：

1. **聚合操作符编码在包头**：每个需要聚合的包在 IB 头中携带一个 SHARP 操作符字段（4-8 bits），指示交换机应对此包执行什么归约操作——SUM、MIN、MAX 等。同一个 SHARP 树可以混合不同的操作符号（虽然这在实践中很少见）。

2. **数据类型编码**：负载的每个元素的数据类型（INT32、FP16、FP32、FP64）也在 SHARP 扩展头中编码。交换机需要知道数据类型才能正确执行加法器的数据路径布局——FP16 加法器和 INT32 加法器是完全不同的硬件 block。

3. **payload 对齐**：为了硬件实现的简洁，SHARP 要求聚合 payload 以 16 字节对齐（一个 IB 的 4 字）。如果模型梯度的大小不是 16 字节的整数倍（在 FP16 的矩阵乘法结果中几乎总是），NCCL 需要在软件层做 padding。

4. **chunk 级流控和完成追踪**：SAT 模式下，每个 chunk 在 SHARP 树中独立流控。当某个交换机接收的 chunk 因为下游拥塞而无法转发时，该交换机对上游暂停针对该 chunk 的信用——但仅该 chunk，其他 chunk 流不受影响。这就是为什么 SAT 的带宽利用率高——一个慢 chunk 不会阻塞整个消息。

这里的关键工程判断是**粒度的选择**。chunk 太大 → 流控粒度粗，排队延迟高。chunk 太小 → 每个 chunk 的头部开销占比大，有效带宽下降。SHARP 的 chunk 大小是内部可调的（根据交换机缓冲区配置），通常和 MTU 有密切关系。

### 决策三：NCCL 集成 —— 用户缓冲区注册消除拷贝

仅有机子层面的 SHARP 硬件是不够的。网内计算需要和上层软件生态深度对接。NCCL（NVIDIA Collective Communication Library）是这一对接的关键界面。

NCCL 和 SHARP 的集成经历了两个阶段。第一阶段（2017-2020 年间，SHARP v1）主要是"透明启用"——NCCL 检测到硬件支持 SHARP 后自动选择 SHARP 作为 AllReduce 的 collective 后端。对用户几乎全透明，只需要在 nccl.conf 中置一个环境变量。

但透明模式有一个性能瓶颈：**数据拷贝**。在透明模式下，NCCL 要将 GPU 的梯度数据从 NCCL 的用户缓冲区拷贝到注册的 SHARP 通信缓冲区（这些缓冲区是 IB HCA 管理的 RDMA 区域），做完 AllReduce 后再拷回来。对于小消息问题不大，对于大消息（BERT 的 680MB 梯度、GPT-3 的 350GB 梯度）——两个拷贝就是双倍的 GPU HBM → CPU PCIe → NIC 的数据移动。

2021 年，NCCL 团队引入了 **用户缓冲区注册（user buffer registration）** 功能，解决了这个问题。核心思路是：让 NCCL 使用指向 GPU 内存的指针直接和 SHARP 交互，不需要任何中间拷贝。

具体流程：

1. **预注册**：训练启动时，NCCL 将 PyTorch/JAX 的梯度 tensor 所在的 GPU 内存区域注册到 SHARP。注册建立了一个 NVLink/PCIe 地址和 IB 地址之间的映射——让 IB HCA 可以通过 GPUDirect RDMA 直接读写 GPU HBM。
2. **零拷贝 AllReduce**：运行时，NCCL 调用 SHARP AllReduce 时，传给 SHARP 的不是数据副本的地址，而是梯度本身的 GPU 内存指针。SHARP 交换机从 GPU HBM 读取梯度（通过 IB HCA + NVLink），在网内聚合，直接将结果写回 GPU HBM——全程没有任何 CPU 内存的参与。
3. **异步完成**：NCCL 将 SHARP AllReduce 作为一个 CUDA stream 上的操作发起。GPU 可以在 AllReduce 进行时继续执行其他 CUDA kernel（前提是这些 kernel 不访问正在聚合的梯度）。这是 computation-communication overlap 的必要前提。

用户缓冲区注册带来的收益不仅是一两次内存拷贝。它消除了 CPU 瓶颈——在大规模训练中，CPU 内存带宽（一般是 200-300GB/s）远低于 GPU HBM 带宽（H800 有 3.35TB/s）。如果每次 AllReduce 都经过 CPU 内存，CPU 可以成为比网络更早的瓶颈。

英伟达的博客提到"某大型服务提供商使用 SHARP 在其内部 AI 工作负载中获得 10-20% 的性能提升"。虽然未点名，但从描述（"大型服务提供商"）和其硬件环境（NDR IB + H100）来看，几乎可以确定是在指微软 Azure 或类似规模的云提供商。

### 决策四：NVLS —— 把网内计算带入 NVSwitch 域

SHARP 的一个重大局限是它只工作在 InfiniBand 交换机上。多 GPU 节点内部的 NVSwitch 域，AllReduce 仍然走传统的 NVLink 路径——GPU 之间直接通过 NVSwitch 进行数据交换，由 GPU 上的 SM 执行归约计算。

NVLink SHARP（NVLS）改变了这一点。NVLS 将网内计算的逻辑嵌入 NVSwitch ASIC 内部，让节点内的集体操作也能在交换机内完成聚合。

NVLS 首次出现在 Hopper 架构（2022 年）。它的核心硬件组件是 NVSwitch 内部的 **multimem 单元**——一套轻量级的共享内存语义的网内计算引擎。与 IB SHARP 不同，NVLS 操作在共享内存域上，包的内容不是 RDMA 消息，而是直接的 load/store 指令。这使它更快（延迟更低——NVSwitch 内的往返在 1 微秒以内），但也更不灵活（只能服务一个 NVSwitch 域内的 GPU，最多 8 个在标准 DGX 配置，或 72 个在 NVL72 配置）。

NVLS 提供两个核心 primitives：

- **multimem.st**：加速 AllGather。源 GPU 发送一个写请求，NVSwitch 将该请求复制（multicast）到目标组中的每一个 GPU。一个写操作 → N 个目标 GPU 同时收到数据。在标准 AllGather（无 NVLS）中，源 GPU 需要向每一个目标 GPU 分别发送一个写请求，总带宽消耗是数据量的 (N-1) 倍。NVLS 把它压缩到 1 倍——一个写，交换机组播。

- **multimem.ld_reduce**：加速 Reduce-Scatter。源 GPU 发起一个读请求，NVSwitch 从目标组中的每一个 GPU 读取数据，在交换机内做规约，然后只将归约后的结果返回给源 GPU。在标准 Reduce-Scatter 中，目标 GPU 各自把数据发给源 GPU，源 GPU 逐个做归约——数据流量同样是 (N-1) 倍。NVLS 把"逐 GPU 发送+逐 GPU 归约"变成了"交换机并发读取+交换机归约"，流量降到 1 倍。

NVLS 的设计引入了一个新的架构概念：**共享内存多播地址（multimem address）**。这不是传统意义上的物理内存地址——它是一个逻辑标识，代表"在这个集体操作中，所有 GPU 上被分配为相同代数索引的数据块"。当 GPU 对 multimem 地址执行 multimem.st 时，NVSwitch 知道该将数据写入哪些目标 GPU（以及在各目标 GPU 上写入哪个偏移量），因为目标集在树建立时已经预配置好了。

这个设计让它极快——每个包不需要携带 N 个目标地址，只需要一个 multimem 地址——但也意味着它极不灵活。**NVLS 只能支持静态集体操作**——目标集固定的 AllGather、Reduce-Scatter、AllReduce（可以组合出来）。对于 MoE 的 Dispatch 和 Combine，目标集在运行时变化（token 去哪个专家取决于路由网络的输出，这个输出每个 batch 都不同），NVLS 无法直接使用。

这就揭示了 NVLS 和 IB SHARP 的功能分工：

- **IB SHARP**：加速跨节点的集体通信（AllReduce 是最主要的应用）。处理粗粒度的大消息聚合。使用 SAT 流式协议解决大消息带宽问题。多租户支持使其适合云环境。
- **NVLS**：加速节点内 NVSwitch 域的集体通信。处理细粒度的共享内存操作（load/store 的组播和归约）。低延迟（sub-microsecond 单跳）。但受限于静态集体操作。

两者组合形成两层次结构：**节点内** NVLS 做 GPU-group 内的聚合归约，**节点间** IB SHARP 做跨节点的大规模归约。对 MoE 训练来说，这提供了一个强力的组合——非 MoE 的 AllReduce（如注意力层的梯度同步）被双层加速，释放更多节点间/节点内带宽给 MoE 专属的 All-to-All 通信。

### 决策五：SHARP 对 MoE 的影响 —— 为什么释放带宽就是加速 All-to-All

SHARP 不能直接加速 MoE 的 All-to-All（截至 SHARP v3，不包括还处于学术阶段的 DySHARP 等工作）。但它通过**间接方式**显著影响 MoE 的训练效率。

MoE 训练的通信不是只有 All-to-All。在一个 MoE 层前向的任意时刻，网络上传送着至少三种不同的通信流：

1. **All-to-All dispatch（MoE 层前向）**：token 从源 GPU 发往目标 GPU。
2. **All-to-All combine（MoE 层前向）**：专家输出从目标 GPU 传回源 GPU。
3. **AllReduce（非 MoE 层的前向和反向）**：数据并行组内梯度同步，或者张量并行组内的激活同步。

第三种通信——AllReduce——通常在 MoE 训练中不单独构成瓶颈，但它在**同一张网络卡上**与 All-to-All 竞争带宽。如果 AllReduce 使用了 40% 的 IB 带宽，那留给 All-to-All 的就只剩 60%。对于 All-to-All 主导的训练来说，AllReduce 不是瓶颈，**但它的带宽占用挤占了 All-to-All 的可用带宽**。

SHARP 的效果是：它将所有 AllReduce 的网络数据量砍掉一半甚至更多。这直接释放了 IB 带宽。在 DeepSeek-V3 的训练配置中（16-way PP、64-way EP、跨 8 个节点），每步训练在非 MoE 层上有显著的 AllReduce 流量。如果能通过 SHARP 将所有 AllReduce 的带宽需求减半，那释放出来的带宽等同于将 All-to-All 的有效可用带宽提升 20-30%——取决于 AllReduce 和 All-to-All 的占比。

这就是为什么"SHARP 让 BERT 训练吞吐提升 17%"的数据点对 MoE 训练如此重要。在 Dense BERT 中，17% 的提升来自 AllReduce 本身的加速。在 MoE 中，**同样的 AllReduce 加速带来的是 All-to-All 可用带宽的增加**——因为 AllReduce 不再独占那么大的份额。实际增益取决于各层的通信占比，但在 All-to-All 严重的配置（如 DeepSeek-V3 的 64-way EP 跨 8 节点），SHARP 的间接收益可能和它在 Dense 训练中的直接收益一样显著。

DeepSeek-V3 团队在 Section 3.5 的建议中对此做了最明确的背书。他们在 H800 集群上训练 V3 时，缺少 SHARP（H800 的 InfiniBand EDR 交换机和 H200 集群的 Quantum-2 NDR 交换机有代差）。因此他们不得不将 132 个 SM 中的 20 个专用于All-to-All 通信处理。这 20 个 SM 上没有任何 tensor core 计算——它们每时每刻都在搬运数据、管理内存布局、执行归约操作。DeepSeek 团队的原话是对 SHARP 类硬件最清晰的期待：

> "We aspire to see future vendors developing hardware that offloads these communication tasks from the valuable computation unit SM, serving as a GPU co-processor or a network co-processor like NVIDIA SHARP."

这段话点出了 SHARP 在 MoE 时代最本质的价值——不只是让网络更快，更是**把 GPU 的宝贵 SM 从通信杂务中解放出来**。20 个 SM 的 15% 算力释放，乘以 2048 个 GPU，再乘以几个月的训练时间——这笔账算下来非常可观。

---

## 横向对比：SHARP 与网内计算的相关工作

### IB SHARP vs NVLS：两张网的分工

| 维度 | IB SHARP (v3/v4) | NVLink SHARP (NVLS) |
|------|------------------|---------------------|
| **工作域** | 节点间 InfiniBand 网络 | 节点内 NVSwitch 域 |
| **扩展规模** | 数千节点 | 8-72 GPU（NVL72） |
| **通信模型** | 消息传递（RDMA） | 共享内存（load/store） |
| **协议引擎** | LLT + SAT | Multimem 寻址 |
| **延迟** | 微秒级（IB 往返） | 亚微秒级（NVSwitch 单跳） |
| **支持的集体操作** | AllReduce, Broadcast, Reduce, Barrier | AllGather (multimem.st), Reduce-Scatter (multimem.ld_reduce) |
| **多租户** | 支持（v3+） | 单工作负载 |
| **对 MoE Dispatch** | 不支持 | 不支持（仅静态） |
| **对 MoE Combine** | 不支持 | 不支持（仅静态） |
| **对 MoE AllReduce** | 直接加速（5-10× 延迟改善） | 直接加速节点内部分 |
| **集成层** | NCCL + IB verbs | CUDA PTX + multimem ISA |

### 网内计算在学术界的其他方向

SHARP 和 NVLS 是工业界的网内计算旗舰实现，但学术界在过去五年贡献了丰富的探索：

- **SwitchML/ATP (NSDI'21)**：利用可编程交换机（P4）实现网内聚合。不同于 SHARP 的固定硬件聚合，SwitchML 使用交换机上的可编程 pipeline 动态配置聚合函数。灵活性高，但受限于 P4 交换机的有限算力和内存，聚合吞吐远低于 SHARP。

- **Flare (SC'21)**：灵活的网内 AllReduce 系统。允许 AllReduce 的不同归约操作按照各异步粒度和调度策略运行。与 SHARP 互补——SHARP 提供硬件速度，Flare 提供调度灵活度。

- **CAIS (HPCA'26)**：专注于 GPU 多芯片系统上的计算感知网内计算。在 NVSwitch 内部加入更多计算能力以支持张量并行中的 partial 矩阵聚合。是 NVLS 的学术型进化版。

- **TRACI (ISCA'25)**：将网内计算扩展到推荐模型（DLRM）的输入动态通信。和 DySHARP 相似的问题动机（动态通信模式超出静态网内计算能力），但面向 DLRM 的 embedding lookup 通信模式。

这些工作共同推动网内计算从"加速静态集体操作"走向"处理动态应用层通信"，后者是 MoE 时代的核心诉求——也是下一篇 DySHARP 的主题。

### NCCL+SHARP vs 标准 NCCL AllReduce：实际性能差异

让我们把这个对比具象化。在一个典型的 8 GPU DGX A100 节点上，在 InfiniBand HDR 网络中，对 100MB 的梯度做 AllReduce：

| 配置 | AllReduce 耗时 | 网络总流量 | GPU 参与度 |
|------|--------------|-----------|-----------|
| 标准 NCCL Ring AllReduce | ~2.1ms | ~700MB（7×100MB） | GPU SM 做加法 |
| NCCL + IB SHARP SAT | ~1.0ms | ~200MB（2×100MB） | 交换机做加法 |
| 加速比 | 2.1× | 3.5× 流量减少 | — |

对于 1MB 的小梯度（常见于小模型或专家梯度）：

| 配置 | AllReduce 耗时 | 瓶颈 |
|------|--------------|------|
| 标准 NCCL Ring | ~35μs | 延迟主导（ring 有 7 跳） |
| NCCL + IB SHARP LLT | ~7μs | 树状（log_2(8)=3 跳） |
| 加速比 | 5× | — |

小消息场景的加速显著更高——因为传统 AllReduce 在大量小消息时的软件栈开销（NCCL kernel launch、barrier sync、多跳延迟）被 SHARP 的硬件融合消除了。

---

## 总结与清单：SHARP 网内计算的核心认知

### 核心要点

1. **网内计算 = 数据流经时归约**。SHARP 把 AllReduce、Broadcast、Barrier 等集体操作的交换逻辑植入交换机 ASIC，将网络从被动管道变成主动计算参与者。面积开销不到交换机芯片 0.1%，性能提升可达 5-9×（取决于操作和消息大小）。

2. **两套协议引擎应对两种场景**。LLT（低延迟树）针对小消息优化延迟——少跳数、硬件 Barrier 状态机。SAT（流式聚合树）针对大消息优化带宽——流式处理、逐 chunk 流水线、消除 store-and-forward 延迟。

3. **四代演进各有主题**。v1 解决 HPC 小消息 → v2 解决 AI 大消息（SAT + GPUDirect RDMA）→ v3 解决云环境多租户 → v4 扩展到更广泛的集体操作类型。

4. **NVLS 是网内计算的 NVSwitch 版本**。用 multimem 寻址在 NVSwitch 域内执行 AllGather 和 Reduce-Scatter 的硬件加速。比 IB SHARP 更快（亚微秒延迟），但局限于静态集体操作和节点内范围。

5. **IB SHARP + NVLS = 双层网内计算**。节点内 NVLS 归约 + 节点间 IB SHARP 归约，将非 MoE 层的 AllReduce 流量压缩到接近下限，释放出来的带宽直接回馈给 MoE 的 All-to-All。

6. **DeepSeek-V3 的背书是对 SHARP 价值最好的度量**。在实际训练 671B 参数的 MoE 模型中，缺乏 SHARP 导致需要将 15% 的 GPU SM 专用于通信搬运。SHARP 的价值不只是加速通信——它释放的是宝贵的 GPU 计算单元。

### 工程师决策清单

- [ ] **确认集群的 IB 交换机代际**——只有 EDR (SHARP v1)、HDR (v2)、NDR (v3)、XDR (v4) 才支持 SHARP。做硬件选型时优先考虑 NDR 或更高。
- [ ] **检查 NCCL 版本是否支持用户缓冲区注册**——NCCL 2.9+ 支持，这是零拷贝 SHARP AllReduce 的前提。若 NCCL 过旧，再好的 SHARP 硬件也会因数据拷贝而损失效果。
- [ ] **对于 MoE 训练，将 AllReduce-intensive 的非 MoE 层（如注意力、LayerNorm 的 DP 同步）配置使用 SHARP**——即使 SHARP 不能直接加速 All-to-All，释放的网络带宽也会间接加速 MoE 通信。
- [ ] **如果 AI 集群是多租户共享的（云环境），SHARP v3 的多租户隔离是关键特性**——v2 单工作负载限制在共享集群中几乎是不可用的。
- [ ] **对于小规模节点内训练（1-8 GPU 单节点），考虑 NVLS**——它在 NVSwitch 域内的延迟优势对张量并行的 AllReduce 特别有效。但 MoE 的 Dispatch/Combine 暂时无法用 NVLS 加速——这是下一篇 DySHARP 要解决的问题。

---

*下一篇：[第六篇：网内计算（下）—— DySHARP 与 MoE 动态网内计算]。我们将深入 DySHARP 的全栈设计——如何通过动态 multimem 寻址和 token-centric kernel fusion，将 MoE 的动态通信也推入交换机的加速路径。*
