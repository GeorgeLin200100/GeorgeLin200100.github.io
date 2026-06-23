---
layout: post
title: Load/Store vs Send/Recv —— AI互联各层次的语义选择
subtitle: AI Interconnect Semantic Model
date: 2026-06-23
author: George Lin
header-img: img/post-bg-gen.png
catalog: true
tags:
- Interconnect
---

## 一个让人睡不着觉的问题

2024 年，NVIDIA 的 NVLink 已经到了第五代（Blackwell），SerDes 跑 224 Gbps PAM4，有效速率 200 Gbps/lane，单 GPU 双向带宽 1.8 TB/s。一个 NVL72 机柜里，72 块 Blackwell GPU 通过 NVSwitch 全互联，机柜内总带宽 130 TB/s。这些数字大到已经不太好想象。

但如果你是一个芯片架构师，在做下一代 AI 芯片的互联设计，你大概率睡不着觉。不是因为数字不够大，而是因为你面前摆着一个不知道怎么回答的问题：

**在互联的每一个层次上，到底是让计算单元向远端发一个 load/store，还是让它发一条 message？**

这个问题单拎出来看似乎很无聊。load/store 和 send/recv 不都是"把数据从 A 弄到 B"吗？但如果你真的坐下来做架构 trade-off，你会发现它在不同物理尺度、不同工作负载、不同规模要求下，答案完全不同。更麻烦的是，你选了其中一种语义，意味着你的整个硬件栈都要围绕它来建。地址翻译、缓存一致性、流控、故障隔离、编程模型——全捆在一起。

这篇文章想把这件事掰开来，但不是为了给答案。是为了给一组问题。如果你读完能拿着这组问题去拷打你手上的设计 spec，那就可以了。

先交代一句范围。这篇讨论的"语义"分两层。**硬件实现层面**是主线，load/store 和 send/recv 在硅片上到底意味着什么。**编程模型层面**是辅线，它们暴露给软件写作者的是什么。两者相关但不相同，后面会反复出现这个区分。

---

## 第一部分：把两种语义按在硬件上看清楚

先做一件事：把 load/store 和 send/recv 这两套东西按在硬件实现上看清楚它们的区别。这一步不能省。后面所有层次的讨论都建立在"它们根本不是一回事"这个前提上。

### Send/Recv：接收方说了算

Send/Recv 语义的核心是一个看似平淡、实则决定一切的事实：**接收方拥有目标缓冲区。谁来放数据、放在哪里、什么时候放完——全部由接收方决定。**

在硬件实现上，send/recv 对应的是这些东西：

- 发送方有一个发送队列（Send Queue, SQ），接收方有一个接收队列（Receive Queue, RQ）。两边各自往自己的队列里塞 work request，硬件从队列里取活干。
- 发送方的 work request 说的是"把地址 X 处长度为 L 的数据发出去"。它不指定数据到了对端要放在哪里。那是接收方的事。
- 接收方的 work request 是提前 posted 的："我准备了一块 buffer 在地址 Y，大小为 M，谁来都可以往里面写"。发送方发来的数据到达后，硬件把数据写入 Y，然后往接收方的 Completion Queue（CQ）里塞一个条目："数据到了，放在了 Y"。
- 发送方在 send 完成后也会收到一个 completion，但这个 completion 只表示"数据已经发出去了"或者"数据已经被对端确认收到了"（取决于协议），不包含"对端把数据放在了哪里"这样的信息。

这个机制的直接后果是：**发送方的数据在到达接收方之前，接收方必须已经为它准备好了内存。** 如果接收方没有提前 post buffer，数据要么被丢弃，要么触发一个"未预期消息"的错误路径。InfiniBand 把这种叫 Unexpected Message。SRQ（Shared Receive Queue）设计出来就是为了缓解这种尴尬——多几条 QP 共享一个 buffer pool，不用每条都预分配。但 SRQ 只是把问题缩小了，没消除。

为什么把这件事说得这么细？因为"接收方预分配 buffer"这个机制是 send/recv 一切硬件代价和一切硬件优势的根源。

代价很明显：接收方必须猜测发送方什么时候会发多少数据。猜少了，buffer 不够，丢包。猜多了，buffer 空了，浪费内存。这个猜的过程没有完美的解法。要么引入额外握手延迟（发送方先发一条"我准备发 N 字节"的预告），要么承受 buffer 浪费和丢包的风险。

优势不那么明显，但在大规模系统中才体现出来：因为 buffer 是接收方自己的，接收方天然拥有对自己内存的全部控制权。没有外部实体可以在接收方不配合的情况下往它的内存里写东西。这意味着**故障隔离是天然的**，一个节点崩溃不会污染另一个节点的内存。也意味着**流控是显式的**，接收方不 post buffer，发送方就发不出来，背压自动生效。

### Load/Store：发起方说了算

Load/Store 语义的核心和 send/recv 正好反过来：**发起方直接操作远端的地址空间**。它不需要接收方提前准备 buffer，也不关心对端是否"期待"这次访问。发起方说"读地址 Z"或者"写地址 Z"，硬件负责把这件事完成。

在硬件实现上，load/store 对应的是一套完全不同的机制：

- 发起方看到的不是"队列"和"消息"，而是一个地址。这个地址可能映射到本地 DRAM，也可能映射到远端设备的某个内存区域。从发起方来看，两者没有区别，都是一条 load 指令或 store 指令。
- 要让这件事成立，需要一套地址翻译机制：发起方发出的物理地址是它的 PCIe 域或系统域里的地址，这个地址必须能被翻译成远端设备内部的地址。x86 上这个活是 IOMMU 干的，ARM 上是 SMMU，CXL 里叫 ATS（Address Translation Services）。叫什么不重要，做的事是同一件：把发起方的地址映射到接收方的实际物理页。
- 访问粒度是 cache line 大小（通常是 64B）。这不是 load/store 语义本身要求的，而是因为 load/store 依赖缓存一致性协议来维护数据正确性，而缓存一致性协议天然以 cache line 为粒度运作。
- 缓存一致性意味着，如果多个发起方同时访问同一个地址，硬件必须保证它们看到的数据是一致的。这件事通过 snoop 机制来实现：当一个发起方要写一个 cache line 时，互联网络向所有可能持有该 cache line 副本的节点发 snoop，把它们手里的副本无效化（或要求回写脏数据），然后才允许写。

