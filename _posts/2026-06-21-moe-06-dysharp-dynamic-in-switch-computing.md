---
layout:     post
title:      DySHARP Dynamic In-Switch Computing
subtitle:   DySHARP Dynamic In-Switch Computing
date:       2026-06-21
author:     George Lin
header-img: img/post-bg-gen.png
catalog: true
tags:
- moe
---

## 引言：两台交换机的距离

想象一个场景：你坐在 GPU 0 上，手里握着一个 token。路由网络告诉你，这个 token 需要专家 2 和专家 3 来处理。专家 2 在 GPU 2 上，专家 3 在 GPU 3 上。在标准的 MoE dispatch 流程中，你要把这个 token 的 hidden state 原封不动地**发送两次**——一次从 GPU 0 到 GPU 2，一次从 GPU 0 到 GPU 3。两次传输的是完全相同的数据，走的是同一根 NVLink 出向链路（GPU 0 到 NVSwitch），被交换机分发给两个不同的目标 GPU。

这还没完。Combine 阶段是相反的对称浪费：GPU 2 和 GPU 3 各自把专家输出传回 GPU 0，而这两个输出需要在 GPU 0 上做加权求和。换言之，**两份可以提前在交换机里加在一起的数据，却各自占了一条链路，跑到 GPU 0 的 HBM 里才做加法**。

DySHARP 论文的作者在模拟 GH200 NVL32 上跑 DeepSeek-V3 时测出了一个令人不安的数字：**近 50% 的 MoE 总通信流量是冗余的**。如果把每字节都换成钱，相当于训练一个 MoE 大模型时，一半的通信账单是你"多付的"——重复的多播和可以归约但未归约的分段传输。

交换机网内计算（in-switch computing）正是针对这类冗余提出的解决方案。交换机在转发数据的同时做计算：遇到同一个 token 要发给多个 GPU，交换机帮你复制；遇到来自多个 GPU 的归约输出，交换机帮你加起来。这不是新概念——NVIDIA 自从 Hopper 代就在 NVSwitch 里集成了 NVLink SHARP (NVLS)，支持 in-switch multicast 和 in-switch reduction。AllReduce 和 AllGather 这些**静态集体操作**已经享受网内计算的好处有三代产品了。

但 MoE 的通信不是静态的。

所有 token 每轮迭代选择不同的专家子集。所有 token 在每个 GPU 上的内存位置不一样。目标集动态变化，内存地址不对称——这和 NVLS 的设计前提完全冲突。强行套用 NVLS 去加速 MoE，方法是把动态的 dispatch/combine 重新解释为静态的 AllGather/Reduce-Scatter——这相当于"让所有 GPU 向所有 GPU 发送所有数据"，产生了 **340% 的无用流量**，把网内计算省下来的带宽全部还了回去。

DySHARP 要解决的问题就是：**如何让交换机网内计算支持 MoE 的动态、不规则的通信模式？**这不是"换个配置"就能解决的。它需要对包格式、指令集、GPU 微架构、交换机微架构、CUDA 运行时做全栈改造。更关键的是，即使搞定了通信原语，还有一个方向性带宽不对称的坑在前面等着——消除了 dispatch 方向的冗余，GPU→交换机流量骤降；消除了 combine 方向的冗余，交换机→GPU 流量骤降；但如果 dispatch 和 combine 各跑各的，未被优化的方向仍然拖后腿，总体加速为零。

本文是网内计算系列的第二篇。上一篇我们回顾了 SHARP 和 NVLS 的架构演进，建立了网内计算的"理论基础"。这一篇我们进入工程落地——看 DySHARP 如何把动态网内计算从"理论有可能"变成"实测 1.79 倍加速"。

---

## 系统剖析：为什么 MoE 通信有一半是冗余？

### 冗余从哪里来？

要理解 DySHARP 的设计，必须先精确理解冗余的物理根源。

MoE 的专家并行（Expert Parallelism, EP）引入两个通信操作：

