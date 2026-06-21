---
layout:     post
title:      GPU NoC Physical Foundation
subtitle:   GPU NoC Physical Foundation
date:       2026-06-21
author:     George Lin
header-img: img/post-bg-gen.png
catalog: true
tags:
- moe
---

## 引言：170 个周期的延迟差意味着什么？

在 MoE 训练中，我们通常把通信的焦点放在 GPU 之间——NVLink 够不够快、InfiniBand 有没有阻塞、All-to-All 要不要走 NVSwitch。但在这一切发生之前，**token 的 hidden state 必须先在本 GPU 内部从计算核心（SM）走到高速缓存（L2），再从 L2 走到片外 HBM 的发送缓冲区**。这一段路程——片上网络（Network-on-Chip, NoC）——经常被视为"固定成本"而被忽略。

它不应该被忽略。

2024 年，KAIST、UBC 和 Microsoft 的研究者在 MICRO 上发表了一项研究，首次系统地实测了 NVIDIA V100、A100、H100 三代 GPU 的片上网络特性。他们在实测中发现了一个惊人的数字：**从同一个 SM 访问不同物理位置的 L2 分片，延迟可以差到 70%**。在 V100 上，最快 175 个周期，最慢 248 个周期——差了 73 个周期。在 A100 上，跨 GPU 分区的 L2 访问延迟可以从 ~200 个周期跳到 ~400 个周期。

这些数字对 MoE 有直接影响。在 All-to-All dispatch 的第一步，每个 SM 上的 token 数据从寄存器走到 HBM 发送缓冲区的过程中，要穿过这个 NoC。如果某些 token 的路径比其他 token 长——物理上更长——那么 dispatch 操作的完成时间就不是均匀的。在同步通信模型下，发完最快的一批 token 的 SM 必须等待最慢的那个 SM 完成——因为 All-to-All 在所有参与 GPU 间有一个隐式的 barrier。

**这就是我们要从芯片内部开始讲通信的原因：你无法在 node 间消除一个从 chip 内部就没有解决的延迟问题。**

---

## 第一层：GPU NoC 到底是什么？

### 不仅仅是"连线"

GPU 的 NoC 不是一个简单的全互联矩阵（crossbar）——至少在现代 GPU 上不是。随着 GPU 规模的扩大（V100 80 SM → H100 132 SM），单纯的全互联 crossbar 需要 O(n²) 的连线，面积和功耗都无法接受。

因此，NVIDIA 采用了**层次化互联拓扑**。虽然 NVIDIA 没有公开过内部 NoC 的精确拓扑图，但基于芯片照片（die photo）、性能实测和结构推断，学术界达成了相对一致的认知：

```
                     ┌─── HBM stack ───┐
                     │                  │
        ┌────────────┤  Memory Controller (MC) + L2 slices  ├────────────┐
        │            │                  │            │
        │            └──────────────────┘            │
        │                                            │
   ┌────┴────────────────────┐              ┌────────┴───────────────────┐
   │   GPU Partition Left    │              │    GPU Partition Right     │
   │   ┌─── GPC 0 ───────┐  │              │   ┌─── GPC 4 ───────┐      │
   │   │ TPC0 TPC1 ... │  │              │   │ TPC0 TPC1 ... │      │
   │   │ [SM0][SM1]... │  │              │   │ [SM64][SM65]..│      │
   │   └────────────────┘  │              │   └────────────────┘      │
   │   ┌─── GPC 1 ───────┐  │              │   ┌─── GPC 5 ───────┐      │
   │   │ ...            │  │              │   │ ...            │      │
   │   └────────────────┘  │              │   └────────────────┘      │
   └────────────────────────┘              └────────────────────────────┘
```

这个层次结构的关键层级是：

| 层级 | V100 | A100 | H100 | 说明 |
|------|------|------|------|------|
| **SM** | 80 | 108 | 132 | 基本计算单元 |
| **TPC** | 2 SM/TPC | 2 SM/TPC | 2 SM/TPC | 2 SM 通过 mux 共享 NoC 入口 |
| **CPC** | 无 | 无 | 2-3 TPC/CPC | H100 新增的中间层级 |
| **GPC** | 6 GPC, 最多 14 SM/GPC | 7 GPC, 最多 16 SM/GPC | 8 GPC, 最多 18 SM/GPC | 图形/计算处理集群 |
| **GPU Partition** | 无（单分区） | 2 分区（左右） | 2 分区（左右） | A100/H100 引入，中央 crossbar 分割 |
| **L2 Cache** | 6MB, 32 分片 | 40MB, 80 分片 | 50MB, 80 分片 | L2 分片分布在 Memory Controller 附近 |

