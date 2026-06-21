---
layout:     post
title:      NVLink Generational Evolution
subtitle:   NVLink Generational Evolution
date:       2026-06-21
author:     George Lin
header-img: img/post-bg-gen.png
catalog: true
tags:
- moe
---

## 引言：一根 400 GB/s 的绳子绑着一头 989 TFLOPS 的猛兽

2023 年底，DeepSeek 团队在做一件让所有系统工程师血压升高的事：用 H800 GPU 训练一个 671B 参数的 MoE 模型。

H800 是一张"绑着绳子"的卡。它的单卡 FP8 算力是 989 TFLOPS——和 H100 一模一样。但它的 NVLink 带宽被美国商务部产业安全局（BIS）的出口管制规则砍到了 400 GB/s，不到 H100 原版 900 GB/s 的一半。对于 Dense 模型训练，这 400 GB/s 在 8 卡节点内做 Tensor Parallel 的 AllReduce 还算够用。但对于 MoE，All-to-All 通信是跨所有 GPU 的——**NVLink 带宽直接决定了 dispatch 和 combine 的吞吐上限**。

DeepSeek 团队的应对不是抱怨。他们在技术报告里平静地记录了一个事实：在 2048 张 H800 组成的集群上，计算和通信的时间比大约是 1:1。然后他们做了一整套工程改进——从通信 kernel 的 SM 调度、IB 转发的流水线设计、到 DualPipe 的计算-通信重叠——硬是把一个受限制系统训练到了对标 GPT-4 的水平。

这个故事的起点，就是我们今天要讲的主角：NVLink。

NVLink 不是一夜之间冒出来的。从 2016 年 P100 上的第一代到 2024 年 B200 上的第五代，NVIDIA 用了八年时间，把 GPU 间的互联带宽从 160 GB/s 推到了 1.8 TB/s——翻了 11 倍。与此同时，单个 NVLink domain 的总带宽从几十 TB/s 级跃迁到 1 PB/s。这中间的每一步，都不是简单的"把 SerDes 速率翻倍"。物理层、拓扑、交换芯片、信号完整性、热设计——每一个层面都有选择，每一个选择都有代价。

这篇文章我们就从 2016 年 P100 上的 NVLink 1.0 开始，一直讲到 B200 上的 NVLink 5.0 和 NV-HBI。我们会一块一块拆解：为什么 NVLink 被发明？每一代 SerDes 是怎么提升的？NVSwitch 解决了什么问题？NVLink-C2C 和 NV-HBI 跟片间互联有什么关系？FlexLink 这种"废物利用"式的带宽聚合有什么启示？

---

## 系统剖析：NVLink 的物理本质与架构层次

### 受够了 PCIe：NVLink 的诞生动机

要理解 NVLink，首先要理解它要取代的是什么。

在 Pascal 架构之前（2016 年以前），多 GPU 之间的通信只有一条路：PCIe。GPU A 要发数据给 GPU B，数据流是 **GPU A → PCIe → CPU（或 PCIe Switch）→ PCIe → GPU B**。这条路径有三个致命弱点：

**第一，带宽严重不足。** PCIe Gen3 x16 的双向带宽是约 32 GB/s。而 2016 年的 P100 配备的是 720 GB/s 的 HBM2 显存带宽。这意味着芯片从显存取数据的速度是芯片间通信速度的 **22.5 倍**。多 GPU 训练时，所有 GPU 的梯度同步都要走这条窄路——PCIe 成了整个系统的串行瓶颈。

**第二，效率极差。** 实测数据显示，在 DGX-1 (P100) 上，PCIe 的 GPU-to-GPU 数据拷贝只能达到理论带宽的 62%。如果经过 PCIe Switch（所谓"PIX"拓扑），双向拷贝的有效带宽甚至跌到理论值的 28%。剩下的带宽去哪了？被协议开销、CPU 中转延迟、以及 PCIe 本身不是为 GPU 直连设计的这个事实吃掉了。

**第三，缺少缓存一致性。** PCIe 是一个非一致性的 I/O 总线。这意味着两个 GPU 无法透明地访问彼此的内存——每次数据移动都必须显式地通过软件管理，在 CUDA 层面就是 cudaMemcpyPeer。没有硬件缓存一致性，意味着没有原子操作、没有统一地址空间——编程模型的复杂度随 GPU 数量指数增长。

NVLink 的设计就是冲着这三个痛点去的：

- **更高带宽**：第一代就做到 160 GB/s，是 PCIe Gen3 x16 的 5 倍
- **更高效率**：直接 GPU-to-GPU 链路，不需要 CPU 中转，实测可达理论带宽的 88%
- **缓存一致性**：硬件支持 peer-to-peer 原子操作，允许构建统一地址空间

Pascal 是起点。从 P100 开始，NVLink 开始了一段八年的硬件工程马拉松。

### 协议栈：PHY / DL / TL 三层设计

NVLink 采用了类似以太网和 PCIe 的分层协议设计，从底层到上层分为三层：

**物理层（PHY Layer）**：包含串行器/解串器（SerDes）。这是整个 NVLink 最关键的硅 IP——把并行的芯片内部数据转换成差分对上的高速串行信号。每一代 NVLink 的带宽提升，核心就来自 SerDes 速率的提升。NVLink 1.0 用的是 20 Gbps NRZ 编码，NVLink 4.0 用的是 112 Gbps PAM4 编码，NVLink 5.0 推到了 224 Gbps PAM4。

