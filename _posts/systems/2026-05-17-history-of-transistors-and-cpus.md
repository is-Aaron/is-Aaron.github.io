---
title: "晶体管与 CPU 的演进史：从一个开关到千亿级开关"
date: 2026-05-17 00:00:00 +0800
categories: [计算机系统, 硬件]
tags: [CPU, 晶体管, 集成电路, 微处理器, 半导体, 计算机体系结构]
---

如果把计算机比作一座城市，CPU 就像它的调度中心，而晶体管就是这座城市里最基础的“开关”。今天的高端处理器、SoC 或 AI 加速器已经可以容纳数十亿、上百亿乃至千亿级晶体管：例如 Apple 2025 年发布的 M3 Ultra 通过 UltraFusion 将两颗 M3 Max die 连接为软件可见的一颗统一芯片，总计 1840 亿个晶体管；NVIDIA Blackwell 架构 GPU 则达到 2080 亿个晶体管。这样的进步并不是简单地“把开关做小”。它经历了材料、制造工艺、集成电路、指令集、缓存、流水线、功耗控制和软件生态的多次共同演化。

理解晶体管和 CPU 的历史，最重要的不是背年份，而是看清一条主线：**人类先学会用半导体做可靠开关，再学会把开关批量制造到同一块芯片上，最后才把整个中央处理器压缩进一枚微处理器。**

## 先厘清三个概念

### 晶体管：可控的电子开关

晶体管本质上是一种用小信号控制大信号的半导体器件。它既可以做放大器，也可以做数字电路里的开关。对 CPU 来说，晶体管最重要的角色是受控开关：数字电路通常用电压高低来表示逻辑 1 和 0，再由多个开关组合成逻辑门，继续组合出加法器、寄存器、控制器、缓存和各种执行单元。

几个和 CPU 历史密切相关的器件/电路概念包括：

- **BJT（Bipolar Junction Transistor，双极型晶体管）**：速度快，但功耗和集成密度不如后来成为主流的 MOS 系器件。
- **MOSFET（Metal-Oxide-Semiconductor Field-Effect Transistor，金属氧化物半导体场效应晶体管）**：更适合大规模集成，是现代 CPU 的基础器件。
- **CMOS（Complementary MOS，互补式 MOS）**：不是单只晶体管类型，而是用 NMOS 和 PMOS 搭配实现逻辑的电路/工艺风格；它静态功耗低，成为现代数字芯片的主流实现方式。

### CPU：执行指令的中央处理器

CPU（Central Processing Unit，中央处理器）是执行程序指令的核心部件。它通常包含：

- **控制单元**：负责取指、译码、调度执行流程。
- **算术逻辑单元（ALU）**：负责整数运算和逻辑运算。
- **寄存器**：CPU 内部最快的小型存储。
- **缓存（Cache）**：缓解 CPU 与内存速度差距。
- **执行单元、分支预测、流水线、乱序执行等结构**：让多个指令尽可能并行推进。

注意：**CPU 不一定等于微处理器**。早期 CPU 可以由许多真空管、分立晶体管或多块集成电路组成；微处理器则是把 CPU 集成到一块芯片上的产物。

### 微处理器：一块芯片里的 CPU

微处理器（Microprocessor）通常指单芯片 CPU。它让计算机从“由一柜子电子部件组成的机器”逐渐走向个人电脑、嵌入式设备、手机和今天的云计算基础设施。

所以这段历史可以分成三层：

| 层次 | 关键问题 | 历史突破 |
| :--- | :--- | :--- |
| 器件层 | 如何制造可靠、低功耗、可批量生产的开关？ | 晶体管、MOSFET、CMOS |
| 制造层 | 如何把大量开关连接在同一块芯片上？ | 平面工艺、光刻、集成电路 |
| 架构层 | 如何让这些开关高效执行程序？ | 微处理器、指令集、缓存、流水线、多核 |

## 真空管时代：计算机先于现代晶体管出现

在晶体管之前，电子计算机主要依赖真空管。真空管也能充当开关和放大器，但它有明显缺陷：体积大、发热高、耗电多、寿命短。早期电子计算机能工作，但维护成本极高，机器规模也很难继续扩大。