- **Dispatch（分发）**：每个 token 从其"家"所在的 GPU（源 GPU）发往持有被激活专家的 GPU（目标 GPU）。如果 1 个 token 激活 top-k 个专家，它就产生 k 次源→目标的发送。当 k 个专家分布在不同的 GPU 上时，**同一个 token 的数据要从源 GPU 的 NVLink 出向链路发出 k 次**。
- **Combine（汇聚）**：每个专家计算完毕后，输出需要从专家所在 GPU 传回 token 的源 GPU 做加权求和。如果 k 个专家分布在 k 个不同的 GPU 上，**k 份可以提前求和的数据从交换机流向了同一个目标 GPU**。

用论文 Figure 1(b) 的具象化例子：Token B 在 GPU 0 上，激活了 GPU 2 和 GPU 3 上的两个专家。Dispatch 阶段，GPU 0 必须把 Token B 发给 GPU 2 和 GPU 3——这是**两次**从 GPU 0 到交换机的传输，传输的是同样的数据。Combine 阶段，GPU 2 和 GPU 3 的计算结果都要传回 GPU 0——这是**两次**从交换机到 GPU 0 的独立传输，两份结果本来可以直接在交换机里加好再送回来。

这种冗余和 top-k 直接相关。论文的 Figure 2(a) 量化了 DeepSeek-V3 在不同数量的激活专家下的冗余比例：**当 top-k >= 8 时，冗余数据接近总流量的 50%**。DeepSeek-V3 默认就是 top-8。

这个数字不是巧合。当每个 token 激活 k 个专家时，dispatch 阶段产生 k 倍于最小必要流量的数据传输，combine 阶段同样产生 k 倍。理论上，通过交换机多播（dispatch 方向）和交换机归约（combine 方向），这些冗余可以完全消除。Figure 2(b) 计算了消除冗余后的理想通信加速——**接近 2 倍**。

### NVLS 为什么不行？

NVLS (NVLink SHARP) 已经在 NVSwitch 里实现了 in-switch multicast 和 in-switch reduction。它的核心机制是**multimem 寻址**（multimem addressing）：

- Packet 只携带**一个地址**——multimem 地址。
- 交换机根据**预配置的目标集**把这个地址解析为多个目标 GPU 的多个内存位置。
- 因为目标集是预配置的（在 AllGather 场景中通常是"所有参与的 GPU"），一个包进来，交换机知道该发给谁。

这个机制能工作的前提是两条规则：

1. **固定目标集（Fixed target sets）**：一个通信操作中的所有 token 始终与同一组 GPU 通信。AllGather 里当然是——所有 GPU 都要收数据。
2. **对称寻址（Symmetric addressing）**：token 在各 GPU 上的内存偏移是相同的。AllGather 把同一份数据写到所有 GPU 的相同偏移位置。

MoE 的 dispatch/combine **同时违反这两条**：

- **变化目标集（Varying targets）**：Token A 被路由到 GPU {0, 1, 2}，Token B 被路由到 GPU {2, 3}。每个 token 的目标 GPU 子集不同。
- **非对称寻址（Asymmetric addressing）**：Token 在每个 GPU 上的内存位置是独立动态分配的。GPU 0 上 Token A 可能在偏移 0，GPU 1 上在偏移 2，GPU 2 上在偏移 1。

预配置目标集无法应对变化的目标。单一地址无法描述非对称的多目标内存位置。

### 340% 无用流量的代价

既然 NVLS 只支持静态 collectives，一个"只需改软件"的 workaround 是把 MoE 通信重新解释为静态 collectives：

- Dispatch → AllGather：让所有 GPU 向所有 GPU 发送所有 token。
- Combine → Reduce-Scatter：让所有 GPU 参与全局归约。

但这意味着**每个 GPU 都要发送和接收它实际不需要的数据**。如果 GPU 0 只需要给 GPU 2 和 3 发数据，AllGather 却让它给所有 32 个 GPU 都发一份。论文在 DeepSeek-V3 + GH200 NVL32 配置上实测：**这种转换产生的无用流量达到原始有用流量的 340%**。这比原始的冗余还要糟糕——本来 dispatch/combine 只有 ~50% 冗余，现在因为所有无关 GPU 都被迫参与进来，总流量反而比不做网内计算还要大。

