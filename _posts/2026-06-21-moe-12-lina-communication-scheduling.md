---
layout:     post
title:      Lina Communication Scheduling
subtitle:   Lina Communication Scheduling
date:       2026-06-21
author:     George Lin
header-img: img/post-bg-gen.png
catalog: true
tags:
- moe
---

## 引言：同一种通信操作，两个截然不同的瓶颈

如果你在 2023 年问一个做 MoE 系统工程的人"All-to-All 到底慢在哪"，你大概率会得到模棱两可的回答——"带宽不够吧"、"卡太多互相等"、"不均衡"——然后就没有然后了。这些回答不是全错，但它们把原因和结果搅在了一起，导致你在做优化时无从下手。

Hong Kong 的几位研究者——以 CityU 的 Jiamin Li 为首，联合 ByteDance 的 Yimin Jiang——在 2023 年做了一件事：他们把分布式 MoE 训练和推理中的 All-to-All 通信分别拆开来看，找出了两种场景下 All-to-All 成为瓶颈的**不同根因**。这件事看似简单，但实际上是一个重要的认知拐点——在此之前，没人真正系统化地论证过训练和推理的 All-to-All 问题是本质不同的。

他们的答案很清晰：

**训练的 All-to-All 慢，是因为它在反向传播中和 AllReduce 抢带宽。** 在混合并行（EP + DP）下，MoE 层的 All-to-All 和 Dense 层的 AllReduce 同时运行在两条独立的 CUDA stream 上，两者对 NIC 的发送队列进行无协调的公平共享。AllReduce 是不可或缺的——没有它就无法聚合 Dense 层的梯度——但因为 AllReduce 消息更小、更频繁，它在带宽公平分配下吃掉了一半以上的带宽。All-to-All 数据量大（每个 token 的 hidden state 都要跨 GPU），被这种竞争直接拉长——中位数 1.83x 的 slowdown，最差 4.14x。

**推理的 All-to-All 慢，是负载不均导致的。** 推理中没有 AllReduce，所以不存在带宽竞争。但专家热度分布是极端偏斜的——最热专家收到的 token 是最冷专家的 4-5.56 倍。热门专家所在的 GPU 的链路被塞满，冷门专家的链路几乎闲置。这种不均衡有两个恶果：(1) 热门专家的计算量大，其他 GPU 要等它算完才能发起 All-to-All；(2) All-to-All 中不同链路的传输量不同，总完成时间由最慢的链路决定。

这两个问题的本质完全不一样：一个是**资源竞争**问题（训练），一个是**负载分配**问题（推理）。Lina 系统（"丽娜"或"李娜"，作者 Jiamin Li 的名字）的价值不在于它发明了某种全新的通信原语，而在于它把这两个问题的边界画得清清楚楚，并用针对性的机制分别解决。

在这篇博客里，我们详细拆解 Lina 对训练和推理的两种调度方案，然后追问一个更根本的问题：**为什么必须分离？合在一起会怎样？** 最后，我们提炼出可以泛化到其他通信优化场景的一般性原则。

---

## 系统剖析（上）：训练的优先级调度 —— 用微操作化解带宽竞争

### 问题形态：反向传播中的双流碰撞

要理解 Lina 在训练侧的设计，必须先把反向传播（backward pass）的通信形态搞清楚。

在一个标准的分布式 MoE 训练中（数据并行 + 专家并行混合），反向传播阶段每一层产生两种通信需求：

**MoE 层的通信：**
- All-to-All 需要做两次：一次是梯度从 expert 端传回 token 原始端（combine backward），一次是梯度从 token 端传向 expert 端（dispatch backward）。
- 这些 All-to-All 是由专家并行（Expert Parallelism）进程组管理的，跑在一条专用的 CUDA stream 上。

**非 MoE 层的通信：**
- Attention 层、LayerNorm、Embedding 等 Dense 层的梯度需要通过 AllReduce 聚合。
- AllReduce 由数据并行（Data Parallelism）进程组管理，跑在另一条 CUDA stream 上。

在原生的 DeepSpeed（以及类似系统）中，这两条 stream 是**独立的、无协调的**。NCCL 底层公平地分配 NIC 带宽给所有活跃的通信操作。结果是：All-to-All 和 AllReduce 同时跑，各自分一半带宽。

AllReduce 可以容忍这种带宽减半——它的数据量相对小、消息粒度细、可以和后续计算重叠。但 All-to-All **不行**。All-to-All 是一个阻塞性质的操作：在它完成之前，后续的 expert 计算（FFN forward/backward）无法开始。All-to-All 被拉长了多少，计算就多等了多久——GPU 的 SM 利用率在 All-to-All 期间只有 3.7%（Lina 论文对 20 个 step 的 PyTorch Profiler 结果）。

Lina 对训练侧的问题做了量化。在 16x A100 + 100Gbps InfiniBand 的测试环境下，他们采集了 1500 个 All-to-All 操作在反向传播中的实际完成时间，绘制了 slowdown factor 的 CDF 分布。结果触目惊心：