**数据链路层（DL Layer）**：负责包帧（packet framing）、错误检测和流控。这个层级保证物理链路上的原始比特流能被可靠地组织成可处理的数据包。这是 NVLink 区别于简单的 SerDes 直连的关键——它在包级别提供了可靠性保证。

**事务层（TL Layer）**：处理高层事务协议，包括地址翻译、缓存一致性消息、原子操作等。这是 NVLink 能支持 GPU peer-to-peer 内存访问（通过 GPUDirect P2P）和统一地址空间的核心。

这三层的协同设计是 NVLink 能持续提升带宽的基础。如果只提升 PHY 的 SerDes 速率而不优化 DL 层的帧效率和 TL 层的协议开销，物理带宽的提升会被协议开销吃掉。

### 信号调制：从 NRZ 到 PAM4 的分水岭

NVLink 代际演进中有一个无法回避的技术分水岭：NRZ 到 PAM4 的信号调制切换。

**NRZ（Non-Return-to-Zero，不归零编码）** 是 NVLink 1.0 到 2.0 使用的基础调制方式。每个时钟周期传输 1 个比特（0 或 1，两种电平）。NRZ 的优势是实现简单、功耗低、信号完整性相对好控制。劣势也很明确：在给定的信道带宽下，数据率翻倍意味着奈奎斯特频率翻倍——物理信道的损耗随频率指数增长，总有一个天花板。

**PAM4（4-Level Pulse Amplitude Modulation，四电平脉冲幅度调制）** 在 NVLink 3.0 时代被引入。每个时钟周期传输 2 个比特（00、01、10、11，四种电平）。在同样的物理信道上，PAM4 可以传输两倍的数据率。代价是信噪比（SNR）要求更严苛——四种电平之间只有三分之一的电压间隔（相比 NRZ 的两种电平），意味着更复杂的均衡算法、更高的 DSP 功耗、以及更短的物理传输距离限制。

这个 NRZ→PAM4 的切换，是整个 NVLink 演进中最关键的一次物理层决策。它出现在 NVLink 2.0（25 Gbps NRZ）到 NVLink 3.0（50 Gbps PAM4）的代际交替中。如果不切换，NRZ 继续翻倍到 50 Gbps 在信道损耗和功耗上都将不可接受。切换 PAM4 后，NVIDIA 在同一个物理信道上获得了翻倍的带宽——但这个切换也意味着每一条 NVLink 链路的物理传输距离被缩短了，这直接推动了 NVSwitch 的引入（我们稍后展开）。

下表总结了 NVLink 代际的核心物理参数：

| 代际 | 架构 | GPU | SerDes 速率 | 调制方式 | 每链路带宽（双向） | 链路数 | 总带宽（双向） | 年份 |
|------|------|-----|------------|---------|-----------------|--------|--------------|------|
| NVLink 1.0 | Pascal | P100 | 20 Gbps | NRZ | 40 GB/s | 4 | 160 GB/s | 2016 |
| NVLink 2.0 | Volta | V100 | 25 Gbps | NRZ | 50 GB/s | 6 | 300 GB/s | 2017 |
| NVLink 3.0 | Ampere | A100 | 50 Gbps | PAM4 | 100 GB/s | 12 | 600 GB/s | 2020 |
| NVLink 4.0 | Hopper | H100 | 100 Gbps | PAM4 | 200 GB/s | 18 | 900 GB/s | 2022 |
| NVLink 5.0 | Blackwell | B200 | 200 Gbps (224G) | PAM4 | 400 GB/s | 18 | 1.8 TB/s | 2024 |

带宽数字背后的核心工程决策，我们逐代拆解。

---

## 关键技术决策

### 决策一：NVLink 1.0 — 不完美但开路（P100, 2016）

**当时的技术背景**：2016 年，PCIe Gen3 x16 是多 GPU 系统的事实标准互联。每 GPU 双向带宽约 32 GB/s。而这一年的 P100 具有 720 GB/s HBM2 和 21.2 TFLOPS FP16。从存储器、计算单元到互联总线的带宽台阶是 720 → 21.2 TFLOPS → 32 GB/s——最大的漏斗在最外面。

**NVIDIA 的决策：** 从零开始发明一个 GPU 专有互联，而不是等 PCIe Gen4。

这是一个看似冒险但实际逻辑清晰的决策。PCIIe 标准由 PCI-SIG 几百家成员共同制定，更新周期约 3-4 年——PCIe Gen4 从 2017 年才正式发布，x16 双向带宽约 64 GB/s。如果 NVIDIA 依赖 PCIe 的演进节奏，P100 的互联上限将被锁死在远低于片内计算和存储能力的水平。

**物理实现**：NVLink 1.0 采用 20 Gbps NRZ 信号，每个链路 8 对差分线（8 TX + 8 RX），单链双向 40 GB/s。P100 配备 4 条链路，总双向带宽 160 GB/s——约 PCIe Gen3 x16 的 5 倍。

**拓扑选择**：P100 时代没有 NVSwitch。4 条 NVLink 链路的直接连接能力限制了拓扑选择——最多 4 卡的全互联（每个 GPU 用 3 条链路各连一个 peer，第 4 条做冗余或回环）。在 DGX-1 的 8 卡配置中，GPU 们被分成两组 4 卡 cube。组内 NVLink 直连，组间走 PCIe/QPI——这是一个"半 NVLink"系统。