如果说 send/recv 的一切源于"接收方拥有 buffer"，那 load/store 的一切源于**一致性**。

一致性的优势是编程模型的天堂：你不需要显式地管理"谁在什么时候给谁准备了什么 buffer"。你在代码里写 `a = b[c]`，硬件自己搞定数据在不在本地、不在的话从哪拉、拉到本地之后如果有其他人也在用这个地址怎么办。这种透明性是共享内存编程的前提，而共享内存编程是当前大多数并行编程框架的基础。

一致性的代价在单芯片上已经够贵了，出了芯片只会更贵：

- **Snoop 的广播代价**。每一次写操作都可能触发广播查询。在总线上（如 ACE 协议）这是一条物理线，所有 snooped master 并行响应。在网络里（如 CHI 的分布式 snoop filter 或目录协议），这是一组点对点消息。不管哪种实现，参与者数量一多，snoop 流量就是 O(N²) 级别。
- **Snoop filter 的存储开销**。为了不把 snoop 广播给所有人，通常会在 Home Node 维护一个 snoop filter，一个记录了"哪些节点可能持有某个 cache line"的查找表。这个表的容量和互联节点数正相关，也和共享模式（Inner/Outer Shareability Domain）有关。但划分域只能控制 snoop 范围，不能消除 snoop filter 本身的存储和查表开销。
- **地址翻译的硬件代价**。每一笔 load/store 都要经过地址翻译——发起方的虚拟地址到发起方的物理地址再到接收方的物理地址，至少涉及两级页表 walk。为了不每笔都 walk，需要缓存翻译结果，就得有 ATC（Address Translation Cache）以及配套的 ATS 协议来回 invalidate ATC 条目。在 CXL 里，这套东西叫 CXL.cache + CXL.mem，占的 die area 不小。
- **一致性的延迟代价**。一笔跨节点的 load，在拿到数据之前可能要先经历：查本地 cache（miss）→ 发 snoop 查远端 cache → 等远端 cache 响应 → 如果远端是 Dirty，从远端 cache 取数据；如果远端是 Clean，从远端内存取数据。每一步串行，每一步都在加延迟。对比一下 send/recv：接收方提前准备好 buffer，发送方直接把数据推过来，硬件把数据放进 buffer，结束。没有 snoop，没有地址翻译。

### 灰色地带：RDMA 和 Atomics 到底站哪边

有人说：RDMA 一边发消息一边又可以直接读写远端内存，这不就是两者的混合？

RDMA 确实提供了"单侧发起"的操作——RDMA Read 和 RDMA Write。发起方指定远端的内存地址（通过 rkey 来做权限检查），数据直接写进去或读出来，远端 CPU 完全不知道这件事发生了。从编程模型角度看，这确实很像 load/store。

但从硬件实现角度看，RDMA Read/Write 底层跑的还是消息。发起方的 RNIC 把操作封装成一个消息包，里面带着 rkey、远端地址、数据载荷。远端 RNIC 收到后，验证 rkey，通过的话执行 DMA 操作把数据写入/读出远端内存，然后（对于 RDMA Read）把结果打包发回来。整个过程没有涉及缓存一致性。远端 CPU 的 cache 上如果有这个地址的脏数据，RDMA 写入是无视它的，反之亦然。这就是为什么 RDMA 编程模型要求显式的内存注册（MR）和显式的 fence——在软件层面保证一致性，不依赖硬件 coherence。

Atomic 操作同理。infiniBand 的 atomic compare-and-swap、atomic fetch-and-add，从编程模型看很像 load/store 世界里的原子操作，但底层是消息。RNIC 收到一个 atomic 消息，在 NIC 内部（或通过 PCIe 的 atomic op）完成操作，返回结果。

Put/Get 操作（CCIX 或某些 NoC 协议中定义的）倒更接近 load/store 的实现。它们通常直接映射到缓存一致性协议里的事务（如 CHI 的 ReadUnique、ReadShared）。但 put/get 仍然需要一个中间层（Home Node）来协调一致性和地址翻译，所以不是"纯粹的" load/store，更像是用消息原语实现 load/store 语义。

这个灰色地带的关键教训是：**不要被编程模型的相似性迷惑。看硅片上到底在发什么信号。**

---

## 第二部分：四个层次，逐层拆开

有了"两种语义在硬件上到底是什么"这个共识，开始拆层次。

每一层用五个问题审视：物理约束是什么？AI 工作负载在这一层的主流通模式是什么？当前主流设计选了什么？为什么？哪里开始出现张力？

### 2.1 片上互连（NoC）：Load/Store 的绝对主场

**物理约束**

片上互连的特征用四个数字就够了：距离 ~mm、延迟 ~1ns、带宽 ~TB/s、误码率低到可以忽略。H100 内部，一个 SM 到最远的 L2 分片，物理距离不超过一张 reticle。在这个尺度上，一个 round-trip 大约是几纳秒，对 GPU 的 1-2 GHz 时钟来说就是几个到几十个周期。

**工作负载模式**

AI 芯片的 NoC 上跑的数据流大致分两类：

- **权重驻留（Weight-stationary）**：权重固定在 PE 本地，激活值和部分和在 PE 之间流动。TPU 的脉动阵列就是这么做的。
- **输出驻留（Output-stationary）**：部分和固定在 PE 本地，权重和激活值流动。