- 中位数 slowdown：**1.83x**
- 最差 slowdown：**4.14x**

也就是说，有一半的情况下 All-to-All 因为和 AllReduce 同时跑而被拉长了将近一倍；极端情况拉长了四倍多。这就是训练中 All-to-All 瓶颈的根因——不是 All-to-All 本身慢，而是它从来没拿到过完整的带宽。

### 直觉方案："那就让 All-to-All 优先运行，AllReduce 等一等"

这个问题最直接的答案就是优先级调度：All-to-All 到了就优先发送，AllReduce 只有在没有 All-to-All 等待或进行时才发起。

但 NCCL 不支持抢占。NCCL 的通信原语是高度优化的黑盒，一旦被调用，其完整的传输策略（包括 ring/halving-doubling 算法的选择、chunk 分割、连接建立）就全部推送到 CUDA stream 上，内部没有"暂停"或"释放带宽"的控制点。所以如果 All-to-All 到来时有一个 AllReduce 已经在跑了，All-to-All 只能等。

更糟的是，简单的优先级没有考虑操作的**到达时间**。考虑这个场景：

1. 反向传播开始，Dense 层的梯度先算完，AllReduce 被触发并开始执行（因为队列里只有它）。
2. MoE 层的梯度随后算完，All-to-All 被触发——但 AllReduce 已经在半路上，All-to-All 只能等。
3. All-to-All 等待期间，更多的梯度产出更多的 AllReduce 进入队列。
4. 等到 AllReduce 跑完，All-to-All 开始——但晚了一步，整个 MoE 层的后续计算都跟着晚了。

Lina 论文的 Figure 7b 给出了实际的时间线对比。在这种"naive priority"方案下，MoE 层梯度计算的总完成时间是 **30.9ms**，比不做优先级（baseline）的 **29.0ms** 更差。精心设计的优先级不但没加速，反而倒退了。

### 关键设计：Tensor 分区 → 微操作 → 灵活调度

Lina 的解决方案优雅但简单得令人意外。核心思想是：**把大的通信操作切成小的微操作（micro-op），然后在微操作级别做优先级调度。**

#### Step 1: Tensor Partitioning（张量分区）

传统的 PyTorch DistributedDataParallel 做 AllReduce 时会采用**梯度融合（gradient bucketing）**：把多个参数的梯度合并到一个 bucket 中，然后对每个 bucket 执行一次 AllReduce。这减少了通信次数，提高了大消息的带宽利用率。但问题是：bucket 的大小是变动的（在梯度边界对齐），而且一旦一个 bucket 的 AllReduce 开始，就没有控制机会了。

Lina 反其道而行之：**不合并，反而拆分。**

每个梯度 tensor 被切分成等大的 chunk（实验中的默认值是 30MB），每个 chunk 形成一个独立的 allreduce micro-op。同理，All-to-All 的 tensor 也可以被分区为多个 all-to-all micro-op。

这个设计带来了两个根本性的好处：

**好处一：粒度统一。** 每个 micro-op 的大小是相同的（30MB），这意味着它们的传输时间相近且可预测。优先级调度不需要估计"这次 AllReduce 要跑多久"——每一个 micro-op 的运行时大致相等。

**好处二：调度灵活。** 一个大的 AllReduce 一旦开始就必须跑完，可能阻塞后续的 All-to-All 很多毫秒。但一个 micro-op 跑完只要很短的时间——如果调度器发现一个 AllReduce micro-op 结束后有一个 All-to-All micro-op 在等待，它可以立即切换。

实际的调度规则很简单：
- All-to-All micro-op 的优先级高于 AllReduce micro-op
- 当 All-to-All micro-op 正在执行时，AllReduce micro-op 不能启动
- 当没有 All-to-All micro-op 在等待或执行时，AllReduce micro-op 可以运行
- All-to-All micro-op 到来时不能抢占正在运行的 AllReduce micro-op，但因为 micro-op 很短（~几毫秒），等待代价不大

论文的 Figure 8a 展示了效果。在将梯度切成 5 个 chunk 后：
- 在第一个 All-to-All 到来之前，调度器先发射了 3 个 AllReduce micro-op（约 30MB × 3 的传输量）
- 第一个 All-to-All 到来后，停止 AllReduce，专属带宽给 All-to-All
- 第一个 All-to-All 结束后，在 expert 计算期间（1.6ms），再穿插一个 AllReduce micro-op
- 梯度 i-2 的 AllReduce 比 naive priority 方案提前了 **6.6ms（21.7%）** 完成

#### Step 2: Pipelining —— 让计算不再空等

Tensor 分区不仅让调度更灵活，还开启了一个新的优化维度：**通信和计算流水线**。

传统方式下，All-to-All 必须完全结束后才开始 FFN 计算——所有 token 的 hidden state 都到齐了，expert 才开始处理。但如果把 All-to-All 切成 micro-op，expert 可以在收到第一批 token 后就开始计算——因为 FFN 操作在 token 粒度上是独立的。

