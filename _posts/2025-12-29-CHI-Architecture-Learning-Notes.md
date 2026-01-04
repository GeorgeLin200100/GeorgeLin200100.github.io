---
layout: post
title: CHI-Architecture-Learning-Notes
subtitle: 有你有我雪中送火，翻天覆海不枉最初
date: 2025-12-29
author: George Lin
header-img: img/post-bg-gen.png
catalog: true
tags:
- AMBA
---



# 系列博客说明

本系列博客将围绕片内一致性总线CHI协议展开。首篇博客将基于ARM官方的Learn the architecture - Introducing AMBA CHI文档进行总结，意在对CHI协议的基本定义、事务流程及事务类型三个方面有初步的认知。本博客不会照搬协议，会掺杂一些个人理解，不当之处请指正。之后的博客可能会聚焦CHI协议性能优化方案、CMN-700具体实现、CHI-C2C协议、gem5-c2c建模或CHI与UCIe的交互四方面展开，意在通过约8-10篇博客，初步掌握CHI协议、一致性实现。

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

#### UDP (Unique Dirty Partial)

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

#### UCE (Unique Clean Empty)

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

#### Unique/Shared, Clean/Dirty

一句话总结：Unique一定是Unique的，Shared不一定是Shared的，Dirty一定是Dirty的，Clean不一定是Clean的

Unique：该缓存行只在这个cache中存在

Shared: 该缓存行不一定只在这个cache中存在（可能只在这个cache，也可能在多个cache中）

Dirty: 该缓存行的数据与主存不同，且有责任更新主存

Clean：该缓存行的数据可能与主存不同（可能相同也可能不同），但不负有更新主存的责任

### SAM（System Address Map）

地址映射表，RN中的SAM用于将地址翻译成HN节点索引，HN中的SAM用于将地址翻译成SN索引。需要注意的是，同一个地址被不同RN中的SAM路由到的HN应该一致。

### CHI节点的六个通道

![image-20251230142357181](../images/2025-12-29-CHI-Architecture-Learning-Notes.assets/image-20251230142357181.png)

每个RN-F都有六个通道，往外发的有TXREQ, TXDAT, TXRSP三个，往内收的有RXSNP, RXDAT, RXRSP三个。

具体功能看图。这里仅举几个例子。

比如RN-F要发起一笔写，用到的就是TXREQ, TXDAT, RXRSP. 最后还要用TXRSP跟HN说声。

RN-F要发起一笔读，用到的就是TXREQ, RXDAT, RXRSP，最后还要用TXRSP跟HN说声。

RN-F被snoop了，用到的就是RXSNP和TXRSP，如果snoop成功了，还得用到TXDAT.



比较惊艳的是，从CHI-E开始，允许复制通道，进而灵活地控制带宽。这种复制可以是两个粒度：

- 粗粒度：复制一整套六个通道，如图3-4
- 细粒度：复制单个通道，如图3-5

![image-20251230143204081](../images/2025-12-29-CHI-Architecture-Learning-Notes.assets/image-20251230143204081.png)

![image-20251230143214255](../images/2025-12-29-CHI-Architecture-Learning-Notes.assets/image-20251230143214255.png)



### Flits

![image-20251230143503549](../images/2025-12-29-CHI-Architecture-Learning-Notes.assets/image-20251230143503549.png)

因为是片内总线，CHI里的Flit不用像PCIe一样序列化，上图展示了Request Flit的基本结构

有几个要点值得注意:

- 每个Flit都伴有一根valid线
- Opcode域最重要，标明了传输的类型（比如ReadOnce, CleanInvalid等）
- 四个ID：SrcID, TgtID, TxnID, DBID。不同类型Flit中需求的不同ID如下表：
  - SrcID标明是谁发的这个Flit
  - TgtID表明这个Flit要发去哪里
  - TxnID类似AXI里面的ID，是点对点传输中不同的transaction的区分，功能是支持flit的outstanding（最大256或1024）
  - DBID（Data Buffer ID）比较有意思，只在Response和Data Flit中出现。 我在下面专门一个开section说明。

![image-20251230150655436](../images/2025-12-29-CHI-Architecture-Learning-Notes.assets/image-20251230150655436.png)

### DBID

DBID在写/读操作中的作用不一样。在写操作中，主要用于数据流向控制，即告诉发送方往哪里填数据（如果DBID耗尽，写请求会卡死）；在读操作中，主要用于资源的清理（如果DBID释放慢，HN的Tracker会满，无法接新的读请求）。

##### 写操作中的DBID

当请求者（Requester）发送写请求后，响应者（Completer，如 HN-F）如果准备好了接收数据的 Buffer，会返回一个带有 **DBID** 的响应（如 `DBIDResp`）。这个 DBID 就代表了接收方内部那个特定的数据缓冲区。发送方在后续发送数据包（Data Packet）时，会将收到的这个 **DBID** 填入数据包的 **TxnID** 字段中。这样接收方看到数据包时，立刻就能通过 DBID 知道这组数据属于哪个之前的写请求，并直接存入预留的 Buffer。通过 DBID，接收方可以精确控制流入的数据量。只有发放了 DBID，发送方才能传数据，这有效防止了接收端 Buffer 溢出。

##### 读操作中的DBID

读操作中，DBID主要用于两个方面：

首先是在SN与HN的交互中，作为读请求中的回传标识（Read Receipt）。当 HN 向 SN 发起读请求时，SN 可能会返回一个 `ReadReceipt`（读收据）。SN 通过这个响应告诉 HN：“我已经收到你的读请求了，并且给这个请求分配了一个 **DBID**（槽位）。这样 HN 就知道这个请求已经在 SN 的队列里排上号了，HN 可以根据这个确认信号来管理自己的事务追踪器（Tracker）。