换句话说：**没有动态支持，NVLS 对 MoE 不仅没有加速，反而在帮倒忙。**

### 通信时间占比：70.4% 的瓶颈

冗余不仅涉及流量量，更关系到时间。论文在模拟 GH200 NVL32 上运行 DeepSeek-V3 时测量到：**通信消耗了 MoE 单层执行时间的 70.4%**。这个数字与 ByteDance 在 COMET 中的发现（50-80%）一致，但更具体——它说明在 NVL32 这种全互联 NVSwitch 系统中，即便链路带宽密度已经非常惊人（900 GB/s per NVLink），通信仍然是 MoE 层的主导开销。

更值得警惕的是趋势线。NVIDIA Blackwell (GB200) 和 Rubin 架构的计算能力增速远超通信带宽增速——B200 的 FP8 算力相比 H100 提升了 2.5 倍，但 NVLink 5.0 的单链路带宽只从 900 GB/s 提升到 1.8 TB/s（2 倍）。通信-计算比正在进一步恶化。这意味着如果不做体系结构层面的干预（比如动态网内计算），**未来 MoE 层的通信时间占比还会继续上升**。

### DySHARP 的解题思路

面对功能缺口（NVLS 不支持动态通信）和调度缺陷（计算-通信隔离导致方向性带宽不平衡），DySHARP 提出了一个两件套方案：

1. **动态 multimem 寻址（Dynamic Multimem Addressing）**——通信原语层：扩展 NVLS 的 multimem 寻址框架，使其支持动态变化的目标集和非对称寻址，从而消除 ~50% 的冗余流量。
2. **Token-Centric Kernel Fusion**——调度层：将 Dispatch、Computation、Combine 三个原本隔离的执行阶段融合为一个以 token 为粒度的流水线，消除方向性带宽不平衡，把流量减少转化为实际加速。

两个组件不是可选的加法——它们是**同一个解的两个半部分**。没有动态 multimem 寻址，kernel fusion 没有冗余可消除；没有 kernel fusion，动态 multimem 寻址消除的冗余因为方向性不对称而无法转化为加速。论文在 Figure 4 中以六张时间线图清晰地证明了这一点。

---

## 关键技术决策

### 决策一：动态 multimem 寻址 vs. 显式寻址——为什么不在包里写满地址？

给交换机提供动态多目标信息，最直接的方案是**显式寻址（explicit addressing）**：在请求包里把所有目标 GPU 的完整目标地址都写进去。包到了交换机，交换机读出头里的 N 个地址，各自转发。

DySHARP 的设计者没有选这条路。他们给出了两个理由：

**1. 有效载荷效率低。** NVLink 4.0 的 flit 大小是 16B。如果目标为 8 个 GPU，每个目标地址需要 8B（64-bit GPU 物理地址），那么仅地址头就需要 8 个 flit——请求包和响应包各带一份。在典型的数据传输粒度（比如一个 token 128B=8 flits）下，有效载荷效率从接近 80% 掉到 69%。这就是在传输的数据里掺了 ~11% 的地址税。

**2. 软件开销大。** 显式寻址下，发送方 GPU 必须知道每个目标 GPU 上对应的内存地址。这意味着发送方要自己跟踪"这个专家在这个 GPU 上已经收了多少 token"——等同于每个源 GPU 维护一组远程 token 计数器，并且每次 dispatch 前要计算出准确的远程写入地址。这类软件管理已被证明消耗 10-20% 的 GPU 计算资源（DeepSeek-V3 报告中提到的），并且需要跨 GPU 同步（额外 >5% 性能损失，参考 DeepEP）。

**DySHARP 的替代方案**：扩展 multimem 寻址范式——包只携带一个"代数索引"（algebraic index）和一个轻量级的"目标专家列表"（target expert list）。目的地 GPU 自己在本地完成"代数索引到本地布局索引"的映射。这个选择的核心洞察在于：

