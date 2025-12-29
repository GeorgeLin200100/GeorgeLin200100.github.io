---
layout: post
title: CHI-Architecture-Learning-Notes
subtitle: 有你有我雪中送火，翻天覆海不枉最初
date: 2025-12-17
author: George Lin
header-img: img/post-bg-gen.png
catalog: true
tags:
- AMBA
---



# 系列博客说明

本系列博客将围绕片内一致性总线CHI协议展开。首篇博客将基于ARM官方的Learn the architecture - Introducing AMBA CHI文档进行总结，意在对CHI协议的基本定义、事务流程及性能优化方案支持三个方面有初步的认知。本博客不会照搬协议，会掺杂一些个人理解，不当之处请指正。之后的博客可能会聚焦CMN-700具体实现、CHI-C2C协议、gem5-c2c建模或CHI与UCIe的交互四方面展开，意在通过约8-10篇博客，初步掌握CHI协议、一致性实现。

# CHI协议的定位

CHI协议的定位是片内多核互连总线。ARM官方提到，CHI协议不会定死使用的具体NoC拓扑，但其明确支持的仅有ring, mesh和crossbar三种，如下：

![image-20251229191533302](../images/2025-12-29-CHI-Architecture-Learning-Notes.assets/image-20251229191533302.png)

其中，ring拓扑实现简单，但点对点延迟随着节点数量增多而线性增大，因而适合中等规模系统；crossbar拓扑是全连接，跳数最少，但实现代价很大，适合小规模系统；mesh比较复杂，点和点之间可以有多种路由方案，适合大规模系统。



# CHI协议的演进

CHI从2014年发布以来，已经来到了Issue. G。

CHI-A已有：事务排序模型，独占式访问，Distributed Virtual Memory (DVM)操作。

CHI-B添加：原子事务，Reliability, Availability, and Serviceability (RAS)支持，直接内存传输（Direct Memory Transfer）和直接缓存传输（Direct Cache Transfer）(这俩都是用来在读返回的时候绕过HN直接到RN从而减少延迟的，挺有用)

CHI-C和CHI-D都是一些细微的改动。

CHI-E改动较大，比如加了个与DMT和DCT对偶的Direct Write-data Transfer



# CHI协议的基础概念

### 节点（Node）

节点大体分三种：RN, HN和SN。还有一类MN。

RN (Request Node)类似AXI中的master，可以发起读写请求。

SN（Subordinate Node）类似AXI中的slave，可以是DDR这类memory controller或者外设。

HN（Home Node）比较有意思，类似一个中转/集散中心。比如，HN收到RN发来的request，另起一个request向SN发起请求，请求完成后再返回给RN。HN也不止承担这类点对点功能。比如，某个RN想要让自己的某个cache line从shared状态变成unique状态，就跟HN说，去，你给跟这个cache line有关联的RN都发个snoop，让这些RN如果有这个cache line的就把这个cache line invalidate掉，再通过你（HN）汇总消息了告诉我(RN)。

RN分为三类，Fully Coherent的RN-F, IO Coherent的RN-I, 和IO Coherent加上DVM支持的RN-D。RN-F有支持coherence的cache，可以应答snoop。RN-I不带有支持coherence的cache，不可以应答snoop，RN-D不带有支持coherence的cache，不可以应答snoop，但可以接收DVM messages。

HN分为三类，Fully Coherent的HN-F，Non-Coherent的HN-I和Miscellaneous的MN。前两种很好理解，最后一种指不支持Coherence但是可以处理DVM请求。

SN分为两类，Fully Coherent的SN-F和IO Coherent的SN-I，不再赘述。

总结表如下：

![image-20251229222801077](../images/2025-12-29-CHI-Architecture-Learning-Notes.assets/image-20251229222801077.png)

![image-20251229222817528](../images/2025-12-29-CHI-Architecture-Learning-Notes.assets/image-20251229222817528.png)

### IO Coherent和Full Coherent

IO Coherent 描述的是不带cache的设备（如 DMA、GPU 或加速器）如何与拥有缓存的处理器（如 CPU）安全地共享数据。

**Full Coherent (全一致性):** 比如 CPU 之间。大家都有 Cache，数据可以互访，且大家都需要被别人“监听”（Snoop）。

**IO Coherent (IO 一致性):** 只有一方（CPU）有 Cache，另一方（IO）没有 Cache（或者其 Cache 不对系统可见）。IO 设备能看到 CPU 的 Cache，但 CPU 不需要去 Snoop IO 设备。**IO Coherent** 意味着：当一个 IO 设备（请求者）发起访问时，硬件会自动检查系统的 Cache 状态。

- **读操作：** 如果数据在某个 CPU 的 Cache 里是“脏”的（Modified），硬件会自动把最新的数据传给 IO 设备。
- **写操作：** 如果 IO 设备要写入数据，硬件会自动使所有 CPU Cache 中对应的缓存行失效（Invalidate），确保下次 CPU 读取时能拿到 IO 设备写入的新值。