其次是在RN收到数据后发给HN的CompAck（完成确认）中，帮助HN释放对应buffer资源。对于某些读操作（如 `ReadShared`），当 RN 收到数据后，需要发送一个 `CompAck`（完成确认）给 HN。此时，HN 在之前给 RN 发送数据时，可能会带上一个 **DBID**。RN 在回复 `CompAck` 时，会把这个 **DBID** 带回去。这告诉 HN：“我收到数据了，你之前在内部为这个事务所占用的那个 Buffer（由该 DBID 标识）现在可以被释放，给别人用了。”

### Request Order和Endpoint Order

这俩都属于CHI的保序机制。Order字段在最初的请求中定义。

Request Order（请求顺序）是较窄范围的保序约束。它保证来自**同一个发起者 (Requester)** 且发往**同一个内存地址 (Same Address)** 的多个事务，按照它们发出的顺序被处理，即：约束范围仅限于相同地址。发起者（如 RN）在发出后续请求前，必须先收到前一个请求的确认（如 `ReadReceipt` 或 `DBIDResp`），以确保前一个请求已经到达了系统的“保序点”（Point of Serialization）。典型的应用场景包括“读后写 (RAW)”或“写后读 (WAR)”同一变量。

Endpoint Order（端点顺序）是更宽、更强的保序约束。它保证来自**同一个发起者**且发往**同一个端点地址范围 (Same Endpoint Address Range)** 的所有事务，按照它们发出的顺序到达该端点，即：约束范围是整个**地址段**（通常对应一个具体的 Slave 节点，如某个内存控制器或外设）。典型应用场景比如顺序访问外设寄存器。例如，你先配置 DMA 的源地址寄存器，再配置目的地址寄存器，最后启动 DMA。这三个操作地址不同，但必须按序到达同一个外设端点。

可见，**Endpoint Order 包含 Request Order。** 如果你指定了 Endpoint Order，那么同一地址的顺序自然也被保证了。

发起者怎么判断何时可以发起下一个请求？CHI协议给出的答案是：如果是读操作(ReadNoSnp和ReadOnce)，发起者收到ReadReceipt信息之后，即可发起下一个请求。如果是写操作（WriteNoSnp和WriteUnique），发起者收到DBIDResp信息后，即可发起下一个请求。

可以发现，Order字段和HN中序列化点（PoS, Point of Serialization）作用似乎相同，都是用来保序的。那为什么对于非一致性事务（ReadNoSnp/WriteNoSnp）和弱一致性事务（ReadOnce（读到后用完不保留副本）/WriteUnique（推出数据后RN不留副本）），要多此一举加上一个Order字段呢？非一致性事务很好理解，一般不过HN,直接到SN, 因而不会参加HN保序。假如此时有保序需求（比如刚刚提到的顺序访问DMA寄存器），加上Order字段是一种轻量化的告知SN需要保序的手段。对于弱一致性事务，由于不需要同步Cache状态，它们经常绕过 PoS 逻辑以追求低延迟。`Order` 字段是 RN 给互连结构的指令，告诉它在不走一致性流程时，依然要维护逻辑上的先后顺序。

### CHI的Retry机制

CHI的Retry机制有些特别。当传输不成功时，发起者并不会一直重发请求，而是会等待接收者告知发起者特定槽位已经空出，可以发起重传后，才会发起一次重传，且该次重传可以确保成功。具体事务流程如下：

1. 发起者在Request Flit中设置AllowRetry=1，并且把credit类型字段PCrdType设置为0，发给接收者。
2. 接收者如果Requester Buffer已经满了，就返回给发起者一个RetryAck信息。该信息中，会把PCrdType字段设置为一个特定值，比如2.
3. 当接收者能接收这个Request了，就会通过TXRSP通道向发起者发送一条PCrdGrant信息（指明PcrdType=2）
4. 发起者收到PCrdGrant信息并确认其与RetryAck中的PcrdType match之后，就会把AllowRetry设置为0，把刚刚的Request Flit发给接收者。接收者必须接收。



# CHI事务流程

CHI协议在底层架构上和Non-Coherent的AXI总线显著不同，其事务流程也更为复杂。下面将按照Opcode分类进行transaction流程的说明，分为两个层面，一是事务流，二是四个ID怎么用。

### ReadNoSnp

假设有RN 0-2, HN 3-4, SN 5。RN0是读的发起方，SN5是目标。

1. RN0通过SAM选定HN3，通过TXREQ通道向HN3发送ReadNoSnp请求。
2. HN3通过SAM选定SN5，通过TXREQ通道向SN5发送ReadNoSnp请求。
3. SN5通过TXDAT通道向HN3返回CompData信息，包含数据。
4. HN3向RN0返回CompData信息，包含数据，走的是HN3的TXDAT, RN0的RXDAT通道。
5. 如果在最开始的ReadNoSnp请求中ExpCompAck标志位（Expect Completion acknowledgement）被设置为1，则RN0需要向HN3返回CompAck信息
6. HN3收到CompAck信息后会放行对RN0刚刚读到这个缓存行的Snoop（保证Sequential Consistency）

![image-20251231101620524](../images/2025-12-29-CHI-Architecture-Learning-Notes.assets/image-20251231101620524.png)

### WriteNoSnp

同样的，假设有RN 0-2, HN 3-4, SN 5。RN0是写的发起方，SN5是目标。

1. RN0通过TXREQ通道向HN3发送WriteNoSnp请求。
2. HN3通过TXRSP通道向RN0发送CompDBIDResp信息，表明自己可以通过某个ID的Data Buffer接受写数据.
3. 以下两个事件可以以任何次序发生：
   1. HN3向SN5发送WriteNoSnp信息，SN5向HN3返回CompDBIDResp信息，表明自己可以通过某个ID的Data Buffer接受写数据.
   2. RN0向HN3通过TXDAT通道发送WriteNoSnp所需的写数据
4. HN3把写数据通过TXDAT通道发送给SN5