论文的 Figure 8b 展示了流水线效果。在理想情况下，FFN 计算时间（1.6ms）可以完全隐藏在 All-to-All 通信时间内——等最后一批 token 到达时，前面批次的计算已经完成了。这意味着 **MoE 层的 total wall-clock time 缩短了整整一个 FFN 计算周期**。

现实中流水线效率受制于一个关键因素：**FFN micro-op 的计算时间远短于 All-to-All micro-op 的通信时间**。因为 100Gbps InfiniBand 上传输一个 30MB chunk 的时间远大于一个 FFN 对 30MB 数据的处理时间。这导致了流水线"气泡"——计算 stream 跑完了一批 micro-op 后要等通信 stream 传送完下一批。

#### Step 3: Expert Packing —— 增大计算粒度以匹配通信

Lina 用 Expert Packing 解决了这个粒度不匹配问题。

既然一个 expert 的 FFN 算不过一个 All-to-All micro-op，那就把多个 expert 的计算打包在一起。让每个 GPU 托管多个 expert，使得 FFN micro-op 的总计算量增大到和 All-to-All micro-op 的通信量大致相等，流水线才能高效。

具体策略：从一个 expert 每 GPU 开始，迭代翻倍（1, 2, 4, …），直到 FFN 计算时间超过 All-to-All micro-op 的通信时间。在大多数实验配置下，2 个 expert 每 GPU 是最优的。

Expert packing 带来了显著的流水线效率提升：
- Transformer-XL（16 expert）：流水线效率从 33% 提升到 86%（2.6x）
- GPT-2（16 expert）：从 36% 提升到 85%（2.36x）
- BERT2GPT2（16 expert）：从 34% 提升到 79%（2.32x）

平均提升 **2.43x**。当 GPU 显存不够装多个 expert 时，Lina 使用 DRAM offloading——把不活跃的 expert 参数换出到主机内存。

### 训练的端到端效果

在 16x A100 + 100Gbps InfiniBand 的环境下，Lina 的训练性能数据如下：

**训练步时加速（vs DeepSpeed Baseline）：**
- 2-expert 模型：1.71x speedup
- 4-expert 模型：1.37x speedup
- 8-expert 模型：1.73x speedup
- 16-expert 模型：1.47x speedup

2-expert 和 8-expert 的加速最大，原因很直观：2-expert 场景退化成纯数据并行（没有跨节点的 All-to-All，所有通信都是 AllReduce，Lina 的优先级调度有效消除了竞争），而 8-expert 场景下每节点 4 个 GPU（两个节点），所有 All-to-All 都是节点内的——避免了 inter-node 的带宽瓶颈。

**All-to-All 专用加速：**
- 4-expert：2.21x
- 8-expert：2.39x
- 16-expert：2.31x

All-to-All 的加速远高于整体步时加速，这是因为 MoE 层只占总步时的一部分——Dense 层的计算时间不受 Lina 影响。论文数据显示 All-to-All 平均占 MoE 层时间的 74.9%，占总体步时的平均 34.1%。这意味着 All-to-All 哪怕加速了 2.3x，整体步时加速也只能到 1.47-1.73x——这部分是 Amdahl's Law 的直接体现。

**GPU 利用率：** 在 16-expert 模型上，GPU SM 利用率从 baselin 的 66.2%（Transformer-XL）/ 62.3%（GPT-2）/ 63.5%（BERT2GPT2）提升到 83.4% / 78.2% / 82.5%，平均改善 **17.6%**。这个改善几乎全部来自于 All-to-All 阻塞期的缩短——GPU 不再空等通信完成。

**MoE 层反向传播加速：** 在 2-expert 和 8-expert 场景下分别达到 2.41x 和 2.32x——比前向的加速（1.84x 和 1.89x）更大，因为反向传播中 All-to-All 和 AllReduce 的竞争才是主要瓶颈，前向中没有这个竞争。

---

## 系统剖析（下）：推理的负载调度 —— 预测专家热度，动态调整部署

### 问题形态：不是竞争，是胖瘦不均

推理侧的问题和训练侧完全是另一个故事。推理中不存在 AllReduce（没有梯度需要同步），所以不存在带宽竞争。但出现了一个新的问题：**专家热度分布极端偏斜。**

原因在于 MoE 的特质——虽然在训练期间有 auxiliary load balancing loss 把 token 往各专家身上均匀推，但这个 loss 是"鼓励"而不是"强制执行"均匀分布。在推理时，gating network 根据输入 token 的实际语义特征做路由决策，没有任何人为干预。结果就是：

- 4-expert MoE 推理：最热专家收到的 token 是最冷的 **4.02x**
- 16-expert MoE 推理：最热专家收到的 token 是最冷的 **5.56x**