**每一级之间的带宽不是线性的。** SM 的输出带宽在 TPC 层被 2:1 共享，在 GPC 内被进一步共享。这种层级化带宽共享意味着——物理上靠近某个 L2 分片的 SM 在通信路径上比远离那个分片的 SM 有结构性的优势。

### Hierarchical Crossbar + 输入加速（Input Speedup）

实测数据表明，GPU NoC 的带宽供给遵循一个关键原则：**输入加速（input speedup）**。

输入加速是指：当多个 SM 同时向 NoC 注入流量时，NoC 提供的总带宽并不简单等于"SM 数 × 单个 SM 的带宽"。因为 SM 共享 TPC 和 GPC 级别的带宽入口，如果 input speedup 不够，多个 SM 同时通信时每个 SM 能拿到的带宽会下降。

论文的实测结果：

- **TPC speedup（读）**：V100/A100/H100 三代都达到了满带宽——即 2 个 SM 同时读取时，每个 SM 都能获得和单独读取时相同的带宽
- **TPC speedup（写）**：V100 只有 ~1.09×（远不到满带宽的 2×），而 A100 和 H100 达到了满带宽——说明写入带宽共享在 V100 代是个瓶颈，后来被修复了
- **GPC 级 speedup（local）**：V100 只达到 ~50% 的满带宽（即 7 个 TPC 全部注入时，平均每个 TPC 只有单独时的一半带宽），H100 接近 85%
- **GPC 级 speedup（global，跨分区）**：A100 和 H100 相比 V100 有显著提升——说明新一代 GPU 的跨分区 NoC 带宽被大幅加强

**这对 MoE 的启示**：在 All-to-All dispatch 的节点内阶段，所有 SM 上的 token 都在竞争 NoC 入口。如果 dispatch kernel 的设计没有考虑 input speedup 的限制——比如让太多 SM 同时写 L2——实际的通信带宽可能远低于理论的 L2 带宽，导致 dispatch 延迟被人为拉长。

---

## 第二层：延迟的非均匀性 —— 实测数据

### V100：同一个 GPC，同一个人，不同的命运

论文中的 Figure 1(a) 是整篇研究最核心的一张图。它展示了从 V100 上的 SM24 到所有 32 个 L2 分片的访问延迟分布：

- 最低延迟：~175 cycles
- 最高延迟：~248 cycles
- 平均延迟：~212 cycles
- 差值：73 cycles，约 **33% 的非均匀性**

同一个 SM，同样的访问类型（L2 hit），同样的数据大小——唯一不同的就是 L2 分片的物理位置。73 个周期在 1.38GHz 的 V100 上大约是 53 纳秒。对于一个 GPU kernel 来说这就是几个 warp 切换的开销，但对于一个紧耦合的通信循环来说，这 53ns 在几千次重复后累积成可感知的延迟差异。

更值得注意的是 GPC 间的差异。论文的 Figure 1(b) 和 Figure 2 显示：

- GPC0 的延迟分布：μ = 213 cycles, σ = **13.9 cycles**——宽分布，延迟变化大
- GPC2 的延迟分布：μ = 209 cycles, σ = **7.5 cycles**——窄分布，延迟更均一

为什么 GPC2 的延迟更均一？因为它位于芯片中间（根据芯片照片的推断），物理上到达所有 L2 分片的路径长度差异更小。而 GPC0 位于芯片边缘，离一侧的 L2 分片近但离另一侧远。

**这意味着，一个 GPU kernel 在 GPC2 上运行和 GPC0 上运行，其通信延迟的确定性是不同的。** 对于 MoE dispatch，如果你不幸把通信密集的 SM 线程块调度到了 GPC0，你会观察到比 GPC2 更大的 tail latency。而在同步 All-to-All 中，tail latency 决定全体的等待时间。