注意如果在初始的WriteNoSnp请求中，ExpCompAck=1，则在RN发送完写数据并且确认收到了CompDBIDResp之后，RN会向HN发送一个CompAck消息，HN收到CompAck后，菜回真正关闭（Deallocate）这个事务在内部Tracker中的记录。

### MakeUnique

MakeUnique请求中包含snoop操作。

假设还是一样的，RN 0-2, HN 3-4, SN 5。RN0是MakeUnique的发起方，HN3是Snoop的发起方。

1. RN0对HN3发出MakeUnique消息，指示想把A地址的缓存行独占。
2. HN3向RN1和RN2发出SnpMakeInvalid请求（属于Snoop）
3. RN1和RN2将地址A的缓存行Invalidate之后，按任意顺序向HN3返回SnpResp消息
4. HN3在收到RN1和RN2的SnpResp之后，向RN0返回Comp_UC消息（Unique Clean），表明Snoop已经都返回了
5. 此时假如RN2向HN3发起读地址A的ReadShared请求，HN3会阻塞这个请求，因为RN0还没有给CompAck给HN3
6. RN0向HN3发送CompAck消息，正式终结这个事务。
7. HN3收到CompAck消息之后放行刚刚RN2的ReadShared请求，向RN0和RN1发送SnpShared消息。
8. 以下两个事件以任意顺序完成：
   1. RN0返回SnpRespData，把最新的A地址的数据给到HN3
   2. RN1返回SnpResp，表明自己没有A地址的数据
9. 收到RN0返回的数据后，HN3把数据给到RN2
10. RN2向HN3发送CompAck信息，该事务完成。



# CHI事务类型

CHI事务类型繁多，可分为Read、Write、Dataless、Combined Write、Atomic、Other六类。

### 读事务（Read Transactions）

#### 一句话搞清楚CHI不同读事务的语义

CHI 协议中复杂的读事务可以用以下的比喻快速理解，我们把 **互连网络（ICN）** 比作“图书馆管理员”，把 **Requester** 比作“读者”：

- **ReadNoSnp**：跳过管理员直接去私人书库拿书，不管别人手里有没有副本（非一致性读取）。
- **ReadNoSnpSep**：和 ReadNoSnp 一样，但允许管理员先把“书找到了”的消息和“书本内容”分两次告诉我。
- **ReadOnce**：路过顺便看一眼，不打算把书借走存放在自己家里（不存入 Cache）。
- **ReadOnceCleanInvalid**：我看一眼就走，而且建议管理员把馆里所有副本都擦干净并放回仓库（Clean & Invalidate 提示）。
- **ReadOnceMakeInvalid**：我看一眼就走，而且顺手把馆里其他人的副本全撕了，还不负责修补（强制 Invalidate 且不写回）。
- **ReadClean**：我要借书回家看，但我有洁癖，全系统谁手里的书要是脏了，必须洗干净还给图书馆（强制写回内存）。
- **ReadNotSharedDirty**：我要借书，但我绝不当那个负责洗干净书的人，谁爱洗谁洗（拒收 SharedDirty 责任）。
- **ReadShared**：我要借书，怎么快怎么来，书脏不脏、谁负责洗我都不在乎。
- **ReadUnique**：我要借书而且我要在书上涂改，请管理员立刻把别人手里的书全部回收并销毁。
- **ReadPreferUnique**：我大概率要改书，如果现在没人用就给我独占权，要是有人正用着，我也能接受先共享着看。
- **MakeReadUnique**：书我已经借到手了，现在我想申请改书的权限，请管理员把别人的副本都撤了。



#### 一句话搞清楚何时选用何种CHI读事务

**ReadNoSnp**：当你访问**非相干区域**（如配置寄存器或私有内存）时使用，因为不需要和其他核心商量，速度最快。

**ReadNoSnpSep**：场景同上，但当你希望在数据还没准备好时，先让总线确认“收到请求”以**释放资源**时选用。

**ReadOnce**：当你只需要读一次数据（如**搬运数据包或流媒体**）且不想让它占用你宝贵的缓存空间时选用。

**ReadOnceCleanInvalid**：当你读完数据后确定**短期内没人会再用**，想顺便帮系统“清理垃圾”并把脏数据刷回内存时选用。

**ReadOnceMakeInvalid**：当你读完数据后**确定该数据已作废**（如已处理的丢弃包），想直接从系统中抹除它且不在乎数据丢失时选用。

**ReadClean**：当你需要长期持有数据，且为了**后续能快速关机/休眠**，想逼系统现在就把所有脏副本洗干净写回内存时选用。

**ReadNotSharedDirty**：当你作为消费者读数据，但你的**缓存很小**，不想在被踢出（Evict）时背负写回内存的沉重负担时选用。

**ReadShared**：当你只想做最通用的**普通读取**操作，不带特殊目的，只求以最灵活、最低延迟的方式拿到数据副本时选用。

**ReadUnique**：当你准备**立即修改**某个变量（如执行 `store`）时选用，一步位到位拿走所有人的权限。

**ReadPreferUnique**：当你准备做**原子操作（自旋锁）**时选用，它在不打断别人进度的前提下，尽量帮你提前抢到独占权。

**MakeReadUnique**：当你手里**已经有这行数据**，但突然想从“只读”升级为“修改”模式时，为了省带宽不重传数据而选用。



#### CHI协议上对读事务的详细阐述

##### ReadNoSnp

从RN向Non-snoopable的地址区间读数据，或者从HN向任何地址区间读数据

##### ReadNoSnpSep

从HN向SN读数据，且要求SN仅发data response。Sep是把data response和非data的response Seperate开的意思。

##### ReadOnce

向Snoopable的地址区间拿数据的Snapshot。换言之，读完不会保留在自己的缓存行中。

##### ReadOnceCleanInvalid

在ReadOnce的基础上，建议（hint）持有该缓存行的Node把该缓存行CleanInvalid掉（如果Dirty，要先写回memory）