这不是一个可以"再调调辅助 loss 权重"来解决的问题——推理的输入是真实世界的请求，其 token 分布天然不均。你不能在推理时加辅助 loss（没有梯度更新），也不能用 capacity factor 丢 token（推理丢 token 直接影响输出质量）。

偏斜的专家热度造成了两个维度的损害：

**损害一：计算 waiting。** 热门专家所在的 GPU 要算的 token 多，计算时间长。但 attention 层处理整个序列，必须等所有 token 的 MoE 输出都到齐才能继续。这意味着冷门专家算完了也得等——论文实测冷门专家的最大空闲时间为 batch 推理时间的 **29.4%**。

**损害二：通信量不均。** 在专家并行下，每个 GPU 固定负责一个专家。热门专家的 GPU 的链路承担了不成比例的通信量，而冷门专家 GPU 的链路几乎是空的。由于 All-to-All 是同步操作，总完成时间取决于通信量最大的那条链路。

### 设计挑战：你怎么提前知道谁会热？

最自然的方案是"动态调度"——根据 gating network 的输出，把热门专家复制到更多 GPU 上，冷门专家打包到更少的 GPU 上，以此平衡负载。但这里有一个棘手的先有鸡还是先有蛋问题：

- 要调度，你需要知道 gating network 的输出（每个 token 选了什么专家）
- gating network 的输出要等 gating 计算完成才知道
- gating 计算在每个 MoE 层的开始进行——这意味着如果你等 gating 输出再做调度，调度本身（重新分配专家、协调 All-to-All 的 split、更新 device mapping）会阻塞整个前向计算
- 而且这个问题在每一层都重复出现——每一层的专家热度分布都不一样

Lina 论文提供了实证证据：同一批输入在不同 MoE 层的 top-4 热门专家完全不同（Table 2）。因此不能在一层做了调度就全模型复用，必须在每一层重新做。如果每层都等 gating 输出再调度，累积的调度延迟将吞噬所有加速收益。

### 突破点：Token 级专家选择的跨层模式

Lina 的关键洞察来自于对 MoE 模型内部运行机制的细致观测：**一个 token 在不同层选择的专家之间存在相关性。**

他们追踪了已训练模型中 token 的专家选择行为。对于在层 i 中选择了同一个专家的所有 token（形成一组），他们计算这些 token 在层 i+1 中也选择了相同 top-k 专家的比例。结果令人惊讶：

- k=1（只看 top-1 专家）：平均 **41.94%** 的 token 保持了"专家一致性"——在相邻层选了同一个专家
- k=2（看 top-2）：平均 **54.59%** 保持了一致
- 越深的层，比例越高（在模型后半段出现明显的上升趋势）

为什么会有这种现象？Lina 论文给出的解释很直观：MoE 的 gating network 架构很简单（一个线性层 + softmax），它的路由决策主要基于 token 的相对浅层特征——如词性（名词/动词）、语义类别（数字/时间）等。这些特征是**固定的**——同一个 token（"apple"）在任何一层都是"名词"，gating 在不同层中会对相似的语义特征做出相似的路由决策。同时，专家倾向于关注 token 的局部句法信息而非序列内的交叉依赖，所以不同层中的专家对同一 token 的判断有底层的一致性。

这个模式虽然不够精确到预测每个 token 的具体选择（41% 的 top-1 准确率不算高），但**对整体热度估计来说已经足够了**——因为热度是聚合指标，不需要精确到每个 token。

### Lina 的热度估计方法

基于上述模式，Lina 的热度估计流程分为两个阶段：

**离线的 Profiling 阶段（训练结束后、推理前）：**

1. 训练中负载均衡 loss 收敛后，收集所有 MoE 层的 gating 输出
2. 将 token 按它们在"路径"（从层 i-l 到层 i 选择的专家序列）分组
3. 对每组，统计其在层 i+1 的专家选择分布 Ψ
4. 路径长度 l 控制精度-成本 trade-off：l 越大估计越准，但 profiling 数据量和查询成本越高。Lina 默认用 l=3

**在线的估计阶段（每个推理 batch，每一层）：**

1. 对每个 token t，跟踪它过去 l 层的专家选择序列 j(t)（"采样路径"）
2. 从离线统计中查找路径 j(t) 对应的下一层专家概率分布 π(j(t))
3. 汇聚所有 token 的 π，得到下一层每专家 e 的估计热度：∑(t) π(j(t))(e) / N_tokens
4. 按估计热度分配设备：热门专家拿到更多 GPU，冷门专家打包到更少 GPU

### 两阶段调度：估计先行，实测校准

Lina 每个 MoE 层采用两阶段调度：

**第一阶段：估计驱动的预调度（与计算重叠）。**
在 gating network 开始计算的同时，调度器基于估计热度计算新的 expert-device mapping。所有调度相关的通信（估计热度收集、新的 mapping 分发）piggyback 在常规的 All-to-All 上——设备 0 在第一次 All-to-All 中收集各设备的估计信息，在第二次 All-to-All 中下发新的 mapping。由此调度开销几乎完全被模型计算覆盖。