这解释了为什么晶体管不是“锦上添花”的改进，而是一次底层替换。它把计算设备从笨重、脆弱、耗电的真空管时代，带向小型、可靠、可量产的半导体时代。

## 1947：点接触晶体管诞生

1947 年 12 月 16 日，贝尔实验室的 John Bardeen、Walter Brattain 在探索固态放大器的研究团队中做出了第一只可工作的点接触晶体管；12 月 23 日，他们向实验室管理层演示了这个器件。William Shockley 是这个方向上的关键负责人之一，并在随后提出结型晶体管理论。这个点接触器件使用高纯锗材料和两个非常接近的金触点，证明了半导体能够实现信号放大。

这一步的历史意义在于：电子开关不再必须依赖真空中的电子流动，而可以在固体材料内部完成。1956 年，Bardeen、Brattain、Shockley 因半导体研究和晶体管效应获得诺贝尔物理学奖。

不过，点接触晶体管并不适合大规模稳定生产。1948 年，Shockley 构想了基于 p-n 结的结型晶体管结构；到 1951 年，Bell Labs 宣布了性能超过点接触晶体管的 grown-junction transistor，让结型晶体管路线向更稳定、可生产的方向迈进。再加上后来硅材料、氧化层和光刻工艺的成熟，晶体管才逐渐从实验室器件走向工业产品。

## 1950 年代：从锗到硅，从单个器件到可制造工艺

早期晶体管常使用锗。锗容易做出器件，但耐高温和稳定性不如硅。硅真正成为半导体工业的基础，不只是因为它是半导体材料，更因为硅可以形成稳定的二氧化硅绝缘层。这个看似材料学的小细节，后来变成了 MOSFET、平面工艺和现代集成电路的关键。

1959 年，Fairchild 的 Jean Hoerni 发明平面工艺。它把晶体管结构做在硅片表面，并利用氧化层保护和隔离器件，使晶体管更可靠，也更适合用光刻方式批量制造。平面工艺的重要性常被低估：它不是又一种晶体管，而是让“很多晶体管可以在同一片硅上稳定制造”的工艺基础。

## 1958-1959：集成电路把“连接问题”变成“制造问题”

当工程师开始用越来越多晶体管搭建计算机时，一个新问题出现了：即使单个晶体管足够小，手工焊接和布线也会迅速变成灾难。电路越复杂，连接点越多，故障率越高，成本越不可控。

集成电路解决的正是这个问题。

1958 年，Texas Instruments 的 Jack Kilby 演示了早期集成电路，在半导体材料上形成晶体管、电容、电阻等元件，再用细金线把它们连成电路。这个方案证明了“把电路功能集成到半导体材料中”的可行性，但飞线连接并不适合大规模生产。1959 年，Fairchild 的 Robert Noyce 基于 Hoerni 的平面工艺提出更适合量产的单片集成电路方案，用沉积金属互连取代飞线连接。两者共同奠定了 IC（Integrated Circuit，集成电路）的历史起点：**电路不再由一个个分立元件手工拼起来，而是越来越多地作为一个整体被制造出来。**

这一步对 CPU 的意义极大。没有集成电路，CPU 可以存在，但会昂贵、庞大、难维护；有了集成电路，CPU 才有机会进入越来越小的设备。

## 1959-1960：MOSFET 打开高密度集成的大门

1959 年，贝尔实验室的 Mohamed Atalla 和 Dawon Kahng 做出了第一只成功的绝缘栅场效应晶体管；到 1960 年，这一路线以 MOS 场效应放大器的形式被展示出来，也就是后来 CPU 中最重要的 MOSFET 技术谱系。MOSFET 的优势在于结构适合缩小，输入端由氧化层隔离，便于大规模集成。

早期微处理器并不是一开始就采用现代 CMOS，例如 Intel 4004 依赖的是硅栅 MOS 技术；但 MOS 技术为大规模集成提供了方向。随着 PMOS、NMOS、CMOS 工艺相继成熟，芯片可以容纳更多逻辑，同时把功耗控制在可接受范围内。今天我们谈 CPU 制程、栅极、沟道、漏电、FinFET、GAAFET，本质上都还在 MOSFET 这条技术谱系里。