用法：某个缓存行长远来看仍然有用，但用完这次之后短期内不会再被用到，就可以发起一个ReadOnceCleanInvalid

注意事项：

1. 该缓存行的CleanInvalid是个hint，不能保证完成。
2. ReadOnceCleanInvalid 可能会意外地破坏其他处理器（Master/Agent）正在进行的“独占访问”（Exclusive Access/Atomic 操作），从而导致系统性能下降或逻辑重试。比如，假设 CPU A 正在对地址 `0x100` 进行独占访问（准备写入）。此时 CPU B 发起了一个 `ReadOnceCleanInvalid` 访问同一个地址 `0x100`。根据协议，`ReadOnceCleanInvalid` 可能会导致 CPU A 缓存里的那行数据被 **Invalidate（失效）**。 一旦 CPU A 的缓存行失效，它的**独占监视器（Exclusive Monitor）就会重置**。结果就是 CPU A 的独占访问会**失败（Fail）**，必须重新开始。也就是说，如果你对一个正被频繁进行原子操作（如锁、信号量）的热点内存区域使用这个指令，会导致其他核心的原子操作不断失败和重试，进而引发严重的**缓存颠簸（Cache Thrashing）**，降低系统效率。

##### ReadOnceMakeInvalid

与ReadOnceCleanInvalid的唯一不同是，建议持有该缓存行的Node把该缓存行MakeInvalid掉（如果Dirty，可以直接丢弃，不需要写回memory）

用法：某个缓存行用完这次就不会再被用到了，就可以发起一个ReadOnceMakeInvalid，相比ReadOnceCleanInvalid节省一次WriteBack to memory

注意事项：

1. 该缓存行的MakeInvalid是个hint，不能保证完成
2. 与ReadOnceCleanInvalid一样，ReadOnceMakeInvalid可能会意外地破坏其他处理器（Master/Agent）正在进行的“独占访问”（Exclusive Access/Atomic 操作），从而导致系统性能下降或逻辑重试。
3. ReadOnceMakeInvalid会导致Dirty的缓存行丢失，是有损的。
4. 在ReadOnceMakeInvalid事务中，“失效操作（Invalidation）”与“数据返回（Read Data Response）”之间具有时序与可见性约束。即：
   1. 在数据交给请求者之前，必须先锁定（commit）“让别人失效”这件事。commit不代表动作已经全部完成，而是代表这个失效请求已经在总线互连矩阵中排好队，且**不可撤回**。（否则，如果系统先把数据给了你（请求者），但还没把其他人的缓存标记为“准备失效”，那么在微观时序上，可能会出现短暂的窗口期，导致两个 Agent 认为自己都拥有对该数据的合法控制权，或者看到不一致的值。）
   2. 另一方面，需要保证，任何在 拿到该缓存行的数据之后才开始的对该缓存行的写入操作，都绝对不会被 A 的那个失效动作所干扰。（如果没有这个规则：假设Agent A 发起 `ReadOnceMakeInvalid`。数据发回给了 A。紧接着，Agent B 发起了一个新的 `Write` 事务（更新了该数据）。此时Agent A 延迟的那个“使无效”信号才慢慢悠悠地到达，结果把 Agent B 刚刚写好的新数据给“误杀”使无效了。）
5. 返回给Requester的数据应该处于I或UC或UD状态

##### ReadClean

ReadClean 的语义是：“我需要这份数据，但**我不打算修改它**，如果系统中有脏数据（Dirty Data），请帮我把它洗干净（写回内存/下级缓存）。”也就是，Requester拿到的数据一定是UC或者SC状态。

用法：

1. Requester打算把这个读回来的数据放进不能承担Dirty写回功能的cache里面，所以强制要求别人给自己的是clean状态。
2. 强制去脏。比如当一个集群（Cluster）或核心准备进入**休眠或关断状态**时，它必须清理掉所有的 Dirty Cache lines。如果此时另一个 Agent（比如 GPU 或另一个 CPU）使用 **ReadClean** 读取了这些数据，它实际上“顺手”帮正在准备休眠的核心完成了“写回内存”的工作。这样，当核心真正进入休眠指令时，需要处理的 Dirty 数据量已经大大减少，从而加快了进入低功耗状态的速度。
3. 避免Dirty状态的来回传递让系统管理变得复杂。如果 Dirty 数据一直在核心之间通过 ReadShared 传递（Dirty Tick-tack），那么总有一个核心必须承担“最后将数据写回内存”的责任。**ReadClean** 提供了一个明确的契机，在读取的同时解除这种“更新内存”的责任负担，保持系统状态的“整洁”。

##### ReadNotSharedDirty

ReadNotSharedDirty的语义是：拒绝在共享状态下承担 Dirty 责任。也就是返回的数据可以是UC, UD或SC，但绝对不可以是SD.

##### ReadShared

ReadShared是最基础，最通用的读取事务。ReadShared允许请求者接收任何合法的共享状态（包括UC、UD、SC和SD）

##### ReadUnique

仅允许返回的数据是UC或UD状态，保证自身独占性。

##### ReadPreferUnique

ReadPreferUnique是相对比较温和的ReadUnique。如果另一个核心正在进行独占操作，读到的数据会是shared状态而不是unique状态。你可以把 **ReadPreferUnique** 想象成一个**“有礼貌的预定”**： “老板，我想买这张桌子（Unique），如果现在没人订，直接给我；如果已经有人在付钱了，我就先站在旁边看看（Shared），不打扰人家结账。”

用法：

