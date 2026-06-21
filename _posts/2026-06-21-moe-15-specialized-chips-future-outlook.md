---
layout:     post
title:      Specialized Chips and Future Outlook
subtitle:   Specialized Chips and Future Outlook
date:       2026-06-21
author:     George Lin
header-img: img/post-bg-gen.png
catalog: true
tags:
- moe
---

**系列编号**：第 15 篇（终篇）
**系列主题**：面向工业 MoE 大模型训练的互联通信技术深度研究
**所属系列**：工业 MoE 通信体系：从芯片互联到集群拓扑的实战手册
**目标读者**：工业架构师、AI 芯片设计者、分布式系统研究员、高性能网络工程师
**建议阅读顺序**：建议在阅读本篇之前，至少完成本系列前 5 篇（MoE 基础 + 通信原语 + NVLink/NVSwitch + 集群拓扑 + 调度策略）的阅读

---

## 1. 引言——互联的战场，远不止 GPU 一家

本系列前 14 篇文章，我们几乎把可讨论的 MoE 通信话题全部放在 GPU 生态语境下展开：NVLink 的带宽演进、NVSwitch 的全互联逻辑、InfiniBand 与 RoCE 的取舍、all-to-all 的调度算法、expert placement 的拓扑感知策略……这些内容覆盖了当下工业界 95% 以上的 MoE 训练场景。但这种覆盖本身也暗示着一种认知局限——当我们说"AI 芯片"的时候，潜意识里想到的永远是一块插着 HBM 的 GPU。

这是危险的。互联技术从来不是 GPU 的专利。事实上，在 GPU 之外，有三家公司的互联哲学截然不同，且每一家的设计决策都直指 GPU 体系中最脆弱的部分。Graphcore IPU 用 BSP 全交换将所有核心的本地 SRAM 聚合为逻辑上的统一内存；Cerebras WSE 用 2D mesh + 多播电路交换将 85 万个处理单元织成一片硅晶圆级的数据流网络；SambaNova RDU 则用可重构数据流 + 三级存储体系，将算子融合提高到接近极致。这三种哲学没有一家成功了大规模 MoE 训练——但它们的失败恰恰揭示了互联设计的核心矛盾。