**第二阶段：细粒度校准（仅在必要时）。**
gating 计算完成后，调度器比较实际专家热度分布和估计值的差异。如果差异在可接受范围内（top-2k 专家列表一致），不做任何调整，推理继续。如果偏差较大（意味着一些被估计为冷门的专家实际上是热门的——它们被错误地打包在了少数设备上），则触发重新调度。这不是常见的路径：实验显示第二阶段触发的概率平均约 23-25%，而且重新调度的时间（~6.2ms）远小于因负载不均衡导致的设备空闲时间。

论文对比了不使用两阶段调度的替代方案，证实了这个设计的必要性：
- **只做调度不做估计**（每层等 gating 输出后再调度）：16-expert Transformer-XL 的 median 推理时间恶化了 **24.0%**——因为每层阻塞等待调度的开销太大
- **只做估计不做校准**：16-expert Transformer-XL 的 tail（95%ile）推理时间恶化了 **26.7%**——因为估计偏差没有被纠正，导致热门专家被错误打包

### 推理的端到端效果

**中位数推理时间加速：**
- Transformer-XL 4-expert：1.54x；16-expert：1.45x
- BERT-Large 4-expert：1.36x；16-expert：1.46x

**95% 分位推理时间加速（尾延迟）：**
- Transformer-XL 16-expert：**1.82x**
- BERT-Large 16-expert：**1.68x**

尾延迟的改善比中位数大，因为负载不均衡对 tail latency 的影响天然更大——一个 batch 中总有少数 token 落到热门专家上，它们的完成时间决定了整个 batch 的 95%ile。Lina 通过平衡专家负载，直接缩小了最大和最小专家处理时间之间的 gap。

**MoE 层时间（95%ile）：**
- Transformer-XL 16-expert：1.77x
- BERT-Large 16-expert：1.81x

**All-to-All 时间（16-expert）：** 平均 1.96x 加速，最好的层提升 2.50x。All-to-All 时间的改善直接说明负载平衡的有效性——每条链路的传输量变均匀了，最慢的链路不再成为瓶颈。

**热度估计准确率：** Transformer-XL 平均 60.4%（l=3），BERT-Large 平均 63.5%（l=3）。对比来看 l=1 的准确率只有 31.6% 和 28.3%——3 层路径比 1 层路径的准确率翻了一倍。l=6 提升到 71.4% 和 66.0%，但推理时间改善不明显——因为热身延迟（前三层没有调度）和阶段二校准已经足够弥补准确性差异。

**不同任务的可迁移性：** Lina 的热度估计是针对每个具体任务做 offline profiling 的。论文在四个下游任务上测试了泛化性——情感分析（IMDB、Twitter）、翻译（WMT 法语、俄语）——热度估计准确率维持在 62.3%-68.8%，95%ile 推理时间达到理想值的 1.04x-1.11x。这说明跨层专家选择模式是 MoE 模型的内在属性，不是某个数据集的偶然特征。

---

## 关键技术决策：训推分离的必然性

Lina 最有价值的贡献不是单独的训练优化或单独的推理优化——而是在同一个系统中明确地将训练和推理的通信调度视为**两个不可混合的问题**。这个决策的基础来自几个根本性的差异：

### 差异一：是否存在跨通信类型的带宽竞争

**训练中存在竞争，推理中没有。**

反向传播阶段同时存在两种集体通信操作：All-to-All（专家并行）和 AllReduce（数据并行）。它们分别由独立的 CUDA stream 管理，在 NCCL 层面共享 NIC 带宽。这种竞争是训练特有的——推理没有任何梯度聚合的需求，所有 NIC 带宽都归属于 All-to-All。

这意味着训练的优化目标是**协调**：如何在两种通信操作之间分配带宽使得整体阻塞时间最小。推理的优化目标是**均衡**：如何让所有链路的传输量和所有 GPU 的计算量均匀分布。

如果强行用一种机制同时处理竞争和均衡，结果就是两边都不讨好。想象一下，如果把 Lina 的推理方案（动态调整 expert-device mapping）拿来做训练——因为需要移动 expert 参数、重建进程组、重新协调 All-to-All 的 split，这些开销在每次迭代都发生，训练吞吐会被压垮。反过来，把训练的优先级调度拿到推理——推理没有 AllReduce，优先级队列里始终只有 All-to-All，等于退化回 baseline。

### 差异二：优化指标不同——吞吐 vs 延迟

**训练追求吞吐（throughput），推理追求延迟（latency），尤其是尾延迟。**

训练中你关心的是"平均每 step 花多少时间"——这个值乘以 step 总数就是总训练时间。一个 batch 偶尔慢了没关系，只要平均步时短就行。因此训练优化可以用"平均数"来评估——Lina 的训练评估就是 averaging over 50 steps。