1. 比如某个RN需要进行Atomic操作，如果没有这个事务，处理器可能先用 `ReadShared` 读到数据（获得 Shared 状态），发现要修改时，必须再发一个 `CleanUnique` 事务来申请独占权。这需要**两次**往返总线。有了 ReadPreferUnique，相当于直接告诉总线：“我一会儿大概率要改，如果方便的话，直接给我 **Unique** 状态。” 如果成功拿到 Unique，后续的写操作就是“本地操作”，无需再次申请权限，极大地提升了效率。
2. 在多核争抢同一个锁（Exclusive Access）时，如果所有核都强行要求 `ReadUnique`，它会强制让其他所有核心的缓存行失效（Invalidate）。后果是：如果 A 刚拿到 Unique 还没来得及写，B 的 `ReadUnique` 就把 A 给失效了；接着 C 又把 B 失效了。这会导致“缓存颠簸”（Cache Thrashing），谁也无法完成任务。ReadPreferUnique 的聪明之处在于它是一种**温和的请求**。如果 HN-F 发现现在正好有另一个核心正在进行独占操作（Exclusive sequence），它会降级只给你 **Shared** 状态。这允许请求者至少能先“读”到值（比如在自旋锁中轮询状态），而不会粗鲁地打断那个正在执行关键写入操作的核心。

##### MakeReadUnique

MakeReadUnique事务的语义是：当请求者已经拥有数据，但缺乏“修改权限”时，以最小的代价获取独占权（Unique 权限）。

用法：

1. 与ReadUnique相比，优化了带宽，只传权力，不传数据。**ReadUnique:** 无论请求者有没有数据，都会触发Data Response。MakeReadUnique如果互连网络（HN-F）确认请求者已经拥有最新的数据副本，它可以**只返回权限确认**（Completion），而不传输数据。在写密集型应用中，这极大地节省了总线数据带宽，降低了功耗。

注意事项：

1. 在复杂的总线环境中，可能会发生这种情况：**Agent A** 发出 `MakeReadUnique` 想要升级权限。**与此同时，Agent B** 发出了一个 `ReadUnique`，导致总线向 Agent A 发送了一个 **Snoop Invalidate**（要求 A 删掉自己的数据）。如果按照简单的逻辑，Agent A 的数据没了，它的权限升级请求似乎应该失败并重试。 但 CHI 的设计是， 如果 Agent A 在等待 `MakeReadUnique` 结果时，数据被别人的 Snoop 给“抢”走了。总线（HN-F）**必须保证**在最后的响应中，重新把最新的数据再发还给 Agent A。即：Agent A 永远不需要因为“数据中途被抢”而重新发送请求。这保证了**前向进度（Forward Progress）**，避免了在高竞争下的死锁或无限重试。

### 不带数据的事务（Dataless Transactions）

#### 一句话搞清楚CHI不同Dataless事务的语义

##### 1. Stash 系列（提前送货上门）

- **StashOnceUnique**：管家主动把包裹塞进你的储物柜，并顺便把开锁的独占钥匙交给你，方便你待会直接修改。
- **StashOnceSepUnique**：管家先告诉你“包裹马上就位”，让你先去忙，他随后再异步把包裹和独占钥匙送进你的柜子。
- **StashOnceShared**：管家把包裹放进你的柜子，但只给你一份复印件（读权限），因为他知道你只是想看看，不想负责修改。
- **StashOnceSepShared**：管家先发个短信确认收到了送货请求，然后后台慢慢把这份“只读”的包裹副本送达你的柜子。

##### 2. Clean 系列（大扫除与同步）

- **CleanShared**：管家要求你把手里的脏包裹复印一份存入总仓库（PoC），但允许你继续留着这份包裹慢慢看。
- **CleanSharedPersist**：管家不仅要求你把包裹复印件交回仓库，还要亲眼盯着仓库管理员把它锁进永不掉电的保险柜（PoP）。
- **CleanSharedPersistSep**：管家先确认数据已交回仓库，让你先恢复工作，过一会再通知你保险柜（PoP）已经彻底锁好。
- **CleanInvalid**：管家要求你把手里修改过的包裹交回总仓库，然后必须当面把你自己柜子里的那份烧毁（Invalid）。
- **CleanInvalidPoPA**：管家执行跨空间大扫除，把包裹彻底刷过物理别名点（PoPA），确保在其他平行物理空间也能看到这件货。
- **CleanInvalidStorage**：最彻底的清空，管家要求把包裹从所有缓存销毁，并确认数据已经物理写入了最底层的闪存颗粒（PoPS）。

##### 3. 权限与状态切换（所有权流转）

- **MakeInvalid**：管家冷酷地通知你：“不管你手里的包裹是不是新的，立刻把它扔了，我不需要你交回仓库。”
- **CleanUnique**：你告诉管家：“我手里有这件货，但我现在想改它，请帮我收回别人手里的钥匙，让我一个人独占。”
- **MakeUnique**：你告诉管家：“我要彻底重画这幅画，把别人的副本都烧了，我也不需要旧的数据，直接给我独占写权限。”
- **Evict**：你主动告诉管家：“我的柜子满了，这件货我也不打算用了，权限我还给你，但这货没改过，我就不麻烦你存回仓库了。”



#### 一句话搞清楚何时选用上述何种CHI Dataless事务

##### 1. 获取写权限（我要修改数据）

- **CleanUnique**：当你只有读权限（Shared）但想**修改部分字节**时，用它来拿独占权并同步最新数据。
- **MakeUnique**：当你准备**覆盖整行数据**时，用它来最快获取独占权（因为它不产生数据传输）。

##### 2. 缓存清理（我想同步或腾位置）

- **CleanShared**：想把脏数据**推回内存**供他人查看，但自己还想留一份继续读时选用。
- **CleanInvalid**：想同步脏数据到内存，且同步完后**彻底放弃**该缓存行时选用。
- **MakeInvalid**：本地缓存行已无用，想**直接丢弃**且不需要写回内存（哪怕它是脏的）时选用。
- **Evict**：本地缓存行是干净的（Clean），仅仅因为**空间不足**想通知家乡节点（HN）你要放弃权限时选用。

##### 3. 持久化与存储（我要确保数据不丢）