Dispatch 和 Combine 本质上是 AllGather 和 Reduce-Scatter 的**动态变体**。AllGather 中每个 token 广播到所有 GPU，数据在所有 GPU 上占据相同的代数索引。Dispatch 中每个 token 只广播到选中的 GPU 子集——**代数索引仍然跨 GPU 相同**，只是每个 GPU 上分配的具体内存位置（布局索引）因为"只有部分 GPU 收到该 token"而产生碎片化，需要各自压缩存储。

因此，包格式可以这样设计：
- **Flit 0**：48-bit multimem 地址（代数索引）+ 1-bit 阶段标记（Dispatch/Combine）+ 15-bit 目标计数
- **目标扩展 Flit**：每个目标专家 ID 占 16-bit，一个 flit 可装 8 个目标
- **后续 Flit**：标准的 byte-enable 和 payload，完全不变

相比显式寻址，这种格式在 8 目标场景下只需要 flit0 + 1 个目标扩展 flit = 2 flit 的包头 vs. 显式寻址的 8+ flit。

### 决策二：全栈改造——在哪四层动手？

动态 multimem 寻址不是一个"改个包头格式就行"的参数调整。DySHARP 对 **packet 格式、ISA、GPU 微架构、交换机微架构、CUDA 运行时**做了五层联动的修改。我们来逐层看：

**层 1：Packet 格式扩展（数据链路层）**

基于上面讨论的格式设计，DySHARP 在 NVLink 的数据链路层包格式上做了最小侵入的修改：在 flit0 中把原来的 64-bit 地址域拆成 multimem 地址 + 阶段标记 + 目标计数。目标扩展 flit 是新增的，但符合已有的 flit 结构。论文强调这种修改"建立于现有数据路径之上，不改变任何原有功能"。

**层 2：ISA 扩展（指令集层）**

基于 NVLS 的 `multimem.st`（用于 AllGather 的 in-switch multicast 写）和 `multimem.ld_reduce`（用于 Reduce-Scatter 的 in-switch reduction 读），DySHARP 定义了 `dymultimem.st` 和 `dymultimem.ld_reduce`：

- `dymultimem.st`：r1=数据操作数, r2=multimem 地址, r3=目标数量, r4=目标列表基地址
- `dymultimem.ld_reduce`：r1=接收归约结果的寄存器, r2=multimem 地址, r3=目标数量, r4=目标列表基地址

两个新指令各多了两个寄存器（r3, r4）来指定目标列表。目标列表通常预加载到共享内存（shared memory）中以减少取指开销。

注意 `dymultimem.ld_reduce` 有意**不支持加权归约**。加权归约会在交换机硬件里引入乘法器，复杂度太高。DySHARP 的策略是把加权放在 **GEMM-2 的 epilogue** 里完成：每个专家计算输出 o_i 后，立即在 epilogue 中乘以门控权重 w_i，然后传出去就是 w_i·o_i。交换机做的是无权重加法 Σ(w_i·o_i)，结果直接等于加权和。这是一个典型的"计算便宜则在计算处做，通信复杂则简化通信处"的 trade-off。

**层 3：源 GPU 微架构扩展（SM LSU）**

在 SM 的 Load-Store Unit (LSU) 中新增一个独立的 **MultimemQ**（与原有 LSQ 分离）。当 LSU 遇到 `dymultimem.*` 指令时：
1. 先从 shared/global memory 中取目标列表（利用 r3, r4 指定的计数和基地址）。
2. 目标列表就绪后，组装完整的 dymultimem 请求包（multimem 地址 + 目标列表 + 数据）。
3. 通过片上网络发出，再经片间网络进入交换机。

这个 MultimemQ 很小——论文在评估中说总共只有 32 个 entry。它是一个等待目标列表取回的暂存队列，不与常规 LSQ 争抢资源。

**层 4：交换机微架构扩展（Route & Reduction）**

在交换机的 Route 模块中，对 dymultimem 包做**目标感知转发**：
- 对每个目标 expert_id，计算输出端口：`OutPort_i = Target_i / #experts_per_GPU`。
- 交换机按输出端口复制请求包，并在每个复制中只保留该端口的对应目标。

在 Reduction Logic（针对 `dymultimem.ld_reduce`）中：
- 记录每个归约请求关联的目标数量。
- 每当一个部分响应到达时，计数器递减。
- 计数器归零时意味着所有专家的输出已到齐，交换机把最终归约结果发回源 GPU。