**这个决策的对与不对**：对的地方在于，NVLink 1.0 验证了专有高速 GPU 互联的价值假设——在多 GPU 训练中，将有效互联带宽从 PCIe 的 ~20 GB/s 提升到 NVLink 的 ~140 GB/s 对于 AlexNet、VGG 等通信密集模型带来显著提升。不对的地方在拓扑——当 GPU 数量超过 4 张时，缺少交换芯片导致跨组通信必须回退到 PCIe，这严重限制了 NVLink 在 8 卡场景下的有效性。

P100 的 NVLink 1.0 是一个典型的 v1.0 产品：它证明了方向是对的，但受限于信号速率、链路数量和缺失的交换芯片，还远没有达到"把多 GPU 变成一台计算机"的愿景。它最重要的作用是为后续代际积累了两个关键组织能力：SerDes IP 的持续迭代和高速差分对在 PCB/基板层面的大规模路由经验。

### 决策二：NVLink 2.0 & 3.0 — 拓扑从"直连挣扎"到"全互联觉醒"（V100, 2017 & A100, 2020）

**V100 的代际增量**：NVLink 2.0 在核心参数上做的是增量提升——SerDes 从 20 Gbps NRZ 提升到 25 Gbps NRZ（25% 提升），链路数从 4 增加到 6（50% 提升），总带宽从 160 GB/s 到 300 GB/s（87.5% 提升）。

但 V100 最大的进步不在物理层，而在拓扑层。6 条链路让 4 卡全互联变得更加充裕——每个 GPU 可以用多条链路连接同一个 peer，实现更高带宽的 GPU pair 通信。在 DGX-1 V100 中，8 卡被分成两组 4 卡 cube，组内 NVLink 全互联，组间通过 PCIe 和 QPI 桥接——这本质上仍然是"4 卡互联 + PCIe 桥"的架构，8 张卡之间缺少全互联能力。

**V100 也有过 NVLink 直连 CPU 的尝试**——IBM POWER9 通过 NVLink 2.0 的 BlueLink 接口直接连接 V100 GPU，实现了约 75 GB/s 的 CPU-to-GPU 带宽（而 PCIe 只有 ~15.75 GB/s 实测）。但 POWER9 生态太小，这没能成为主流。后来 Grace-Hopper 上 NVLink-C2C 的故事，其实是从这里埋下的种子。

**真正的范式转变发生在 A100**。

2020 年发布的 A100 引入了一个新的硅器件——**NVSwitch**。这是 NVLink 历史上最重要的一次架构突破，其重要性不亚于 SerDes 翻倍。

**NVSwitch 解决了什么？** 在没有 NVSwitch 的年代，GPU 之间的 NVLink 是直连的——GPU A 的 NVLink 端口直接接到 GPU B 的 NVLink 端口，中间没有交换机。这种拓扑意味着：
- N 个 GPU 的全互联需要每个 GPU 有 N-1 条链路（或被部分互联）
- 任何两个 GPU 之间的带宽被它们之间的直接链路数限制
- 如果一个 GPU 要和另一个"物理上不直连"的 GPU 通信，必须经过中间 GPU 转发——增加延迟，占用中间 GPU 的链路带宽

NVSwitch 是一个完全不同的思路：**把 GPU 的 NVLink 端口全部接到一个非阻塞 crossbar 交换芯片上**。每个 NVSwitch 有多个端口，可以无阻塞地将任何输入端口的数据交换到任何输出端口。A100 的 12 条 NVLink 3.0 链路全部连到 6 个 NVSwitch 芯片上——每个 NVSwitch 的 2 个端口分配给一个 GPU，总共 6 × 2 = 12 条链路，恰好匹配 A100 的 12 条链路。

在 8 卡 HGX A100 主板上，这个架构让**任何两张 A100 之间都能以 NVLink 全速通信**——不是直连，而是通过 NVSwitch 交换，延迟增加极小（<1.2 微秒的交换延迟）。8 张卡形成了一个逻辑上的全互联 fabric。

**A100 的物理层也在升级**：NVLink 3.0 是 NRZ 到 PAM4 切换的第一代——从 25 Gbps NRZ 跳到 50 Gbps PAM4。信号速率翻倍，链路数翻倍（从 6 到 12），总带宽从 V100 的 300 GB/s 跳到 A100 的 600 GB/s。

**NVSwitch 和 PAM4 之间的关系不是偶然的。** 当 SerDes 速率提升到 50 Gbps 后，高速差分信号的物理传输距离显著缩短。在 PCB 上从 GPU 直接走到另一张 GPU 的走线长度可能已经无法满足信号完整性要求。NVSwitch 位于 GPU 和 GPU 之间，物理上把长链路切成了两段短链路——GPU-to-NVSwitch 和 NVSwitch-to-GPU——每段都可以在 PAM4 的 SNR 预算内可靠运行。换句话说，**NVSwitch 既是拓扑上的交换需求，也是物理层信号完整性驱动的架构选择**。

### 决策三：NVLink 4.0 — 突破节点边界（H100, 2022）

H100 上的 NVLink 4.0 做了一件前代没有做的事：**让 NVLink 可以走出机箱**。

物理层升级：100 Gbps PAM4（50 Gbaud），每链双向 200 GB/s。链路数从 A100 的 12 增加到 18（+50%），总带宽从 600 GB/s 到 900 GB/s（+50%）。