- **CleanSharedPersist**：需要确保数据**掉电不丢失**，但之后仍需高频读取该数据时选用。
- **CleanInvalidPoPA**：在**机密计算**切换内存归属空间（PAS）前，确保旧空间的残余数据彻底物理清除时选用。
- **CleanInvalidStorage**：必须确认数据已**物理落盘**（如 SSD 颗粒）以满足极高可靠性要求时选用。

##### 4. 缓存推送（我把数据喂给别人）

- **StashOnceUnique**：想把数据连带**写权限**一起“推”给特定的 CPU 核心以减少其写延迟时选用。
- **StashOnceShared**：只想把数据“推”给目标 CPU **读**（如处理报文头），且不想让目标 CPU 负责写回时选用。



#### CHI协议中对Dataless事务的详细阐述

这类事务的特点是，数据（Data）不被包含在response中。这类数据通常用来执行一些coherence操作。

这类事务可以分成几类，

- 一类是StashOnceUnique, StashOnceSepUnique, StashOnceShared, StashOnceSepShared这类独立的Stash操作，用于优化性能；
- 一类是单纯缓存一致性的维护操作（CMO, cache maintenance transactions）, 比如CleanShared, CleanSharedPersist, CleanSharedPersistSep, CleanInvalid, CleanInvalidPoPA, CleanInvalidStorage, MakeInvalid；注意在这类操作中，RN不得理会Response中的cache状态信息
- 一类是CMO操作加上独占性操作，比如CleanUnique，MakeUnique；
- 一类是缓存驱逐操作，单指Evict。

##### CleanUnique

一般用于Requester自身已有一份shared状态的缓存行，且希望获得该缓存行的独占权以完成后续对该缓存行的写。其他核如果有Dirty的该缓存行，需要写回主存。当Requester不能保证后续对该缓存行的写为全行写，可能为partial写时，使用该事务。

##### MakeUnique

与CleanUnique的区别是，其他核如果有Dirty的该缓存行，不需要写回主存。MakeUnique仅用于当Requester即将对该缓存行的**全部**字节执行写操作。

##### Evict

用来驱逐处于clean状态的缓存行。Evict用于某缓存行不再被RN需要的时候。

##### StashOnceUnique, StashOnceSepUnique

StashOnceUnique的语义是请求者发送一个 Stash 请求到 HN（Home Node），指示 HN 将特定的缓存行推送到某个目标 CPU（Target），并且要求该目标 CPU 最终处于 **Unique** 状态（即拥有写权限）。StashOnceUnique是一个单一事务流，HN在收到请求之后，会通过snoop机制通知目标CPU去获取数据。

StashOnceSepUnique的语义和StashOnceUnique一直，但其中的Sep代表将推送动作和完成确认两件事情解耦。HN 会先给请求者返回一个 `StashDone` 响应，表示请求已被受理，而数据的实际搬运过程（Data Pull）则在后台异步进行。

使用场景：

1. 由一个非 CPU 节点（如网卡 NIC 或加速器）主动发起请求，将数据及其“写权限”提前搬运到某个特定的 CPU 缓存中。在传统的系统中，如果 IO 设备写数据到内存，CPU 稍后去读，会经历：`内存写回 -> CPU Cache Miss -> 内存读取` 的漫长路径。**StashOnceUnique** 的用意在于，在 CPU 真正需要数据前，就把数据从 IO 或内存推送到 CPU 的 L1/L2。带有 **"Unique"** 后缀意味着不仅推送数据，还要求目标 CPU 获得 **Unique (Writable)** 状态。这样 CPU 拿到数据后可以直接修改（例如修改报文头），无需再发起 `CleanUnique` 事务来获取写权限。

注意事项：

1. StashOnceSepUnique相比StashOnceUnique，其释放资源的速度更快，但所需的硬件支持也更为复杂。

2. DataPull 并不是一个独立的消息类型，而是一个**动作序列**。 当请求者（如加速器）发送 `StashOnceUnique` 给 HN 时，HN 并不会直接强行把数据塞给目标 CPU（这会把 CPU 的内部流水线搞乱），而是通过 **SnpStashUnique** 信号“敲敲门”，问 CPU：“你要不要这行数据？”如果目标 CPU 觉得现在缓存有空位，它就会发回一个 **DataPull 请求**。
   1. 标准的DataPull交互流程是这样的：请求节点（RN-I）发送 `StashOnceUnique` 给 HN。HN 发现是 Stash 请求，向目标 CPU（RN-F）发送 **`SnpStashUnique`**。目标 CPU 接收到该 Snoop。如果它愿意接收这行数据，它会向 HN 发送一个带有特殊标记的 **`ReadUnique`**（这就是所谓的 DataPull）。HN 将数据（通过 `CompData`）发送给目标 CPU。目标 CPU 现在拥有了该行的独占权（Unique Clean/Dirty），可以直接进行写操作。
   2. DataPull机制的好处在于避免“强推”导致的问题。如果数据被强行推入 CPU 缓存，而 CPU 此时正在忙于处理其他高优先级任务，或者缓存已经满了，强推会导致缓存污染或流水线阻塞。**DataPull 让 CPU 拥有拒绝权**：如果 CPU 缓存太忙，它可以选择不发起 DataPull。
   3. 即：DataPull = Snoop 诱导 + CPU 主动拉取。
3. 这里的Once可以理解为不建立“粘性” (Non-Sticky)。在总线协议设计中，Stash 操作有两种潜在的逻辑：1. **非 Once (持久型/预取型)：** 告诉缓存，“这一块数据很重要，请把它加载进来，并且尽可能长时间地保留它，哪怕最近没用到”。2. **Once (一次性推送)：** 告诉缓存，“我把这行数据推给你，**仅供你下一次操作使用**。用完之后，这行数据的替换权重（Replacement Policy）和普通数据一样”。这样可以防止**缓存污染**。如果 IO 设备源源不断地向 CPU 缓存推送数据，而这些数据被标记为“长期保留”，那么 CPU 自己的局部性数据（Local Data）就会被挤出缓存。加上 "Once"，意味着这只是一个**临时的性能暗示**。

