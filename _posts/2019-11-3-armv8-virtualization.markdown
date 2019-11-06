---
layout: post
title:  "Armv8架构虚拟化介绍"
categories: Technology
tags: Arm Virtualization
author: Calinyara
description: 
---

# 1 综述

<br>
本文描述了Armv8-A AArch64的虚拟化支持。包括stage 2页表转换，虚拟异常，以及陷阱。本文介绍了一些基础的硬件辅助虚拟化理论以及一些Hypervisor如何利用这些虚拟化特性的例子，但文本不会讲述某一具体的Hypervisor软件是如何工作的以及如何开发一款Hypervisor软件。通过阅读本文，你会学到两种类型的Hypervisor以及它们是如何映射到Arm的异常级别。你将能解释陷阱是如何工作的以及其是如何被用来进行各种模拟操作。你将能描述Hypervisor可以产生什么虚拟异常以及产生这些虚拟异常的机制。看懂本文需要一定基础，本文假定你熟悉ARMv8体系结构的异常模型和内存管理。

<br>
# 1.1 虚拟化简介
<br>
这里我们将介绍一些基础的Hypervisor和虚拟化的理论知识。如果你已经有一定的基础或是已经熟悉了这些概念，可以跳过这部分内容。我们用Hypervisor这个词来定义一种负责创建，管理以及调度虚拟机(Virtual Machines, VMs)的软件。

<br>
## 虚拟化为什么重要
<br>

虚拟化一种在现代云计算和企业基础架构种广泛使用的技术。开发人员用虚拟机在一个硬件平台上运行多个不同的操作系统来开发和测试软件，以避免对主计算环境造成可能的破坏。虚拟化技术在服务器上非常流行，大多数面向服务器的处理器都需要支持虚拟化功能，因为虚拟化能给数据中心服务器带来如下一些需要的特性：

- **隔离**：利用虚拟化可以对同一个物理核上运行的虚拟机进行隔离。这使得相互间不可信的的计算环境可以共享同一套硬件环境。例如，两个竞争者可以共享同一个物理机器而又不能访问对方的数据。
- **高可用性**： 虚拟化可以在不同的物理机器之间无缝透明地迁移负载。这个技术广泛用于将负载从出错地硬件平台迁移至其他可用平台以便维护和替换出错地硬件而不影响服务。
- **负载均衡**： 为了降低数据中心硬件和功耗成本，需要尽可能充分地利用硬件平台资源。将负载均衡地迁移到不同地物理机上，有利用充分利用物理机资源，降低功耗，同时为租户提供最佳性能。
- **沙箱**：虚拟机可以作为一个沙箱来为运行其中应用屏蔽其他软件的干扰，或者避免其干扰其他软件。例如在虚拟机种运行特定软件，可以避免该软件的bug或病毒导致物理机器上的其他软件损坏。

<br>
## 1.2 Hypervisor的两种类型
<br>

Hypervisor通常被分成两种类型，独立类型Type 1和寄生类型 Type 2。我们先看看Type 2类型Hypervisor。对于Type 2类型的Hypervisor，其寄生的宿主操作系统拥有对硬件平台和资源的全部控制权，包括CPU和物理内存。下图展示了Type 2类型的Hypervisor。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/1 Example of a hosted or Type 2 hypervisor.png"/></div>
<p align="center">图1：Type 2 Hypervisor</p>

<br>
宿主操作系统，指的是直接运行在硬件平台上并为Type 2类型的Hypervisor提供运行环境的操作系统。这类Hypervisor可以充分利用宿主操作系统对物理硬件的管理功能，其只需提供对客户机的管理即可。不知你是否使用过Virtual Box或是VMware Workstation, 这类软件就是Type 2类型的Hypervisor。

<br>
接下来，看看独立类型的Type 1 Hypervisor, 如图2。 这类Hypervisor没有宿主操作系统。其直接运行在物理硬件之上，直接管理各种物理资源如CPU和内存，同时管理运行客户机操作系统。
<br>

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/2 Example of a standalone or Type 1 hypervisor.png"/></div>
<p align="center">图2：Type 1 Hypervisor</p>