但 H100 最重要的不是这一倍带宽。**第三代 NVSwitch 引入了 NVLink Network 能力**。在这之前，NVLink 是纯 node 内的互联——单台服务器的 8 张卡用 NVLink+NVSwitch 全互联，跨服务器必须走 InfiniBand 或以太网。NVLink Network 改变了这个范式：NVSwitch 的所有 64 个端口都可以配置为"外部端口"，通过 OSFP 光模块或铜缆连接到其他服务器上的 NVSwitch——**把 NVLink domain 从单节点扩展到跨节点**。

物理链路上，NVLink 4.0 的信号格式被设计成兼容 400G 以太网和 InfiniBand 的物理层（PHY），这意味着同一套 OSFP 线缆可以承载 NVLink、IB 或 RoCE 流量——单一物理基础设施支持多种逻辑协议。

第二代 NVSwitch 芯片本身也值得一提。它基于 TSMC 4N 工艺，集成了约 251 亿晶体管，294 mm² 的裸片。一个 NVSwitch 提供 64 个 NVLink 4.0 端口，每个端口 50 GB/s（双向），全双工总带宽 3.2 TB/s。四个 NVSwitch 组成一个 8 卡 DGX H100 的 fabric，提供 3.6 TB/s 的全双工内部 bisection 带宽。

**SHARP 是 NVSwitch 的另一个隐藏价值。** 第三代 NVSwitch 集成了约 400 GFLOPS 的 FP32 计算能力（ALU），可以在交换节点内部完成 AllReduce 的归约（reduction），而不需要把数据送回 GPU 做加法。这使得 AllReduce 的有效带宽可以从"GPU 通信带宽"变成"GPU 通信带宽 × reduction factor"——对于大模型训练中的梯度同步，这意味着在不增加 GPU 端 NVLink 带宽的情况下，把聚合吞吐提升了一个数量级。

H100 还引入了一个我们之前讨论过的内部特性：**SM-to-SM 网络和分布式共享内存（DSM）**。虽然在系统层面 DSM 针对的是 GPU 内部通信（我们在第二篇详细分析过），但它和 NVLink 4.0 存在一个协同效应：如果 token 需要在 GPU 内部的不同 SM 之间传递（比如 MoE dispatch 中多个 token 去往同一个 GPU），DSM 可以处理 GPU 内部的"最后一跳"，而 NVLink 4.0 处理 GPU 之间的"主干传输"。两者结合起来，形成了一个层次化的通信架构。

**H100 的 NVLink Network 为后续 Blackwell 的 NVL72 蓝图铺平了道路。** 在 H100 时代，一个 NVLink Network 集群可以扩展到 256 张 GPU——所有 GPU 共享一个逻辑地址空间，任何 GPU 可以访问任何其他 GPU 的 HBM。这是 DGX GH200 的技术基础，也直接启发了 B200 的 NVL72。

### 决策四：NVLink 5.0 — 224G PAM4 和重新定义的物理极限（B200, 2024）

2024 年 3 月的 GTC 上，Jensen Huang 举起了一块 Blackwell GPU。这块 GPU 实际上不是一个芯片——它是两个 GB100 计算 die 通过 TSMC CoWoS-L 封装在一个基板上的"双 die 单卡"。

B200 上的 NVLink 5.0 做到了 1.8 TB/s 的双向带宽——**是 H100 的两倍，是第一代 P100 的 11 倍**。从 H100 到 B200，NVLink 在两年内保持 18 条链路数不变，通过将 SerDes 速率从 100 Gbps 翻倍到 200 Gbps（实际信号速率 224 Gbps，考虑 FEC 开销后的净数据率约为 200 Gbps），实现了带宽翻倍。

224G PAM4 不是一个随意的数字。它是 IEEE 802.3dj 标准中定义的下一代以太网 SerDes 速率——也被称为"超 200G"通道。在单对差分线上传输 224 Gbps 的信号，意味着每个符号周期只有约 8.9 皮秒——光的传播速度是每纳秒 30 厘米，所以一个符号周期内信号在真空中只能传播 2.7 毫米。在现实世界的铜介质中，信号传播速度更慢（约真空中 50-70%），一个符号周期内的传播距离不到 2 毫米。

在这个速度下，信道损耗、串扰、反射、电源噪声——每一项都变成了极难控制的物理问题。224G PAM4 的链路预算（link budget）极其紧张——能容忍的信道插入损耗大约在 30-35 dB，对应的物理传输距离在优质 PCB 材料上大约十几厘米，在铜缆上可能是几十厘米到一米。

**这就是 B200 NVL72 必须使用铜缆背板（copper cable cartridge）而不是 PCB 走线的物理原因。** 在 NVL72 中，72 张 B200 GPU 通过 NVLink 5.0 互连，每个 GPU 有 18 个 NVLink 端口。这些端口不是走在主板上的——它们通过 NVLink 线缆盒（cable cartridge）连接到 9 个 NVLink Switch 抽屉。每根 NVLink 线缆是精心设计的差分对铜缆，在特定的屏蔽和均衡方案下，可以在约一米内承载 224G PAM4 信号。

**NV-HBI：10 TB/s 的双 die 缝合**

B200 最独特的互联创新不在片间，而在封装内。前文提到 B200 是两个 GB100 计算 die 通过 CoWoS-L 硅中介层连接在一起。这个 die-to-die 接口被称为 **NV-HBI（NVIDIA High Bandwidth Interface）**，基于 NVLink 5.0 协议，提供高达 **10 TB/s** 的双向带宽。