##### StashOnceShared, StashOnceSepShared

前面两个事务（StashOnceUnique, StashOnceSepUnique）会把DataPull当成ReadUnique事务看待，而这两个事务（StashOnceShared和StashOnceSepShared）会把DataPull当成ReadNotSharedDirty事务看待。

场景：一般用于使某CPU提前拥有对某缓存行的读权限。这里把DataPull当成ReadNotSharedDirty事务主要是为了避免复杂的SharedDirty状态机。

##### CleanShared

用来让其他的缓存行副本的状态都变成Non-dirty的（Clean或Invalid），dirty的副本必须写回memory。

##### CleanSharedPersist

用来让其他的缓存行副本的状态都变成Non-dirty的（Clean或Invalid），比CleanShared事务更进一步，CleanSharedPersist事务要求dirty的副本必须写回所谓Point of Persistence (PoP)（**Point of Persistence (PoP)** 指的是系统中一个特定的位置，一旦数据到达这里，即便系统**掉电（Power Failure）**，数据也不会丢失。其通常指 **非易失性存储器（NVM）**，如持久内存（Persistent Memory, PMEM）或 Flash。）

##### CleanSharedPersistSep

大体与CleanSharedPersist相同，但是Response可以分两步（Requester也需要支持一步完成的）

##### CleanInvalid

用来让其他的缓存行副本的状态都变成Invalid的，dirty的副本必须写回memory.

##### CleanInvalidPoPA

这个事务比较冷门，主要出现在支持 **Arm 机密计算架构 (Arm CCA)** 或多物理地址空间（Multi-PAS）的系统中。在支持机密计算（如 Arm Realm Management Extension）的系统中，同一个物理内存可能会被映射到不同的 **物理地址空间 (Physical Address Space, PAS)**。例如：**Secure PAS** (安全空间)、**Non-secure PAS** (非安全空间)、**Realm PAS** (机密空间)、**Root PAS** (根空间)。**PoPA**  (Point of Physical Aliasing)是系统中的一个物理位置（通常在内存控制器的前级），在该位置之后，系统不再区分请求来自哪个 PAS，而是直接映射到真正的物理存储介质。也就是它是解决“地址别名（Aliasing）”导致的一致性问题的终点。

CleanInvalidPoPA在将指定地址范围内的所有“脏数据（Dirty）”从 Cache 中推出去且将所有 Cache 副本设为无效（Invalid）的同时规定，这种CMO操作的深度必须达到 **Point of Physical Aliasing** 之后。它确保了当数据经过 PoPA 点后，你在 **PAS A** 写入的数据，能够被 **PAS B** 正确地看到。

##### CleanInvalidStorage

用来让其他的缓存行副本的状态都变成Invalid的，dirty的副本必须写回PoPS (Point of Physical Storage). 

- **PoPS (Point of Physical Storage)指 **物理存储点**，这是最深的一层。它保证数据不仅进入了持久化层（PoP），而且已经写入了**最终的物理介质（如 NAND Flash 的存储单元或磁盘盘片）
- PoPS 确保了“绝对的安全”。在某些严格的合规性场景或文件系统操作中，仅到达 PoP（可能还在闪存控制器的电容保护缓存里）是不够的，必须到达 PoPS。

##### MakeInvalid

用来让其他的缓存行副本的状态都变成Invalid的，dirty的副本允许直接丢弃。



### 写事务（Write Transactions）

CHI中的写事务可以粗略分为覆盖写（Immediate），腾位置（CopyBack），预存（Stash），非一致性写（WriteNoSnp）

#### 一句话搞清楚CHI中不同Write事务的语义

##### 一、 Immediate 类 (直写类：人狠话不多，直接推数据)

- **WriteNoSnp (Full/Ptl):** “发往非一致性区域的普通快递，不查附近邻居，直接送到目的地。”
- **WriteNoSnpDef:** “可以晚点再发的普通快递，允许我一次寄好几件且不用按顺序等回执。”
- **WriteNoSnpZero:** “给目的地发个电报，告诉他：‘把这块地儿全填成零’，但我不用寄实物。”
- **WriteUniqueFull:** “霸道总裁式写回：我手里有全套新货，你们邻居手里的旧货全给我作废，直接存入总库。”
- **WriteUniquePtl:** “精准修改：我只改其中几个零件，但你们邻居手里的整套旧货也得全部作废。”
- **WriteUniqueZero:** “一致性清零：不用寄数据，但要通知全系统把这行旧数据作废，并清空为零。”
- **WriteUniqueFullStash:** “写回并定向安利：我写数据的同时，顺便让管家把这行新数据塞到某个特定邻居的包里。”

------

##### 二、 CopyBack 类 (放回类：把原本就在我这的东西还回去)

- **WriteBackFull:** “我不留了：把我在本地改过的这行脏数据，完整地还给下一级并清空我自己的库存。”
- **WriteBackPtl:** “碎块归还：我本地只有一部分是脏数据，只把这几个改过的碎块还回去，然后我不留了。”
- **WriteCleanFull:** “备份同步：我本地继续留着这行，但同步一份最新版给总库，让我这行的状态变‘干净’。”
- **WriteEvictFull:** “权限转交：这行我没改过，但我原本是独占的，现在我不要了，把这份‘独占特权’连同数据一起转交给下一级。”
- **WriteEvictOrEvict:** “商量着办：我要丢弃这行独占数据，问问管家你要不要存？你要我就寄（WriteEvict），你不要我就直接扔（Evict）。”

#### 一句话搞清楚CHI中不同写事务的使用场景