## 1965：摩尔定律让行业有了节拍器

1965 年，Gordon Moore 在《Electronics》杂志发表文章，观察到集成电路上可经济制造的元件数量快速增长，并预测这种趋势还会持续。后来行业把这个经验规律称为“摩尔定律”。早期表述接近“每年翻番”，后来常被概括为“约每两年翻番”。

摩尔定律不是物理定律，而是产业协同目标。它把材料、光刻、设备、EDA、架构设计、制造良率和市场需求绑在一起，形成了一种节奏：下一代芯片应该更密、更便宜、更强。

对 CPU 来说，摩尔定律带来两个直接结果：

- **晶体管预算不断增加**：从几千个晶体管，到几百万、几亿、几十亿。
- **架构设计不断改变**：新增缓存、流水线、预测器、向量单元、多核和专用加速模块，都依赖越来越大的晶体管预算。

## 1971：Intel 4004 与商用微处理器时代

1971 年，Intel 4004 发布。它通常被称为第一款商用单芯片微处理器，拥有约 2,300 个晶体管，最初面向 Busicom 计算器项目。

4004 的性能按今天标准看微不足道，但它完成了一次概念迁移：**CPU 可以作为一颗通用芯片出售，而不是为每台机器定制一套逻辑板。** 从此以后，计算机产业开始围绕标准化处理器、内存、外设和软件生态展开。

需要精确地区分：4004 不是历史上第一个能执行程序的 CPU，也不是唯一早期处理器方案；它的重要地位在于“商用、单芯片、可编程微处理器”这几个条件同时成立。

## 1970 年代：8 位与 16 位处理器把计算带进个人设备

4004 之后，微处理器迅速从 4 位走向 8 位、16 位：

| 年份 | 代表处理器 | 影响 |
| :--- | :--- | :--- |
| 1971 | Intel 4004 | 商用单芯片微处理器起点 |
| 1974 | Intel 8080 | 早期微型计算机的重要 CPU |
| 1975 | MOS Technology 6502 | 低成本推动家用电脑和游戏机普及 |
| 1976 | Zilog Z80 | 广泛用于嵌入式与早期个人电脑 |
| 1978 | Intel 8086 | x86 指令集起点 |
| 1979 | Intel 8088 | 8086 的 8 位外部总线版本，降低系统成本 |

这一时期的核心变化不是单纯性能提升，而是“计算开始商品化”。处理器可以批量购买，开发者可以围绕某个指令集写软件，厂商可以围绕处理器构建兼容生态。

## 1981：IBM PC 与 x86 生态的放大效应

1981 年，IBM 发布 IBM PC，采用 Intel 8088 作为中央处理器。这个选择的影响远远超过硬件本身：IBM PC 的开放式硬件结构、MS-DOS 软件生态以及后来的兼容机市场，使 x86 成为个人计算时代最重要的指令集之一。

这也是 CPU 历史里一个很典型的现象：**技术上最优的方案，不一定成为生态上最强的方案。** x86 的长期生命力，既来自 Intel 和 AMD 的持续工程投入，也来自巨大的软件兼容性惯性。

随后，Intel 80286 引入更强的保护模式能力，80386 在 1985 年把 x86 推向 32 位，并支持分页机制。操作系统、编译器和应用软件开始依赖更复杂的内存管理，CPU 不再只是“算得更快”，而是在支撑更复杂的软件抽象。

## 1980 年代：RISC 思想挑战复杂指令集

当 x86 等 CISC（Complex Instruction Set Computer，复杂指令集计算机）继续发展时，另一条路线也在兴起：RISC（Reduced Instruction Set Computer，精简指令集计算机）。

RISC 的核心观察是：如果指令更简单、更规整，CPU 就更容易做流水线、更容易提高频率，也更容易让编译器参与优化。IBM 801、Berkeley RISC、Stanford MIPS 等项目推动了这一思想。后来 ARM、MIPS、SPARC、PowerPC 等架构都不同程度继承了 RISC 传统。