推理中，用户体验由 95%ile 甚至 99%ile 的延迟决定。一个用户请求如果落到尾延迟上，会直接感受到响应慢。而且 MoE 推理中的尾延迟往往和热门专家相关——热门专家上的 token 总是等得最久。Lina 的推理评估同时给出 median 和 95%ile 的改善，这是有意的设计——说明作者知道 inference optimization 不能只看平均。

这种指标差异影响了方案设计。训练方案可以接受一些偶尔的调度失衡（比如 partition size 偶尔不够优），只要平均步时改善就行。推理方案必须处理 worst-case——这就是为什么 Lina 在推理中需要两阶段调度，用第二阶段来兜底。

### 差异三：瓶颈的根源不同

我们已经反复强调这一点，但值得用一种更系统的方式总结：

| 维度 | 训练瓶颈 | 推理瓶颈 |
|------|---------|---------|
| **根本原因** | All-to-All 和 AllReduce 的带宽竞争 | 专家热度的偏斜分布 |
| **导致的现象** | All-to-All 被延长（中位数 1.83x，最差 4.14x） | 热门专家处理时间长，链路传输量不均 |
| **通信层面的表现** | All-to-All 不能获得完整带宽 | All-to-All 被最慢的链接限制 |
| **计算层面的表现** | GPU 在 All-to-All 期间空闲（SM 利用率 3.7%） | 冷门专家 GPU 等待热门专家 GPU 完成计算（最大空闲 29.4% of batch time） |
| **最优解法** | 微操作 + 优先级调度 | 基于估计的负载均衡 + 动态设备分配 |
| **核心机制** | Tensor partitioning → 调度粒度细化 → 优先 All-to-All | 跨层模式 → 估计热度 → 复制热门、打包冷门 |
| **预期改善维度** | 步时缩短（吞吐提升） | 中位数和尾延迟降低 |

这张表不只是对 Lina 的总结。对于做 MoE 系统工程的人，它提供了一个诊断框架：当你的系统通信慢时，先判断是训练还是推理，再判断是竞争还是失衡——两种问题的解法和评估标准完全不同。

### 差异四：是否有"已知信息"可以利用

训练中对未来通信没有先验信息——每个 batch 的 token 不同，All-to-All 和 AllReduce 的大小也不同。所以 Lina 的训练侧采用了一种"不需要预测"的方案：用 micro-op 实现细粒度调度，让实际执行顺序自适应。

推理中却有可被利用的先验信息——跨层的专家选择模式。虽然不完美，但足以提供足够的信号来做预调度。这就是为什么 Lina 的推理方案比训练方案更"预测驱动"——两阶段调度中的阶段一完全依赖估计，而训练侧不需要任何估计。

这个对比揭示了一条普适的原则：**当系统中有可被利用的统计模式时，基于预测的调度胜于反应式调度；当没有时，需要通过减小操作粒度来增加调度灵活度。**

### 微操作洞察的通用性

Lina 的训练方案依赖于一个看似简单但深远的设计选择：把 All-to-All 和 AllReduce 都切成 micro-op。这个选择的价值不仅体现在 Lina 自己的数据上，还在后续的工作中被验证。

最直接的呼应来自 DeepSeek-V3。DeepSeek-V3 的 DualPipe 调度将前向和后向计算切成 micro-batch 级别的 chunk，实现前向和反向的流水交错。虽然 DualPipe 的粒度是 micro-batch（含多个 layer），不同于 Lina 的 tensor-level chunk（含多个 token），但底层哲学完全一致：**通过分解大操作来创造调度自由度，然后把最优调度问题转化为排程问题。** 两个系统都证明了一个事实——在分布式 MoE 中，通信的不当排程造成的损失远大于分解操作的额外开销（Lina 测得的 partition/cat overhead 平均仅 1.02% 的步时）。

另一个值得注意的细节是 Lina 的 partition size 选择。实验显示 30MB 是最优的——比它小（10MB）会有传输效率损失（每个 micro-op 的建立和收敛开销占比变大），比它大（50MB+）会失去调度灵活性，因为 micro-op 太大导致 All-to-All 等待 AllReduce micro-op 的时间也变大。这 30MB 的选择也是特定于 100Gbps IB 和 A100 环境的——在不同的硬件配置下，最优分区大小必然不同。

---

## 横向对比：Lina vs DeepSpeed-MoE vs Tutel

### DeepSpeed-MoE：基础设施打磨，缺少通信调度

DeepSpeed-MoE（2022）是 Lina 的基准实现平台，但两者的设计哲学有本质差别。

DeepSpeed-MoE 的核心贡献在几个方面：
- **灵活的并行策略组合**：数据并行、专家并行、以及 Pyramid-Residual MoE 架构（只在模型底层和顶层使用专家，中间层用 Dense，减少通信次数）
- **层次化 All-to-All**：将跨节点和节点内的 All-to-All 分开处理，利用 NVLink 的高带宽做节点内通信，只在节点间走 IB
- **Random Token Dropping**：在训练中随机丢弃一定比例的 token，减少通信量，但牺牲模型质量