不管哪种，数据移动的粒度都是单个标量或向量，不是 MB 级别的大块传输，而是一个 clock 一次的数据推送。数据流高度规律。脉动阵列的数据流在编译时就可以精确预测每一个 PE 在每一个 cycle 要收发什么数据。

GPU 里情况稍微复杂一些。SM 通过 Crossbar 或网状网络访问 L2 分片和 HBM 控制器，访问模式不那么规整，kernel 可以以任意顺序读/写 global memory。但粒度仍然是 cache line（通常 32B 或 64B），而且单个 warp 的访存 pattern 在 warp scheduler 视角下是已知的（coalesced access 就是利用这个做的优化）。

**当前设计选择**

Load/Store 一统天下。ARM 的 AXI/CHI、NVIDIA GPU 的内部 NoC、Google TPU 的片上互联，全部基于 load/store 语义（或者说基于地址的访问）。除了一些特殊的流处理加速器会用自定义的流协议，几乎看不到 send/recv 的影子。

**为什么**

因为在这尺度上，延迟压倒一切。

设想一个 send/recv 模型跑在 NoC 上的极端情况：发送方 PE 要往接收方 PE 发一个激活值。在 send/recv 模型下，接收方必须提前 post 一个 receive buffer。这意味着每次数据传输之前，发送方和接收方之间要有一轮"我准备好了 buffer，你可以发了"的握手。这轮握手在 1 GHz 时钟下可能是 5-10 个周期。如果数据传输本身只需要 2-3 个周期呢？那大部分时间花在握手上，不是在传数据。

而且 NoC 上 PE 的数量是几百到上千个。每个 PE 都要为所有可能向它发数据的 PE 预先分配 buffer，这个 buffer 的容量要 cover 最坏情况下的数据量。即使每个 PE 的 buffer 很小，加在一起也会吃掉一大块宝贵的片上 SRAM。

Load/Store 模型下这一切都不存在。发送方直接把数据写到接收方的本地 buffer 地址，接收方不需要显式地"期待"这次写入。地址分配是静态的或者编译时决定的，不需要运行时协商。省掉的是延迟、面积、功耗。在 NoC 这个尺度上，这三者一样都不能浪费。

那一致性怎么办？在 NoC 上一致性是可管理的。不是因为一致性变简单了，而是因为几百个节点的全一致性在芯片内可以用硬件跟得上（snoop filter、层次化一致性域、目录协议），并且这个代价芯片架构师愿意付。共享内存编程模型对软件生产力的价值太大，花钱买是值的。

**张力**：芯片越做越大。Cerebras 的 WSE（Wafer Scale Engine）在整片晶圆上做了接近一百万个 PE。在这种规模下，全 chip 级别的 cache coherence 代价太大了，不再可行。Cerebras 的选择是在 PE 之间采用消息传递，本质上就是 send/recv，只不过不叫这个名字。这是物理的强制，不是理论的胜利。当全一致性撑不住的时候，硬件设计师会自动退回到消息传递。这件事后面会反复看到。

### 2.2 Die-to-Die / 多芯粒封装：延迟仍然主导，但张力开始冒头

**物理约束**

Die-to-Die 互联的范围是 ~mm 到 ~cm，延迟 ~1-10ns。物理层走高密度封装的 microbump（间距 25-55μm，取决于具体工艺），信号速率从 UCIe 标准封装的 4-16 GT/s 到先进封装的 24-32 GT/s。

和片上 NoC 相比，die-to-die 的延迟高了一个数量级但还是极低的。和片间互联（如 PCIe）相比，它又低了接近一个数量级。UCIe 标称 PHY 层延迟不超过 2ns，加上协议栈也比 PCIe 的几十纳秒快一个数量级。

**工作负载模式**

对 AI 芯片来说，多 die 封装的动机通常是把大到一颗 die 装不下的设计拆开。比如计算 die 叠 HBM die（H100 的 CoWoS 封装），或者计算 die 之间互联（Blackwell 的双 die 设计）。在训练场景下，如果模型被切分到多个 die 上，die 间通信的模式和 NoC 上类似：张量并行（TP）的 AllReduce 是最主要的需求，粒度是细粒度的激活值和梯度。

**当前设计选择**

Load/Store 再次主导，形式比 NoC 上更多样：

- UCIe 的协议层原生支持 CXL.mem 和 CXL.cache。CXL.mem 让 CPU 以 load/store 语义直接访问远端内存（不带缓存一致性，纯地址访问）。CXL.cache 则更进一步，通过硬件 snoop 维护主机和设备的 cache coherence。这两种模式的硬件代价不在一个量级——CXL.mem 只需要地址翻译，CXL.cache 还要 snoop filter、MESI 状态机、snoop 通道。在 D2D 距离上，两种都能跑，但选哪种决定了硅面积和功耗。
148	- NVLink-C2C（Grace-Hopper 互联）：900 GB/s 带宽。底层跑的是 NVLink 专有协议，通过 ATS 做地址翻译，以一致性方式暴露 load/store 语义给 CPU。和 CXL.cache 的定位接近，但带宽高一截。
- CHI-C2C（ARM 的方案）：面向多芯粒 SMP 和 coherent accelerator attach，协议层把 CHI 的事务（ReadUnique、ReadShared 等）映射到 Flit 包上，直接对接 UCIe 的物理层。

这些协议都在干同一件事：**让多 die 伪装成一块大芯片。**

**为什么**

Die-to-die 的延迟仍然在 load/store 的甜区内。相比 NoC 的 ~1ns，die-to-die 的 ~10ns 贵了 10 倍，但这个量级的延迟做一次 cache miss + cache fill 仍然可以接受。HBM 的访问延迟也是 ~100ns 级，die-to-die 比本地 HBM 还快。