### CHI的缓存行状态（Cache line states）

ACE是valid-invalid, unique-shared, clean-dirty三对组合，CHI是valid-invalid, unique-shared, clean-dirty，partial-empty四对组合，但部分组合不被支持。被支持的组合如下：

![image-20251229223115651](../images/2025-12-29-CHI-Architecture-Learning-Notes.assets/image-20251229223115651.png)

这里重点讲下CHI新增的两个状态，UDP (Unique Dirty Partial)和UCE (Unique Clean Empty)。这两者基本功能都是省了一次内存到缓存的缓存行读，最终优化了写入性能，降低带宽浪费。

### UDP (Unique Dirty Partial)

UDP解决的是**“我只想写一部分，而且我懒得（或者没能力）先把旧数据读回来合成”** 的问题，它把“合并数据”这个重活累活从 CPU/设备端甩给了总线系统（HN-F）。它是为了高性能 SoC 处理碎片化数据写入而引入的特殊状态。

在没有 UDP 以前，如果你只想修改 Cache Line 里的 4 个字节（即 **Partial Write**），流程是：

1. **ReadUnique：** 从内存读回整行（64字节）数据。
2. **Merge：** 在 CPU 内部把你的 4 字节新数据和读回来的 60 字节旧数据合并。
3. **Dirty：** 标记为 Unique Dirty 状态。

有了 **UDP** 以后，流程简化为：

1. **发送写请求：** 发起一个 `WriteUniquePtl`（Partial）请求。
2. **直接传输：** 节点不从内存读数据，而是**直接把这 4 字节数据发给中心节点（HN-F）**。
3. **进入 UDP：** 在某些实现中，RN 可以在本地暂存这部分数据并标记为 UDP。
4. **下游合并：** 最终由 HN-F 负责从内存读出旧数据，并与这 4 字节合并。

对于发起请求的 CPU 或 IO 设备（RN）来说，它**确实省掉了一次把数据读进自己 Cache 的过程**。本质上是把合并数据的任务从RN丢给了HN。

UDP 状态在 **IO Coherent** 场景（比如 DMA 或 PCIe 设备接入）中有独特价值：首先，**其简化了IO模块的硬件逻辑。** 很多小的 IO 模块内部并没有复杂的“合并（Merge）”电路。如果非要它先读再写，它还得额外准备一块 Buffer 来存读回来的数据。其次，他提高了IO设备的**带宽利用率。** 如果一个设备只是不断地往内存写零碎的状态位，使用 UDP 模式可以让数据“单向奔流”，只有写，没有读。

需要注意，在 UDP 状态下，虽然 RN 本地的数据是不完整的，但 CHI 协议通过 **Byte Enable (BE)** 信号解决了问题。 当 UDP 状态的数据被写回（Evict）或者被其他核 Snoop 时，它会明确告诉总线：*“这 64 字节里，只有第 0-3 字节是我的新数据，剩下的字节请以内存或下游 Cache 为准。”*

### UCE (Unique Clean Empty)

CHI 协议引入 **UCE (Unique Clean Empty)** 的意义在于**解耦了“所有权”和“数据内容”**。

在没有 UCE 这种机制的系统中，所有的写缺失（Write Miss）都会遵循 **“先读再改”** 的逻辑：

1. **Read-Allocate：** 即使是写操作，硬件也会先发起一个“读”请求，把数据搬到 Cache 里。
2. **Modify：** CPU 在 Cache 中修改那几个字节。
3. **Complete：** 标记该行为 Dirty。

**即使你打算覆盖全部 64 字节**，传统的硬件逻辑通常比较“呆板”，它无法预知你接下来的指令是只写 1 字节还是写满整行。为了保证逻辑的一致性和简化设计，它统一采取“先读回来占个坑”的做法。

但UCE状态优化了这个流程，即明确给了软件一个“承诺”：*“我保证会写满整行，请只给我权限，不要浪费带宽给我传数据。”*

典型的例子是 **C 语言中的 `memset` 操作** 或者 **DMA 大批量数据传输**。当程序明确知道要初始化一片内存时，使用 CHI 的 `MakeUnique` 指令进入 UCE 状态，可以省去大量无意义的内存读取，从而让系统带宽几乎翻倍。

可以看有UCE和没有UCE时写一个完整缓存行的流程对比

| **场景**     | **传统做法 (ReadUnique)**       | **CHI 优化做法 (MakeUnique)**    |
| ------------ | ------------------------------- | -------------------------------- |
| **步骤 1**   | 请求独占权 + **请求数据**       | **仅请求独占权** (MakeUnique)    |
| **步骤 2**   | 等待内存/其他 Cache 返回数据    | 系统返回确认，**无数据传输**     |
| **步骤 3**   | 覆盖数据，状态变为 Unique Dirty | 状态进入 **UCE**，直接填入新数据 |
| **带宽消耗** | **高** (Data 占用了总线带宽)    | **极低** (只有控制信令)          |

### 