### V100 的物理布局：延迟的地图

论文通过系统地测量 SM-L2 分片的延迟对，并结合 V100 的芯片照片（die photo），推断出了近似的逻辑布局（Figure 4）。关键发现：

- **延迟由 SM 在 GPC 中的位置 + L2 分片在 Memory Partition 中的位置共同决定**
- 物理上靠近的 SM-L2 对（如 SM64 和 L2 slice 17）延迟最低（180 cycles）
- 物理上远离的 SM-L2 对（如 SM4 和 L2 slice 15）延迟最高（217 cycles）
- 同一个 GPC 内的 SM 具有高度相关的延迟分布——Pearson 相关系数可达 0.998

论文使用了 Pearson 相关系数热力图（Figure 6）来展示 SM 之间的延迟模式相似性：

- V100：同一 GPC 内的 SM 几乎完美相关（r ≈ 1.0），相邻 GPC 间高度相关（r ≈ 0.8-0.9），远处的 GPC 间相关性显著降低
- 特别是两组 GPC（GPC0&1 和 GPC4&5）在延迟模式上相似——它们很可能在芯片的两端对称位置——而中间的 GPC2&3 具有不同的延迟特征

**这个物理布局信息对 MoE kernel 的启发是直接的**：如果你能控制 token dispatch 时 SM 的分配——比如让处理 dispatch 操作的 SM 尽量靠近目标 L2 分片——你可以节省显著的单跳延迟。这不是一个容易的优化（SM 分配通常由 GPU 线程块调度器自动决定），但在设计 kernel 时值得考虑。

### A100：GPU 分区的代价

A100 引入了 GPU 分区（partition）的概念——芯片被一条中央 crossbar 分成左右两个分区，每个分区有自己的 GPC、L2 分片和 Memory Controller。

论文的 Figure 6(b) 显示 A100 的 Pearson 相关热力图相比 V100 有显著变化：同一个 GPC 内的 SM 不再具有完美相关性，延迟分布更加碎片化。**芯片变大了，物理距离的差异也变大了。**

但真正的发现在 Figure 8(b) 中：**跨分区 L2 访问的延迟惩罚**。

当 SM 访问同分区（near partition）的 L2 分片时，延迟约 200-250 cycles——与 V100 类似。但当 SM 访问跨分区（far partition）的 L2 分片时，延迟跳到约 400 cycles——**几乎翻倍**。这 150-200 cycles 的额外延迟来自数据在 GPU 内部穿越中央 crossbar 的往返路径。

换句话说，A100 的 NoC 实际上是一个**两跳网络**：SM → 同分区 L2 是一跳，SM → 中央 crossbar → 跨分区 L2 是两跳。这种两级延迟结构意味着一个 SM 的"通信带宽"取决于它访问的数据分布在哪个分区。

### H100：新层级（CPC）和分布式共享内存

H100 引入了一个新的层级——CPC（Compute Processing Cluster）——在 TPC 和 GPC 之间。这是 NVIDIA 官方文档没有明确标注的层级，但论文通过详细的延迟和带宽测量确认了它的存在：

- 一个 GPC 包含 3 个 CPC
- 每个 CPC 包含 2-3 个 TPC（4-6 个 SM）
- CPC 内部有 SM-to-SM 网络——支持 H100 新引入的**分布式共享内存**（Distributed Shared Memory, DSM）

论文的 Figure 7(b) 展示了 CPC 间的 SM-to-SM 通信延迟：

- CPC0 内部 SM 间通信：196 cycles
- CPC2 内部 SM 间通信：~213 cycles
- **CPC 间通信的延迟随 CPC 的距离增加而增加**

这个 SM-to-SM 网络是 H100 的一个重要创新——它允许一个 SM 直接访问另一个 SM 的共享内存（shared memory），而不需要通过 L2。对于 MoE 来说，这意味着如果两个 token 的专家碰巧在同一个 CPC 内（或者更理想，同一个 SM 的不同 warp），token 的 hidden state 可以通过 SM-to-SM 网络快速传递，而不走完整的 L2 路径。