更重要的是，如果多 die 之间不维持一致性，软件写作者就必须显式地管理"这块数据在哪个 die 上"以及"什么时候搬迁"。这把多 die 芯片的复杂性直接暴露给软件，而软件行业花了三十年才勉强学会写共享内存的多核程序（大多数还没学会）。硬件架构师的判断是值的：在硬件上多花面积和功耗维持一致性，换软件能继续当单芯片写。芯片面积是一次性的，软件生态是持续迭代的。

**张力**

问题出在 die 的数量上。

一个双 die 的 Blackwell 维持全一致性需要几个 Home Node 和一套 snoop 通道，代价可控。但如果你想把 16 个 die 连在一起呢？32 个呢？全一致性的 snoop 流量和 snoop filter 开销会随 die 数量非线性增长。这时候你面临选择：要么继续咬牙维持全一致性（代价越来越大），要么退到部分一致性（只在一组 die 内部维持一致性，组间用消息传递），要么干脆放弃一致性。

这正是 CXL 3.0 和 UALink 在探索的方向。共享内存的"域"可以大到什么程度？物理上，延迟随距离线性增长。架构上，一致性的复杂度随参与者数量超线性增长。两条曲线一定会在某个点交叉。交叉点之后，load/store 的一致性代价超过收益，send/recv 的简单性开始胜出。

目前这个交叉点还没有在 die-to-die 层面被清楚定义，因为 die 的数量还不够多。但当 chiplet 生态再成熟一些、单封装内的 die 数从个位数涨到十位数时，这个问题会非常实际。

### 2.3 节点内 GPU 间：两套语义正面交锋的地方

**物理约束**

NVLink 第五代：每个 lane 224 Gbps PAM4（有效 200 Gbps），x18 link 双向 1.8 TB/s。走线长度在 0.5-3 米之间（NVL72 机柜的线缆背板）。延迟估计在 100ns-1μs。

这个区间是两套语义拉锯最激烈的地带：够低，低到 load/store 的一致性代价不算不能接受；也够高，高到 send/recv 的消息开销（post buffer + completion）可以被摊还。

**工作负载模式**

节点内 GPU 间通信的模式是 AI 训练中最复杂的：

- 张量并行（TP）：单层权重切到多 GPU，每个 GPU 算完自己那部分后需要 AllReduce 汇总。粒度是细粒度的激活值（每次前向都是一批激活值跨 GPU），延迟敏感。这是最接近 load/store 使用场景的模式。
- 流水线并行（PP）：不同层在不同 GPU，GPU 间以 micro-batch 为单位传递激活值。粒度中等，延迟敏感度中等。
- 专家并行（EP，MoE 专属）：token 跨 GPU 找专家，All-to-All 通信。粒度是中等大小的 chunk，但模式动态。对带宽敏感超过延迟。

同一组 GPU，同一个 hour，可能同时跑着这三种通信模式。这就是为什么这个层次最复杂。没有一种语义可以覆盖全部需求。

**当前设计选择**

NVLink + NVSwitch 的架构做了一个看似矛盾的设计：

- NVLink 的编程接口暴露了 load/store 语义。通过 ATS（Address Translation Services），GPU 可以直接访问远端 GPU 的 HBM，就像访问自己的 HBM。NVIDIA 把这叫 Unified Memory。做过 profiling 的人都知道它的性能特性和本地 HBM 完全是两回事，但接口确实是 load/store。
- NVLink 的底层传输是消息。物理层跑的是基于 flit 的包交换（flit 大小为 128 bits，16 bytes，这是公开文献中的数字；NVLink 4/5 的精确 flit 格式未公开但大概率类似）。每个 flit 带着地址和 CRC，在这个层面它和一个消息没有本质区别，只是粒度小得多。
- NVSwitch 是包交换交换机。它做的事情和 InfiniBand 交换机在逻辑上没有本质区别：收到 flit，查路由表，转发到目标端口。区别在于端口数少（8-16 个）、带宽极高（每个端口 1.8 TB/s）、延迟极低（~100ns），而且是电路交换和包交换的混合体。
- SHARP/NVLS 做网内计算。Reduce 操作在交换机上完成。GPU 发数据到交换机，交换机做加法，返回结果。功能上类似 AllReduce 的 ring/tree 算法，但不需要 GPU 参与通信循环。从 GPU 角度看，它发了一个"reduce"请求，拿到了结果，这更像 RPC 而不是 load/store。

**为什么是混合**

因为 TP 和 EP 的通信特征差得太远了。TP 的 AllReduce 需要 ~1μs 级别的延迟才能不让计算空等。这个延迟下 send/recv 的 post buffer + completion 的 round-trip 开销太大。你需要 GPU 能直接"看"到远端的激活值，像在本地一样。这就是 load/store 的优势。

但 EP 的 All-to-All 是另一回事。数据量非常大（DeepSeek-V3 上每次是 GB 级别），token 的目标 GPU 是动态的。如果全用 load/store 来做，每笔 cache-line size 的访问都要走一遍地址翻译 + snoop 流程。GB 级别除以 64B 等于上千万笔 cache line 访问，每笔都要查 ATS 表。这个开销不可接受。所以实际上 All-to-All 底层跑的是 bulk DMA transfer，本质是 send/recv，只是 GPU 的 kernel 把它包装成了"发给 GPU 3 的专家"这样的语义。

**张力**