这种修改被评估为"在现有 NVLS 数据路径上增加一个周期的控制逻辑"，面积开销不到 NVSwitch 芯片的 0.1%。

**层 5：目标 GPU 微架构扩展（Hub Memory Manager）**

目标 GPU 的 Hub 中新增一个**硬件内存管理器（Hardware Memory Manager）**，核心任务是把 multimem 地址（代数索引）翻译为虚拟地址（布局索引）。这是整个动态 multimem 寻址设计的精髓所在。

管理器的核心数据结构是 **AL Table**（Algebraic-Layout Table），存储在 GPU DRAM 中。每一对"代数块→布局块"的映射占 4 字节（1-bit Valid + 31-bit Layout Index）。以 1M token 为例，需要的表大小仅 4MB——与 H100 的 80GB HBM 相比可忽略不计。

加速 AL Table 查找的是一个 **AL TLB**（类 CPU TLB 的硬件缓存），使用 CAM（Content-Addressable Memory）做快速标签匹配。论文通过设计空间探索发现 512-entry 是最佳甜点——近理想的命中率且面积开销极小。因为 MoE 通信中同一 token vector 内的元素访问地址连续，具有较强的时间局部性。

Dispatch 阶段的 AL Table 工作流：
1. `dymultimem.st` 请求到达目标 GPU。
2. AL TLB 查找：命中则直接得到布局索引 → 地址翻译完成。
3. AL TLB 未命中：查 AL Table（DRAM）。
4. 如果是新的代数块（首次访问），管理器分配一个新的布局块（累积分配，确保碎片化的代数张量被压缩成稠密的布局张量），并登记映射关系。
5. 地址翻译完成 → 数据写入对应虚拟地址。

Combine 阶段复用已建立的 AL Table——因为专家计算不改变 token 顺序，Dispatch 和 Combine 可以共享同一套映射。

**层 6：CUDA 运行时扩展**

CUDA 层面新增了 `CUDymulticastObjectProp` 结构和对应的 API 调用（`cuDyMulticastCreate`, `cuDyMulticastBindAddr`），程序员需要指定：
- `bsize`：块大小（token vector 的粒度）
- `hsize`：共享同一套 AL 映射的 multimem 区域数量（对于 Dispatch+Combine 是 2）
- `ntoken`/`nactive[expert]`：multimem 和虚拟地址空间的大小

这些参数的确定依赖于路由网络在 Dispatch 前生成的 token 分配信息——而这恰恰是 MoE 计算图在每次迭代中都需要做的。

### 决策三：Token-Centric Kernel Fusion——为什么光有通信原语不够？

动态 multimem 寻址消除了冗余流量，但论文作者发现了一个反直觉的现象：**消除冗余并不自动等于加速。**

原因在于网内计算带来的流量减少是**方向性不对称的**：

- In-switch multicast（dispatch 方向）：消除了 GPU→交换机的重复发送。GPU→交换机方向的流量大幅下降。
- In-switch reduction（combine 方向）：消除了交换机→GPU 的重复接收。交换机→GPU 方向流量大幅下降。

当 Dispatch 和 Combine 隔离执行时：
- Dispatch 阶段：GPU→交换机快（多播省了），但交换机→GPU 方向仍然满负荷（很多 token 要去很多 GPU）→ **交换机→GPU 成为瓶颈**。
- Combine 阶段：交换机→GPU 快（归约省了），但 GPU→交换机方向仍然满负荷（很多 GPU 要上传输出）→ **GPU→交换机成为瓶颈**。

两个阶段的瓶颈方向互补，但如果你一个阶段一个阶段地跑，每个阶段的总时间由各自的瓶颈方向决定——**两个瓶颈不同时存在，但轮流拖慢整个流程**。

Token-centric kernel fusion 的核心思路是：**让 Dispatch 和 Combine 同时运行**。

但这不能大粒度地做——你不能等 Dispatch 全部完成再跑 Compute。Token-centric kernel fusion 把 MoE 层建模为一个以 token（或 token tile）为粒度的流水线：