H100 在跨分区延迟上也做了优化。论文的 Figure 8(c) 显示 H100 的 L2 延迟分布比 A100 更均匀——NVIDIA 的 Hopper 架构白皮书提到 H100 的 L2 "caches data for memory accesses from SMs in GPCs directly connected to the partition"，这意味着 L2 缓存策略被设计成倾向于缓存本分区的数据，减少了跨分区访问的频率。

但在 L2 未命中时的跨分区惩罚仍然存在——Figure 8(f) 显示 H100 的 L2 miss penalty 不是恒定的，而是依赖于数据被缓存在哪个分区。这为 MoE dispatch 的缓冲区放置增加了一层复杂性：**如果你把 dispatch 发送缓冲区放在"错误"分区控制的 HBM 区域，你在 L2 miss 时会付出额外的往返延迟。**

---

## 第三层：带宽的均匀性 —— 好消息和坏消息

### 好消息：带宽基本均匀

论文的一个核心发现是：**延迟虽然不均匀，但带宽基本是均匀的。**

- V100 上单个 SM 对一个 L2 分片的带宽：平均 ~34 GB/s，σ = 0.147 GB/s——分布极窄
- V100 上一个 GPC 对一个 L2 分片的带宽：平均 ~85 GB/s，σ = 0.06 GB/s——分布更窄

这说明 GPU NoC 的带宽分配被精心设计成对所有 SM-L2 对公平——即使某些路径有更高的延迟，带宽并没有被牺牲。这是通过在 NoC 内部提供足够的**输入加速（input speedup）**来实现的——更远的路径使用更宽的内部链路来补偿额外的延迟。

**带宽均匀性对 MoE 的意义**：All-to-All dispatch 是一个带宽密集操作（每次传输大量 token 数据），而不是延迟敏感操作。因此，只要带宽是均匀的，延迟的非均匀性对 dispatch 总吞吐量的影响比我们直觉上想象的要小——除非延迟差异造成了 SM 级别的负载不均。

### 坏消息：跨分区带宽下降

A100 引入了 GPU 分区后，跨分区 L2 的带宽不再是均匀的。

论文的 Figure 12 揭示了这一现象：

- SM0（位于左分区）访问左分区 L2（near partition）：~39.5 GB/s
- SM0 访问右分区 L2（far partition）：~26 GB/s——**下降约 34%**
- SM2（位于右分区）访问左分区和右分区的带宽互换——但模式相同

这种带宽不对称在小并发场景下尤为明显。论文的 Figure 14 显示，当只有 1-2 个 SM 发送流量时，跨分区带宽的降幅可达 **28%**。只有当足够多的 SM 同时发送流量（~8 个 SM 以上）时，跨分区带宽才能饱和到和同分区相近的水平。

**根据 Little's Law**，带宽 = 并发请求数 / 平均延迟。跨分区延迟更高 → 单个 SM 在远端 L2 上的"飞行请求"（in-flight requests）更少 → 带宽更低。当更多 SM 同时发送请求时，飞行请求总量增加，带宽才得以饱和。

这对 MoE dispatch 的启示是：如果 dispatch kernel 只使用了少量的 SM 和少量的通信 warp（比如 DeepSeek-V3 只用 20 个 SM 做通信），那这些 SM 的物理位置和它们访问的 L2 分片的位置关系就很重要——如果通信 SM 集中在左分区但 dispatch 缓冲区的数据在右分区的 HBM 上，跨分区带宽的下降会让 dispatch 吞吐受到惩罚。

### H100 的"补丁"：局部化 L2 缓存

H100 对这个问题做了一个架构级的修补——L2 缓存被配置为倾向于缓存来自"直接相连的 GPC"的数据。这意味着如果 MoE dispatch kernel 处理的数据碰巧被缓存在本地分区的 L2 中（这在 dispatch kernel 刚写入数据后是常见的——数据还在 L2 中），H100 的延迟和带宽都会很优秀。但如果数据需要从另一个分区的 HBM 读取（L2 miss），惩罚依然存在。

---

## 第四层：从 NoC 到 MoE —— 为什么这对你很重要

### 1. Token dispatch 的 tail latency 问题

