---
layout:     post
title:      Full System Power, Thermal and Parallelism
subtitle:   Full System Power, Thermal and Parallelism
date:       2026-06-21
author:     George Lin
header-img: img/post-bg-gen.png
catalog: true
tags:
- moe
---

## 引言：跑满不代表跑好

A100 GPU 标称的 FP16 峰值是 312 TFLOPS，一块卡。如果有人告诉你，拿 3072 块 A100 绑在一起，你的训练吞吐能稳在每块 163 TFLOPS、合计 502 PFLOPS —— 也就是理论峰值的 52% —— 你会觉得这是及格还是优秀？

Megatron-LM 团队 2021 年在 SC 会议上给出的答案就是这个 52%。按今天大模型训练的标杆来看，这个数字不算低也不算高。但真正值得追问的问题不是"52% 好不好"，而是"另外 48% 去哪了"。

答案不在 GPU 核心里，不在 HBM 带宽里，甚至不只在 NVLink 带宽里。答案在**它们三个的交叉地带**。

当你把 1000 块 GPU 放进一个机柜群落，它们之间就会发生一些单卡 scale 时根本不会出现的物理效应：某些 GPU 因为处在气流下游，比邻居恒高 20°C；某些 PCIe 链路上的 SendRecv 调用太稀碎，导致 100Gbps 的 InfiniBand 实际只跑出了 30Gbps 的有效吞吐；你加了 EP（Expert Parallelism），TP 的 AllReduce 就被挤到了瓶颈链路上；你调大了 microbatch size 想摊薄通信开销，结果峰值功耗冲破了功耗墙，SM 频率从 1.98GHz 被硬生生压到 1.7GHz，同步训练让所有人等着最慢的那块卡。

这些效应在单个 GPU 的 benchmark 里都是隐形的。**但在千卡级别，它们形成了一条环环相扣的因果链：通信逼着 GPU 等待→ GPU 在等待和计算之间频繁切换→ 功耗曲线被撕裂出尖峰→ 散热系统跟不上→ 时钟频率被降→ 同步训练拖慢所有人→ 通信效率进一步恶化。** 这不是某一个瓶颈的问题，而是一张网的每一个节点都在互相拉扯。