与此同时，编译器研究也在积极将互联通信提升为一等公民。Elk (MICRO'25)、T10 (SOSP'24)、SuperMesh (MICRO'25) 三篇顶会论文代表了一个趋势：编译器不再仅关心计算分块和寄存器分配，而是开始系统性建模 inter-core 数据交换、HBM 带宽争用、preload 与 execute 的重叠调度。这些技术虽然没有直接解决 MoE 通信的 all-to-all 瓶颈，但它们的模型、抽象和优化方法为下一代互联感知的 MoE 编译器奠定了方法论基础。

最后，硬件互联本身也在经历一场从私有协议到开放标准的范式转移。UALink 1.0 的发布标志着业界首次以联盟形式对抗 NVLink 的封闭生态；CPO（共封装光学）和 OCS（光路交换）则在物理层推动"带宽-热量-成本"三角的重构。

因此，这篇终章文章有三个任务：第一，深入剖析三种非 GPU 互联哲学的技术架构和设计动机；第二，梳理三篇顶会论文如何将互联通信作为编译器的一等公民对待；第三，展望未来十年 MoE 通信技术的关键趋势和开放问题。

---

## 2. 系统剖析——三种互联哲学，三重根本差异

### 2.1 Graphcore IPU (BSP 模型) —— 确定性通信的极端实验

Graphcore IPU MK2（Colossus）是一块在互联设计上走得最极端、也最具学术趣味的芯片。理解它需要先忘掉 GPU 的整个心智模型。

**硬件层面**：一块 IPU MK2 拥有 1,472 个独立的可编程核心（tiles）。每个核心配备 624KB 的本地 scratchpad SRAM——注意，是 scratchpad，不是 cache。这意味着 IPU 完全没有 cache 层次结构：没有 L1、没有 L2、没有 L3。所有数据处理都是显式软件管理的。在这 1,472 个核心之间，IPU 部署了一套 all-to-all exchange fabric：每个核心可以以 5.5 GB/s 的单向带宽访问任意另一个核心的本地 SRAM。1472 × 5.5 GB/s ≈ 8 TB/s，这就是 IPU 的聚合 inter-core 带宽。

对比是惊人的：A100 GPU 的 on-chip cache 总容量约 60MB，而 IPU 的分布式 SRAM 则达到 896MB（1472 × 624KB）。A100 的 HBM 带宽为 2TB/s，而 IPU 内部 inter-core 带宽是它的 4 倍。在纸面上，这是对"内存墙"的正面攻击。

**编程模型**：IPU 采用 BSP（Bulk Synchronous Parallel）执行模型。所有核心执行相同的程序（MIMD 架构，每个 tile 有 6 个硬件线程），计算阶段和通信阶段交替进行。每个计算阶段结束后，所有核心经过一个 barrier 同步点，然后进入通信阶段进行 all-to-all 数据交换，再进入下一个计算阶段。整个过程中，通信模式是完全确定性的——不是"best-effort"的数据搬运，而是像编译器调度寄存器一样可以静态分析、静态优化。

**多芯片扩展**：IPU-Link 连接多块 IPU，每根链路 64 GB/s 双向，单芯片聚合 320 GB/s。一个 IPU-POD4 系统包含 4 块 IPU MK2，共 5,888 个核心、3.5GB 片上 SRAM、640 GB/s 片间带宽。

这种 BSP + all-to-all 的哲学与 GPU 的根本差异在于：GPU 的设计逻辑是"尽可能让计算单元饱和，通信能异步就异步"，而 IPU 的逻辑是"计算和通信同步、交替，每一步的数据路由在编译期就已确定"。

### 2.2 Cerebras WSE (Dataflow 模型) —— 硅晶圆上的 2D Mesh

如果说 Graphcore 的逻辑是"把通信变成编译器可推理的确定性行为"，Cerebras 的逻辑则是"把整个网络变成一张超大规模的硅晶圆，让数据流动的距离尽可能短"。

**硬件层面**：WSE-2（Wafer-Scale Engine 第二代）包含约 850,000 个处理单元（PE），每个 PE 配备 48KB 本地 SRAM，通过 2D mesh 网格互联。每个 PE 的 router 管理 5 个双向链路：4 个连接相邻 PE（北/南/东/西），第 5 个是连接到本地处理器的 RAMP 链路。每个链路每周期 32 位，单跳延迟约 1 纳秒。WSE-2 的总 fabric 带宽达到惊人的 220 Pb/s（220,000 Tb/s）。

**颜色路由与多播**：WSE 的互联最具特色的特性是"基于颜色的电路交换路由"。每 32 位数据包（称为 wavelet）被分配一个颜色标签。Router 根据 wavelet 的颜色和到达方向来决定转发路径。关键特性是：Router 支持硬件级多播——一个 wavelet 可以在 router 处被复制并同时发送到多个方向，不增加额外成本。每个 tile 最多可以使用 24 种颜色通道。

**近最优 wafer-scale reduce**：ETH Zurich 团队在 HPDC'24 上发表的"Near-Optimal Wafer-Scale Reduce"是第一个系统性研究 WSE 上的 Reduce/AllReduce 算法的工作。他们提出了一套包含深度（Depth）、距离（Distance）、竞争（Contention）和能量（Energy）四个维度的性能模型，模型预测误差小于 4%。

他们发现的关键洞察是：对于不同的向量长度和 PE 数量组合，最优的 Reduce 算法完全不同。Star Reduce 在标量（向量长度 = 1）时表现最好，Chain Reduce 在长向量时最优，Tree Reduce 适合中等长度，而 Two-Phase Reduce 在中间范围表现最佳。基于这一模型，他们开发了 Auto-Gen Reduce——自动生成针对特定输入规模的优化树结构，在 1D 场景下最多比 vendor 实现快 3.27 倍，在 2D 场景下（512×512 PE）快 3.27 倍。

WSE 的哲学核心是"空间计算"（spatial computing）——数据和计算被物理地映射到硅晶圆上的特定位置，通信距离直接等于物理距离。这不是 von Neumann 模型的变体，而是一个全然不同的计算范式。

### 2.3 SambaNova RDU (Reconfigurable Dataflow) —— 融合到极致

SambaNova SN40L RDU 的互联哲学与前两者又不同：它的目标不是提供通用的核心间通信 fabric，而是通过可重构数据流网络（Reconfigurable Dataflow Network, RDN）将算子融合推到绝对极致，从根本上减少对 inter-core 通信的需求。

**硬件层面**：SN40L 采用 TSMC 5nm 工艺，Chip-on-Wafer-on-Substrate (CoWoS) 2.5D 封装，包含两块 RDD（Reconfigurable Dataflow Dies）和 HBM。每块 RDU 有 1,040 个 Pattern Compute Units (PCUs) 和 1,040 个 Pattern Memory Units (PMUs)，通过 RDN（2D mesh packet-switched 网络）互联。RDN 包含三条物理 fabric：vector fabric（tensor 数据，packet-switched）、scalar fabric（metadata 和地址）、control fabric（电路交换的控制 tokens）。RDN 支持多播、静态流路由、可编程的包序重排（通过 sequence ID）和软件可配置的带宽节流。

**三级存储体系**：这是 SN40L 与 GPU 和其他 AI 芯片最关键的区别——520 MiB 片上分布式 SRAM + 64 GiB co-packaged HBM + 最多 1.5 TiB 插拔式 DDR DRAM。DDR 到 HBM 的拷贝带宽超过 1 TB/s（单节点 8 socket 聚合）。这使 SN40L 能够以极低的模型切换成本支持"Composition of Experts"（CoE）架构。

**Kernel Looping 与算子融合**：SN40L 的核心优势在于可以融合 20+ 个算子到单个 kernel 调用中，包括任意复杂的访问模式（shuffle、transpose 等）。传统 GPU 上，跨算子融合受限于共享内存/Cache/HBM 的数据搬运模型；而 SN40L 的 streaming dataflow 模型允许 operator 之间通过 on-chip stage buffer 直接传递数据，消除了大量的 off-chip 访存。结果显示：HBM 带宽利用率超过 90%（对比 GPU 上通常约 21% 的能力），大部分 decoder 层被融合为单个 kernel 调用。

**Samba-CoE**：SambaNova 将 150 个 Llama2-7B 专家和 1 个 router 部署在单节点（8 socket）上，总参数量 1 万亿。与 DGX H100 相比，整体 CoE 推理加速 3.7 倍（batch size=8），与 DGX A100 相比加速 6.6 倍；模型切换速度分别快 15 倍和 31 倍。

SN40L 的哲学是：**最好的互联，是不需要的互联**——通过极致的融合减少通信、通过多级存储消除模型切换的通信开销。

---

## 3. 编译器——将互联通信提升为一等公民

一个重要的趋势是：在顶级系统会议上，深度编译器正日益将 inter-core 通信视为需要独立建模和优化的"一等公民"，而非计算的附属品。

### 3.1 Elk (MICRO'25) —— 三维权衡：计算 × 通信 × I/O

Elk 由 UIUC 和 Microsoft Research 联合开发，是第一个正式将 ICCA 芯片（Inter-Core Connected AI chip）的三维性能空间——per-core 执行、inter-core 数据交换和 off-chip HBM 访问——统一纳入编译器优化的框架。

**核心洞察**：这三个维度在共享资源（SRAM 容量、互联带宽、memory access ports）上存在根本性的冲突。Elk 将它们形式化为可配置的编译器参数：
- **执行空间（execution space）** 越大 → per-core tile 越大 → 计算效率越高，inter-core 通信越少
- **预取空间（preload space）** 越大 → 可以提前加载更多 operator → HBM 带宽利用率越高，计算与 HBM 访问的重叠越多
- **每个预取 operator 的 preload space 越大** → 更多共享数据在预取时广播 → 执行时的 inter-core 通信越少

这三个目标在有限的片上 SRAM 中相互冲突。

**算法设计**：Elk 的核心贡献是 inductive operator scheduling（归纳式算子调度）+ cost-aware memory allocation（成本感知内存分配）。调度算法以 O(K²N) 复杂度（N 个 operator，最多 K 个可预取），从最后一个 operator 向前归纳地选择每个 operator 的最优预取数量，使"当前到结束"的总执行时间最小。理论证明（Lemma 4.1 + Theorem 4.2）保证了这一归纳步骤的最优性（在给定 cost model 下）。

**性能结果**：在基于真实 IPU-POD4 构建的 emulator 上，Elk 达到理想 roofline 性能的 94%，相比 basic baseline 平均加速 1.87×，interconnect 带宽利用率 89.52%，HBM 利用率接近理想。更重要的是，Elk 显式讨论了扩展到 MoE 模型的路径——在 MoE 中，每个 expert 的在同一 operator 中可能有不同选择，Elk 可以在编译期根据通用的 expert 形状优化执行计划，然后在运行时根据 expert routing 的结果进行专家参数预取。

Elk 还显式提到了扩展到 GPU 的可能性——H100 的 inter-SM 连接的聚合带宽已接近 HBM 带宽，这意味着 GPU 也面临 ICCA 芯片相同的内存空间争用和互联带宽争用问题。

### 3.2 T10 (SOSP'24) —— Distributed On-Chip Memory 的第一性原理

T10 是第一个充分利用 inter-core 互联带宽和分布式片上内存的 DL 编译器。它的核心贡献是提出 rTensor（RotatingTensor）抽象和 compute-shift 执行模型。

**rTensor 抽象**：T10 将 tensor 的分布和旋转形式化为三个关键参数：
- **Spatial partition factor**：沿每个维度如何切分 tensor 为 sub-tensor
- **Temporal partition factor**：每个 sub-tensor 如何进一步在共享它的核心间切分
- **Rotating pace**：每个 compute-shift 步骤中，sub-tensor partition 沿每个维度旋转（shift）的元素数量

这三个参数统一表达了从"全复制无通信"到"最大切分最大通信"的连续设计空间。

**Compute-Shift 模式**：传统 GPU 的 load-compute-store 模式使用 Virtual Global Memory 来模拟共享内存，造成了大量冗余的 inter-core 通信（占执行时间的 50%-74%）和内存容量浪费（sub-operator 大小可增加 22%-180% 如果去掉 VGM）。T10 的 compute-shift 模式将 DNN 计算组织为同步的计算-移位循环——每个核心在每一步计算一个 sub-task，然后将持有的 tensor partition 循环 shift 到邻居核心。这种方式消除了对全局共享内存的需求，均衡化了 inter-core 通信负载，并使 cost model 可以精确到单核单步。

**结果与局限**：T10 在 Graphcore IPU MK2 上实现最高 3.3× 的加速（平均 1.69×）。但它只关注片上执行优化，没有考虑 off-chip HBM 管理，因此无法支持超出片上容量的模型。Elk 可以视为 T10 在"片上+片外"维度的自然扩展。

### 3.3 SuperMesh (MICRO'25) —— 拓扑即算法

SuperMesh 来自 Texas A&M 的研究团队，切入点看起来小——只是给 2D mesh 的边界节点加了额外的短链接——但它揭示了 AI chiplet 互联中的一个结构性缺陷，以及最小拓扑修改如何带来 1.77-2.22× 的 ReduceScatter/AllGather 加速。

**问题诊断**：在标准的 2D mesh 中，边界节点（尤其是角落节点）的连接数天然少于内部节点。在 collective communication（AllReduce/ReduceScatter/AllGather）中，所有节点必须发送/接收等量的数据。边界节点因为链路少，成为瓶颈，导致内部节点长时间空闲等待。这种结构性不对称此前没有被充分重视——人们习惯上认为 mesh 是"scalable"的，但 scalable 不等于 efficient。

**解决方案**：SuperMesh 提出两种变体：
- **SuperMeshBi (SMBi)**：在所有边界链路上添加并行的双向链路，使角落节点从 2 条链路变为 4 条
- **SuperMeshAlter (SMAlter)**：在边界节点间交替添加双向链路，不需要修改 router 度（保持 5 port）

**协同设计的集体通信算法**：SuperMesh 的关键贡献不仅是拓扑，更在于协同设计了一套 pipeline 化的集体通信算法。在标准 mesh 上，pipeline AR 只能构造 3 棵不相交树（因为角落节点链路不够），必须牺牲一个 chiplet 不参与计算。而 SuperMesh 的 4 棵不相交树使所有节点都参与计算和通信，AllReduce 加速 1.18-1.33×。更显著的是 ReduceScatter 和 AllGather 的 1.77-2.22× 加速——这两种集体操作在 ZeRO 训练和 tensor parallelism 中是关键通信原语。

**对 GPU NoC 的启示**：SuperMesh 的核心理念——"在边界增强就能解决 collective 瓶颈"——对 GPU 内部的 NoC（如 H100 的 SM-to-SM 互联）同样适用。随着 inter-SM 链路的聚合带宽接近 HBM 带宽，GPU 也会遇到类似的边界链路拥塞问题。

### 3.4 编译器趋势总结

这三篇论文共同指向一个趋势：**互联拓扑正在成为编译器 IR 的一部分**。传统 compilers 的优化范围止步于单个 core 的计算分块和寄存器分配。而 Elk/T10/SuperMesh 将优化范围扩展为：如何在给定互联拓扑上安排数据的生产、消费和传输，以最大化全局效率。这与 MoE 场景中 all-to-all 通信的编译优化有直接的延续关系——MoE 的 expert 分发和 token 聚合同样可以用"computing + shifting"或"preload + execute"的范式来表达。

---

## 4. 关键技术决策——为何这些芯片未能部署大规模 MoE 训练？

### 4.1 三种设计哲学在 MoE 面前的系统性困局

IPU、WSE、RDU 虽然都展现了对 GPU 体系某些弱点的优雅克服，但到目前为止，没有一家在工业级大规模 MoE 训练中取得成功。这不是偶然的——每一家的核心设计决策都直接或间接地阻碍了这一场景。

**IPU 的困境**：BSP 模型在确定性方面是优势，但在 MoE 的动态路由场景下成为灾难。MoE 的 all-to-all 通信模式高度依赖于每层的 token-to-expert 分配结果，而这个结果是运行时动态生成的。BSP 要求编译期确定所有通信模式，与 MoE 的动态性根本矛盾。另外，IPU 的单片片上 SRAM 只有 ~900MB，远不足以容纳 MoE 模型（即使是单个 expert 的参数也远超此容量），必须依赖 off-chip HBM——但 off-chip memory latency 在 BSP 模型中会直接阻塞所有核心的同步屏障。

**WSE 的困境**：WSE 的片上 fabric 带宽确实惊人（220 Pb/s），但 850,000 个 PE 每个只有 48KB SRAM。MoE 的每个 expert（即使是最小的 1B 参数模型）的 FFN 层也需要数十 MB 的空间。在 WSE 上部署 MoE 意味着将 expert 参数分散到成千上万个 PE 的微小内存中，任何一次 expert 切换都需要在 fabric 上进行大量数据移动。这种"全分散"映射对于 GEMM 和 Conv 等规整计算是可行的，但对于 MoE 的动态 expert 选择和 all-to-all 通信，调度复杂度指数级增长。而且，WSE 是基于颜色电路交换的——可用的颜色数量只有 24 个——在 MoE 的复杂多播-聚合模式下，24 个颜色通道很快就会耗尽。

**RDU 的困境**：SN40L 是三种芯片中最接近实际部署的，Samba-CoE 也展示了令人印象深刻的推理性能。但关键在于——CoE 不是 MoE。CoE 的每个请求只路由到一个 expert，没有 token 级别的 expert 并行性，也就没有 all-to-all 通信。真正 MoE 训练需要在一个 batch 内将不同 token 分发到不同 expert 并同时处理，这涉及高度不规则的 all-to-all + 计算重叠——这与 RDU 的 streaming dataflow + 静态编译模型存在深层不匹配。而且 RDU 的核心优化方向是"融合以减少通信"，而 MoE 的通信不可避免——这是两种设计哲学的"硬碰撞"。

### 4.2 通用矛盾：静态编译 vs. 动态路由

我们可以提炼出一个通用矛盾：当前所有这些专用芯片都从不同的角度追求"**编译期确定通信**"——IPU 的 BSP 全交换、WSE 的颜色路由、RDU 的静态 flow routing。而 MoE 的本质是"**运行时决定通信**"——哪个 token 去哪个 expert、expert 之间的负载如何分布，全是动态的。

这种矛盾在当前阶段没有完美的芯片级解决方案。这也是为什么带有灵活通信 API（NCCL、NVSHMEM）和成熟动态路由支持的 GPU 生态仍然统治 MoE 训练场景。

### 4.3 Elk 项目如何看待 MoE 扩展

有趣的是，Elk 论文在"Discussion and Future Work"中显式讨论了 MoE 支持。它的思路是：在编译期，所有 expert 具有相同的 shape，Elk 基于通用 expert 优化执行计划。在运行时，expert routing operator 决定使用哪个 expert 的索引后，芯片根据编译期生成的 partition plan 预取对应的 expert tensor。这种"编译期做计划、运行时选专家"的模式可以处理 MoE 的静态-all-expert-同构假设，但无法处理异构 expert（不同 expert 大小不同）或动态 expert 创建的场景。

---

## 5. 硬件未来趋势——产业链与技术栈的重构

### 5.1 UALink：用开放标准打破封闭生态

UALink（Ultra Accelerator Link）1.0 于 2025 年 4 月发布，这是一个旨在成为 NVLink 开放替代品的行业联盟标准。成员包括 AMD、Broadcom、Intel、Google、Meta、Apple、Microsoft 等。

**技术参数**：
- 信号速率：200 Gbps/lane（基于标准 802.3 以太网 PHY——200GBASE-KR1/CR1）
- 规模：最多 1,024 个加速器互联
- 延迟：64B payload 时的 round-trip < 1μs；port-to-port hop 约 100-150ns
- 首个部署产品：AMD MI400 "Helios"，预计 2026 年

UALink 的战略意义不在于技术独创性（它本质上是基于成熟以太网标准的 scale-up 互联），而在于生态经济学。NVLink 每代升级意味着 NVIDIA 锁定了从芯片到互联到交换机的整个堆栈。UALink 的出现使 AMD、Intel 等加速器厂商可以在一个共同的互联底座上竞争——有点像 PCIe 之于系统互联的作用。

对 MoE 的意义：UALink 如果能成为跨厂商的互联标准，那么异质加速器集群（GPU + AI chiplet）的 expert placement 和 all-to-all 通信将有一个统一的高速 fabric。但这需要 UALink 的生态成熟到可以承载 all-to-all 的重负载——目前至少还需要 2-3 代产品迭代。

### 5.2 CPO（共封装光学）：物理极限的突破

CPO（Co-Packaged Optics）代表了互联物理层的最激进变革。传统的光模块（pluggable optics）位于 PCB 边缘，电信号需要穿过 PCB traces 到达模块——这段距离的损耗和串扰是带宽继续提升的最大瓶颈。CPO 将光学引擎直接封装在交换芯片的基板（substrate）上，信号从交换 ASIC 到光纤几乎是直接转换。

**关键玩家**：
- Broadcom 已经出货 51.2T CPO 交换芯片
- Ayar Labs 提供基于硅光（silicon photonics）的光学 I/O chiplet
- Lightmatter / Celestial AI 分别推进光学 interposer 和光学 fabric 方案

CPO 对 MoE 互联的影响是间接的但深远的。现阶段，超大规模 MoE 训练集群的互联受限于"热量-带宽-成本"三角：更高的交换机带宽意味着更高的功耗和热量（电信号在铜介质中的损耗随频率指数增长），而更高的功耗限制了一个机架内可容纳的交换端口数（限制了 all-to-all 的全带宽能力）。CPO 可以在大幅降低每 bit 功耗的同时提升单端口带宽——这正是 MoE all-to-all 通信需要的。

### 5.3 OCS（光路交换）：从谷歌到谁？

谷歌已经在其 TPU v4 和 v5 数据中心中大规模部署了 OCS（Optical Circuit Switching），实现了光层面的拓扑重配置。OCS 的核心机制是通过 MEMS 微镜阵列，在光域中提供跨机架的 direct circuit 连接，绕过传统的 packet-switched spine-leaf 网络。

OCS 对 MoE 的潜在意义极具想象力：如果未来可以将 MoE 的 expert 部署在不同机架上的加速器中，通过 OCS 动态建立 expert-to-expert 或 token-to-expert 的光路直连——这种"动态拓扑重构"目前仍处于研究阶段，但它揭示了一个可能的方向：MoE 通信的未来可能不是在静态网络拓扑上优化调度，而是让拓扑本身随通信需求而改变。

### 5.4 技术趋势交汇：对 MoE 的全栈影响

将 UALink（开放 scale-up）、CPO（光 I/O 和交换）、OCS（光路动态重构）三条线叠加来看，未来 5-8 年的 MoE 训练集群可能呈现这样的形态：

- **节点内**：通过 UALink 或类似开放标准，多个加速器 die/chip 以 1+ TB/s 的带宽互联，支持 expert 参数的高效共享
- **节点间**：CPO 驱动的超带宽低功耗交换机，支持无收敛的 all-to-all 通信
- **机架间**：OCS 提供可重配置的拓扑，允许根据 expert popularity 或 token 分布动态调整通信路径

但需要强调的是，这些硬件变革必须与前文所述的**编译器和调度层创新**配合才能真正释放价值。硬件提供更多带宽和更灵活的拓扑，编译器来决定如何在这些能力上最优地组织 MoE 的通信-计算重叠。

---

## 6. 总结与开放问题——十五篇文章之后，我们站在何处？

### 6.1 全系列回顾：一份 MoE 通信技术的知识地图

本系列 15 篇文章覆盖了从芯片内部互联到全球数据中心级集群拓扑的完整 MoE 通信技术栈。以下是简要的系列地图：

| 篇号 | 主题 | 核心命题 |
|------|------|----------|
| 1 | MoE 基础原理 | Expert 并行如何从稀疏激活到通信负担 |
| 2 | All-to-All 通信原语 | 分发的数学本质与实现多样性 |
| 3 | NVLink 与 NVSwitch | GPU 内部 scale-up 互联的演进逻辑 |
| 4 | InfiniBand 与 RoCE | Scale-out 网络的两种哲学与实践 |
| 5 | 集群拓扑与 Expert Placement | 如何将 experts 映射到物理拓扑 |
| 6 | 通信调度与重叠 | 计算与通信的 pipeline 优化 |
| 7 | All-to-All 算法优化 | Reduce-Scatter / AllGather 的 custom 实现 |
| 8 | 异构专家与负载均衡 | 非均匀 expert 大小的通信挑战 |
| 9 | 稀疏计算与 MoE 推理 | 推理场景下的通信优化特殊性 |
| 10 | 全栈系统视角 | PathWay / MegaBlocks / DeepSpeed-MoE 等系统 |
| 11 | AllReduce 优化 | Ring / Tree / 混合算法的深度分析 |
| 12 | 容错与弹性训练 | MoE 下的故障恢复与弹性调度 |
| 13 | 成本建模与经济性 | 互联成本如何影响 MoE 训练经济学 |
| 14 | 前沿实践与产业案例 | 工业界的 MoE 训练集群真实部署 |
| 15 | 专用芯片 + 未来展望 | 三种互联哲学 + 下一代硬件趋势 |

这一系列的核心线索可以概括为：**MoE 的性能瓶颈不在计算，而在通信；通信的瓶颈不在带宽绝对值，而在通信模式与硬件拓扑之间的匹配效率。** 十五篇文章从不同角度反复印证和深化了这一命题。

### 6.2 核心开放问题

作为终篇，我们需要明确当下 MoE 通信领域最紧迫的几个开放问题：

**问题 1：跨厂商互联互操作性何时成熟？**

UALink 倡议虽然令人振奋，但 NVLink 已经迭代了 5 代，生态成熟度极高。开放标准要追上私有协议的性能和生态，至少需要：一致的 SDK/库支持（类似 NCCL 之于 NVLink）、多个加速器供应商的实际产品、以及在实际 MoE 训练中证明其 all-to-all 性能。乐观估计仍需 3-5 年。

**问题 2：异构集群中的统一通信调度框架是否存在？**

当数据中心同时部署 GPU、AI chiplet、FPGA 等不同加速器时，如何在一个统一的通信模型下调度 MoE 的跨设备 all-to-all？这需要通信中间件不仅感知网络拓扑，还要感知设备的计算-通信特征（带宽不对称、延迟差异、内存层次差异）。目前没有成熟的解决方案。

**问题 3：光电混合拓扑的实时重配置能否实用化？**

OCS 已经证明可以在数据中心中工作，但目前的切换粒度是分钟级（或更慢）。如果要将 OCS 用于面向 per-training-step 的 MoE 拓扑重配置，切换延迟需要降到毫秒级甚至更低。这在 MEMS 技术的物理极限上仍然不明确。

**问题 4：推理场景下的极端延迟优化。**

本系列主要关注训练场景，但 MoE 推理对延迟的要求（per-token < 10ms, per-expert switching < 1ms）是另一种极端。如何在保持 all-to-all 带宽的同时实现亚毫秒级的 expert 切换？SN40L 的三级存储 + DDR-HBM 快速切换是一条路径，但这条路径能否在 GPU 体系上实现或超越？

**问题 5：专用 AI 芯片能否为 MoE 训练找到"杀手拓扑"？**

我们分析了 IPU、WSE、RDU 的互联哲学，它们各自在某些方面（确定性、距离最小化、融合极致化）达到了理论边界，但都没有解决 MoE 训练的核心矛盾——动态路由与静态编译之间的张力。是否存在一种新的互联拓扑（可能是 2D mesh + 可编程 bypass + 动态电路交换的混合体），能够同时满足 MoE 对低延迟点对点通信和高带宽集体通信的需求？

### 6.3 写在最后

回到这篇终章的标题——"专用芯片互联实验 + 未来展望"。十五篇文章的旅程告诉我们一个朴素但经常被忽视的事实：**在 AI 系统研究中，互联不是一个"配套"问题，而是一个"定义性"问题。** 你选择的互联拓扑决定了你的编程模型、你的调度策略、你的容错方案，甚至决定了你的商业模式（开放 vs. 封闭）。

回头看 Graphcore IPU、Cerebras WSE 和 SambaNova RDU——它们就像 AI 芯片史上的三次大胆实验。它们在商业上或许都没有打败 GPU，但它们的互联哲学——BSP 的确定性、数据流的空间性、可重构的融合性——为未来芯片的设计者提供了极其宝贵的参照系。MoE 训练的下一个拐点，很可能不是在 GPU 上通过增量优化达到的，而是通过吸收这些实验的洞察，在一个新的互联范式上重新构建全栈方案。

这也是这 15 篇文章最想表达的：**互联即体系，通信即计算。**

---

**系列完结。感谢您走过这段从单芯片互联到全球数据中心拓扑的技术旅程。**

> **声明**：本系列文章基于公开的学术论文、工业技术报告和产品规格撰写，不代表任何芯片厂商或云计算公司的立场。文中技术判断仅为作者分析，存在时效性和局限性的可能。读者在实际系统设计时应以最新规格和 peer-reviewed 文献为准。

---

**参考文献（本篇直接引用）**

[1] Liu Y, Xue Y, Crawford N, Xue J, Huang J. "Elk: Exploring the Efficiency of Inter-core Connected AI Chips with Deep Learning Compiler Techniques." MICRO'25, Seoul, 2025. arXiv:2507.11506.

[2] Liu Y, Xue Y, Cheng Y, Ma L, Miao Z, Xue J, Huang J. "Scaling Deep Learning Computation over the Inter-core Connected Intelligence Processor with T10." SOSP'24, Austin, TX, 2024. arXiv:2408.04808.

[3] Laskar S, Majhi P, Muzahid A, Kim EJ. "SuperMesh: Energy-Efficient Collective Communications for Accelerators." MICRO'25, Seoul, 2025.

[4] Luczynski P, Gianinazzi L, Iff P, Wilson L, De Sensi D, Hoefler T. "Near-Optimal Wafer-Scale Reduce." HPDC'24, 2024. arXiv:2404.15888.

[5] Prabhakar R, et al. "SambaNova SN40L: Scaling the AI Memory Wall with Dataflow and Composition of Experts." 2024. arXiv:2405.07518.

[6] Knowles S. "Graphcore Colossus Mk2 IPU." IEEE Hot Chips 33, 2021.

[7] Lie S. "Multi-Million Core, Multi-Wafer AI Cluster." IEEE Hot Chips 33, 2021.

[8] Lie S. "Cerebras Architecture Deep Dive: First Look Inside the HW/SW Co-Design for Deep Learning." IEEE Micro, 2023.

[9] Zheng L, Li Z, Zhang H, et al. "Alpa: Automating Inter- and Intra-Operator Parallelism for Distributed Deep Learning." OSDI'22, 2022. arXiv:2201.12023.