10 TB/s 是什么概念？它是 NVLink 5.0 片间带宽（1.8 TB/s）的 **5.5 倍**，是 HBM3e 显存带宽（8 TB/s）的 1.25 倍。两个 die 之间的通信带宽超过了 GPU 与显存之间的带宽——这意味两个 die 在逻辑上表现得就像一个完整的大芯片，不会因为数据在不同 die 上而产生性能差异。

NV-HBI 之所以能提供如此高的带宽，是因为它的物理规模优势：CoWoS-L 中介层上布满了数以万计的微凸块（microbump）连接——实际上是成千上万条超短距离的并行物理连接。这些连接的每比特能耗远低于跨越 PCB 或线缆的 SerDes 通道。在 ~1.3 毫米的硅中介层上，信道的插损几乎为零，不需要复杂的 DSP 均衡——直接 NRZ 信号就能达到高速率。

**NV-HBI 解决的是制造可行性问题，而不是通信架构问题。** 单个 GB100 die 约 460 mm²，在 TSMC 4NP 工艺下，这个尺寸已经接近光罩（reticle）极限——即单次曝光能制造的最大芯片面积。再大就做不出了（或者良率崩盘）。NV-HBI 让 NVIDIA 可以用两个 reticle-limit die 组成一张逻辑上统一的大 GPU，绕过了单 die 面积限制。这是一个典型的**封装驱动架构**决策。

**NVL72：把 72 张 GPU 连成一台计算机**

GB200 NVL72 是 NVLink 5.0 的终极部署形式。一个机架内 18 个计算节点，每个节点包含 2 个 Grace CPU 和 4 个 Blackwell GPU。72 张 GPU 通过 9 个 NVSwitch 抽屉实现全互联——所有 GPU 对之间的 NVLink 5.0 带宽都是对称的。

总聚合带宽？72 GPU × 1.8 TB/s = **130 TB/s**。在一个机架内。

NVIDIA 声称 NVLink 5.0 可以扩展到最多 576 张 GPU 的 NVLink domain，聚合带宽超过 **1 PB/s**（即 1000 TB/s）。相比之下，第一代 NVLink 1.0 在 8 卡 DGX-1 中的总聚合带宽约 640 GB/s——**这是 8 年内超过 1500 倍的提升**。

NVL72 的一个关键工程决策是**铜缆背板**而不是光互联。在前面物理限制的讨论中我们讲到，224G PAM4 信号的铜缆传输距离只有约 1 米。NVL72 的设计恰好利用了这一点——同一机架内的 GPU 间距离在 ~1 米以内，铜缆足够覆盖。铜缆相比光模块的优势是巨大的能耗节约——NVIDIA 声称在同一机架中用铜缆替代光互联节省了约 20 kW 的功耗，同时消除了光模块的成本和故障率。

**NVL72 正在改变 MoE 训练和推理的游戏规则。** 在 NVL72 上，一个 MoE 模型的专家可以分布在一个 NVLink domain 内的 72 张 GPU 上，所有的 All-to-All 通信都在 NVLink 5.0 的 130 TB/s fabric 上完成——没有 InfiniBand，没有跨机架拥塞，没有 PCIe。这直接带来了 NVIDIA 宣称的性能提升：GPT-MoE-1.8T 在 32k GB200 NVL72 上的训练速度是同等 H100 数量的 4×，推理速度是 30×。

### 决策五：NVLink-C2C — 当 CPU 和 GPU 也需要 NVLink

Grace Hopper Superchip（GH200，2023 年发布）引入了一种新的互联类型：**NVLink-C2C（Chip-to-Chip）**。它不是一个片间 GPU-GPU 互联，而是一个 900 GB/s 双向的、缓存一致性的 die-to-die 接口，连接 Grace CPU 和 Hopper/Blackwell GPU。

**为什么 CPU-GPU 连接也需要 NVLink？或者说，为什么不能继续用 PCIe？**

答案的核心是缓存一致性（cache coherence）。PCIe 是一种 I/O 总线——它允许设备访问主存，但不提供硬件级的缓存一致性协议。当 GPU 通过 PCIe 访问 CPU 的 LPDDR5X 内存时，如果 CPU 的缓存中有被修改的副本，软件必须显式地刷缓存（cache flush）——这是一个昂贵的操作，延迟在微秒级别。

NVLink-C2C 通过硬件一致性协议解决了这个问题。Grace CPU 的 Scalable Coherency Fabric（SCF）通过 NVLink-C2C 直接延伸到 Hopper GPU 的 L2 缓存。CPU 和 GPU 可以透明地访问对方的内存，硬件自动维护缓存一致性，不需要任何软件干预。这使得一个关键能力成为可能：**GPU 可以把 CPU 的 LPDDR5X 内存当作自己的扩展显存使用**。

在 GH200 上，Hopper GPU 有 96 GB HBM3。通过 NVLink-C2C，GPU 可以透明地访问 Grace CPU 上最多 512 GB 的 LPDDR5X。对于 GPU 来说，总可寻址内存是 608 GB——比单张 H100 的 80 GB 大了 **7.6 倍**。

NVLink-C2C 的 900 GB/s 带宽是 PCIe Gen5 x16 的约 **7 倍**（PCIe Gen5 x16 双向带宽约 128 GB/s）。能效方面，NVLink-C2C 每比特传输消耗约 1.3 皮焦耳——是 PCIe Gen5 的 **5 倍能效**。这得益于 CoWoS 封装中介层上的超短物理连接——和 NV-HBI 一样，不需要长距离 SerDes 驱动。