这并不是“RISC 一定比 CISC 好”的简单二分。现代高性能 x86 CPU 内部通常会把复杂指令解码成更小粒度的微操作再执行；而现代 RISC CPU 也会加入复杂的向量扩展、预测器和乱序执行。真正的历史趋势是：**指令集是软件看到的契约，微架构才是硬件真正发力的地方。**

## 1990 年代：频率、流水线、缓存与乱序执行

进入 1990 年代，CPU 性能提升越来越依赖微架构技巧：

- **更深流水线**：把指令执行拆成更多阶段，提高时钟频率。
- **超标量执行**：一个周期内发射多条指令。
- **乱序执行**：只要数据依赖允许，就不必严格按程序顺序执行。
- **分支预测**：提前猜测程序会走哪条分支，减少等待。
- **多级缓存**：用 L1/L2/L3 缓解内存访问延迟。

这些技术都需要大量晶体管。也就是说，晶体管数量增加并不只是让 ALU 变多，而是让 CPU 拥有更复杂的“调度系统”。现代 CPU 的很多晶体管并不直接做算术，而是在做预测、缓存、重命名、排队和数据搬运。

## 2000 年代：功耗墙迫使 CPU 转向多核

1974 年，Robert Dennard 等人系统量化了 MOSFET 的缩放规律：在理想的恒场缩放下，晶体管尺寸缩小的同时，电压、电流等参数也按比例下降，从而提升速度、降低单器件功耗和成本，并让单位面积功耗在一段时间内保持可控。这个规律支撑了很长一段时间的“更小、更快、功耗还能接受”。

但到了 2000 年代，电压无法按比例继续下降，漏电流和散热问题变得突出。CPU 不能再简单依赖“频率越来越高”获得性能。行业开始明显转向多核、并行计算和能效优化。

这就是所谓“功耗墙”的影响：

- 单核频率继续提升会遇到散热和功耗限制。
- 多核把晶体管预算分散给多个执行核心。
- 软件必须更多考虑并发、并行和数据局部性。
- GPU、DSP、NPU 等专用或半专用处理器的重要性上升。

从这个阶段开始，CPU 历史不再只是 CPU 自己的历史，而变成了 CPU、GPU、内存、互连、编译器、操作系统和应用负载共同演化的历史。

## 2003 以后：64 位、移动端、多核与异构计算

2003 年，AMD Opteron 把 x86-64 带入服务器市场。它的重要之处在于：在保持 x86 软件兼容性的同时，提供 64 位地址空间和寄存器扩展。相比完全另起炉灶的 64 位路线，x86-64 的兼容性策略更容易被市场接受。

随后十多年，CPU 发展出现几条并行主线：

- **服务器 CPU**：更多核心、更大缓存、更强内存带宽和 I/O 能力。
- **桌面 CPU**：在单线程性能、游戏负载、多核创作负载之间平衡。
- **移动 CPU**：以能效为中心，big.LITTLE、SoC、专用加速器成为常态。
- **云和 AI 时代的 CPU**：CPU 负责通用控制和复杂系统调度，GPU/NPU/TPU 等负责高吞吐矩阵计算。

这说明 CPU 没有被取代，而是角色发生了变化：它不再独自承担所有计算，而是成为异构计算系统里的通用协调者。

## 近年趋势：先进制程之外，封装和架构越来越重要

当晶体管继续逼近物理极限，单纯依赖制程缩小已经越来越难。现代处理器开始从多个方向继续扩展：

- **FinFET 与 GAAFET**：改进晶体管结构，增强栅极对沟道的控制。
- **Chiplet**：把大芯片拆成多个小芯粒，提高良率并支持灵活组合。
- **2.5D/3D 封装**：通过先进封装缩短芯片之间的数据距离。
- **专用加速器**：为图形、视频编码、AI、加密、安全隔离等任务提供更高能效。
- **软硬件协同**：编译器、运行时、操作系统调度与硬件特性绑定更紧。

如果说 1970 年代的关键词是“把 CPU 放进一颗芯片”，那么今天的关键词更像是“把不同类型的计算单元放进一个高带宽、低延迟、可制造的系统里”。

## 未来技术路线：从器件缩放走向系统级优化

往未来看，CPU 和先进处理器不会只沿着“制程数字继续变小”这一条路前进。更现实的路线是多条技术并行推进：晶体管结构继续演进，供电和互连重新设计，封装从“保护芯片”变成“组织系统”，软件也要更主动地配合硬件。