<br>
在开源社区常见的Hypervisor, Xen (Type 1) 和 KVM (Type 2)就分属这两种不同的类型。其他开源的或知识产权的Hypervisor，参见 [WiKi](https://en.wikipedia.org/wiki/Comparison_of_platform_virtualization_software)。

<br>
## 1.3 全虚拟化和半虚拟化
<br>

关于虚拟机，经典定义是：虚拟机是一个独立的隔离的计算环境，这种计算环境让使用者看起来就像在使用真实的物理机器一样。尽管我们可以在基于ARM的硬件平台上模拟真实硬件，但这通常不是有效的做法，因此我们常常不这么做。例如，模拟一个真实的以太网设备是非常慢的，这是因为对任何一个模拟寄存器的访问都会陷入到Hypervisor当中进行模拟。比起直接访问物理寄存器来说，这种操作的代价要昂贵得多。一个替代方案是修改客户操作系统，使之意识到自身运行在虚拟机当中，通过在Hypervisor中模拟一个虚拟设备来给客户机使用。以此来换取更好得I/O性能。严格来说，全虚拟化需要完全模拟真实硬件，性能上会比较差。开源项目Xen推进了半虚拟化，通过修改客户机操作系统的核心部分使其更适合在虚拟环境中运行，以此来提高性能。

另一个使用半虚拟化的原因是早期的体系结构并不是为虚拟化而设计的，存在虚拟化漏洞。因为虚拟化要求所有敏感指令或访问敏感资源的指令都能被截获模拟。对于存在虚拟化漏洞的体系结构，则需要通过半虚拟化的方案来填补漏洞。而今，大多数体系机构都支持硬件辅助虚拟化，包括Arm。这使得操作系统的核心部分无需修改也能获得较好得性能。只有少数存储和网络相关的I/O设备仍然采用半虚拟化的方案来改善性能，这类半虚拟化的方案如，virtio 和 Xen PV Bus。

<br>
## 1.4 虚拟机（VM）和虚拟CPU (vCPU)
<br>

非常有必要区分虚拟机（VM）和虚拟CPU(vCPU)。这有利于理解本文的后续部分。例如，一个内存页面可以被分配一个虚拟机，因此所有属于该VM的所有vCPUs都可以访问它。而一个虚拟中断只是针对某个vCPU，因此只有该vCPU可以收到。虚拟机（VM）和虚拟CPU(vCPU)的关系如图3所示。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/3 Virtual Machine and Virtual CPUs.png"/></div>
<p align="center">图3：VM vs vCPU</p>
<br>

**注意**：ARM体系结构有定义处理单元（Processing Element, PE）一词，现代CPU可能包含多个内核或线程，PE用来指代单一的执行单元。同样的这里的vCPU严格来说应该是vPE。

<br>

# 2 AArch64的虚拟化
<br>
对于ARMv8, Hypervisor运行在EL2异常级别。只有运行在EL2或更高异常级别的软件才可以访问并配置各项虚拟化功能。
- **Stage 2转换**
- **EL1/0指令和寄存器访问**
- **注入虚拟异常**

<br>
保护状态和非保护状态下的异常级别及可运行的软件如图4所示
<br>

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/4 Virtualization in AArch64.png"/></div>
<p align="center">图4：AArch64的虚拟化</p>
<br>

**注意**：保护状态的EL2用灰色显示是因为，保护状态的EL2并不总是可用，这是Armv8.4-A引入的特性。

<br>
## 2.1 Stage 2 转换
<br>
### 什么是Stage 2 转换
<br>

Stage 2 转换允许Hypervisor控制虚拟机的内存视图。具体来说，其可以控制虚拟机是否可以访问特定的某一块物理内存，以及其出现在虚拟机内存空间的位置。这种能力对于虚拟机的隔离和沙箱功能来说至关重要。这使得虚拟机只能看到分配给它自己的物理内存。为了支持Stage 2 转换， 需要增加一个页表，我们称之为Stage 2页表。操作系统控制的页表转换称之为stage 1转换，负责将虚拟机视角的虚拟地址转换为虚拟机视角的物理地址。而stage 2页表由Hypervisor控制，负责将虚拟机视角的物理地址转换为真实的物理地址。虚拟机视角的物理地址在Armv8有特定的词描述，叫中间物理地址(intermediate Physical Address, IPA)。

stage 2转换表的格式和stage 1的类似。但也有些属性的处理不太一样，例如,判断内存类型 是normal 还是 device的信息被直接编码进了表里，而不是通过查询MAIR_ELx寄存器。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/5 VA to IPA to PA address translation.png"/></div>
<p align="center">图5：地址转换, VA to IPA to PA</p>

<br>
### VMID
<br>
每一个虚拟机都被分配一个ID号，称之为VMID。这个ID号用于标记某个特定的TLB项属于哪一个VM。VMID使得不同的VM可以共享同一块TLB缓存。VMID存储在寄存器VTTBR_EL2中，可以是8或16比特，由VTCR_EL2.vs比特位控制，其中16比特的VMID支持是在armv8.1-A中扩展的，是可选的。需注意，EL2和EL3的地址转换不需要VMID标记，因为它们不需要stage 2转换。

<br>
### VMID vs ASID
<br>
TLB项也可以用ASID(Address Space Identifier)标记，每个应用都被操作系统分配有一个ASID，所有属于同一个应用的TLB项都有相同的ASID。这使得不同应用可以共享同一块TLB缓存。每一个VM有它自己的ASID空间。例如两个不同的VMs同时使用ASID 5，但指的是不同的东西。对于虚拟机而言，VMID结合ASID同时使用至关重要。

<br>
### 属性整合和覆盖
<br>
stage 1 和 stage 2映射都包含属性，例如存储类型，访问权限等。内存管理单元（MMU）会将两个阶段的属性整合成一个最终属性，整合的原则是选择更有限制的属性。且看如下例子：
<br>

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/6 Combining stage 1 and stage 2 attributes.png"/></div>
<p align="center">图6：映射属性整合</p>

在上面的例子中，Device属性比起Normal属性更具限制性，因此最终结果是Device属性。同样的原理，如果你将顺序调换一下也不会改变最终属性。

<br>
属性整合在大多数情况下都可用工作。但有些时候，例如在VM的早期启动阶段，Hypervisor希望改变一下默认的行为。可以通过如下寄存器比特来改变默认的正常行为。
<br>
- HCR_EL2.CD: 控制所有stage 1属性为Non-cacheable。
- HCR_EL2.DC：强制所有stage 1属性为Normal，Write-Back Cacheable。
- HCR_EL2.FWB (Armv8.4-A引入)：使用stage 2属性覆盖stage 1属性，而不是使用默认的限制性整合原则。

<br>
### 模拟MMIO
<br>

与物理机器的物理地址空间类似，VM的IPA地址空间包含了范围内存与外围设备的区域。如下图所示

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/7 Emulating IMMO new.png"/></div>
<p align="center">图7：模拟MMIO</p>

<br>
VM使用外围设备区域来访问真实的物理外围设备，这包含了直通设备和虚拟外围设备。虚拟设备完全由Hypervisor模拟，如下图所示

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/8 Stage 2 mappings for virtual and assigned peripherals.png"/></div>
<p align="center">图8：stage 2映射</p>

<br>

一个直通设备被直接分配给VM并映射到IPA地址空间，这使得VM中的软件可用直接访问真实的物理硬件。一个虚拟的外围设备由Hypervisor模拟，其stage 2的转换项被标记为fault。虽然VM中的软件看来其是直接与物理设备交互，但实际上这一访问会导致stage 2转换错误，从而进入相应的异常处理程序由Hypervisor模拟。

<br>
为了模拟一个外围设备，Hypervisor需要知道哪一个外围设备被访问，外围设备的哪一个寄存器被访问，是读访问还是写访问，访问长度是多少，以及使用哪些寄存器来传送数据。

<br>
当处理stage 1 faults时，FAR_ELx寄存器包含了触发异常的虚拟地址。但虚拟地址不是给Hypervisor用的，Hypervisor通常不会知道客户操作系统如何配置虚拟地址空间的映射。对于stage 2 faults，有一个专门的寄存器HPFAR_EL2，该寄存器会报告发生错误的IPA地址。IPA地址空间由Hypervisor控制，因此可用利用此寄存器里的信息来进行必要的模拟。

<br>
ESR_ELx寄存器用于报告发生异常的相关信息。当loads或stores一个通用寄存器触发stage 2 fault时，相关异常信息由这些寄存器提供。这些信息包含了，访问的长度，访问的原地址或目的地址。Hypervisor可以以此来判断对虚拟外围设备的访问权限。下图展示了一个 **陷入(trapping) -- 模拟(emulating)** 的访问过程。
<br>

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/9 Example of emulating an access to MMIO.png"/></div>
<p align="center">图9：外围设备模拟</p>

1. VM里的软件尝试访问虚拟外围设备，这个例子当中是虚拟UART的接收FIFO。
2. 该访问被stage 2转换block住，导致一个abort异常被路由到EL2。
   - 异常处理程序查询ESR_EL2关于异常的信息，如访问长度，目的寄存器，是load还是store操作。
   - 异常处理程序查询HPFAR_EL2，取得发生abort的IPA地址。
3. Hypervisor通过ESR_EL2和HPFAR_EL2里的相关信息对相关虚拟外围设备作模拟，模拟完成后通过ERET指令返回vCPU，并从发生异常的下一条指令继续执行。

<br>
### 系统内存管理单元(System Memory Management Units, SMMUs)

<br>
到目前为止，我们只考虑了从处理器发起的各种访问。我们还需要考虑其他主设备如DMA控制器发起的访问。我们需要一种方法来扩展stage 2映射以保护这些主设备的地址空间。如果一个DMA控制器没有使用虚拟化，那它看起来应该如下图所示

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/10 DMA controller that does not use virtualization.png"/></div>
<p align="center">图10：没有虚拟化的DMA访问</p>
<br>
DMA控制器通常由内核驱动编程控制。内核驱动会确保不违背操作系统层面的内存保护原则，也就是或一个应用不能使用DMA访问其没有权限访问的其他应用的内存。

<br>
下面让我们考虑操作系统运行在虚拟机中的场景。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/11 system with the OS running within a VM new.png"/></div>
<p align="center">图11：虚拟化下没有SMMU的DMA访问</p>
<br>

在这个系统中，Hyperviosr通过stage 2映射来隔离不同VMs的地址空间。这是基于Hypervisor控制的stage 2映射表实现的。而驱动则直接与DMA控制器交互，这会产生两个问题：

<br>
- **隔离**：DMA控制器访问在虚拟机之间没有了隔离，这破坏了虚拟机的沙箱功能
- **地址空间**： 利用两级映射转换，是内核以为PAs就是IPAs。但DMA控制仍然看到的是PAs。因此DMA控制器和内核看到的是不同的地址空间，为了解决这个问题，每当VM与DMA控制器交互时就需要陷入到Hypervisor中做必要的转换。这种处理方式是及其没有效率的，其容易出错。

<br>

解决的办法是将stage 2的机制推广到DMA控制器。这样的话这些主设备控制器也需要一个MMU，Armv8称之为SMMU（更常见的一个叫IOMMU）。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/12 System Memory Management Unit.png"/></div>
<p align="center">图12：虚拟化下通过SMMU的DMA访问</p>
<br>

Hypervisor负责设置SMMU，以使DMA控制器看到的地址空间与kenrel看到地址空间相同。这样就能解决上述两个问题。

<br>
## 2.2 指令的陷入与模拟
<br>
有时Hypervisor需要模拟一些操作，例如VM里运行的软件试图配置处理器的一些属性，如电源管理或是缓存一致性时。通常你不会允许VM直接配置这些属性，因为这会打破隔离性，从而影响其他VMs。这就需要通过以陷入的方式产生异常，从而在异常处理程序中做相应的模拟。Armv8包含一些陷入控制来帮助实现 **陷入(trapping) -- 模拟(emulating)**。如果对相应操作配置了陷入，则这种操作发生时会陷入到更高的异常级别，便于Hypervisor模拟。

<br>
举个例子，执行等待中断指令**WFI**通过会使CPU进入低功耗状态。然而，当配置HCR_EL2.TWI==1时，如果在EL0/EL1执行**WFI**则会导致EL2的异常。
（注：陷入不是为虚拟化而设计的，例如你会发现有陷入到EL3和EL1的异常，但异常对虚拟化实现至关重要。）

<br>
对于 **WFI**的例子里， 操作系统通过在一个idle loop里执行 **WFI**指令，但虚拟机中的操作系统执行该指令时，会陷入到Hypervisor里模拟，这时Hypervisor通常会调度另一个vCPU执行。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/13 Example of trapping WFIs from EL1 to EL2.png"/></div>
<p align="center">图13：WFI指令模拟</p>
<br>

## 2.3 寄存器的访问
<br>
**陷入 -- 模拟**的另一个用途是用来呈现虚拟寄存器的值。例如寄存器ID_AA64MMFR0_EL1是用来报告处理器内存相关特性的，操作系统可能会读取该寄存器来决定在内核中开启或关闭某些特性。Hypervisor可能会给VM呈现一个与实际物理寄存器不同的值。这是怎么实现的呢？首先Hypervisor需要开启对该寄存器读操作的陷入。然后，在陷入的异常处理中判断异常相关的信息并进行模拟，在这个例子中就是设置一个虚拟的值。最后ERET返回。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/14 Example of trapping and emulating an operation.png"/></div>
<p align="center">图14：寄存器访问的陷入模拟</p>
<br>

MIDR and MPIDR
<br>
**陷入 -- 模拟** 的开销是很大的。这种操作需要先陷入到EL2，然后由Hypervisor做相应模拟再返回客户操作系统。对于某些寄存器如 **ID_AA64MMFR0_EL1**，操作系统并不经常访问，**陷入 -- 模拟**的开销还是可以接受的。但对于某些经常访问的寄存器以及性能相关的代码，陷入太频繁的话会对系统性能造成很大影响。对于这些情况，我们需要尽可能地优化 **陷入**。

- **MIDR_EL1**: 存有处理器类型信息
- **MPIDR_EL1**：亲和性配置

Hypervisor可能希望在访问上述两个寄存器时不要总是陷入。对这些寄存器，Armv8提供了其对应地不需要陷入的版本。Hypervisor可以在进入VM时先配置好这些寄存器的值。当VM中读到 MIDR_EL1 / MPIDR_EL1时会自动返回VPIDR_EL2 / VMPIDR_EL2的值而不发生陷入。

- **VPIDR_EL2**：读取 **MIDR_EL1**返回 **VPIDR_EL2**的值避免陷入
- **VMPIDR_EL2**：读取 **MPIDR_EL1**返回 **VMPIDR_EL2**的值避免陷入

注意：VPIDR_EL2 / VMPIDR_EL2 在硬件reset后没有初始化的值，它们必须由软件启动代码初始化一个合理的值。

<br>
## 2.4 异常虚拟化

<br>
中断是硬件通知软件的机制，在一个使用虚拟化的系统中，中断处理会变得更为复杂。有些中断会由Hypervisor直接处理，有些中断被分配给了VM，需要由VM中的处理程序处理，并且还有可能在接收到这个中断时，对应的VM并没有被调度运行。这意味着我们不仅需要支持在EL2中直接处理中断，还需要一种机制能将收到的中断转发给相应VM的vCPU。Armv8提供了vIRQs, vFIQs, 和vSErrors来支持虚拟中断。这些中断的行为和物理中断（IRQs, FIQs, 和 SErrors）类似，只不过只有当系统运行在EL0/1是才会收到，运行在EL2/3是收不到虚拟中断的。

<br>
### 开启虚拟中断

<br>
虚拟中断也是根据中断类型控制的。为了发送虚拟中断到EL0/1, Hypervisor需要设置 **HCR_EL2**中相应的中断路由比特位。例如，开启vIRQ，你需要设置 **HCR_EL2.IMO**， 这意味着物理IRQ中断将被发送到EL2，同时虚拟中断将被发送到EL1。理论上，Armv8可以配置成VM直接接收物理FIQs和虚拟IRQs。但在实际应用中，通常只配置VM接收虚拟中断。

<br>
### 产生虚拟中断
<br>
有两种方式产生虚拟中断
<br>
1. 配置HCR_EL2，由内部CPU核产生
2. 使用GICv2及以上版本的外部中断控制器

<br>
我们先来看第一种机制，HCR_EL2中有如下的控制比特位
<br>
- **VI**: 配置vIRQ
- **VF**: 配置vFIQ
- **VSE**: 配置vSError
<br>
设置上述比特位等同于中断控制器向vCPU发送中断信号。和常规物理中断一样，虚拟中断受PSTATE控制。这种机制简单易用，但有个明显的缺点，需要由Hypervisor来模拟中断控制器的相关操作，一系列的 **陷入 -- 模拟**将带来性能上的开销。

<br>
第二种方式是使用Arm的通用中断控制器(Generic Interrupt Controller, GIC)来产生虚拟中断。从GICv2版本开始，GIC可以通过物理CPU interface 和 虚拟CPU interface发送物理中断和虚拟中断。见下图：

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/15 The GIC virtual and physical CPU interfaces.png"/></div>
<p align="center">图15：GIC中断发送</p>
<br>

这两个CPU interface是等同的，区别是一个发送物理中断信号，另一个发送虚拟中断信号。Hypervisor可以将虚拟CPU interface映射给VM，以便VM可以直接和GIC通信。这种方式的好处是Hypervisor只需建立映射，不需要做任何模拟，从而提升了性能。（**PS：虚拟化性能提升的关键就在优化陷入，减少次数，优化流程**）

<br>
### 中断转发给vCPU的例子
<br>
上面介绍了虚拟中断是如何开启和生产的。让我们来看一个中断转发给vCPU的例子。考虑一个物理外围设备，该设备被分配给了某个VM，如下图所示：

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/16 Example sequence for forwarding a virtual interrupt.png"/></div>
<p align="center">图16：虚拟中断转发的例子</p>
<br>
具体步骤如下：
<br>
1. 物理外围设备发送中断信号给GIC
2. GIC产生物理中断异常，可能是IRQ或FIQ。由于配置了HCR_EL2.IMO/FMO，这些异常会被路由到EL2。Hyperviosr发现该设备已被分配给了某个VM，于是检查需要将该中断信号需要转发给哪个vCPU。
3. Hypervisor配置了GIC将该物理中断以虚拟中断的形式转给某个vCPU。GIC于是发送vIRQ/vFIQ信号，如果此时还运行在EL2，这些信号会被忽略。
4. Hypervisor将控制权返还给vCPU。
5. 处理器运行在EL0或EL1，来自GIC的虚拟中断被接收（受PSTATE控制）

<br>
上面的例子展示了如何将一个物理中断以虚拟中断的形式转发给VM。如果是一个虚拟中断，Hypervisor可以直接注入虚拟中断，而不要将虚拟中断绑定到某个物理中断。

<br>
### 中断屏蔽
<br>

我们知道中断屏蔽比特位PSTATE.I, PSTATE.F,  PSTATE.A分别对应IRQs, FIQs和SErrors。如果运行在虚拟化环境中，这些比特位的工作方式由些许不同。

<br>
例如，对于IRQs，设置EL2.IMO意味着
- 物理IRQ路由至EL2
- 对EL0/EL1开启vIRQs
<br>

这同时也改变了PSTATE.I 屏蔽的含义， 当运行在EL0/EL1是，如果 HCR_E2.IMO==1, PSTATE.I针对的是虚拟的vIRQs而物理的pIRQs。

## 2.5 通用时钟虚拟化

<br>
<br>
...待续


<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-66555622-4"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'UA-66555622-4');
</script>