在 GB200 Superchip（Grace CPU + 2× B200 GPU）中，NVLink-C2C 的 900 GB/s 带宽在三方（1 CPU + 2 GPU）之间共享。这个配置的物理含义是：两个 B200 GPU 和 Grace CPU 在 CoWoS 封装上共享同一个缓存一致性域，所有三方的内存形成一个统一地址空间。

从 MoE 训练的角度看，NVLink-C2C 带来的最实际的好处是：**内存容量扩展和细粒度的 CPU-GPU 协同**。在大规模 MoE 模型中，专家的参数和优化器状态会耗尽 GPU HBM——NVLink-C2C 允许模型状态溢出到 CPU LPDDR5X，带宽虽然比 HBM 低，但比 PCIe 高得多。对于 MoE 推理，KV cache 可以利用 CPU 的大容量 LPDDR5X 内存，减少显存压力。

---

## 横向对比：从 NVLink 1.0 到 NVLink 5.0 的完整画像

### 代际参数总表

| 参数 | NVLink 1.0 | NVLink 2.0 | NVLink 3.0 | NVLink 4.0 | NVLink 5.0 | NVLink-C2C |
|------|-----------|-----------|-----------|-----------|-----------|------------|
| **架构** | Pascal | Volta | Ampere | Hopper | Blackwell | Grace Hopper/Blackwell |
| **发布年份** | 2016 | 2017 | 2020 | 2022 | 2024 | 2023 |
| **SerDes 速率** | 20 Gbps | 25 Gbps | 50 Gbps | 100 Gbps | 200 Gbps (224G) | CoWoS 中介层 |
| **调制方式** | NRZ | NRZ | PAM4 | PAM4 | PAM4 | 直接信号 |
| **每链路带宽（双向）** | 40 GB/s | 50 GB/s | 100 GB/s | 200 GB/s | 400 GB/s | — |
| **每 GPU 链路数** | 4 | 6 | 12 | 18 | 18 | 1（CoWoS D2D） |
| **每 GPU 总带宽（双向）** | 160 GB/s | 300 GB/s | 600 GB/s | 900 GB/s | 1.8 TB/s | 900 GB/s |
| **交换芯片** | 无 | 无 | NVSwitch 1.0 | NVSwitch 3.0 | NVSwitch 5.0 | 无（直连） |
| **GPU 拓扑（8 卡）** | 2×4 cube + PCIe | 2×4 cube + PCIe | 全互联（6× NVSwitch） | 全互联（4× NVSwitch） | 全互联（9× NVSwitch） | 1 CPU + 1/2 GPU |
| **物理介质** | PCB + 桥接器 | PCB + 桥接器 | PCB + 基板 | PCB + OSFP 铜/光缆 | 铜缆背板 | CoWoS 硅中介层 |
| **跨节点能力** | 无 | 无 | 无 | NVLink Network | 全 NVLink Domain | 无 |
| **最大 Domain GPU 数** | 8 | 8 | 8（单节点） | 256（NVLink Network） | 576 | 32（NVL32） |
| **与 PCIe 带宽比** | 5× vs Gen3 | 10× vs Gen3 | 18× vs Gen3 | 14× vs Gen5 | 14× vs Gen5 | 7× vs Gen5 |

### 拓扑演进：从直连到全互联到跨机架域

NVLink 的拓扑演进可以总结为三个阶段：

**阶段 1（NVLink 1.0/2.0）：直连 cube。** GPU 之间 NVLink 点对点连接，没有交换芯片。拓扑受限于链路数——P100 的 4 条链路最多连 4 卡 cube，V100 的 6 条链路让 4 卡 cube 更充裕。8 卡必须分组，组间桥接靠 PCIe 或 QPI。这个阶段的 NVLink 本质上是一个"GPU cluster 内的快速小圈"。

**阶段 2（NVLink 3.0/4.0）：全互联 fabric。** NVSwitch 的引入让任意 GPU 对之间都能直通——不需要中间 GPU 转发。NVSwitch 的 crossbar 架构确保了非阻塞交换：任何一个输入端口的流量不会因为其他端口的流量而被阻塞。H100 的 NVLink Network 把这个 fabric 的概念拓展到了跨节点。

**阶段 3（NVLink 5.0）：全机架域。** NVL72 把一个机架变成了一个 NVLink domain。72 张 GPU 全互联，带宽足够高、延迟足够低，使得跨 GPU 的 MoE All-to-All 可以从"需要仔细调度"变成"随便走"。NVIDIA 给出的 576 GPU domain 规划意味着未来的 NVLink domain 可能横跨 8 个机架。

### 物理介质选择：铜缆、PCB 与光的工程取舍

NVLink 代际演进中，物理介质的选择一直和 SerDes 速率紧密耦合：

- **NVLink 1.0-2.0（20-25 Gbps NRZ）**：PCB 走线足够。相对低速的信号对信道损耗不敏感，在标准 FR4 PCB 上走几十厘米不是问题。
- **NVLink 3.0-4.0（50-100 Gbps PAM4）**：需要更高质量的 PCB 基板材料和更短的走线。NVSwitch 的位置被精心设计在 GPU 之间的"中点"，缩短每条链路的物理长度。H100 跨节点走 OSFP 铜缆或光缆——光缆的传输距离可达 20 米以上，铜缆约 2-3 米。
- **NVLink 5.0（224G PAM4）**：铜缆的极限。224G PAM4 信号在铜介质上的有效传输距离约 1 米。NVL72 的铜缆背板设计正好匹配这个距离限制——同一机架内 GPU 到 NVSwitch 的距离刚好在这个范围内。如果未来需要跨机架的 NVLink 5.0（超过 1 米），光互联几乎不可避免。