```
TokenTile i: Dispatch → GEMM-1 → GEMM-2 → Combine
TokenTile i+1:                    Dispatch → GEMM-1 → GEMM-2 → Combine
```

关键基础设施是 **Token Tracker**——一个硬件/固件组件，跟踪三个依赖链路：

1. **Dispatch → GEMM-1**：当某个专家的一个 tile（128 token）全部到达后，对应的 GEMM-1 tile 即可发出。
2. **GEMM-1 → GEMM-2**：当 GEMM-1 的一整行 tile 计算完成后，GEMM-2 对应的 tile 行即可发出。
3. **GEMM-2 → Combine**：当一个 token 的所有 top-k 个专家的输出都已生成后，该 token 即可开始 Combine。

Tracker 使用三张表实现：

- **Tile Status (TS) Table**（1024 entry, on-chip）：跟踪每个 token tile 的 dispatch 到达进度、GEMM-1 完成进度、GEMM-2 完成进度。
- **Token ID (TID) Table**（DRAM）：记录每个 token tile 包含哪些 token ID。GEMM-2 tile 行完成后，用它来通知源 GPU。
- **Output Readiness (OR) Table**（1024 entry, on-chip）：在源 GPU 上跟踪每个 token 的 top-k 输出就绪状态。计数器到达 top-k 即触发 Combine。

Token-centric scheduler 在软件中通过 megakernel + 持久线程块（Persistent TB）实现。它将 SM 划分为四组——Dispatch、GEMM-1、GEMM-2、Combine——每组"就绪门控"（readiness-gated）：只有当 tracker 标记 tile 就绪且 SM 组有空闲资源时，才发射对应的工作。

当流水线稳定后，Dispatch（GPU→交换机密集）和 Combine（交换机→GPU 密集）自然并发。两个互补的非对称流量模式同时占用双向带宽——总体带宽利用率大幅提升。

### 决策四：为什么 tile 大小选 128？

Token-centric kernel fusion 中，同步的 tile 大小被固定为 **128 个 token**。这个选择的理由链很清晰：

- 128 匹配 GEMM 的 tile 大小（在 H200 上，GEMM 的最优 tile 通常为 128×128 或 128×256）。选更小的 tile 会迫使 GEMM 采用次优 tile 尺寸，损害计算利用率。
- 更小的 tile 会增加同步开销——tracker 需要更频繁地检查就绪状态、发射通知。
- 更大的 tile 会粗化 overlap 粒度——如果 tile 是 256 token，那 256 token 全部 dispatch 完才能开始 GEMM-1，流水线阶段间的空隙变大。

论文在 Small-8 配置上验证了不同 tile 大小的效果（Figure 30），确认 128 是最优点。

### 决策五：多节点扩展——统一网络接口

虽然论文的主体在 NVL32 单节点系统上，但 Discussion 一节提出了多节点扩展方案。这种扩展的动机来自 DeepSeek-V3 自身的需求——跨节点专家并行需要 InfiniBand。而 InfiniBand Quantum 交换机同样存在类似的通信冗余，网内计算在跨节点层面同样有意义。

DySHARP 的多节点扩展方案抽象整个集群为**统一的共享内存系统**：

- 程序员继续用 `dymultimem.st` 和 `dymultimem.ld_reduce`，不需要关心物理拓扑。
- NVSwitch 和 Quantum IB Switch 协同做全局路由。
- 以 multicast 为例：源 GPU 复制请求——一份走 NVSwitch 做节点内多播，另一份经扩展的 IBGDA（InfiniBand GPUDirect Async）把 NVLink 包翻译为 IB 包，发给远程节点的 IB 交换机，远程交换机再做节点内多播。
- 进入 IB 之前，小的 NVLink 包先做聚合（类似 FinePack 的思路），以提升网络效率。

初步评估中，4/8 个 DGX-H100 节点和 2/4 个 NVL32 节点上，扩展 DySHARP 相比 DeepEP 和 DualPipe 均展示出了优势（Figure 31）。

---

## 横向对比：DySHARP 在方法和路线图中的位置