NVSwitch 的物理互联规模随 GPU 数线性增长：每个 GPU 有 18 条 link，共 G × 18 条物理链路；每片 NVSwitch 芯片有 72 个 port，需要 S = ceil(G × 18 / 72) 片 Switch。每个 GPU 把 18 条 link 均匀分发到所有 Switch 芯片（每条 link 连到不同的 Switch），每片 Switch 通过内部 crossbar 提供对所有 GPU 的全端口无阻塞交换。NVL72 用了 18 片 NVSwitch，每片内部 crossbar 72×72 端口。换句话说，GPU 之间不是全互联布线——是 Clos 式拓扑，crossbar 承担了全互联的逻辑功能。

NVL72 机柜内部总计用了约 5,184 根铜缆，累计长度约 3.2 公里，每根实际不过 1-3 米，是从 compute tray 拉到中间 NVSwitch tray 的短距 DAC。N 到 72 还撑得住铜缆。再往上 NVL576，铜缆受限于 224 Gbps PAM4 在铜介质上的信号衰减（超过 2-3 米后 raw BER 快速恶化），光学是唯一选择。光互联的核心问题不是 raw BER（光纤天然 pre-FEC BER ~10⁻¹⁵，比铜缆的 ~10⁻⁴ 好得多），而是引入了更多收发器延迟波动和链路训练复杂度，对 load/store 所需的严格确定性延迟不利。

NVSwitch 底层是包交换，这已经暗示了一个判断：在这个距离和规模上，电路交换（load/store 的天然伙伴）已经不太可行。NVSwitch 选择包交换（send/recv 的天然伙伴）作为物理基础，然后在上层搭建了一层 load/store 的抽象。这层抽象不是没有代价。地址翻译、page fault 处理、TLB shootdown，这些开销在单 GPU 上是成熟的，在多 GPU 上是被动接受的，在 72 GPU 全互联上是当前工程极限，在 576 GPU 上是什么还不清楚。

### 2.3.1 插曲：UALink —— Load/Store 的去一致性实践

在 NVLink 主导的 scale-up 互联领域，2024 年 5 月出现了一个不可忽视的新角色：UALink（Ultra Accelerator Link）。它由 AMD、Intel、Google、Meta、Microsoft、Cisco、HPE 联合发起，2025 年 4 月发布了 1.0 规范，目前已经有超过 85 家成员。AMDA 计划在 MI400（2026）上首发支持。

UALink 和本文主题高度相关，因为它在设计上做了一个明确的选择：

- **语义层**：使用 load/store + atomics 语义。每个加速器可以直接以地址访问远端内存。
- **一致性层**：软件管理的一致性（software-managed coherency），不是硬件 cache coherence。没有 snoop，没有硬件维护的 MESI 状态机。
- **规模目标**：每个 pod 最多 1,024 个加速器。
- **物理层**：200 GT/s per lane，基于 IEEE 802.3dj 以太网 SerDes，但换了更轻的 FEC 以降低延迟。
- **延迟**：sub-1μs round-trip（和 PCIe switch 相当）。

UALink 的位置刚好在 NVLink 和 InfiniBand 之间：它保留了 load/store 的编程便利性，但砍掉了硬件一致性的硅代价和扩展性负担。一致性完全扔给软件——在 AI 训练这个特定场景下（数据流已知、通信模式可预测），这个 trade-off 很可能是值的。它说明了一件事：**load/store 语义不需要和硬件一致性捆绑销售。** 至少在 scale-up 的规模（1024 加速器），去掉一致性可以让 load/store 推得更远。硬件一致性不是 load/store 语义的宿命，而是 load/store 实现的一个选项——一个在 N 小的时候很值、N 大到一定程度就太贵的选项。

### 2.4 节点间：Send/Recv 的主场，但 Load/Store 在敲门

**物理约束**

节点间互联：距离 ~10-100m，延迟 ~1-10μs，误码率不再是"可以忽略"。InfiniBand 的 BER 要求是 10⁻¹⁵，实际运行中 10⁻¹² 到 10⁻¹³ 是常态。带宽单端口 NDR 400 Gbps（~50 GB/s），XDR 800 Gbps（~100 GB/s），靠多端口聚合做到更高的有效带宽。

几微秒的延迟看起来只比节点内高了一个数量级，但对 load/store 来说一笔 cache miss 的代价从 ~100ns 变成了 ~10μs，差了 100 倍。这个差距大到"把远端内存当成本地内存用"在性能上完全不可行，除非你的计算稀疏到每几万个 cycle 才需要一次远端访问。

**工作负载模式**

节点间通信以数据并行（DP）的梯度 AllReduce 和 MoE 的跨节点 All-to-All 为主。两种模式的共同特征是：数据量极大（每步梯度量等于模型参数量），对延迟不敏感（可以 pipeline 隐藏），对带宽极端敏感。NCCL 的 ring allreduce 在节点间就是典型的 send/recv：每个 GPU 只和两个邻居通信，数据以 chunk 为单位（不是 cache line）沿着环流动。

**当前设计选择**

Send/Recv 主导。MPI、NCCL、Gloo，所有主流节点间通信库都基于消息模型。RDMA 提供了一些接近 load/store 的操作（Read/Write），但仅限于没有缓存一致性的情况。在需要一致性的场景下，这些操作需要显式的 fence 和 memory barrier，本质上是用软件做一致性。

**为什么**

三个根本原因。

**第一，规模。** 一个 2048 GPU 的集群，节点数是 256（每节点 8 GPU）。如果每个节点都试图把其他 255 个节点的内存当成自己地址空间的延伸，地址翻译表的规模是天文数字。而且 snoop 流量会把网络彻底塞满，不是"可能塞满"，而是一定塞满。每一次对共享内存的写入，在 load/store 一致性协议下，都要向所有可能持有该地址副本的节点发 snoop。255 个目标，每一笔写入 255 条 snoop 消息。就算有目录协议优化（只向实际持有副本的节点发 snoop），目录本身的存储和查找在 256 个节点的规模下也不是免费的。