**NVIDIA 对铜缆的坚持是一个值得注意的工程选择。** 在带宽需求爆炸式增长的时代，直觉上应该"上光"。但 NVIDIA 坚持在机架内用铜缆——铜缆的单位比特能耗比光模块低 5-10 倍，可靠性更高（没有激光器老化问题），成本更低。在 NVL72 的 120 kW 机架功率预算中，省下 20 kW 给互联本身就是巨大的竞争优势。NVIDIA 的哲学似乎是：**光互联只在物理上非用不可的时候才用**。

---

## FlexLink：当你的 NVLink 被砍了 55%，怎么把带宽"凑出来"？

H800 的存在是某个工程假设的反面教材。NVIDIA 设计 H100 的时候，假设 NVLink 4.0 总是 900 GB/s——所以 NCCL 的通信策略就是"能走 NVLink 全走 NVLink"。但当 H800 的 NVLink 被法规砍到 400 GB/s，NCCL 的策略不再是"够用就好"——它低效了。PCIe 和 RDMA NIC 两条路径在整个 collective 操作期间几乎完全闲置——即使它们加起来可以提供额外的几百 GB/s。

蚂蚁集团的 FlexLink（2025 年的论文，arXiv:2510.15882）正面回答了这个问题：**能不能把 NVLink、PCIe 和 RDMA NIC 三条异构路径动态地聚合成一个虚拟的通信 fabric，在不影响精度、不需要改应用代码的前提下，把 H800 的受限带宽补回来？**

答案是可以。而且做得很漂亮。

### FlexLink 的核心思想

FlexLink 的本质是一个**带宽聚合层**，在 NCCL 的集体通信原语下面，做两件事：

1. **在初始化时，通过粗粒度调参（大约 10 秒的 profiling），找到 NVLink、PCIe、RDMA 三条路径上的最优流量分配比例**
2. **在运行时，通过轻量级监控器持续监测每条路径的完成时间，动态微调分配比例，防止慢路径拖累快路径**

调参算法的核心逻辑很简单：让所有活跃路径在大致相同的时间完成它们各自的数据传输。如果 NVLink 最快但任务太重，就向 PCIe/RDMA 卸载一部分流量。如果某个辅助路径太慢变成了新瓶颈，就减少卸载比例。步长减半的阻尼机制防止振荡。

### 具体性能数据

在 8 卡 H800 服务器上：

| 操作 | GPU 数 | 消息大小 | NCCL 基准带宽 | FlexLink 带宽 | 提升 | PCIe+RDMA 流量占比 |
|------|--------|---------|-------------|-------------|------|-------------------|
| AllReduce | 2 | 256MB | 139 GB/s | 175 GB/s | **+26%** | 12% + 9% |
| AllReduce | 4 | 256MB | 98 GB/s | 118 GB/s | **+20%** | 12% + 2% |
| AllReduce | 8 | 256MB | 107 GB/s | 109 GB/s | +2% | 1% + 1% |
| AllGather | 2 | 256MB | 132 GB/s | 163 GB/s | **+23%** | 14% + 5% |
| AllGather | 4 | 256MB | 49 GB/s | 62 GB/s | **+27%** | 12% + 10% |
| AllGather | 8 | 256MB | 21 GB/s | 26 GB/s | **+24%** | 12% + 7% |

几个有趣的发现：

**AllReduce 在 8 卡场景下几乎没有提升（仅 +2%）**。原因在于 Ring AllReduce 的算法结构——对于 N 个 GPU，Ring AllReduce 需要 2(N-1) 步顺序通信步骤。当 N=8 时是 14 步。而 Ring AllGather 只需要 N-1=7 步。PCIe 和 RDMA 路径的高延迟在 14 步中被放大了——每一步累积的延迟开销压过了带宽卸载的优势。FlexLink 的调度器正确地检测到这一点，主动将 PCIe/RDMA 的流量降到几乎为零，让低延迟的 NVLink 独占通信。这种"知难而退"式负载均衡，恰恰是 FlexLink 比 naive 静态分区更智能的地方。

**AllGather 的提升最显著**，因为它的通信步数少（N-1 步），延迟累积不严重，PCIe/RDMA 的带宽贡献能充分释放。

**2 GPU 和 4 GPU 场景下 PCIe+RDMA 的卸载比例通常能达到 14-21%**，在 8 GPU 的 AllGather 场景也能达到约 19%。这些被卸载的流量原本会导致 NVLink 阻塞——NVIDIA 给出的 400 GB/s NVLink 并不是"400 GB/s 随便用"，在多个 GPU 对之间同时走 NVSwitch 时，存在跨端口竞争。FlexLink 通过把一部分流量引到 PCIe/RDMA，实际上减轻了 NVSwitch 的内部拥塞。

### FlexLink 对 MoE 的启示

FlexLink 呈现的数据直接关联到 MoE 训练的通信瓶颈。论文引用的数据点：
- MoE 训练中，通信开销可占前向传播的 **43.6%**
- 长序列推理中，Flash Communication 的通信开销可高达 **65.9%**
- 32B 模型在 64K 序列长度的 prefill 中，通信占执行时间的 **36%**