在 All-to-All dispatch 中，所有参与 GPU 执行一个逻辑上同步的操作：每个 GPU 把自己的 token 分发到其他 GPU。在节点内（NVLink 域），这个分发是并行的——GPU 0 向 GPU 1、2、3... 同时发送数据。但**发送的完成时间由最慢的一条链路决定**。

内部 NoC 的非均匀延迟在这条链路的最前端引入了一个变数：GPU 0 上的 SM 在把数据从 register file 搬到 NVLink 发送缓冲区时，经过的 NoC 路径不同，延迟也不同。如果某些 SM 的路径延迟比其他 SM 高 30-50%，dispatch 的有效完成时间就会被这些"慢 SM"拖累。

在 NVSwitch 的全互联架构下（所有 GPU 同时收发），这个问题被放大了：**所有 GPU 上的最快 SM 都在等最慢的 SM。**

### 2. 通信 SM 的放置策略

DeepSeek-V3 的经验是：把 20 个 SM 专门划出来做通信。这些 SM 不做计算，只做数据搬运（IB↔NVLink 转发）。那么这 20 个 SM 应该选哪些？

根据论文的分析，最佳策略是：**选择分布在不同 GPC 上的 SM**，而不是把通信 SM 集中在一两个 GPC 中。原因有两个：

- **NoC 带宽共享**：同一 GPC 内的 SM 共享 GPC 级的 NoC 入口带宽（input speedup 有限）。如果通信 SM 集中在同一个 GPC，它们的有效总带宽会被 input speedup 瓶颈限制。分散到多个 GPC 可以利用更多并行的 NoC 入口。
- **L2 分片覆盖**：不同 GPC 到达不同 L2 分片的延迟不同。分散通信 SM 可以让延迟分布更均匀，减少 tail latency。

论文的 Figure 15(b) 给出了实证：当 14 个 SM 集中在 1 个 GPC 时，相比分散在 6 个 GPC，性能下降约 62%。这是一个巨大的差异——仅仅来自 SM 的物理位置选择。

### 3. H100 的分布式共享内存（DSM）机会

H100 引入了跨 SM 的共享内存访问（通过 SM-to-SM 网络）。对于 MoE dispatch，这意味着：

如果 token 的目标专家在同一个 GPC 内的另一个 SM 上（这需要专家的 GPU 内部分布信息），理论上可以通过 DSM 直接传递，绕开 L2 和 HBM 路径。SM-to-SM 网络的延迟（~200 cycles CPC 内，~213 cycles CPC 间）远低于 L2 访问（212-400 cycles），而且不需要占用 L2 带宽。

这是一个尚未被大规模探索的优化方向。目前没有公开的 MoE 训练系统利用 DSM 做 token dispatch——都走标准的 L2/HBM 路径。但在未来，随着 H100/B200 的 DSM 变得更成熟，GPU 内部的 MoE token 路由可能成为节点内层次化 All-to-All 的第一跳。

### 4. MoE kernel 设计的 NoC 感知

目前的 MoE dispatch kernel（包括 DeepSeek-V3 的那些）在设计时主要考虑的是：如何最大化 NVLink 和 IB 的利用率。GPU 内部的 NoC 被视为"足够快"而不被专门优化。

但论文的数据告诉我们，这个"足够快"的背后有不均匀的结构。未来的 MoE kernel 设计可以考虑：

- **亲和性感知的 SM 绑定**：如果编译时知道哪些 SM 靠近哪些 L2 分片（通过类似论文的 micro-benchmarking），可以把通信密集的 SM 绑到"有利"的物理 SM ID 上
- **分区间负载均衡**：在跨分区场景下，确保通信 SM 均匀分布在两个分区，避免所有通信流量只走一个分区的 NoC
- **利用 CPC 的 SM-to-SM 网络**：对于 GPU 内部的 token 路由（如果多个专家在同一个 GPU 上），通过 DSM 绕开 L2 路径

---

## 第五层：NoC 设计的未来方向

### 更大的芯片，更大的非均匀性？

从 V100 → A100 → H100，芯片面积在持续增长。H100 的芯片面积约 814mm²（TSMC 4N），Blackwell B200 更大（双 die 方案）。随着芯片面积增长，NoC 的物理距离也在增长——延迟的非均匀性只会加剧，不会消失。