- **WriteUniqueFull**：**【内存初始化/DMA】** 我手里有一整行新数据要“强行占领”这个地址，所有人手里的旧货立刻扔掉。
- **WriteUniquePtl**：**【非对齐写/碎片修改】** 我只想改一行里的某几个字节，但也要霸道地让别人手里的整行失效。
- **WriteUniqueFullStash**：**【生产者-消费者模式】** 我刚算出一行结果，存入内存的同时，顺便“推”到 CPU 缓存里让它赶紧处理。
- **WriteBackFull**：**【Cache 替换】** 缓存满了，我得把这一行改过的（Dirty）数据还给内存，腾地方给别人。
- **WriteCleanFull**：**【内存快照】** 数据我改好了，先发一份给内存备份（防止丢数据），但我自己还要留着继续读写。
- **WriteEvictFull**：**【特权转移】** 我没改数据，但我之前是“独占”的，现在我不需要了，把这个“独占干净”的状态传给下一级。
- **WriteEvictOrEvict**：**【带宽优化】** 我想扔掉一行没改过的数据，问问下一级缓存：“你要不要存？你要我才发数据，不要我就直接删了”。
- **WriteNoSnp (Full/Ptl)**：**【非一致性/IO设备】** 发给显存或不用查缓存的设备，直接暴力存入，不跟其他 CPU 核心打招呼。

#### CHI协议中对不同写事务的详细阐述

CHI中的写事务可以分为两类：一类是所谓Immediate write transaction，流向可以是RN到HN，也可以是HN到SN。特点是在写之前RN不需要取得该缓存行的独占权（因为RN写完之后不想存该缓存行），而是把维护一致性的锅推给HN完成。另一类是CopyBack write transaction，这类的数据流向是去往下一级cache或者memory，同样也不需要对其他RN的snoop。

##### WriteNoSnpFull

从RN向Non-snoopable的地址区间写一整个（full）缓存行，或者从HN向SN写一整个缓存行。所有Byte Enable (BE)位必须为高

##### WriteNoSnpPtl

和WriteNoSnpFull相比，BE位可以全部为0，可以部分为0（写缓存行的一部分），也可以全部为1（写缓存行的全部），很灵活

##### WriteNoSnpDef

和WriteNoSnpFull,相同的是从RN向Non-snoopable的地址区间写一整个（full）缓存行，或者从HN向SN写一整个缓存行。所有Byte Enable (BE)位必须为高。

不同的是，这些写是可推迟的（Deferrable），即只要数据被传送到路径中的一个**中间节点**（例如 Home Node 或缓存控制器），并且该节点保证之后一定会负责把数据写完，就可以立即返回确认信号。（因为这种写入不会触发其他核心缓存的查询（Snoop），因此系统可以放心地在中间环节拦截并确认，而不必同步等待全局一致性检查。）这种机制主要是为了降低延迟和提高带宽效率。

注意事项：

- 允许同一个RN发出多个Outstanding的Deferrable Write事务

##### WriteNoSnpZero

与WriteNoSnpFull相比，WriteNoSnpZero指示目标核向目标地址缓存行写全0. 因为写的数据已经确定（全0），后续RN或者HN不会再传输写数据。

##### WriteUniqueFull

向某块snoopable的地址区间发起全行写（full）。写前和写后，发起者的该缓存行状态都是Invalid。写数据进入memory或者SLC中。对于发起者RN来说，可以不理会一致性的维护问题，全权交给HN来解决。

##### WriteUniquePtl

与WriteUniqueFull相同，其他RN的该缓存行副本都会被invalidate；与WriteUniqueFull的唯一不同是，可以不是全行写（某些BE位可以不为1）。为了保证缓存行数据的完整性，写的部分缓存行数据直接从RN发向HN，在HN端完成合并。

用法：某些只写不读的小流量数据（如日志记录，状态标志更新），可以显著降低其延迟；某些计算单元产生碎片化结果，直接丢给下游 SLC 处理，而不需要在本地保留副本。

注意事项：WriteUniquePtl只是把合并这个脏活丢给了HN。

##### WriteUniqueZero

向某块snoopable的地址区间发起全0的全行写。同样的，后续写数据不会被传输。

##### WriteUniqueFullStash

发起者指示HN向某个RN发起stash的snp hint，如果该RN决定接受这个hint，就会发起datapull，把建议的对应地址的缓存行全行拉进自己对应的cache位置。

##### WriteUniquePtlStash

与WriteUniqueFullStash的唯一不同是，datapull的可以是缓存行的一部分。

##### WriteBackFull

把一整行Dirty状态的缓存行推给下一级缓存或者memory

##### WriteBackPtl

把Dirty状态的缓存行的一部分推给下一级缓存或者memory

##### WriteCleanFull

把一整行Dirty状态的缓存行推给下一级缓存或者memory的同时，保留一个clean的副本在本层级cache中

##### WriteEvictFull

把一整行UniqueClean状态的缓存行推给下一级缓存，当前层级cache中不保留副本。

一般用于RN还想要保持对某个缓存行的独占权，但是当前层级cache放不下了，就把缓存行驱逐到下一个层级。

##### WriteEvictOrEvict

语义是：我这里有一行 UniqueClean（独占且干净）的数据要丢弃，我想顺便把数据传给下一级缓存（SLC），但如果你（HN）不想要，我就直接把它删了

让HN根据自身情况自行决定。

流程如下：RN 发送 `WriteEvictOrEvict` 请求给 HN。HN 根据自己当前的负载、SLC 的剩余空间等情况做决定：如果HN想要数据，就回复 `CompDBID`（包含 Data Buffer ID），告诉 RN：“把数据发给我吧”。如果HN不需要数据，就仅回复 `Comp`。RN 收到后直接在本地把该行设为 Invalid，不需要发送数据。

RN可以通过LikelyShared字段对HN进行提示。如果 RN 认为这行数据以后可能被多个核心共享，它会在字段中标记。HN 看到这个提示后，更有可能决定“接收数据”并把它留在 SLC 中，以方便后续的读取请求。