但这些优化都属于"**让 All-to-All 本身跑得更快**"的范畴。DeepSpeed-MoE 没有解决"All-to-All 和 AllReduce 同时跑时会互相拖慢"的问题——它依赖 NCCL 的默认公平共享策略，而这正是 Lina 发现的核心瓶颈。

Lina 对 DeepSpeed-MoE 的超越不在于实现更好的 All-to-All——在单操作测量上两者的 All-to-All 本身可能差不多快——而在于**改写了操作之间的关系**。通过优先级调度和微操作，Lina 确保了 All-to-All 总是拿到完整带宽，因此 All-to-All 的完成时间直接减少了 2.21x-2.39x。

一个有趣的细节：Lina 在评估时关闭了 DeepSpeed 的 Random Token Dropping 和层次化 All-to-All，以确保实验的公平性——对比的是纯调度策略的差异，而非工程实现的差异。

### Tutel：自适应并行切换，互补而非竞争

Tutel（2022，同样来自 Microsoft）在通信方面的主要贡献是**层次化 All-to-All** 和**自适应并行切换**：

- 层次化 All-to-All：根据节点内/节点间 GPU 拓扑结构，将 All-to-All 分解为两步——先在节点内做 all-to-all（利用 NVLink），再做节点间的 all-to-all（利用 IB）。这和 DeepSpeed-MoE 的设计一致。
- 自适应并行切换：根据 expert 的计算负载动态调整数据并行和专家并行的配比，以平衡计算和通信。

Lina 的论文中将 Tutel 作为第二 baseline。结果显示 Tutel 的表现与 DeepSpeed 接近（差距很小），Lina 对 Tutel 的加速略低于对 DeepSpeed 的加速。这说明 Tutel 的层次化 All-to-All 和拓扑感知优化有一定效果，但仍未解决通信竞争的根本问题——Tutel 和 DeepSpeed 一样，没有 All-to-All 和 AllReduce 之间的协调调度。

Lina 和 Tutel 的关系是互补的：Tutel 的层次化 All-to-All 和自适应并行可以降低 All-to-All 本身的绝对时间，Lina 的优先级调度则确保 All-to-All 不被其他通信操作干扰。**理论上可以在 Lina 的调度框架上应用 Tutel 的拓扑感知 All-to-All，同时获得两边的收益。**

### Lina 的独特位置

放在 2023 年 MoE 系统的全景中，Lina 的独特贡献可以这样概括：

- 它是第一个**同时涵盖训练和推理**且**分别识别其瓶颈根源**的 MoE 通信优化系统
- 它第一次系统性地论证了**All-to-All 和 AllReduce 的竞争是训练瓶颈的根因**，并给出了量化的 slowdown 分布
- 它率先提出了**利用跨层专家选择模式做推理预调度**，这在当时是非常规的思路
- 它的**微操作调度器**是一种通用的设计模式——不依赖任何关于未来通信的先验信息，通过分解操作来创建调度自由——这个思想在后续的 DualPipe 等工作中得到延续

Lina 的局限性也需要诚实面对：
- 所有实验在最多 16 个 A100 GPU 上进行——这是一个小规模集群，是否在 100+ GPU 的规模上依然有效未经验证。随着 GPU 数量增加，All-to-All 的同步开销会非线性增长
- 热度估计的准确率在 60% 左右——意味着 40% 的情况下估计不够准确，需要第二阶段校准。在极端偏斜的情况下（如某个专家突然暴热），估计可能完全失败
- Expert packing 导致的 GPU 显存压力——16-expert 模型下 Transformer-XL 和 GPT-2 用满了 100% 的 GPU 内存并依赖 DRAM offloading，而 offloading 会引入额外的 PCIe 带宽竞争
- 调度器只在单节点上运行——如果 device 0 故障，整个推理管线会中断

---

## 总结与清单：通信调度的五项原则

Lina 帮我们澄清了一个在 MoE 通信优化中最易被混淆的问题：**训练和推理的 All-to-All 瓶颈是两种不同的病，需要不同的药。**

从 Lina 的设计中，我们可以提炼出以下五条可以跨系统泛化的通信调度原则：

### 原则 1：先诊断，后开方——识别是竞争还是失衡

在动手优化之前，回答这个问题：你的 All-to-All 慢，是因为其他通信操作在抢带宽（训练），还是因为某个设备上的负载比其他设备高（推理）？前者应该用优先级或带宽隔离，后者应该用负载重分配。混淆二者的代价是写出一个看起来很完整但实际不解决核心问题的方案。

### 原则 2：大操作要能切碎（Chunk Everything）

Lina 的 micro-op 设计是最值得被移植的模式。无论你面对的训练框架是 PyTorch、JAX 还是自研框架，只要你的通信库支持对 tensor chunk 的独立操作，就能套用这个思路。粒度越细，调度越灵活，优先级的生效越及时。代价是 partition 和 concatenation（Lina 实测 1.02% 步时开销）以及更多的小消息传输开销（1.7% 的传输时间延长），通常在可接受范围内。