论文在结论部分指出：**"随着 GPU 规模的持续增长，理解 NoC 的非理想特性对确保整体系统不被 NoC 瓶颈所限变得至关重要。"** 基于模拟的 NoC 研究（假定矩形 2D Mesh 或 crossbar）不再能准确反映真实 GPU 的 NoC 行为。真实 GPU 的 NoC 是层次化的、通过 input speedup 精心平衡过的、但存在显著非均匀性的混合架构。

### Designing AI Chip 的视角

Bjarke Hammersholt Roune（前 Google TPU 技术负责人）在他的技术指南中从另一个角度讨论了芯片互联的设计：

> "对于面向 AI 的芯片，片内互联应该是可预测的——确定性路由、均匀延迟、充足的带宽——因为 AI 工作负载的通信模式是被编译时已知的，不需要适应动态流量变化。"

这与 GPU NoC 的现实形成了有趣的对比。GPU 的设计哲学来自图形处理，需要适应各种不规则的数据访问模式——因此 NoC 被设计成具有弹性带宽分配能力但牺牲了延迟均匀性。TPU 的设计哲学来自张量处理，编译时已知通信模式——因此可以使用更确定性的互联（如 TPU v4 的 Torus）。

哪种哲学更适合 MoE？在这个问题上，DeepSeek-V3 的经验更偏向 TPU 的哲学——MoE 的通信模式是半确定的（token 路由在运行时已知，但可以通过容量和负载均衡来控制），一个更可预测的 NoC 会让通信 kernel 的设计和优化变得更容易。

---

## 关键技术决策 Checklist

**1. 你做过 GPU NoC profiling 吗？**

在你的 GPU 集群上运行类似论文中的 micro-benchmark（算法 1 和 2），确认 NoC 的非均匀延迟和跨分区带宽在你的具体 GPU 型号上的实际值。论文的数据是在 V100/A100/H100 上测的，但你的具体硬件可能因为芯片变体（如 H800 vs H100）而有差异。

**2. 通信 SM 是分散分布还是集中分布？**

如果你像 DeepSeek-V3 那样使用专用的通信 SM，检查它们是否分散在不同的 GPC 上。如果通信 SM 集中在 1-2 个 GPC 中，考虑重新绑定它们。论文的 Figure 15(b) 显示集中分布可能导致 62% 的性能下降。

**3. 通信缓冲区的物理位置在哪里？**

dispatch 缓冲区的 HBM 物理地址落在哪个分区？如果它恰好是通信 SM 所处分区的"远分区"，跨分区带宽下降（A100 上最多 28%）会影响 dispatch 吞吐。考虑使用 GPU 的 NUMA 感知内存分配（如果平台支持）。

**4. MoE kernel 是否可以利用 H100 的 DSM？**

如果多个专家在同一个 GPU 的不同 SM 上（当专家数足够多时会自然发生），现有的代码是通过 L2/HBM 路径做数据交换。评估是否可以利用 H100 的 SM-to-SM 网络（distributed shared memory）来做 GPU 内部的 token 路由——这可以省去 L2 带宽和延迟。

---

## 下一篇预告

第三篇将跨越芯片边界，进入层级 1-2：NVLink 和 NVSwitch。从 NVLink 1.0 (P100) 的裸线直连到 NVLink 5.0 (B200) 的 1.8TB/s 交换架构，每一代带宽翻倍背后的 SerDes 演化、物理取舍、以及对 MoE All-to-All 通信模式的直接适配。紧接着第四篇将深入 NVSwitch 4.0 的芯片级设计——251 亿晶体管、72 条 200Gb/s 端口、14.4Tb/s 交换芯片——以及 NVL72 的机柜工程极限。

---

**本文引用的主要资料**：
1. Jin et al., "Uncovering Real GPU NoC Characteristics: Implications on Interconnect Architecture", MICRO 2024. DOI: 10.1109/MICRO61859.2024.00070
2. Roune, B. H., "Designing AI Chip Hardware and Software", 2026（个人技术指南）
3. NVIDIA, "NVIDIA H100 Tensor Core GPU Architecture", Whitepaper, 2022
4. NVIDIA, "NVIDIA A100 Tensor Core GPU Architecture", Whitepaper, 2020