**第二，故障。** 在一个机柜里，GPU 0 出问题了，整个机柜训练停掉可以接受，这就是 NVLink 域内的故障模型。但在数据中心里，256 个节点中每天有一个出故障是常态，不是意外。load/store 语义下如果一个持有脏 cache line 的节点突然挂掉，其他节点对那个地址的后续访问会怎样？要么等到超时（几十毫秒甚至几秒），要么触发 Machine Check Exception 把相关进程全部 kill。两种结果在大规模训练中都不能接受。超时意味着所有 GPU 空等，MCE 意味着整个训练任务崩溃。send/recv 语义下一个节点挂了只影响它参与的那些 QP，其他 QP 不受牵连。因为 buffer 是接收方自己的，不存在"别人的脏数据在我这里"的问题。

**第三，流控。** load/store 语义下如果搭配的是无端到端信用机制的传输层（比如 PCIe 的原生流控依赖链路层 credit + PFC），写请求是发起方驱动的——发起方不管接收方有没有准备好就发。网络拥塞会导致交换机 buffer 堆积，触发 PFC 反压，而 PFC 对同一链路上所有流量无差别反压，造成头阻塞。send/recv 通常搭配端到端信用机制（比如 InfiniBand 的 credit-based 流控），接收方通过控制信用发放来做端到端流控，这是逐 QP 粒度的。但需要说明：这是传输层实现的选择，不是语义本身决定的。CXL.mem 用的是 load/store 语义但底层也用 credit-based 流控，不走 PFC。所以准确的说法是：send/recv 的队列模型和端到端流控天然匹配；load/store 可以做端到端流控（CXL 证明了这一点），但历史上通常和链路级反压绑定在一起。

**张力**：CXL 和 RDMA 在把 load/store 往上推。

CXL 3.0 的跨机架共享内存可以让多个主机通过 CXL Switch 访问同一个 CXL 内存池，使用 load/store 语义。但它的一致性实现方式值得一提。CXL 3.0 用的是 Home Agent 目录协议——每个内存地址有一个 Home Agent 作为权威，所有设备的请求先到 Home Agent，Home Agent 维护 MESI 状态目录，对需要的一致性操作发 snoop。这是硬件全一致性（所有设备参与 MESI），但不是对称 broadcast-snooping，而是层次化的 directory-based。CXL 还引入了 Back-Invalidation Snoop、多级级联 Switch 的 snoop 路由、Peer-to-Peer 直访（device 可以 DMA 到 peer device 的内存，但仍由 Home Agent 仲裁一致性）。O(N²) 的 snoop broadcast 被压成了 Home Agent 的 O(N) 目录查找——但代价是 Home Agent 成了单点瓶颈和单点故障。

而且在物理距离上，CXL 的 load/store 延迟已经让很多软件隐式假设失效了。单芯片上一个 cache-to-cache transfer 可能 50ns，CXL 跨机架场景下变成 500ns+，10 倍差距。依赖这个延迟假设的软件——自旋锁的实现、NUMA 感知的内存分配器、Linux 内核的调度器——都可能在这个延迟特性下表现异常。

不是要说 CXL 没价值。内存池化可以大幅提高数据中心内存利用率。但它不是"让所有节点看起来像一块大芯片"的魔法——它只是一层硬件抽象，这层抽象之上仍然需要软件知道自己是在和远端内存打交道。

另外值得注意的一点是 CXL.mem 和 CXL.cache 的区别。CXL.mem 是 CPU 以 load/store 语义直接访问设备内存，但**不带缓存一致性**——远端内存的 cache line 不会被 CPU cache，访问时没有 snoop。CXL.cache 才是带硬件一致性的版本（设备可以 cache host 内存，主机和设备的 cache 通过 snoop 保持一致）。两者的硬件代价差一个数量级。很多讨论把 CXL.mem 的"load/store"等同于"一致性"，这是错的。CXL.mem 的 load/store 只是寻址方式，和一致性无关。

---

## 第三部分：六个问题：一个给架构师的决策框架

看过四个层次的具体情况之后，应该能提炼出更一般的东西了。下面六个问题，是架构师在决定某个接口用哪种语义时应该依次回答的。顺序不是随便排的。越靠前的问题决定力越强。

### Q1：延迟预算还剩多少？

这是最硬的约束。延迟不由架构师说了算，由物理（光速）和工作负载（计算密度）联合决定。

< ~100ns：Load/Store 别无选择。

Send/Recv 的 post buffer + completion round-trip 在这个延迟区间内是找死——原因在前面 NoC 那节（2.1）已经讨论过了，send/recv 的握手开销让 ns 级延迟不可接受。在 NoC 上这个判断是绝对的。

> ~10μs：Send/Recv 的开销被摊还。

当传输的 baseline 延迟在微秒级，send/recv 的 post buffer 和 completion overhead（几十到几百纳秒）相对就不算什么了。更重要的是，在这个延迟量级上 load/store 的 cache miss penalty 从"可容忍"变成"无法接受"。一笔远端 cache miss 要 10μs，意味着如果密集地做随机远端访问，IPC 会跌到地板。

中间地带（100ns - 10μs）：两种语义都可以，答案取决于其他问题。

这正是 NVLink 所处的区域，也是两种语义拉锯最激烈的地方。延迟单独不够做判断。

### Q2：访问粒度是什么？

Cache-line 大小（32-64B）、不规则访问 → Load/Store。

Load/Store 以 cache line 为粒度，不额外消耗带宽。不规则访问意味着你无法预测下一个访问的目标地址，这正是 load/store 的随机访问能力擅长的。

大块传输（KB-MB）、规则模式 → Send/Recv。

Send/Recv 的 work request 可以指向任意大小的 buffer，不需要把大块传输拆成无数 cache-line 大小的事务。每笔事务的 overhead（post work request + completion）可以在大块数据上摊还。