### 与 NVLS 的关系：继承与突破

DySHARP 不是推翻 NVLS，而是**在 NVLS 的 multimem 寻址框架上的一个扩展**。这两者的关系可以这样理解：

| 维度 | NVLS | DySHARP |
|------|------|---------|
| 通信模式 | 静态 collectives（AllReduce, AllGather, Reduce-Scatter, Barrier） | 动态 operators（MoE dispatch/combine） |
| 地址机制 | 单一 multimem 地址 + 预配置目标集 | 单一 multimem 地址 + 包内目标专家列表 |
| 内存管理 | 统一（所有 GPU 对称布局） | 分布式（每 GPU 独立管理布局，AL Table 本地翻译） |
| 指令集 | multimem.st, multimem.ld_reduce | dymultimem.st, dymultimem.ld_reduce（额外 r3, r4 寄存器） |
| CUDA API | cuMulticastCreate, cuMulticastBindAddr | cuDyMulticastCreate, cuDyMulticastBindAddr（额外阶段和容量参数） |

这种继承性意味着 DySHARP 可以被集成到现有系统中而不破坏已有功能——NVLS 的静态 collectives 继续正常工作，MoE 的动态通信走 Dymultimem 路径。论文强调硬件修改"构建于现有数据路径之上，不改变任何原有功能"。

### 与 FlexLink 的互补性

FlexLink（第七篇会详细展开）从另一个完全不同的角度切入通信瓶颈：**不改变通信模式，而是聚合更多可用链路**。

FlexLink 观察到在 H800 上，NVLink 带宽被合规性限制砍半到 400 GB/s，而 PCIe（128 GB/s 双向）和 RDMA NIC（50 GB/s each）在 NVLink 通信期间几乎闲置。FlexLink 通过两阶段自适应负载均衡，把一部分集体通信流量卸载到 PCIe 和 RDMA 路径上，实现 AllGather +27%、AllReduce +26% 的带宽提升。

DySHARP 和 FlexLink 解决不同层面的问题：

| 维度 | DySHARP | FlexLink |
|------|---------|----------|
| 优化层面 | 通信**模式**——消除冗余 | 通信**路径**——聚合闲置链路 |
| 硬件修改 | 需要（包格式、ISA、微架构） | 不需要（纯软件方案，NCCL API 兼容） |
| 适用场景 | NVSwitch 全互联系统（H100/H200/B200/GB200 NVL） | NVLink 带宽受限的场景（H800/A800/合规芯片） |
| 性能上限 | ~2× 通信理论加速（消除冗余） | ~30% 带宽提升（受限于空闲链路带宽） |
| 对 MoE 的特殊性 | 针对 dispatch/combine 的动态不规则性 | 针对集体操作的链路聚合，非 MoE 特定 |

两者理论上可以叠加使用——FlexLink 聚合更多链路扩大总带宽，DySHARP 消除传输冗余降低总通信量。但目前还没有论文报告这种组合实验。

### 与 COMET 等 overlap 方案的关系

COMET（COMmunication-CompuTation Overlapping for MoE）是 2025 年 ByteDance 提出的细粒度 overlap 方案。DySHARP 将其作为 SOTA baseline，并取得了 1.79× 加速。

这 1.79× 来自哪里？关键实验是 ablation study（Figure 16）：

- **DeepEP（无 overlap, 无 in-switch）**：通信瓶颈严重，作为基准
- **COMET（fine-grained overlap, 无 in-switch）**：overlap 隐藏了部分通信，但仍受冗余流量限制
- **DySHARP-Basic（in-switch 通信原语, 无 overlap）**：流量减少约 50%，但因方向性不对称，速度反而不如 COMET
- **DySHARP-COMET（in-switch + COMET overlap）**：流量减少 + overlap，但方向性不对称仍存在，改善有限
- **Kernel fusion only（overlap 流水线, 无 in-switch）**：和 COMET 打平——没有流量减少，光靠流水线不够
- **DySHARP full（in-switch + kernel fusion）**：流量减少 + 方向性平衡 = 1.79×