### 1. 晶体管结构：从 FinFET 走向 GAAFET / nanosheet

FinFET 曾经解决了平面 MOSFET 缩小后的栅极控制问题，但继续缩小时，漏电、变异性和制造复杂度仍会加重。下一步主流方向是 GAAFET（Gate-All-Around FET）及其 nanosheet / RibbonFET 等实现：让栅极更充分地包围沟道，从而增强对电流的控制。

TSMC 的 N2 技术采用 nanosheet transistor，A16 会把 nanosheet 与 Super Power Rail 结合，A14 则计划继续沿 nanosheet 和 NanoFlex Pro 方向演进；Intel 18A 则把 RibbonFET 作为核心特性之一。它们的共同目标不是“让单个晶体管无限变快”，而是在更小尺寸下继续维持可接受的漏电、性能和能效。再往后，业界还会研究 CFET（Complementary FET，把 nFET 与 pFET 纵向堆叠）等结构，但这类技术距离大规模商业落地还存在制造、散热、设计规则和成本挑战。

### 2. 供电网络：背面供电会变得更重要

传统芯片的信号线和供电线主要都在晶圆正面金属层中竞争空间。随着晶体管和互连越来越密，供电压降、金属线电阻和布线拥塞会限制性能。背面供电（backside power delivery）把部分供电网络移到晶圆背面，目标是让正面更专注于信号互连，同时降低供电阻抗。

Intel 将 PowerVia 作为 Intel 18A 的重要特性，TSMC 也在 A16 路线中提出 Super Power Rail。需要注意的是，背面供电不是“新型晶体管”，也不是直接替代先进制程；它更像是工艺集成和布线体系的重构，收益会体现在供电稳定性、面积利用率和能效上。

### 3. 光刻：High-NA EUV 延续缩小，但不会单独解决所有问题

EUV 光刻已经是先进制程的重要基础。下一阶段的 High-NA EUV 通过把数值孔径从 0.33 提高到 0.55，提升可打印特征的分辨率。ASML 公开资料中提到，High-NA EUV 平台面向更小关键尺寸和更高晶体管密度。

不过，光刻能力只是先进制程的一环。真实产品是否采用某项光刻路线，还要看设备成本、吞吐量、良率、掩模、光刻胶、设计规则和客户产品节奏。换句话说，High-NA EUV 会继续推动缩小，但 CPU 进步不会只由光刻机决定。

### 4. Chiplet 与 2.5D / 3D 封装：CPU 会越来越像一个系统

当单片大芯片的成本和良率压力上升，把不同功能拆成多个 chiplet 再封装到一起，会越来越有吸引力。CPU 核心、I/O、缓存、GPU、AI 加速器、HBM 控制器可以根据成本、性能和良率选择不同工艺节点，再通过先进封装连接。

TSMC 的 3DFabric 包括 SoIC、CoWoS、InFO 等技术，Intel 的 EMIB 和 Foveros 也属于这一方向。UCIe 则试图把封装内 chiplet 互连标准化，推动更开放的 chiplet 生态。它们共同指向一个趋势：未来的“处理器”可能不再是一颗均质大芯片，而是一个由多个芯粒、缓存和存储堆叠组成的封装级系统。

但 chiplet 也不是免费午餐。跨芯粒通信会带来延迟、功耗、带宽、热密度和验证复杂度问题。先进封装能缓解单片芯片的制造压力，却会把难题转移到系统集成、散热和软硬件协同上。

### 5. 内存墙：性能瓶颈会越来越多地来自数据搬运

很多现代负载并不是“算术单元不够”，而是数据搬运太慢、太耗能。CPU 未来的关键能力会更多体现在缓存层级、内存带宽、片上互连、NUMA 拓扑和数据局部性上。HBM、3D 堆叠缓存、近内存计算、CXL 内存扩展与内存池化，都是围绕内存墙展开的路线。

CXL 的定位尤其值得注意：它是面向 CPU、内存扩展设备、加速器和智能 I/O 的开放一致性互连。它有助于数据中心做内存扩展、共享和池化，但它不会让远端或扩展内存拥有和本地 DDR / HBM 完全一样的延迟特性。软件需要知道数据放在哪里，才能真正吃到这些硬件能力。