NVLink 对这两种模式的处理就很说明问题：TP 的细粒度 AllReduce 走 load/store（通过 ATS + coherence），数据并行的梯度同步走 bulk DMA（本质 send/recv）。不是设计缺陷，是设计选择。同一条物理链路，根据访问模式用不同语义。

### Q3：有多少参与者？

少数参与者、共享内存模式 → Load/Store。

"少数"没有绝对值，由两个因素决定：snoop filter 的容量能不能 cover，coherence 协议的广播会不会吃掉全部带宽。经验法则：单芯片内（< 100 个 Agent）没有压力，die-to-die（< 10 个 die）也没有压力。节点内 GPU（< 72 个）目前在工程极限内。

大规模参与者、消息传递模式 → Send/Recv。

很简单：N 个参与者做全一致性 = O(N²) 的 snoop 交互。N 很大的时候这不是"代价"，是物理上跑不通。

一个推论：如果你的系统必须支持千级甚至万级互联节点，你不是在"选择" send/recv。send/recv 是唯一物理可行的选项。一致性协议 O(N²) 的增长特性在物理上是硬天花板，不是"还没人做到"，而是做不到。

### Q4：故障模型长什么样？

单一故障域 → Load/Store 可以。

如果一个组件挂了整个系统都挂，那就没必要为故障隔离多做设计。一致性域内的故障本来就是全局的。

独立故障域 → Send/Recv 的所有权模型是天然优势。

大规模集群里，孤立节点挂掉是日常。send/recv 语义下，一个节点挂了，它的 QP 进入 error state，其他 QP 不受影响，故障被隔离在 QP 级别。load/store 语义下，一个持有脏 cache line 的节点挂了，所有可能依赖那个 cache line 的节点都可能受影响，故障被扩大到整个一致性域。

这不是"理论上的安全隐患"，是跑过大集群的人的日常。你需要一个节点挂了不影响其他节点。send/recv 给你这个保证，不是因为它有什么神奇的故障处理，而是因为它不建立跨节点的共享状态。没有共享状态就没有共享状态的故障传播。

### Q5：缓冲和流控模型是什么？

生产者-消费者紧耦合、反压天然存在 → Load/Store。

NoC 上如果接收方的 buffer 满了，互联可以直接 stall 发送方。物理距离短、延迟可控，紧耦合的背压在芯片内是可行的。

需要弹性缓冲、生产者和消费者解耦 → Send/Recv。

在节点间，你不能因为某个节点接收 buffer 满了就 stall 整个网络。send/recv 的显式排队让你在 NIC 层面做流控，不用反向传播背压。显式队列还让你能做 QoS，不同优先级的 QP、不同 service level 的 traffic class。

不是说 load/store 不能做 QoS（ACE 的 QoS 信号、CHI 的 QoS 也在做）。但 send/recv 的队列模型天然适合做优先级调度：每个 QP 有自己的队列，调度器可以按优先级轮转。load/store 中所有事务共享一个互联，优先级需要通过额外信号来表达。

### Q6：编程模型需要什么？

这个问题放在最后，不是因为它不重要，而是因为它可以被前面五个问题的答案否定。

共享内存抽象对软件生产力有巨大价值 → 尽量暴露 Load/Store。

让多核编程看起来像单核编程，是硬件过去三十年给软件的最大礼物。写过 MPI 的人都知道显式消息传递有多痛苦。不是因为 MPI 的 API 不好，是因为你需要在大脑里维护"数据在哪个节点上"的全局地图。对张量并行这种细粒度共享场景，共享内存抽象是刚需。没有它，TP 的实现复杂度会爆炸。

极致性能、愿意管 buffer → Send/Recv。

如果前面五个问题中任何一个指向 send/recv，你就不该在硬件上强行提供 load/store 抽象。那个层次上 load/store 硬件代价会吃掉编程模型收益。但你可以把它推到驱动层或 runtime 层去实现：让硬件跑高效的消息传递，软件提供共享内存假象。NVIDIA 的 Unified Memory 就是这么做的，底层是 page fault + migration（本质是消息），上层是 `cudaMallocManaged`（看起来像共享内存）。区别在于 Unified Memory 的性能特性是"看情况"。如果你理解它在做什么，可以写出好的代码。不理解的话，结果是正确但极慢。

### 综合：两栏对照

把上面六个维度放一张表：

| 决策维度 | 倾向 Load/Store | 倾向 Send/Recv |
|----------|-----------------|----------------|
| 延迟预算 | < ~100ns（NoC、D2D） | > ~10μs（节点间） |
| 访问粒度 | Cache-line 级、不规则 | MB 级大块、规则模式 |
| 参与者规模 | < ~100（芯片内、机柜内） | > ~1000（数据中心级） |
| 故障模型 | 单一故障域 | 独立故障域 |
| 流控模型 | 紧耦合反压 | 弹性排队 |
| 编程模型需求 | 共享内存优先 | 消息传递优先 |

这个表不是绝对的，是偏好的方向。一个实际设计通常会落在中间某个位置，你必须判断它更靠近哪一边。至少能让你知道你在 trade off 什么。

---

## 第四部分：边界在移动，但不会消失

### 物理的天花板

如果你只看协议演进趋势，会觉得 load/store 语义在向上蔓延。CXL 把 load/store 从 PCIe 域推到机架级，NVLink-C2C 让它跨了 die，有些提案在讨论"数据中心级共享内存"。好像方向很明确：总有一天万物皆 load/store。

但物理规律不是协议能绕过去的。