Georgia Tech 的 Go 等人在 2025 年 MICRO 会议上发表的长文《Characterizing the Efficiency of Distributed Training: A Power, Performance, and Thermal Perspective》(MICRO'25)，就是对这张网的最完整"解剖报告"。他们用 32 块 H200、64 块 H100 和 32 块 AMD MI250，把 TP、PP、DP、EP 的每一个排列组合都跑了一遍，同时逐卡采集了功耗、温度、PCIe 带宽、NVLink 流量和 SM 频率。这篇报告打破了"并行策略只看吞吐量"的舒适区，把功耗和热搬上了桌。

今天这篇文章，我们就借助 Georgia Tech 的系统级数据，加上 Megatron-LM 的并行策略经验、Alpa 的自动并行搜索视角、以及 Google Pathways 的异步数据流实践，来回答一个贯穿千卡集群的核心问题：**当通信遇上功耗墙和热节流，谁在为谁买单？**

---

## 系统剖析：通信-计算-显存三角的塌缩

### 通信不只是"搬运数据"

在讨论分布式训练时，有一个流行的简化框架："计算密集型模型看 FLOPS，通信密集型模型看带宽"。这个框架在单节点或者小规模集群上大概是对的。但在千卡集群上，它漏掉了最关键的一环：**等待**。

让我们把训练迭代的微观时间线展开。一个 typical 的训练 step 中，每一块 GPU 的状态大致在这几个阶段之间切换：

1. **计算中**：SM 单元跑矩阵乘法、注意力、激活函数 —— 真正产生有效 FLOPS 的时刻。
2. **通信中**：通过 NVLink/PCIe/IB 收发张量 —— 数据在物理链路上跑，GPU 流水线被通信 kernel 占据。
3. **等待中**：计算做完了，通信还没完成；或者通信做完了，但同步屏障还没解除 —— GPU 什么事都不做。
4. **热身/恢复中**：刚从等待状态唤醒，SM 频率还没恢复到最高点 —— 低效计算 + 高功耗（因为频率提升过程中需要额外电压）。

在 Dense 模型的小规模训练中，状态 3 的占比通常很小。因为 TP 的 AllReduce 可以通过 NVLink 迅速完成（NVSwitch 提供了 900GB/s 全互联带宽），PP 的 P2P 通信量也不大。**但当 MoE 的 All-to-All 通信加入之后，状态 3 的占比会以非线性速率攀升。**

Georgia Tech 的实验数据清晰地展示了这一点。在 32xH200 集群上训练 Mixtral-8x22B 时，稀疏 MoE 模型的 kernel 时间分布中，通信时间占据了主导地位。以 EP8-TP4-PP1 配置为例，通信 kernel 的耗时超过了总 kernel 时长的 50%。相比之下，同等规模的 Dense 模型（如 Llama3-70B）的计算 kernel 占比始终在 50% 以上。

这不是一个简单的"通信带宽不够"的问题。**它是一个"计算变快了、通信也跟着增多、但等待变多了更多"的飞轮效应。** MoE 把单个 token 的计算量压扁了（因为只激活少量专家）—— 这就意味着 GPU 需要处理更多的 token 来保持 SM 单元不空转 —— 更多的 token 意味着更多的 All-to-All 通信—— 更多的 All-to-All 通信意味着更长的等待时间—— 而更长的等待时间又把更多的计算周期"吃"掉了。

### 通信-计算-显存：三个维度缠绕在一起

在千卡集群的全系统分析中，通信、计算、显存不是三个独立的瓶颈，而是一个三角形的三边，每一边的伸缩都会连带挤压另外两边。这种耦合在 MoE 训练中尤其剧烈：

**通信→计算的传递链**：MoE 中专家散布在多个 GPU 上→ 每个 GPU 都要参与 All-to-All → All-to-All 的延迟随参与 GPU 数量指数增长 → 通信延迟里 GPU 空闲 → 计算利用率暴跌。

**计算→功耗的传递链**：为了补偿通信带来的空闲期，通常的做法是增大 microbatch size，让每次通信"搬运"的数据更有"分量"（更好的 compute-to-communication ratio）。但更大的 microbatch 意味着每个 SM 周期内要做更多的矩阵乘法—— 峰值功耗冲高—— 触发功耗墙—— SM 频率下调—— 算力进一步下降。

**功耗→通信的传递链（反向）**：功耗墙导致的频率下调让计算变慢→ 原本被精心编排的通信-计算重叠被打乱→ 通信 kernel 在没有计算 kernel 可以重叠的情况下单独执行→ 通信延迟暴露→ GPU 等待时间更长→ 频率进一步下降（恶性循环）。

Georgia Tech 论文里有一个令人瞠目的数据：**在一次实验中，节点级供电故障导致部分 GPU 的运行速度下降超过 4 倍**，形成了严重的 straggler 效应，整个训练流水线被拖垮。4 倍减速不是 GPU 硬件故障——同样的卡在其他节点就是正常的——而是供电系统的瞬时不足导致 GPU 进入低功耗保护模式。这提醒我们一个很容易被忽略的事实：**在千卡规模下，基础设施的物理极限不再是二阶效应，它们可能随时变成决定训练吞吐的主要矛盾。**

### 微批次大小：一把双刃剑的物理机理

微批次大小（microbatch size）是分布式训练中最常被调节的超参数之一。它的直观逻辑很简单：单个 microbatch 越大→ 计算量越大→ 通信频次越低→ compute-to-communication ratio 越好→ 吞吐应该越高。

但 Georgia Tech 的数据告诉我们，这个逻辑只在某个阈值之前成立。过了阈值，增大 microbatch 反而会损害性能。

数据非常清晰。在 H200 集群上训练 GPT3-175B 时：
- TP8-FSDP 配置下，microbatch size 从 1 增大到 4，step time 获得了 3× 以上的加速。这是因为更大的 microbatch 使通信粗粒度化，更好地利用了有限网络带宽。
- 但在 TP2-PP16 这种 pipeline-heavy 配置下，microbatch size 增大的好处迅速消失，甚至导致效率下降。

为什么 pipeline-heavy 配置对抗大 microbatch 如此敏感？Georgia Tech 给出了三层解释：

**第一层：通信带宽饱和。** 增大 microbatch 意味着每次 P2P 通信传输的数据块变大。在 PP 中，相邻 pipeline stage 之间需要传递激活值，通信量与 microbatch size 严格成正比。当数据块大小超过了网络链路的 BDP（带宽延迟积），链路进入饱和状态，进一步增大数据块不会提升有效带宽，只会增加排队延迟。

**第二层：流水线气泡的反直觉效应。** 在 1F1B 调度下，pipeline bubble 的大小是 (p-1)/m，其中 p 是 pipeline stage 数，m 是 microbatch 数。增大单个 microbatch 的 size 意味着总 batch size 不变时 microbatch 数 m 减少—— 气泡比例上升。这就是把双刃剑：你降低了通信频率，但增大了等待时间的比例。

**第三层：功耗和热，这是最被低估的。** Georgia Tech 的功耗时序数据揭示了一个关键模式：大 microbatch 诱发**突发性执行（bursty execution）**。当每个 microbatch 很大时，GPU 在等待阶段完全空闲（功耗低），进入计算阶段后 SM 单元猛地冲到满负荷（功耗瞬时飙高）。这种"闲→猛冲→闲"的执行模式，比均匀的"中负荷持续跑"在散热上要恶劣得多。原因很简单：热容——GPU 的散热系统有热惯量，瞬时功耗的尖峰来的太快，散热器根本来不及响应，芯片结温（junction temperature）就会在毫秒级时间内上冲到节流阈值。

数据证实了这一点。在 H200 集群上，峰值功耗和峰值温度**总是**随着 microbatch size 增大而升高——无论训练吞吐是否改善。在 TP-heavy 配置下，高功耗触发时钟节流（clock throttling），直接降低计算效率。在 PP-heavy 配置下，流水线气泡产生更 bursty 的执行模式，间歇性地让计算资源处于低利用率状态，但功耗尖峰仍然在——形成"高功耗低产出"的尴尬局面。

总结：**大 microbatch 不是免费午餐。它的代价不只在纸面上的 pipeline bubble 公式里，还在供电系统能不能承受电流尖峰、散热器能不能压住瞬时温升。** 这些在千卡集群里不是细节，而是决定了你的 MFU 到底能稳定在 50% 还是被压在 35%。

### 热 IS 通信问题——统一通信调度能做什么

如果我问你"一块 GPU 比另一块 GPU 温度高 20°C，这是谁的错？" 你大概率会回答"散热设计"或者"物理位置"。但如果我再问"温度高的 GPU 运行慢，拖累整个同步训练，这是谁的错？" 这就进入了一个跨界地带——**热是物理问题，但它造成的性能损失是通过通信机制放大的。**

Georgia Tech 的实验里，NVIDIA HGX 机箱采用前到后（front-to-back）气流设计。气流从前端（冷风口）进入，经过 8 块 GPU 后从后端（热风口）排出。靠近进风口的 GPU（前端 GPU）和靠近出风口的 GPU（后端 GPU）之间存在系统性的温度差。在最极端的情况下，这个温差达到了 27%。

27% 的温差意味着什么？
- 后端 GPU 的结温始终比前端 GPU 高出 15-20°C。
- 当负载加重（比如进入计算密集阶段），后端 GPU 最先撞上热节流温度阈值。
- 热节流一旦触发，SM 时钟频率从 1.98GHz 下调到 1.7GHz 甚至更低。
- 在同步训练中（TP 的 AllReduce 是同步的，PP 的 1F1B 调度也是同步的），**所有 GPU 必须等最慢的那块**。
- 于是前端 GPU 虽然温度低、频率高、算得快——但它们必须等着。等待的时候它们在干嘛？不确定。可能在忙等（spin-wait），可能在低功耗空闲状态。

这就是那条著名的因果链在热维度上的投影：**热不平衡→ 频率不对称→ 同步等待→ 计算资源浪费。** 而这一切，都是因为通信-同步机制在物理上要求所有 GPU 步调一致。

Georgia Tech 在论文中提出了一个非常有穿透力的观察：**当前软件系统几乎都假设 GPU 行为是均匀的（uniform optimization assumption）。** 所有 GPU 跑同样的 kernel，用同样的 microbatch size，做同样的通信调度。这个假设在芯片级层面成立（同一型号的 GPU 的 SM 架构一样），但在系统级层面不成立（物理位置的差异导致热行为不同）。

破局思路是什么？Georgia Tech 实验了**热感知 pipeline stage 分配**（thermal-aware pipeline stage placement）。他们的做法是：把较冷的 GPU（前端 GPU，进气口附近）和较热的 GPU（后端 GPU，出气口附近）分别成组。让冷 GPU 处理更重的 pipeline stage（例如 embedding 和 projection 层，它们的张量更大）。这种非对称分配在两个变体上被验证：

- **对称变体（Symmetric）**：冷 GPU 组负责前几个 pipeline stage，热 GPU 组负责后几个 stage，每个 stage 的计算负载均等。效果：整体效率提升最高 2%，但热方差实际上增大了。
- **非对称变体（Asymmetric）**：冷 GPU 组承担额外一个 transformer 层的计算量，以主动"卸载"热 GPU 的负担。在 Llama3-70B（80 层，4 个 pipeline stage）上，这种 19/21 的层数 split（10% 负载不平衡）带来了 4% 的效率提升和 8% 的温度差距缩小。在 GPT3-175B（96 层，8 个 pipeline stage）上，11/13 的 split（18% 不平衡）缩小了 17% 的温度差距，但效率下降了 7%——因为负载不平衡的代价超过了热改善的收益。

关键结论很明确：**热感知调度有效，但需要精确匹配模型结构和硬件热特征。层数越少（stage 越粗），负载不平衡的惩罚越重。** 这不是一个"开箱即用"的优化，而是一个需要在模型结构和物理拓扑之间做联合搜索的问题。

更激进的思路是：软件层面能不能做**非均匀通信调度**？比如让冷 GPU 承担更多的 AllReduce 中继角色，或者让热 GPU 提前发出通信请求以避免忙等？这些方向目前还没有系统性的实验数据，但从 Georgia Tech 的发现中可以合理推论：**在热不平衡成为定局的环境里，均匀的通信策略在系统层面已经"不均匀"了，因为热已经让计算能力变得不均匀了。**

---

## 关键技术决策：Scale-Up vs Scale-Out，并行耦合的代价

### Scale-Up 还是 Scale-Out —— MoE 通信的答案可能相反

在云计算和传统 HPC 的圈子里，scale-up（更大的单体机器）vs scale-out（更多的小机器）是一个老生常谈的话题。通常的结论是：compute-bound 的工作做 scale-out 有优势（聚合算力大），communication-bound 的工作做 scale-up 有优势（减少跨节点通信）。

但 Georgia Tech 的数据给出了一组非常有意思的反直觉结果。

实验在两套 NVIDIA 集群上对比：**32×H200（scale-up 路径，每 GPU 有 141GB HBM3，4 个 HGX 节点）** 和 **64×H100（scale-out 路径，每 GPU 有 80GB HBM3，8 个 HGX 节点）**。两套系统的总显存容量接近（~4.5TB vs ~5.1TB），但架构差异巨大：
- H200 集群：GPU 数量少但单卡显存大，所有通信集中在一个 4 节点的拓扑内，NVSwitch 域大。
- H100 集群：GPU 数量多但单卡显存小，通信需要跨 8 个节点，IB 流量显著增多。

直觉是什么？64 块 H100 的聚合算力是 64 PFLOPS（FP16），32 块 H200 是 32 PFLOPS，算力差距 2×。如果 H100 能高效利用，吞吐应该是 H200 的两倍。

实测结果呢？

**对于 compute-bound 模型（如 Llama3-70B），H100 确实显著领先。** 这些模型的通信开销低，主要瓶颈在计算，H100 的 2× 算力优势直接转化为吞吐优势。H100 在所有 kernel 类型上的 compute 时间都更短（见图 3 的 kernel latency 分解）。

**但对于 communication-bound 模型（如 GPT3-175B 和 Mixtral-8x22B），差距急剧缩小，在某些配置下 H200 甚至反超。** 

用数据说话：
- GPT3-175B 的 TP2-PP16 配置下，H200 在吞吐和 energy-per-token 上双双超过了 H100。
- Mixtral-8x22B 的 EP8-TP1-PP4 配置下，H200 在能量效率上超过了 H100 —— 而能量效率的倒数是 completion time per joule，对训练成本来说比绝对吞吐更有意义。
- 即便在吞吐略逊的配置里，H200 的能量效率也是 H100 的显著倍数。因为当 GPU 的通信等待时间减少后，功耗浪费也跟着减少。

为什么 H200 在 communication-heavy 配置下有优势？

**第一个原因：更大的 HBM 减少了 TP 的需求。** H200 的 141GB HBM3 对比 H100 的 80GB —— 1.76× 的显存容量差。当模型的参数和激活值可以装进更少的 GPU 时，TP 宽度就可以减小（甚至变成 TP1）。TP 变窄的直接影响是什么？AllReduce 通信量减少。AllReduce 通信量减少的直接后果是什么？PCIe 和 IB 链路上的争用减少——这些被释放的带宽可以被 EP 的 All-to-All 使用，或者被 DP 的梯度同步使用。这是一个**通信总带宽的零和博弈**：一个维度的需求下降，就是另一个维度的供给增加。

**第二个原因：更少的节点意味着更少的 EP 跨节点通信。** 在 H200 的 4 节点集群里，当 TP 宽度减小后，更多的 GPU 在同一个节点内可以被分配给 EP。这让 All-to-All 通信在节点内部通过 NVLink/NVSwitch 完成，避免走 IB 链路。在 Mixtral-8x22B 的实验里，EP8-TP1-PP4 配置实现了 EP 通信的节点内本地化，这是 H200 反超 H100 的配置关键。

**第三个原因：scale-out 系统的 PCIe 争用更严重。** H100 的 8 节点集群需要更多的 IB 通信来做相同的工作。Georgia Tech 的数据显示，当 TP 和 EP 同时存在时，PCIe 链路上会出现大量稀疏的、小粒度的 SendRecv 调用。这些调用不能利用 data chunking（数据分块聚合），导致 PCIe 带宽的实际有效利用率很低。论文的图 6 直观地展示了这一点：PP-heavy 配置（如 TP2-PP16）的 PCIe 流量集中在少数几个边界 GPU 上，通过数据 chunking 提高了有效带宽利用率；而 TP-heavy 配置（如 TP8-FSDP）的 PCIe 流量分散、稀碎，有效带宽利用率远低于标称值。

Georgia Tech 的核心结论是：**"硬件潜力本身并不能保证效率"。** H200 的密度和显存优势只有在精心配置的并行策略下才能兑现——naive 地选择并行策略会让 H200 密度优势被流水线气泡和通信瓶颈抵消。但反过来，当通信密集型工作负载被有效限制在少数节点内部时，32 块 H200 的集群可以在性能和能效上匹配甚至超越 64 块 H100 的集群。

这个发现对 MoE 训练的启示是深远的。MoE 本质上是一个通信密集型负载（All-to-All 不可避免），而 H200 这种"少 GPU、大显存"的 scale-up 路径恰好与 MoE 的通信特点形成了结构性的互补。在 2024-2025 年的产业趋势中，我们已经看到这种互补在 Blackwell B200（192GB HBM3e）的发布中被进一步放大。

### TP+PP+EP+DP：四维并行在一条 PCIe 链路上打架

在现代大模型训练中，TP、PP、EP、DP 四种并行策略通常是组合使用的。Megatron-LM 提出的 PTD-P 策略（Pipeline-Tensor-Data Parallelism）和 DeepSpeed 的 3D 并行是这一领域的标杆方案。一个典型的多维并行配置可能是：节点内 TP=8（用 NVLink），节点间 PP=4（用 IB），全局 EP=8（用 IB），剩余用 DP 填充。

但在千卡集群中，这四个维度并不是各走各的路。它们**共享**一条物理链路，确切地说，它们共享 PCIe 交换机到 NIC（网络接口卡）之间的那一段"最后一公里"带宽。

Georgia Tech 的实验精确地暴露了这个问题。在分析 TP+PP 组合的通信效率时，他们发现：
> "TP+PP 的组合配置触发了跨 GPU 的稀疏、无协调的 SendRecv 调用，这些调用无法利用数据 chunking 或集体通信调度。这些通信模式导致 PCIe 和网络带宽的未充分利用，增加了通信延迟并使计算阶段停滞。"

具体来说，在 32×H200 集群的 GPT3-175B 训练中，TP8-PP4 配置的每个 pipeline stage 包含 8 个 TP rank，每个 rank 需要通过 IB 向下一 stage 的对应 rank 发送激活值。这意味着同一时间有 8 条并发的 IB SendRecv 流在竞争。如果没有 scatter/gather 优化（Megatron-LM 的通信优化，将相同数据跨 TP rank 做 scatter，再用 NVLink 做 all-gather），这 8 条流中的每一条都在传输**完全相同的数据**——因为每个 TP rank 在 MLP 的输出端持有的是相同的张量副本（被 all-reduce 复制过的）。

Megatron-LM 的 scatter/gather 优化正是针对这个冗余：在发送端通过 NVLink 将张量 scatter 成 8 份，每个 TP rank 只通过自己的 IB 链路发送 1/8 的数据，接收端再通过 NVLink all-gather 恢复完整张量。**这个优化把 IB 通信量除以 TP 宽度（通常是 8），直接减小了一个数量级。** 在 Megatron-LM 的 GPT-3 175B 训练中，scatter/gather 优化在 interleaved 1F1B schedule 下带来了 11% 的吞吐提升。

但是，**scatter/gather 优化只适用于 TP rank 之间的输出张量冗余场景。EP 的 All-to-All 通信不享这种冗余——每个专家接收和发送的是不同的 token hidden state，无法做 scatter/gather。** 这就意味着：当 EP 和 PP 同时存在时，EP 的 All-to-All 和 PP 的 P2P SendRecv 在 IB 链路上是直接的竞争对手。

在 NeMo 和 Megatron-LM 的 rank 映射顺序中，并行方式的映射顺序是 **TP → EP → DP → PP**。这意味着 rank 的编号规则是：先填满 TP 组（节点内），再填满 EP 组（节点间），再填 DP 组，最后按 PP 分。这个顺序的一个隐含后果是：**同一 pipeline stage 内的 GPU 可能来自不同的 EP 组，导致 All-to-All 在不同 pipeline stage 之间以不同的带宽模式运行。**

对用户来说，这种耦合表现为一个令人困惑的现象：调大 EP 宽度（更多专家并行度），PP 的吞吐下降了；调大 PP 深度（更多 pipeline stage），EP 的 All-to-All 延迟上升了。这两个维度不是独立的——它们在争夺同一条 IB 链路。

### Alpa 的自动化视角：当搜索空间大到人类无法穷举

到这里，一个问题会自然浮现：如果 TP、PP、EP、DP 之间有如此复杂的相互作用，而且最优配置又高度依赖模型结构和硬件拓扑，那么**人类手动调参的空间还够吗？**

UC Berkeley 的 Alpa 系统（OSDI'22）给出了答案：**不够。** Alpa 是第一个将并行策略搜索形式化为两级优化问题的编译器系统——inter-operator parallelism（PP，跨 stage 的粗粒度流水线）和 intra-operator parallelism（TP/DP，算子的细粒度划分）在两个层级上分别优化，通过 DP（Dynamic Programming）+ ILP（Integer Linear Programming）联合求解。

这套形式化方法的威力在 GShard-MoE 上的结果最能说明问题：

- 在 **2 个节点（16 GPU）** 上，Alpa 自动发现的最优 MoE 并行策略，比手动调参的 DeepSpeed 快了 **3.5×**。
- 在 **4 个节点（32 GPU）** 上，Alpa 的自动策略比 DeepSpeed 快了 **9.7×**。

9.7 倍的差距不是来自更好的 kernel 实现——两种系统跑的是同一套 XLA 后端——而是来自并行策略的差异。DeepSpeed 的 MoE 方案采用的是 GShard 的手工专家并行规则：对 MoE 层的 expert axis 做 partition，对非 expert 层切换回数据并行。这个方案的基础假设是：expert 相关的通信是唯一的瓶颈，所以只优化这一维。但在一台用 25Gbps 跨节点带宽互联的 AWS p3.16xlarge 集群上，通信瓶颈不只在 All-to-All 上——PP 的 P2P 通信和 DP 的 AllReduce 也都是瓶颈。DeepSpeed 没有 inter-op parallelism（因为没有 pipeline parallelism），所以跨节点扩展时通信开销线性增长。

**Alpa 做了什么不同的事？**

Alpa 发现的最优策略是：同时在 intra-op 层级使用 expert parallelism + ZeRO 数据并行（ILP 求解器在数千种算子划分方案中选出来的），同时在 inter-op 层级用 pipeline parallelism 将模型切成多个 stage，让每个 stage 的 CPU 通信量控制在设备 mesh 内部（高带宽链接），只有必要的跨 mesh 通信才走慢链接（25Gbps 跨节点网络）。这个策略组合是 **ILP + DP 两级搜索得到的**，不是人类工程师在配置文件中手工指定的。

Alpa 在 GPT 模型上的结果同样具有启发性。Alpa 自动生成的 GPT 并行策略与 Megatron-LM 的手工最佳策略高度相似——都是均匀 stage 划分，都在显存不紧张时优先用 data parallelism——但 Alpa 额外发现了一个人类工程师忽略的机会：**在原版 Megatron-LM 没有开启的 weight update sharding（即 ZeRO 优化器状态分片）**。这个"多出来"的优化让 Alpa 在 GPT 任务上的吞吐略微超过了 Megatron-LM 的最佳手工配置。

Alpa 的论文里有一句话点出了自动搜索的真正价值，我把它翻译成大白话就是：**你能想到的并行配置，搜索算法都能想到；你没想到的，搜索算法也可能找到。因为搜索算法没有"DP 和 TP 应该分开看"的惯性思维，它只是在一个组合空间里找最小值。**

对于 MoE 训练，这个空间的维度远比 Dense 模型高——因为多了一个 EP 维度，以及 EP 内部的 expert 到 GPU 的映射方式——这使得手工搜索的覆盖基本上不可能穷尽。而 Alpa 在 GShard-MoE 上的 9.7× 加速说明：**这个搜索空间里存在大量人类直觉会忽略的高价值配置。**

### Megatron-LM 的 PTD-P：实践中的权衡

回到手工调参的实践世界，Megatron-LM 在 2021 年 SC 会议上提出的 PTD-P 策略至今仍是训练千亿、万亿参数 Dense 模型的工业标准。它的核心原则有三条：

**原则一：TP 用于节点内，PP 用于节点间。** TP 的 AllReduce 通信对延迟极度敏感——每层 transformer 的 forward 和 backward 各需要两次 all-reduce——高频、小数据量。这种模式在 NVLink（高带宽、低延迟）上运行高效，在 IB（较低带宽、较高延迟）上运行就是灾难。PP 的 P2P 通信则是低频、大数据量——只在 pipeline stage 边界发生——适合 IB 链路的特征。

**原则二：数据并行（DP）是扩展训练规模的主力。** DP 的梯度 AllReduce 每步只发生一次，可以摊薄到全局 batch size 的水平。在 Megatron-LM 的 PTD-P 框架里，DP 的通信开销随 DP 规模增长以 (N-1)/N 的比例摊销——越大的 DP 组，单个 GPU 的 AllReduce 开销反而越小（因为 ring-based AllReduce 的每 GPU 通信量是 2×(N-1)/N×G，G 是梯度大小）。

**原则三：Microbatch size 的选择取决于模型特性。** Megatron-LM 的分析模型给出了 microbatch size 的最优值计算方式，核心权衡是：大 microbatch → 好 GPU 利用率 → 但大 pipeline bubble（因为 microbatch 数 m 减小）；小 microbatch → 小 pipeline bubble → 但低 GPU 利用率。在两个方向之间找到平衡点，在 Megatron-LM 的实验中，最优 microbatch size 可以比次优选择高 15% 的吞吐。

但 Megatron-LM 当年的实验有一个重要的背景：**它跑的是 Dense 模型，不是 MoE。** 在 Dense 模型中，TP 是唯一的 collective 通信大户，PP 的 P2P 通信量适中。**在 MoE 中，EP 的 All-to-All 成为了第二个 collective 通信大户**，而且不像 TP 那样可以被限制在节点内。Megatron-LM 论文没有给出在 MoE 负载上 PTD-P 的表现数据，但从其分析框架可以推断：

- EP 的 All-to-All **绝对不能**和 TP 的 AllReduce 同时使用在节点间链路上。如果 EP 和 TP 都跨节点，两股 collective 通信流会把 IB 链路塞成一个巨大的瓶颈。
- EP 和 PP 可以共存，但需要仔细分配——EP 的 All-to-All 和 PP 的 P2P SendRecv 会共享 IB 带宽。优先让 EP 通信本地化（把同一 EP group 的 GPU 放在同一节点或相邻节点）是减少争用的关键。
- 如果 HBM 足够大（如 H200 的 141GB），可以选择 TP=1（不需要 tensor 并行），释放出 IB 带宽专门用于 EP All-to-All。这恰好与 Georgia Tech 的 scale-up 优势结论一致。

Megatron-LM 最著名的工程成果——在 3072 块 A100 上训练 1 万亿参数模型，达到 502 petaFLOP/s 的聚合吞吐（每 GPU 52% 理论峰值），effective bisection bandwidth 为 pipeline 通信 892 GB/s、data parallel 通信 12.9 TB/s——都是在 Dense 模型上实现的。在 MoE 负载上复现这个级别的效率，需要解决 EP 通信引入的全新瓶颈。

---

## 横向对比：从 SPMD 到异步数据流

### Megatron-LM：PTD-P 的工业实践

Megatron-LM 的 PTD-P 代表的是 **SPMD（Single Program Multiple Data）+ 同步并行**的流派。它的核心假设是：所有 GPU 在任意时刻执行相同的程序，通过同步屏障（pipeline flush）来保证严格的 optimizer 语义。

这套假设的优点是**实现简洁、优化器语义严格、可复现性好**。缺点是**同步屏障本身就是效率损耗**。在 Megatron-LM 的实验中，interleaved 1F1B 调度（每个设备分配 v 个 pipeline stage chunk，交替执行）相比 default 1F1B 调度，吞吐提升在 10% 以上。但如果 v 过大，通信量也增大——"interleaved 调度的通讯量增加了 v 倍"，在没有 scatter/gather 优化时，大 batch size 下 interleaved 调度甚至会逊于 default 调度。

Megatron-LM 本身的实验有一个很说明问题的数字：在 GPT 模型中，**次优的 TP 和 PP 组合可能导致吞吐下降 2 倍**。2 倍——这是在同一个模型、同一批硬件上、只是改了并行配置的差异。这个数字的幕后推手就是前面讨论的并行耦合：错的 TP 宽度→ IB 链路被 AllReduce 吞掉→ PP 的 P2P 通信排队→ pipeline bubble 被放大→ 整个系统的 GPU 利用率崩塌。

### Pathways：异步数据流 + gang-scheduling 的 Google 方案

Google 在 2022 年 MLSys 上发表的 Pathways，代表的是一个与 Megatron-LM 截然不同的流派：**单控制器（single-controller）+ 异步数据流（asynchronous dataflow）+ gang-scheduling**。

Pathways 的核心设计选择是**放弃 SPMD 的"每个 GPU 跑相同代码"假设**，转而采用 MPMD（Multiple Program Multiple Data）：不同的 accelerator island 可以执行不同的编译函数（compiled function），通过一个中心化的资源管理和调度层来协调。这个设计使得：
- Pipeline 的不同 stage 可以运行在不同形状的 accelerator mesh 上——不需要像 Megatron-LM 那样所有 stage 有相同的并行度。
- 跨 island 的通信通过 DCN（Data Center Network）异步完成，不要求 SPMD 的同步屏障。
- Centralized scheduler 可以跨程序做 gang-scheduling，确保通信 collectives 在共享设备上按一致的顺序执行。

Pathways 的实战成绩：在 **6144 TPU v4** 上训练 PaLM（Pathways Language Model），达到 **57.8% MFU（Model FLOPS Utilization）**。考虑到 PaLM 的规模（540B 参数）和 TPU v4 的集群复杂度（多个 pod 通过 DCN 连接），57.8% 的 MFU 是一个非常扎实的数字。

Pathways paper 的几个关键发现：
1. **Pipelined execution 可以和 SPMD 竞争甚至超越。** 在一个 3B 参数的 Transformer 模型上，Pathways 的 16-stage 流水线吞吐（131.4k tokens/s）与 SPMD 相当（125.7k tokens/s）。原因是 SPMD 的 collective 通信在这个规模上的开销已经超过了 pipeline bubble 的开销。
2. **跨 TPU pod 的训练是可行的。** 当模型被切分到 4 个 island（每个 32 个 TPU core）并通过 DCN 连接时，Pathways 达到了与单个 128-core island 完全相同的吞吐——131.4k tokens/s。DCN 传输和计算实现了有效的重叠。
3. **大规模跨 island 训练效率很高。** 在 136B 参数模型的训练中，使用两 island 配置并通过 DCN 做梯度 reduction（跨岛数据量 1030GB），Pathways 达到了单 island 双倍设备方案的 **~97% 吞吐**。3% 的跨岛通信损耗，对于跨 DCN 的分布式训练来说，是令人印象深刻的。

Pathways 对 MoE 通信的启示在于**异步性**。MoE 的 All-to-All 通信天然是一个异步友好的模式——一个 token 发往一个专家，不需要等待其他 token 也完成分发。如果系统的通信层支持 fine-grained 的异步数据交换（Pathways 的 PLAQUE 数据流引擎正是为此设计的），那么 All-to-All 的延迟可以被部分隐藏（通过重叠计算），而不会全部成为同步屏障上的等待时间。这是 SPMD 架构在 MoE 场景下的一个结构性劣势——它要求所有 GPU 在同一时刻完成通信，这放大了 straggler 效应。

但 Pathways 也有它的限制。它的设计深度绑定在 TPU 架构上——TPU 的 XLA 编译器可以编译包含集体通信的长时间运行函数，而 GPU 的通信（NCCL）是由 host 端发起的，两者的异步粒度不同。把 Pathways 的理念移植到 GPU 集群上，需要解决 host-side dispatch latency 的问题（Pathways 的 parallel asynchronous dispatch 正是在解决类似问题）。

### Alpa：用编译器的"上帝视角"重新理解并行

前文已经讨论了 Alpa 在 MoE 上的自动并行策略成果。这里从方法论层面做一个总结对比。

Alpa 把模型并行问题重新分类为 **inter-op**（算子间，对应 PP）和 **intra-op**（算子内，对应 TP+DP+ZeRO）两个层级，这个二分法和 Megatron-LM 的 PTD-P 以及 Pathways 的 MPMD 都有对应关系，但 Alpa 的独特贡献在于：

1. **Inter-op 层级使用了 DP（动态规划）搜索。** 不是手工指定每个 stage 包含多少层，而是让 DP 遍历所有可能的阶段-设备 mesh 配对，找到流水线总延迟最小的那个。DP 的复杂度是 O(K³ NM(N+log(M)))（K 是算子数，N×M 是设备 mesh 尺寸），在算子聚类优化后实际可执行。
2. **Intra-op 层级使用了 ILP（整数线性规划）求解。** 每种算子的并行方案（如 matmul 的 data parallel vs tensor parallel vs ZeRO sharding）被枚举为 ILP 变量，resharding 成本被建模为约束，ILP 求解器在数千种组合中找到通信成本最小的方案。
3. **两级搜索之间存在信息反馈。** Inter-op 的 DP 需要知道每个 stage-mesh pair 的最低代价（由 Intra-op 的 ILP 返回）。这意味着 inter-op 的 stage 划分决策是在充分了解每个 stage 的最优 intra-op 配置的前提下做出的——不是先定 stage、再调 intra-op，而是一起优化。

这三个设计的合力效果是：Alpa 能发现**非均匀**的并行策略——不同 pipeline stage 可以有不同的 intra-op parallelism 配置，不同 layer 的算子划分方式也可以不同。在 Wide-ResNet 这种异构模型上，Auto-sharding 找到的策略（前几层用 data parallelism、后几层用 channel partitioning，stage 间负载不均但总延迟最小）是人类几乎不可能手动指定的。这种非均匀策略在 MoE 模型上同样有价值——MoE 层的通信模式和 Dense 层完全不同，统一的并行配置会在两者之间产生不必要的通信折损。

---

## 总结与清单：千卡集群训练的七条全系统法则

### 核心认知

千卡集群训练 MoE 模型，性能瓶颈从来不是单一的。**通信是显性的瓶颈，功耗是隐性的瓶颈，热是不可忽视的瓶颈放大器。** 三者构成了一条环环相扣的因果链，任何一个环节的恶化都会通过同步机制传递到全局。

从 Georgia Tech 的全系统数据、Megatron-LM 的工业实践、Alpa 的自动搜索方法和 Pathways 的异步架构中，可以提炼出七条全系统法则。

### 法则一：通信-计算-显存三角不可解耦
不要孤立地优化通信或计算或显存。它们是一个三角。MoE 减少计算（稀疏激活）→ 需要更高并行度 → 通信增大 → GPU 空闲 → 功耗效率下降。在千卡级别，这个三角的任何一边被挤压都会在另外两边产生连锁反应。

### 法则二：Scale-up 对 MoE 有结构性优势
H200 的大显存 + 少节点架构 = 减少 TP 需求 + 增强 EP 本地化 = 更好的通信效率。但只有在并行策略与拓扑结构联合优化的前提下，scale-up 的潜力才能兑现。Naive 的并行配置会让 H200 的密度优势完全被抵消。

### 法则三：Microbatch size 过犹不及
大 microbatch 摊薄了通信频次，但引入了三个隐藏成本：(a) pipeline bubble 比例增大，(b) 突发性执行模式诱发功耗尖峰，(c) 热节流因峰值功耗而触发。最优 microbatch size 不是最大能装进显存的那个值，而是上述三个成本的边际平衡点。

### 法则四：EP 的 All-to-All 要尽可能本地化
EP 的 All-to-All 不像 TP 的 AllReduce 那样可以通过 scatter/gather 来压缩数据量。每个 token 的 hidden state 都必须跨 GPU 传输，因此物理距离就是通信开销的直接代理。让同一 EP group 的 GPU 共处一个 NVSwitch 域（节点内），是减少 All-to-All 延迟的最有效手段。

### 法则五：热不平衡破坏同步假设
GPU 集群存在系统性的热不平衡——气流下游的 GPU 恒比上游的热 15-25°C。当热 GPU 被降频，所有 GPU 都必须等待（同步训练）。均匀的通信和计算调度在这种情况下已经不再"均匀"。热感知的 stage 分配和通信调度——让冷 GPU 多干活、让热 GPU 少等待——是改善总体效率的方向。

### 法则六：手动调参已经触及天花板
Alpa 在 GShard-MoE 上比手工 DeepSpeed 快 9.7× 的结果说明：在 TP×PP×EP×DP 的四维搜索空间里，人类直觉能覆盖的只是冰山一角。自动化并行策略搜索（无论是 ILP+DP 还是强化学习）正在成为必需。

### 法则七：异步架构是对 MoE 通信"不均衡"的结构性答案
Pathways 的异步数据流 + gang-scheduling 在 PaLM 训练中达到 57.8% MFU。对于 MoE 的 All-to-All（天然的动态不均衡通信），异步执行允许快的 GPU 不等慢的 GPU，减少 straggler 的放大效应。SPMD 的同步屏障在 MoE 场景下是一个越来越沉重的枷锁。

### 决策清单

以下是一个供工业工程师和研究者参考的千卡 MoE 训练决策清单：

- [ ] **拓扑分析**：集群的 NVSwitch 域有多大？IB 拓扑是什么？GPU 的物理位置导致的温度差是多少？
- [ ] **并行维度分配**：TP 是否严格限制在节点内？EP group 是否与 NVSwitch 域对齐？PP 的 stage 边界是否落在节点边界上以最小化 IB 通信？
- [ ] **HBM 容量判断**：单 GPU 的 HBM 是否足够装下没有 TP 的模型？（如果 141GB H200 够用 1 个 expert，就避免跨节点 TP）。
- [ ] **Microbatch 调优**：是否从最小可行 microbatch 开始往上试，而非从最大能装下的开始往下试？是否同时监控了功耗和温度曲线？
- [ ] **通信优化**：是否开启了 scatter/gather 优化（Megatron-LM）？跨 stage 通信是否避免了冗余？EP 的 All-to-All 是否使用了 NCCL 的 AllToAll 还是自己实现的 P2P 方案？
- [ ] **热均衡**：训练日志中是否有 GPU 间的温度差异超过 15°C？高温度的 GPU 是否对应了更长的 step time？是否可以为"冷 GPU"分配更高的 microbatch 或更重的工作？
- [ ] **自动化搜索**：对于新的模型/集群组合，是否考虑过自动并行搜索（如 Alpa）来发现在手工搜索空间中可能被忽略的策略？

千卡级别的训练不是 1000 个单卡的简单叠加。物理学——功率密度、热传导、信号衰减——在千卡尺度上以非线性的方式介入。通信是一个系统问题，但它的代价是通过功耗和热来支付的，而功耗和热的代价最终又通过同步机制均摊到每一个通信参与者身上。

第十二篇我们将继续探讨跨集群训练的最前沿实践，第十五篇将讨论 Chiplet 架构对 MoE 通信的全新影响。下一篇见。

---

**参考文献与数据来源**

1. Go et al., "Characterizing the Efficiency of Distributed Training: A Power, Performance, and Thermal Perspective," MICRO 2025. (Georgia Tech 全系统功耗/热/并行研究)
2. Narayanan et al., "Efficient Large-Scale Language Model Training on GPU Clusters Using Megatron-LM," SC 2021. (PTD-P, 502 PFLOPS on 3072 A100)
3. Zheng et al., "Alpa: Automating Inter- and Intra-Operator Parallelism for Distributed Deep Learning," OSDI 2022. (ILP+DP 自动并行，GShard-MoE 9.7× speedup)
4. Barham et al., "Pathways: Asynchronous Distributed Dataflow for ML," MLSys 2022. (PaLM 57.8% MFU on 6144 TPU v4, 异步数据流)