### 6. 指令集与软件生态：x86、Arm、RISC-V 会长期共存

未来 CPU 竞争不会只发生在晶体管和封装层，也会发生在指令集、编译器、操作系统、运行时和开发工具链层。x86 仍有强大的兼容性资产，Arm 在移动端、低功耗和服务器市场持续扩展，RISC-V 则以开放、免版税和可扩展 ISA 的方式，为定制处理器、嵌入式、控制核心和专用加速器提供新选择。

但指令集本身并不自动决定性能。真正的产品竞争还取决于微架构实现、制程、缓存、互连、编译器优化、系统软件和生态成熟度。未来 CPU 更像是一组软硬件共同定义的“平台”，而不是单独一颗芯片参数表。

可以把这些路线概括成一张表：

| 技术路线 | 主要解决的问题 | 需要避免的误解 |
| :--- | :--- | :--- |
| GAAFET / nanosheet / RibbonFET | 继续改善小尺寸晶体管的栅极控制和能效 | 不是换个晶体管名字就能自动大幅提频 |
| 背面供电 | 缓解供电压降、布线拥塞和能效问题 | 它是供电/互连体系重构，不是单独的 CPU 架构 |
| High-NA EUV | 提升先进制程可打印特征的分辨率 | 光刻进步仍受成本、良率和设计规则约束 |
| Chiplet 与 3D 封装 | 提高良率、混合不同工艺、靠近逻辑与内存 | 会引入跨芯粒延迟、热设计和验证复杂度 |
| HBM / CXL / 内存池化 | 缓解内存容量、带宽和资源利用率瓶颈 | 扩展内存不等于本地内存，延迟模型不同 |
| RISC-V 与专用 ISA 扩展 | 支持开放定制和领域专用优化 | 开放 ISA 不等于现成高性能 CPU 实现 |

因此，未来 CPU 的主线可以总结为：**晶体管继续变小，但“把数据放在哪里、如何供电、如何封装、如何调度、如何让软件理解硬件”会变得同样重要。** 半导体行业会从单纯追逐晶体管密度，转向更复杂的系统级性能、能效和成本优化。

## 一张时间线总览

| 时间 | 事件 | 对 CPU 发展的意义 |
| :--- | :--- | :--- |
| 1904/1906 | 真空管二极管与三极管先后出现 | 为电子放大和电子开关奠定器件基础 |
| 1940s | ENIAC 等真空管电子计算机 | 证明电子开关可用于通用计算，但体积、功耗、可靠性受限 |
| 1947 | 贝尔实验室点接触晶体管 | 半导体开关时代开始 |
| 1958 | Kilby 演示早期集成电路 | 多个元件可集成在同一材料上 |
| 1959 | Hoerni 平面工艺、Noyce 单片 IC 思想 | 集成电路获得量产路径 |
| 1959-1960 | Atalla 与 Kahng 推动 MOS 晶体管 | 现代高密度数字芯片的器件基础 |
| 1965 | Moore 提出集成电路元件增长趋势 | 半导体产业获得长期节奏 |
| 1971 | Intel 4004 发布 | 商用单芯片微处理器时代开启 |
| 1978 | Intel 8086 发布 | x86 指令集起点 |
| 1980 | IBM 801 展示 RISC 思想 | 指令集和微架构设计出现重要分野 |
| 1981 | IBM PC 采用 Intel 8088 | x86 与 PC 兼容生态扩大 |
| 1990s | 超标量、乱序执行、多级缓存普及 | CPU 用更多晶体管提升单线程性能 |
| 2000s | 功耗墙与多核转向 | 性能增长从高频率转向并行和能效 |
| 2010s-2020s | Chiplet、异构计算、先进封装 | CPU 成为更大计算系统的一部分 |
| 2020s 以后 | GAAFET、背面供电、High-NA EUV、CXL 与开放 chiplet 生态 | 性能增长更多依赖器件、封装、内存和软件协同 |

## 总结：CPU 史就是晶体管预算的使用史