核心信息：**孤立优化哪一层都不够。网内通信原语（硬件/系统软件层）和 token 级流水线调度（软件/运行时层）必须联合设计。**

### 硬件开销评估

可能有人担心全栈改造的硬件成本。论文给出了具体数字（TSMC 12nm 工艺合成）：

- **交换机端**：控制逻辑增加 <0.01mm²，不到 NVSwitch 芯片面积（294mm² for NVSwitch 3.0）的 0.1%。加了一个周期的数据路径延迟。
- **GPU 端**：SM LSU（MultimemQ 32 entries）+ Hub Memory Manager（AL TLB 512 entries + AL Table walker logic）≈ 0.198mm²——约为 H100 芯片（814mm²）的 0.024%。

这些数字说明：从硬件复杂度角度看，支持动态网内计算的门槛很低。真正的障碍在于**全栈协同设计的工程复杂性**——你必须同时改 packet 格式、ISA、CUDA 运行时和通信库。这也是为什么这不是一个"NVIDIA 自己就能轻松加上"的功能——它要求多个团队在同一个 vision 下协同工作。

---

## 总结与清单

DySHARP 讲述的是一个"网内计算从静态到动态"的工程故事，核心结论可以凝练为三条：

**1. 通信冗余是 MoE 专家并行的结构性问题，不是工程疏忽。** Dispatch 阶段的一对多模式天然产生重复传输，Combine 阶段的多对一模式天然产生重复归约。top-k >= 8 时，约 50% 的通信流量是冗余的——这是 top-k routing 的必然副产品，所有 MoE 系统都无法绕开。

**2. 静态网内计算对动态通信模式是"饮鸩止渴"。** NVLS 的 workaround（用 AllGather 模拟 Dispatch、用 Reduce-Scatter 模拟 Combine）产生了 340% 的无用流量——消除 50% 冗余的代价是增加 340% 新冗余，净效果是流量暴涨。这说明：**给动态问题套静态的壳，往往比不做优化更差。**

**3. 通信原语和调度策略是同一枚硬币的两面。** 动态 multimem 寻址消除流量但引入方向性不对称，token-centric kernel fusion 消除不对称但需要流量减少才有意义。单独拿出任何一个都不如已有的 SOTA（COMET）。两者的耦合不是工程选择，而是物理约束的必然——带宽在物理上是双向独立的，而通信模式天然不对称。

### 工程师决策清单

- [ ] **你的 top-k 是多少？** 如果 top-k >= 4，考虑网内计算的收益。top-k 越大，冗余比例越高，Dymultimem 的收益越大。如果 top-1 或 top-2，冗余比例较低（<30%），收益可能不足以 justify 全栈改造的工程成本。
- [ ] **你的系统有 NVSwitch 吗？** Dymultimem 依赖 NVSwitch 内的硬件修改。DGX-H100/DGX-B200/NVL 系统满足条件。基于 PCIe 直连的 8 卡系统（如部分消费级/H800 配置）没有 NVSwitch，无法受益。
- [ ] **你的通信时间占比是多少？** 如果 MoE 层的通信占比没有达到 50%+，1.79× 的 MoE 层加速对端到端训练的影响将被稀释。在 DeepSeek-V3 NVL32 配置下是 70.4%，这是 Dymultimem 的理想目标场景。
- [ ] **你是否在考虑多节点扩展？** Dymultimem 的多节点方案需要 IB Quantum 交换机配合，且需要扩展 IBGDA 来翻译 NVLink 包格式。这是一个额外的集成工作量。
- [ ] **你的 CUDA 工具链能接受 ISA 修改吗？** `dymultimem.st` / `dymultimem.ld_reduce` 是新的 PTX/SASS 指令。如果你的生产环境不允许修改 CUDA 工具链，Dymultimem 硬件的部署会受阻——你的通信库需要 emit 这些新指令。

---

*下一篇预告：第七篇《FlexLink 与异构链路聚合 —— 榨干每一根链路》。在 H800 这种 NVLink 带宽被合规砍半的平台上，PCIe 和 RDMA NIC 在集体通信期间几乎闲置。FlexLink 如何用两阶段自适应负载均衡把这些闲置链路全部用起来？*