### 原则 3：当有模式可循时，预测胜于反应

Lina 推理侧最精彩的决策不是负载均衡本身，而是**利用跨层专家选择模式进行提前预估**。如果等到 gating 输出再做调度，每层都要付出阻塞性的调度开销。这个洞察可以泛化到任何"需要知道未来状态但现在是同步系统"的场景：找到能表征未来状态的统计先验，用它做预决策，只在预决策和实际偏离过大时才触发矫正。

### 原则 4：阻塞操作配全带宽，非阻塞操作填空隙

在 Lina 的训练调度中，All-to-All 拿全带宽，AllReduce 在 All-to-All 不存在的间隙中跑。这是一个通用优先级原则：**同步/阻塞的通信操作（那些完成后才能继续计算的操作）应该被赋予最高优先级和全带宽；异步/可重叠的通信操作可以等。** 这个原则和传统网络中"short flow 先走"的直觉相反——因为短消息（AllReduce）虽然单个时间短，但它可以被重叠，而长消息（All-to-All）虽然单个时间长，但它阻塞着后续计算。所以阻塞性优先于消息大小。

### 原则 5：校准是必须的——预测总会出错

Lina 两阶段调度中的第二阶段（细粒度校准）可能看起来像一个兜底机制，但它是必不可少的。没有它，估计错误导致的性能惩罚（26.7%-33.1% 的尾延迟恶化）会超过调度带来的收益。做任何基于预测的调度系统时，都要为"预测错了"留一条矫正路径——矫正开销应该远小于预测错误的代价，但不能为 0。

### MoE 通信诊断清单（可打印）

---
| 步骤 | 问题 | 你的回答 | Lina 的参考值 |
|------|------|---------|-------------|
| 1 | 你的场景是训练还是推理？ | 训练 / 推理 | 两者不同，不可混淆 |
| 2 | 如果是训练，All-to-All 期间同时有 AllReduce 在跑吗？ | 是 / 否 | 是 → 需要优先级调度 |
| 3 | 如果是训练，All-to-All 期间 GPU SM 利用率是多少？ | __% | Lina 基线 ~3.7% |
| 4 | 如果是训练，All-to-All 的 slowdown factor 是多少？（vs 独享带宽） | __x | Lina 中位数 1.83x |
| 5 | 如果是推理，最热专家和最冷专家的 token 比例是多少？ | __:1 | Lina 4.02x-5.56x |
| 6 | 如果是推理，你能否在 gating 计算前预判专家热度？ | 能 / 否 | 跨层模式可用，准确率 ~60% |
| 7 | 你的 micro-op partition size 可调吗？ | 是 / 否 | Lina 推荐 30MB（100Gbps IB） |
| 8 | 你的 expert packing 策略是否已优化？ | 是 / 否 | Lina 推荐 2 experts/GPU |
| 9 | 调度开销是否与计算流水线重叠？ | 是 / 否 | Lina 阶段一完全重叠 |
| 10 | 预测不准时的兜底机制是否就位？ | 是 / 否 | Lina 两阶段调度 |
---

### 从 Lina 到未来：遗留的开放问题

Lina 发表于 2023 年 7 月，时间上刚好卡在一个关键节点——DeepSeek-V3（2024）和更激进的通信-计算调度策略即将出现。有几个 Lina 没解决的问题成为后来研究的重点：

**1. 更大规模下的调度有效性。** Lina 最多测了 16 个 GPU——当 GPU 数量扩大到 100+ 时，All-to-All 的分区同步开销、优先级调度的可扩展性、以及热度估计在更多专家层上的准确性都需要重新验证。

**2. 和 pipeline/tensor parallel的协同。** Lina 只考虑了 DP + EP，但实际训练中往往还有 TP（Tensor Parallelism）和 PP（Pipeline Parallelism）。这四种并行度下的通信互相干扰是什么样子？优先级调度在四维并行空间中应该如何扩展？DeepSeek-V3 的 DualPipe 给出了一个方向性的答案，但通用解还不存在。

**3. 热度估计的精度瓶颈。** 60% 的 top-1 准确率意味着 40% 的 case 需要第二阶段校准。能否用更复杂的方法——比如训练一个轻量级的预测模型——将准确率提到 80%+？Lina 在讨论中提到了这个可能性，但没有具体实现。

**4. 硬件异质性。** Lina 的所有实验都在同质的 A100 + IB 集群上。当 GPU 型号混合（如 H100 + A100）或互联混合（如 NVLink + IB + RoCE）时，All-to-All 的多速率链路问题会更复杂。

这些开放问题在后面的文章里陆续展开。下一篇，我们把视野从单个系统的通信调度扩展到整个集群的资源拓扑——从 IB 到 RoCE 到 NVSwitch 再到 OCS，看看 MoE 通信如何在物理路由层找到加速的答案。

---

*下一篇预告：[第十三篇：从 InfiniBand 到光交换 —— MoE 通信的物理路由进化]*