晶体管发明解决了“可靠电子开关”的问题；集成电路解决了“如何批量连接海量开关”的问题；微处理器解决了“如何把通用计算能力商品化”的问题；现代 CPU 架构则在回答“如何把越来越昂贵的晶体管预算用在最值得的地方”。

回看这段历史，可以得到一个很朴素的结论：CPU 的进步从来不是单点突破。它依赖物理学、材料科学、制造工艺、电子工程、计算机体系结构、编译器和软件生态长期配合。今天我们写下一行代码，最终能在纳米尺度的晶体管阵列中变成电信号流动，背后正是这七十多年技术积累的结果。

## 术语表

- **ALU**：Arithmetic Logic Unit，算术逻辑单元，负责整数运算和逻辑运算。
- **BJT**：Bipolar Junction Transistor，双极型晶体管，早期重要晶体管类型。
- **CFET**：Complementary FET，互补场效应晶体管研究方向，通常指把 nFET 与 pFET 纵向堆叠以继续提高密度。
- **CISC**：Complex Instruction Set Computer，复杂指令集计算机，典型代表包括 x86。
- **CMOS**：Complementary MOS，互补式 MOS，用 NMOS 和 PMOS 互补搭配实现逻辑的电路/工艺风格，是现代数字芯片的主流实现方式。
- **CPU**：Central Processing Unit，中央处理器，负责执行程序指令。
- **CXL**：Compute Express Link，面向 CPU、内存扩展设备和加速器的一致性互连标准。
- **DTCO**：Design-Technology Co-Optimization，设计与工艺协同优化，让工艺能力和芯片设计规则共同演进。
- **EMIB**：Embedded Multi-die Interconnect Bridge，Intel 的 2.5D 封装互连技术，用于在封装内连接多个芯片/芯粒。
- **FinFET**：鳍式场效应晶体管，通过三维结构改善栅极控制能力。
- **GAAFET**：Gate-All-Around FET，环绕栅晶体管，进一步增强栅极对沟道的控制。
- **HBM**：High Bandwidth Memory，高带宽存储器，常通过先进封装靠近 GPU、AI 加速器或高性能处理器。
- **High-NA EUV**：高数值孔径 EUV 光刻，通过更高数值孔径提升先进制程图形分辨率。
- **IC**：Integrated Circuit，集成电路，把多个器件和连接集成在同一芯片上。
- **MOSFET**：Metal-Oxide-Semiconductor Field-Effect Transistor，金属氧化物半导体场效应晶体管，现代 CPU 的基础晶体管类型。
- **NUMA**：Non-Uniform Memory Access，非一致内存访问，指不同 CPU/核心访问不同内存区域时延迟和带宽可能不同。
- **RISC**：Reduced Instruction Set Computer，精简指令集计算机，强调简单规整指令、流水线友好和编译器优化。
- **SoC**：System on Chip，片上系统，通常把 CPU、GPU、内存控制器、I/O 和专用模块集成到同一芯片上；若跨多个芯粒或封装组合，更准确地说属于 chiplet 或 SiP 等系统级封装路线。
- **UCIe**：Universal Chiplet Interconnect Express，面向封装内 chiplet 互连的开放标准。
- **x86-64**：x86 的 64 位扩展，最早由 AMD 推向主流市场，后被广泛采用。

## 参考文献