- **信号传播延迟**：真空光速 ~3.33 ns/m，但实际介质中更慢。光纤的折射率约 1.5，传播延迟约 5 ns/m；铜缆因介电材料不同，velocity factor 从 0.65 到 0.95 不等，延迟约 3.5 到 5.1 ns/m。10 米跨机架走线，物理传播延迟在光纤上大约 50ns 单程。一个来回 100ns。加上交换机转发、SerDes、协议处理，跨机架的 cache miss 逼近 1μs。这是物理的硬底，不是工程能优化的。
366	- **误码率**：铜缆 DAC 的 raw BER 约 10⁻⁴（224 Gbps PAM4 下），通过 RS-FEC 纠到 FLR ~10⁻¹² 等效。光纤天然 raw BER 即可达 10⁻¹⁵，不需要 FEC。load/store 语义下一个 bit flip 如果发生在地址信号上，你可能往错误的内存地址写了数据。send/recv 有 CRC 和重传，出错了就在那笔消息上解决，代价可控。load/store 的 bit error 修复要 ECC 或更强的编码，代价和复杂度更高。
- **Coherence 天花板**。全一致性的交互复杂度是 O(N²)。有 snoop filter、目录协议、层次化一致性域也不管用，N=1000 的时候 O(N²) 不现实。你看谁在做这件事就明白了。CXL 3.0 不支持 1000 个主机做全一致性，只支持多个主机通过 Switch 共享内存池，层次化的、不是全对等的。不是偶然的。

### 更深一层：不一致性在往上层渗透

这不是说"load/store 有天花板所以 send/recv 永远有用"，这句话太无聊了。我想说的是：**随着 AI 规模继续扩大，系统架构师在主动选择不一致性，而不是被动接受它。**

DeepSeek-V3 的 EP 设计。DeepSeek 选择不做全专家参数的跨 GPU 一致性。每个专家的参数只存在于它所在的 GPU 上。如果一个 token 要访问专家，它通过 All-to-All 把 hidden state 发过去，算完再发回来。这是消息传递，不是共享内存。DeepSeek 没有试图让 token 的原始 GPU"看到"远端专家的参数，它接受了"数据在别处"这个事实，然后用通信来 bridge。

回报是巨大的：没有分布式一致性的开销，GPU 专注于计算和通信，而不是花硅面积和功耗维护一个跨越 64 GPU 的一致性域。代价是编程模型更复杂了，需要写 All-to-All kernel，手动管理 token 的分发和收集。

但考虑到 AI 训练框架本身就是由专业团队开发和维护的，不像通用软件那样需要"任何人都能写"，这个代价在 AI 这个场景下是值得的。这也是为什么 DeepSeek-V3 技术报告里那段"给硬件厂商的建议"值得重新读一遍。他们要的不是把一致性做得更好，他们要的是更好的通信原语：unified address space、multicast、reduce in-network。

**他们要的不是 load/store 的一致性。他们要的是 send/recv 的更高效实现。**

### 把边界推到哪最划算

把从 NoC 到数据中心的整个互联频谱画成一条轴，load/store 在左，send/recv 在右。现在的边界大约在节点的机柜背板位置。NVSwitch 是 load/store 的最后据点，InfiniBand 是 send/recv 的经典领地。CXL 会把边界向右推一点。跨机架的共享内存在物理上可行，只要你不要求延迟特性和本地内存一样。但推到哪会停下来？

我的判断是：推到第一次"一致性管理代价超过共享内存编程模型收益"的地方。这个点在工程上不是固定的，取决于应用场景对编程模型的依赖程度、硬件团队愿意为一致性支付多少硅面积，以及最重要的——部署规模。

用一个 8 GPU 推理小盒子（比如 DGX Station），提供机箱级 cache coherence 完全合理。一致性管理代价在 8 个 Agent 规模下可以忽略，共享内存编程模型让模型部署团队不用操心数据布局。

用 2048 GPU 训练集群，任何超出一个节点的一致性都不可接受。注意力要放在让消息传得更快，而不是让消息看起来不像消息。

两个极端之间，是 2026 年的工程前沿。前移到哪里取决于物理约束、成本结构，和 AI 工作负载对通信模式的实际需求。

---

## 结语：比答案更重要的

回到开头的问题。send/recv 还是 load/store？

诚实回答：看情况。

但"看情况"不是敷衍。它可以被六个问题系统拆解。你需要知道边界划在哪里，为什么要划在那里。你需要知道 die 数从 2 涨到 8 时一致性管理代价怎么涨。你需要知道 GPU 数量从 72 涨到 576 时光互联误码率对 load/store 语义意味着什么。你需要知道 DeepSeek 做了那么多优化之后对硬件厂商的第一诉求为什么不是"更好的 cache coherence"而是"把通信从 SM 卸载出来"。

这些问题的答案不在任何一本教科书里，因为边界本身每年都在移动。而且从 2024 年开始移动速度在加快——UALink 来了，CXL 3.0 在推，NVLink 在往上走，DeepSeek 在往下压。在这样一个时刻，会问问题比会背答案有用得多。

---

**主要参考资料**：

1. ARM, *AMBA AXI and ACE Protocol Specification*, Issue H.c, 2013.
2. ARM, *AMBA CHI Architecture Specification*, Issue E.b, 2019.
3. NVIDIA, *NVIDIA NVLink and NVSwitch*, Technical Overview, 2022-2024.
4. CXL Consortium, *Compute Express Link Specification 3.0*, 2022.
5. UCIe Consortium, *Universal Chiplet Interconnect Express Specification 2.0*, 2024.
6. InfiniBand Trade Association, *InfiniBand Architecture Specification*, Vol. 1, 2023.
7. DeepSeek-AI, "DeepSeek-V3 Technical Report", arXiv:2412.19437, 2024.
8. Cerebras Systems, "Wafer-Scale Deep Learning", Hot Chips 2021.
9. Lebeck et al., "Power Aware Page Allocation", ASPLOS 2000.
10. Roman Glebov et al., "Uncovering Real GPU NoC Characteristics", MICRO 2024.