在这些场景中，H800 的 400 GB/s NVLink 是一个明确的瓶颈。FlexLink 的 27% AllGather 提升和 26% AllReduce 提升，在这些 MoE 通信主导的工作负载中，直接转化为端到端的训练和推理加速。

**FlexLink 的意义不限于 H800。** 论文中的表格 1 显示，即使是 H100/H200（900 GB/s NVLink），也存在 14% 的闲置带宽机会。B200（1800 GB/s NVLink）有 22% 的闲置带宽。未来的 GB300 因为不再有 PCIe 和 NIC 共享路径的争用，闲置带宽机会反而上升到 33%。这意味着 **FlexLink 式的异构链路聚合是一个将持续存在、且需求增长的技术方向**。

从工程哲学的角度，FlexLink 体现了一个原则：**当硬件条件被外部约束锁死时，不抱怨，去充分利用每一条可用路径**。DeepSeek 团队在 H800 上训练 DeepSeek-V3，和 FlexLink 的设计是同一个精神传统——把工程做到极致，让受限的硬件给出不受限的结果。

---

## 总结与清单：八代互联背后的工程法则

NVLink 的八年演进能提炼出几条朴素的工程原则：

**1. 互联带宽必须和计算能力同步增长，否则互联将始终是瓶颈**
NVLink 从 160 GB/s 到 1.8 TB/s 翻了 11 倍。同期，GPU FP8 算力从 P100 的 21.2 TFLOPS (FP16) 到 B200 的 4,500 TFLOPS 翻了约 200 倍。互联带宽的增长速度实际上落后于计算能力的增长——这意味着通信瓶颈正在变得更严重，而不是被缓解。这正是 MoE 训练中 43.6% 通信开销背后的结构性趋势。

**2. 拓扑自由度由链路数和交换芯片共同决定——缺一不可**
P100 没有 NVSwitch，4 条链路不够全互联。V100 的 6 条链路让 4 卡 cube 更充裕，但 8 卡仍然不够。NVSwitch 的引入是拓扑自由度从"线性增长"到"阶跃"的转折点——一下子解决了任意 GPU 对的直接通信问题。但 NVSwitch 也引入了新的约束（芯片功耗、端口数、交换延迟），需要在每一代仔细平衡。

**3. 调制方式的切换（NRZ → PAM4）是一次"不得不做"的跳跃，代价是传输距离**
PAM4 在同样物理信道上提供了两倍的数据率，但代价是 SNR 容忍度下降、功耗上升。这个切换的连锁反应推动了 NVSwitch（缩短链路物理长度）和外部线缆标准（铜缆/光模块）的演进。224G PAM4 进一步将物理传输距离压到约 1 米——这直接塑造了 NVL72"全铜缆背板"的物理形态。

**4. 跨节点互联不是带宽问题，是地址空间问题**
NVLink Network 最重要的不是"NVLink 带宽可以从节点内延伸到节点间"——InfiniBand 早就可以提供高带宽。关键是 NVLink domain 内的统一地址空间和硬件级内存访问。这让跨 GPU 的数据移动从"需要软件管理"变成"硬件透明"，在编程模型上消除了节点边界。

**5. 当硬件受限，把闲置路径"捡起来"是最高性价比的优化**
FlexLink 的 +27% 不是来自新硬件，而是来自充分利用 H800 上被 NCCL 忽略的 PCIe 和 RDMA 路径。这是一种"不花钱买新卡，靠软件把缝隙填满"的思路。在下一代硬件中（GB300+），PCIe 和 NIC 路径不再共享带宽，闲置机会反而更大（33%）——异构链路聚合是一个将持续存在的技术方向。

### 工程师决策清单

- [ ] 你的 GPU NVLink 带宽是多少？你真的用到了吗？建议用 nccl-tests 跑一遍 AllReduce 和 AllGather 实测带宽，而不是相信规格书上的数字
- [ ] 你的 PCIe 路径在 collective 通信期间有多大比例的闲置？如果 NVLink 带宽受限（如 H800 / A800 场景），评估 PCIe 和 RDMA NIC 是否可以被 FlexLink 类方案利用
- [ ] 你的 GPU 拓扑是全互联吗？如果是部分互联（如 P100/V100 时代的 2×4 cube），非直连 GPU 之间的通信路由经过了什么中间节点？
- [ ] 你的 NVSwitch 设备是否支持 SHARP in-network reduction？如果支持，是否在 NCCL 环境中启用了 NCCL_ALGO=NVLS？这可以让 AllReduce 的带宽提升数倍
- [ ] 如果使用 MoE 训练，All-to-All 通信的带宽和计算-通信比是多少？通信占比超过 30% 时，考虑：更大的容量因子是否在浪费带宽？通信 kernel 的 SM 分配是否优化过？
- [ ] 跨节点通信走在什么物理层上——InfiniBand、RoCE、还是 NVLink Network？不同的物理层有不同的拥塞行为、不同的最优化消息大小，需要分别调优
- [ ] 如果硬件受限（如法规限制的 NVLink 带宽），除了 FlexLink 式的带宽聚合，是否考虑通信压缩（如 FP8→FP4 量化、Flash Communication）作为正交优化路径？

---

下一篇将聚焦 NVSwitch 和 NVLink SHARP——从电路交换到包交换，从 static collectives 到动态 in-switch computing，以及 DySHARP 如何在 NVSwitch 上实现对 MoE All-to-All 的 in-network 加速。