- Computer History Museum, [1947: Invention of the Point-Contact Transistor](https://www.computerhistory.org/siliconengine/invention-of-the-point-contact-transistor/)
- Nobel Prize, [The Nobel Prize in Physics 1956](https://www.nobelprize.org/prizes/physics/1956/summary/)
- Computer History Museum, [1946 Timeline: ENIAC](https://www.computerhistory.org/timeline/1946/)
- Encyclopaedia Britannica, [Electronics: The vacuum tube era](https://www.britannica.com/technology/electronics)
- National Museum of American History, [Experimental DeForest "Audion" Tube](https://americanhistory.si.edu/collections/nmah_1289117)
- Nokia Bell Labs, [1956 Nobel Prize in Physics](https://www.nokia.com/bell-labs/about/awards/1956-nobel-prize-physics/)
- Computer History Museum, [1948: Conception of the Junction Transistor](https://www.computerhistory.org/siliconengine/conception-of-the-junction-transistor/)
- Computer History Museum, [1951: First Grown-Junction Transistors Fabricated](https://www.computerhistory.org/siliconengine/first-grown-junction-transistors-fabricated/)
- Computer History Museum, [1958: All Semiconductor "Solid Circuit" is Demonstrated](https://www.computerhistory.org/siliconengine/all-semiconductor-solid-circuit-is-demonstrated/)
- Computer History Museum, [1959: Invention of the "Planar" Manufacturing Process](https://www.computerhistory.org/siliconengine/invention-of-the-planar-manufacturing-process/)
- Computer History Museum, [1959: Practical Monolithic Integrated Circuit Concept Patented](https://www.computerhistory.org/semiconductor/timeline/1959-Noyce.html)
- Computer History Museum, [1960: Metal Oxide Semiconductor (MOS) Transistor Demonstrated](https://www.computerhistory.org/siliconengine/metal-oxide-semiconductor-mos-transistor-demonstrated/)
- Computer History Museum, [1971: Microprocessor Integrates CPU Function onto a Single Chip](https://www.computerhistory.org/siliconengine/microprocessor-integrates-cpu-function-onto-a-single-chip/)
- Intel, [Moore's Law](https://www.intel.com/content/www/us/en/history/virtual-vault/articles/moores-law.html)
- Intel, [Announcing a New Era of Integrated Electronics](https://www.intel.com/content/www/us/en/history/virtual-vault/articles/the-intel-4004.html)
- Smithsonian Institution, [Intel 4004 Microprocessor](https://www.si.edu/object/intel-4004-microprocessor%3Anmah_713495)
- Apple, [Apple reveals M3 Ultra](https://www.apple.com/newsroom/2025/03/apple-reveals-m3-ultra-taking-apple-silicon-to-a-new-extreme/)
- NVIDIA, [Blackwell Architecture](https://www.nvidia.com/en-us/data-center/technologies/blackwell-architecture/)
- TSMC, [Advanced Logic Technologies for Mobile Devices](https://www.tsmc.com/english/dedicatedFoundry/technology/platform_smartphone_tech_advancedTech)
- TSMC, [TSMC Celebrates 30th North America Technology Symposium with Innovations Powering AI with Silicon Leadership](https://pr.tsmc.com/english/news/3136)
- TSMC, [TSMC Unveils A14 Process at Technology Symposium to Advance the AI Future](https://esg.tsmc.com/en-US/articles/366)
- Intel Foundry, [Intel 18A](https://www.intel.com/content/www/us/en/foundry/process/18a.html)
- Intel Newsroom, [Intel Leads the Way with Advanced Packaging](https://newsroom.intel.com/manufacturing/intel-leads-the-way-with-advanced-packaging)
- ASML, [5 things you should know about High NA EUV lithography](https://www.asml.com/en/en/news/stories/2024/5-things-high-na-euv)
- TSMC, [先進封裝解決方案](https://www.tsmc.com/chinese/dedicatedFoundry/services/advanced-packaging)
- UCIe Consortium, [Universal Chiplet Interconnect Express](https://www.uciexpress.org/?lang=en)
- Compute Express Link Consortium, [About CXL](https://computeexpresslink.org/about-cxl/)
- RISC-V International, [About RISC-V](https://riscv.org/about/)
- Intel, [The Intel 8086 and the IBM PC](https://www.intel.com/content/www/us/en/history/virtual-vault/articles/the-8086-and-the-ibm-pc.html)
- Computer History Museum, [Intel 8088](https://www.computerhistory.org/collections/catalog/102631293)
- IBM, [RISC](https://www.ibm.com/history/risc)
- Computer History Museum, [80386 microprocessor, Intel, 1985](https://www.computerhistory.org/revolution/digital-logic/12/330/1575)
- Computer History Museum, [1974: Scaling of IC Process Design Rules Quantified](https://www.computerhistory.org/siliconengine/scaling-of-ic-process-design-rules-quantified/)
- AMD, [Press Release Exhibit, July 16, 2003](https://ir.amd.com/financial-information/sec-filings/content/0001193125-03-020249/dex991.htm)
