---
layout: post
title:  "Armv8架构虚拟化介绍"
categories: Technology
tags: Arm Virtualization 虚拟机 虚拟化 体系结构
author: Calinyara
description: 
---

# 1 综述

<br>
本文描述了Armv8-A AArch64的虚拟化支持。包括stage 2页表转换，虚拟异常，以及陷阱。本文介绍了一些基础的硬件辅助虚拟化理论以及一些Hypervisor如何利用这些虚拟化特性的例子。文本不会讲述某一具体的Hypervisor软件是如何工作的以及如何开发一款Hypervisor软件（对具体实现感兴趣，请参考[aVisor: 基于ARM架构的Hypervisor及操作系统实现](https://calinyara.github.io/technology/2023/02/25/aVisor.html)）。通过阅读本文，你可以学到两种类型的Hypervisor以及它们是如何映射到Arm的异常级别。你将能解释陷阱是如何工作的以及其是如何被用来进行各种模拟操作。你将能描述Hypervisor可以产生什么虚拟异常以及产生这些虚拟异常的机制。理解本文内容需要一定基础，本文假定你熟悉ARMv8体系结构的异常模型和内存管理。

<br>
# 1.1 虚拟化简介
<br>
这里我们将介绍一些基础的Hypervisor和虚拟化的理论知识。如果你已经有一定的基础或是已经熟悉了这些概念，可以跳过这部分内容。我们用Hypervisor这个词来定义一种负责创建，管理以及调度虚拟机(Virtual Machines, VMs)的软件。

<br>
## 虚拟化为什么重要
<br>

虚拟化是一种在现代云计算和企业基础架构中广泛使用的技术。开发人员用虚拟机在一个硬件平台上运行多个不同的操作系统来开发和测试软件，以避免对主计算环境造成可能的破坏。虚拟化技术在服务器上非常流行，大多数面向服务器的处理器都需要支持虚拟化功能，这是因为虚拟化能给数据中心服务器带来如下一些需要的特性：

- **隔离**：利用虚拟化可以对同一个物理核上运行的虚拟机进行隔离。这使得相互间不可信的的计算环境可以共享同一套硬件环境。例如，两个竞争对手可以共享同一个物理机器而又不能访问对方的数据。
- **高可用性**： 虚拟化可以在不同的物理机器之间无缝透明地迁移负载。这个技术广泛用于将负载从出错的硬件平台迁移至其他可用平台，以便维护和替换出错的硬件而不影响服务。
- **负载均衡**： 为了降低数据中心硬件和功耗成本，需要尽可能充分地利用硬件平台资源。将负载均衡地迁移到不同地物理机上，有利用充分利用物理机资源，降低功耗，同时为租户提供最佳性能。
- **沙箱**：虚拟机可以作为一个沙箱来为运行在其中的应用屏蔽其他软件的干扰，或者避免其干扰其他软件。例如在虚拟机中运行特定软件，可以避免该软件的bug或病毒导致物理机器上的其他软件损坏。

<br>
## 1.2 Hypervisor的两种类型
<br>

Hypervisor通常被分成两种类型，独立类型Type 1和寄生类型 Type 2。我们先看看Type 2类型的Hypervisor。对于Type 2类型的Hypervisor，其寄生的宿主操作系统拥有对硬件平台和资源（包括CPU和物理内存...）的全部控制权。下图展示了Type 2类型的Hypervisor。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/1 Example of a hosted or Type 2 hypervisor.png"/></div>
<p align="center">图1：Type 2 Hypervisor</p>
<br>
宿主操作系统，指的是直接运行在硬件平台上并为Type 2类型的Hypervisor提供运行环境的操作系统。这类Hypervisor可以充分利用宿主操作系统对物理硬件的管理功能，而Hypervisor只需提供虚拟化相关功能即可。不知你是否使用过Virtual Box或是VMware Workstation, 这类软件就是Type 2类型的Hypervisor。

<br>
接下来，看看独立类型的Type 1 Hypervisor, 如图2。 这类Hypervisor没有宿主操作系统。其直接运行在物理硬件之上，直接管理各种物理资源，同时管理并运行客户机操作系统。
<br>

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/2 Example of a standalone or Type 1 hypervisor.png"/></div>
<p align="center">图2：Type 1 Hypervisor</p>
<br>
在开源社区常见的Hypervisor, Xen (Type 1) 和 KVM (Type 2)就分属这两种不同的类型。其他开源的或知识产权的Hypervisor，可参见 [WiKi](https://en.wikipedia.org/wiki/Comparison_of_platform_virtualization_software)。

<br>
## 1.3 全虚拟化和半虚拟化
<br>

关于虚拟机，经典定义是：虚拟机是一个独立的隔离的计算环境，这种计算环境让使用者看起来就像在使用真实的物理机器一样。尽管我们可以在基于ARM的硬件平台上模拟真实硬件，但这通常不是最有效的做法，因此我们常常不这么做。例如，模拟一个真实的以太网设备是非常慢的，这是因为对任何一个模拟寄存器的访问都会陷入到Hypervisor当中进行模拟。比起直接访问物理寄存器来说，这种操作的代价要昂贵得多。一个替代方案是修改客户操作系统，使之意识到自身运行在虚拟机当中，通过在Hypervisor中模拟一个虚拟设备来给客户机使用。以此来换取更好得I/O性能。严格来说，全虚拟化需要完全模拟真实硬件，性能上会比较差。开源项目Xen推进了半虚拟化，通过修改客户机操作系统的核心部分使其更适合在虚拟环境中运行，以此来提高性能。

另一个使用半虚拟化的原因是早期的体系结构并不是为虚拟化而设计的，存在虚拟化漏洞。因为虚拟化要求所有敏感指令或访问敏感资源的指令都能被截获模拟。对于存在虚拟化漏洞的体系结构，则需要通过半虚拟化的方案来填补漏洞。而今，大多数体系机构都支持硬件辅助虚拟化，包括Arm。这使得操作系统的核心部分无需修改也能获得较好得性能。只有少数存储和网络相关的I/O设备仍然采用半虚拟化的方案来改善性能，这类半虚拟化的方案如，virtio 和 Xen PV Bus。

<br>
## 1.4 虚拟机（VM）和虚拟CPU (vCPU)
<br>

有必要区分虚拟机（VM）和虚拟CPU(vCPU)。这有利于理解本文的后续部分。例如，一个内存页面可以分配给一个虚拟机，因此所有属于该VM的vCPUs都可以访问它。而一个虚拟中断只是针对某个vCPU，因此只有该vCPU可以收到。虚拟机（VM）和虚拟CPU(vCPU)的关系如图3所示。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/3 Virtual Machine and Virtual CPUs.png"/></div>
<p align="center">图3：VM vs vCPU</p>
<br>

**注意**：ARM体系结构定义了处理单元（Processing Element, PE）一词，现代CPU可能包含多个内核或线程，PE用来指代单一的执行单元。同样的这里的vCPU严格来说应该是vPE。

<br>

# 2 AArch64的虚拟化
<br>
对于ARMv8, Hypervisor运行在EL2异常级别。只有运行在EL2或更高异常级别的软件才可以访问并配置各项虚拟化功能。
- **Stage 2转换**
- **EL1/0指令和寄存器访问**
- **注入虚拟异常**

<br>
安全状态和非安全状态下的异常级别及可运行的软件如图4所示
<br>

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/4 Virtualization in AArch64.png"/></div>
<p align="center">图4：AArch64的虚拟化</p>
<br>

**注意**：安全状态的EL2用灰色显示是因为，安全状态的EL2并不总是可用，这是Armv8.4-A引入的特性。

<br>
## 2.1 Stage 2 转换
<br>
### 什么是Stage 2 转换
<br>

Stage 2 转换允许Hypervisor控制虚拟机的内存视图。具体来说，其可以控制虚拟机是否可以访问特定的某一块物理内存，以及该内存块出现在虚拟机内存空间的位置。这种能力对于虚拟机的隔离和沙箱功能来说至关重要。这使得虚拟机只能看到分配给它自己的物理内存。为了支持Stage 2 转换， 需要增加一个页表，我们称之为Stage 2页表。操作系统控制的页表转换称之为stage 1转换，负责将虚拟机视角的虚拟地址转换为虚拟机视角的物理地址。而stage 2页表由Hypervisor控制，负责将虚拟机视角的物理地址转换为真实的物理地址。虚拟机视角的物理地址在Armv8中有特定的词描述，叫中间物理地址(intermediate Physical Address, IPA)。

stage 2转换表的格式和stage 1的类似，但也有些属性的处理不太一样，例如,判断内存类型 是normal 还是 device的信息被直接编码进了表里，而不是通过查询MAIR_ELx寄存器。

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
TLB项也可以用ASID(Address Space Identifier)标记，每个应用都被操作系统分配有一个ASID，所有属于同一个应用的TLB项都有相同的ASID。这使得不同应用可以共享同一块TLB缓存。每一个VM有它自己的ASID空间。例如两个不同的VMs同时使用ASID 5，但指的是不同的东西。对于虚拟机而言，通常VMID会结合ASID同时使用。

<br>
### 属性整合和覆盖
<br>
stage 1 和 stage 2映射都包含属性，例如存储类型，访问权限等。内存管理单元（MMU）会将两个阶段的属性整合成一个最终属性，整合的原则是选择更有限制的属性。且看如下例子：
<br>

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/6 Combining stage 1 and stage 2 attributes.png"/></div>
<p align="center">图6：映射属性整合</p>
在上面的例子中，Device属性比起Normal属性更具限制性，因此最终结果是Device属性。同样的原理，如果你将顺序调换一下也不会改变最终的属性。

<br>
属性整合在大多数情况下都可以工作。但有些时候，例如在VM的早期启动阶段，Hypervisor希望改变默认的行为，则可以通过如下寄存器比特来实现。
<br>
- HCR_EL2.CD: 控制所有stage 1属性为Non-cacheable。
- HCR_EL2.DC：强制所有stage 1属性为Normal，Write-Back Cacheable。
- HCR_EL2.FWB (Armv8.4-A引入)：使用stage 2属性覆盖stage 1属性，而不是使用默认的限制性整合原则。

<br>
### 模拟MMIO
<br>

与物理机器的物理地址空间类似，VM的IPA地址空间包含了内存与外围设备两种区域。如下图所示

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/7 Emulating IMMO new.png"/></div>
<p align="center">图7：模拟MMIO</p>
<br>
VM使用外围设备区域来访问其看到的物理外围设备，这其中包含了直通设备和虚拟外围设备。虚拟设备完全由Hypervisor模拟，如下图所示

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/8 Stage 2 mappings for virtual and assigned peripherals.png"/></div>
<p align="center">图8：stage 2映射</p>
<br>

一个直通设备被直接分配给VM并映射到IPA地址空间，这使得VM中的软件可用直接访问真实的物理硬件。一个虚拟的外围设备由Hypervisor模拟，其stage 2的转换项被标记为fault。虽然VM中的软件看来其是直接与物理设备交互，但实际上这一访问会导致stage 2转换fault，从而进入相应的异常处理程序由Hypervisor模拟。

<br>
为了模拟一个外围设备，Hypervisor需要知道哪一个外围设备被访问，外围设备的哪一个寄存器被访问，是读访问还是写访问，访问长度是多少，以及使用哪些寄存器来传送数据。

<br>
当处理stage 1 faults时，FAR_ELx寄存器包含了触发异常的虚拟地址。但虚拟地址不是给Hypervisor用的，Hypervisor通常不会知道客户操作系统如何配置虚拟地址空间的映射。对于stage 2 faults，有一个专门的寄存器HPFAR_EL2，该寄存器会报告发生错误的IPA地址。IPA地址空间由Hypervisor控制，因此可用利用此寄存器里的信息来进行必要的模拟。

<br>
ESR_ELx寄存器用于报告发生异常的相关信息。当loads或stores一个通用寄存器触发stage 2 fault时，相关异常信息由这些寄存器提供。这些信息包含了，访问的长度，访问的原地址或目的地址。Hypervisor可以以此来判断对虚拟外围设备访问的权限。下图展示了一个 **陷入(trapping) -- 模拟(emulating)** 的访问过程。
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
DMA控制器通常由内核驱动编程控制。内核驱动会确保不违背操作系统层面的内存保护原则，即一个应用不能使用DMA访问其没有权限访问的其他应用的内存。

<br>
下面让我们考虑操作系统运行在虚拟机中的场景。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/11 system with the OS running within a VM new.png"/></div>
<p align="center">图11：虚拟化下没有SMMU的DMA访问</p>
<br>

在这个系统中，Hyperviosr通过stage 2映射来隔离不同VMs的地址空间。这是基于Hypervisor控制的stage 2映射表实现的。而驱动则直接与DMA控制器交互，这会产生两个问题：

<br>
- **隔离**：DMA控制器访问在虚拟机之间没有了隔离，这破坏了虚拟机的沙箱功能。
- **地址空间**： 利用两级映射转换，使内核看到的PAs实际上是IPAs。但DMA控制器看到的仍然是PAs。因此DMA控制器和内核看到的是不同的地址空间，为了解决这个问题，每当VM与DMA控制器交互时就需要陷入到Hypervisor中做必要的转换。这种处理方式是极其没有效率的，且容易出错。

<br>

解决的办法是将stage 2的机制推广到DMA控制器。这么做的话，这些主设备控制器也需要一个MMU，Armv8称之为SMMU（通常也称为IOMMU）。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/12 System Memory Management Unit.png"/></div>
<p align="center">图12：虚拟化下通过SMMU的DMA访问</p>
<br>

Hypervisor负责配置SMMU，以使DMA控制器看到的物理地址空间与kenrel看到的物理地址空间相同。这样就能解决上述两个问题。

<br>
## 2.2 指令的陷入与模拟
<br>
有时Hypervisor需要模拟一些操作，例如VM里运行的软件试图配置处理器的一些属性，如电源管理或是缓存一致性时。通常你不会允许VM直接配置这些属性，因为这会打破隔离性，从而影响其他VMs。这就需要通过以陷入的方式产生异常，在异常处理程序中做相应的模拟。Armv8包含一些陷入控制来帮助实现 **陷入(trapping) -- 模拟(emulating)**。如果对相应操作配置了陷入，则这种操作发生时会陷入到更高的异常级别，便于Hypervisor模拟。

<br>
举个例子，执行等待中断指令**WFI**通过会使CPU进入低功耗状态。然而，当配置HCR_EL2.TWI==1时，如果在EL0/EL1执行**WFI**则会导致EL2的异常。
（**注**：陷入不是为虚拟化而设计的，有陷入到EL3和EL1的异常，但异常对虚拟化实现至关重要。）

<br>
对于 **WFI**的例子里， 操作系统通过在一个idle loop里执行 **WFI**指令，但虚拟机中的操作系统执行该指令时，会陷入到Hypervisor里模拟，这时Hypervisor通常会调度另一个vCPU执行。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/13 Example of trapping WFIs from EL1 to EL2.png"/></div>
<p align="center">图13：WFI指令模拟</p>
<br>

## 2.3 寄存器的访问
<br>
**陷入 -- 模拟**的另一个用途是用来呈现虚拟寄存器的值。例如寄存器ID_AA64MMFR0_EL1是用来报告处理器内存相关特性的，操作系统可能会读取该寄存器来决定在内核中开启或关闭某些特性。Hypervisor可能会给VM呈现一个与实际物理寄存器不同的值。这是怎么实现的呢？首先Hypervisor需要开启对该寄存器读操作的陷入。然后，在陷入的异常处理中判断异常相关的信息并进行模拟。在如下例子中，就是设置一个虚拟的值，然后ERET返回。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/14 Example of trapping and emulating an operation.png"/></div>
<p align="center">图14：寄存器访问的陷入模拟</p>
<br>

### 避免陷入
<br>
**陷入 -- 模拟** 的开销是很大的。这种操作需要先陷入到EL2，然后由Hypervisor做相应模拟再返回客户操作系统。对于某些寄存器如 **ID_AA64MMFR0_EL1**，操作系统并不经常访问，**陷入 -- 模拟**的开销还是可以接受的。但对于某些经常访问的寄存器以及性能敏感的代码，陷入太频繁会对系统性能造成很大影响。对于这些情况，我们需要尽可能地优化 **陷入**。

- **MIDR_EL1**: 存有处理器类型信息
- **MPIDR_EL1**：亲和性配置

Hypervisor可能希望在访问上述两个寄存器时不要总是陷入。对这些寄存器，Armv8提供了与其对应的不需要陷入的版本。Hypervisor可以在进入VM 时先配置好这些寄存器的值。当VM中读到 MIDR_EL1 / MPIDR_EL1时会自动返回VPIDR_EL2 / VMPIDR_EL2的值而不发生陷入。

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
虚拟中断也是根据中断类型控制的。为了发送虚拟中断到EL0/1, Hypervisor需要设置 **HCR_EL2**中相应的中断路由比特位。例如，开启vIRQ，你需要设置 **HCR_EL2.IMO**， 这意味着物理IRQ中断将被发送到EL2，同时虚拟中断将被发送到EL1。理论上，Armv8可以配置成VM直接接收物理FIQs和虚拟IRQs。但在实际应用中，通常配置VM只接收虚拟中断。

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
1. 物理外围设备发送中断信号给GIC。
2. GIC产生物理中断异常，可能是IRQ或FIQ。由于配置了HCR_EL2.IMO/FMO，这些异常会被路由到EL2。Hyperviosr发现该设备已被分配给了某个VM，于是检查需要将该中断信号转发给哪个vCPU。
3. Hypervisor配置了GIC将该物理中断以虚拟中断的形式转给某个vCPU。GIC于是发送vIRQ/vFIQ信号，如果此时还运行在EL2，这些信号会被忽略。
4. Hypervisor将控制权返还给vCPU。
5. 处理器运行在EL0或EL1，来自GIC的虚拟中断被接收（受PSTATE控制）。

<br>
上面的例子展示了如何将一个物理中断以虚拟中断的形式转发给VM。如果是一个没有物理中断对应的纯虚拟中断，Hypervisor可以直接注入虚拟中断。

<br>
### 中断屏蔽
<br>

我们知道中断屏蔽比特位PSTATE.I, PSTATE.F,  PSTATE.A分别对应IRQs, FIQs和SErrors。如果运行在虚拟化环境中，这些比特位的工作方式有些许不同。

<br>
例如，对于IRQs，设置HCR_EL2.IMO意味着
- 物理IRQ路由至EL2
- 对EL0/EL1开启vIRQs
<br>

这同时也改变了PSTATE.I 屏蔽的含义， 当运行在EL0/EL1是，如果 HCR_E2.IMO==1, PSTATE.I针对的是虚拟的vIRQs而非物理的pIRQs。

<br>
## 2.5 时钟虚拟化
<br>
Arm体系结构中，每个处理器上都有一组通用时钟。通用时钟由一组比较器组成，用来与系统计数器比较。当比较器的值小于等于系统计数器时便会产生时钟中断。在下图中，我们可以看到系统中通用时钟由黄色框部分组成。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/17 System counter module and per-core comparators.png"/></div>
<p align="center">图17：通用时钟比较器与系统计数模块</p>
<br>

下图展示了虚拟化系统中运行两个vCPU的时序。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/18 system with a hypervisor that hosts two vCPUs.png"/></div>
<p align="center">图18：Hypervisor运行有两个vCPU的时序</p>
<br>

物理世界的时间（墙上时间）4ms里，每个vCPU各运行了2ms。如果我们设置vCPU0的比较器在T=0之后的3ms产生一个中断，那么你希望实际在哪个墙上时间点产生中断呢？是vCPU0的虚拟时间的2ms，也就是墙上时间3ms那个点还是
vCPU0虚拟时间3ms的那个点？

<br>

实际上，Arm体系结构同时支持上述两种设置，这取决于你使用何种虚拟化方案。让我们看看这是如何实现的。

<br>

运行在vCPU上的软件可以访问如下两种时钟
<br>
- EL1物理时钟
- EL1虚拟时钟

<br>
EL1物理时钟会与系统计数器模块直接比较，使用的是绝对的墙上时间。而EL1虚拟时钟与虚拟计数器比较。虚拟计数器是在物理计数器的基础上减去一个偏移。Hypervisor负责为当前调度运行的vCPU指定对应的偏移寄存器。这种方式使得虚拟时间只会覆盖vCPU实际运行的那部分时间。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/19 How the virtual counter is generated.png"/></div>
<p align="center">图19：虚拟计数器的计算</p>
<br>

下图展示了虚拟时间运作的原理

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/20 Example of using Virtual Timer.png"/></div>
<p align="center">图20：虚拟时间原理</p>
<br>

在一个6ms的时段里，每个vCPU分别运行了3ms。Hypervisor可以使用偏移寄存器来将vCPU的时间调整为其实际运行的时间。

<br>

## 2.6 虚拟化主机扩展（Virtualization Host Extensions, VHE)
<br>

图21显示了一个Type 1类型的虚拟化系统的软件栈与异常级别的对应关系，Hypervisor部分运行在EL2，VMs运行在EL0/1。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/21 Standalone hypervisor with Armv8-A Exception levels.png"/></div>
<p align="center">图21：Type 1虚拟化系统软件栈与异常级别</p>
<br>

然而，对于一个Type 2类型的系统，其软件栈与异常级别的对应关系可能如图22所示

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/22 Hosted hypervisor pre VHE.png"/></div>
<p align="center">图22：VHE之前的Type 2虚拟化系统软件栈与异常级别</p>
<br>

通常，寄主操作系统的内核部分运行在EL1，控制虚拟化的部分运行在EL2。然而，这种设计有一个明显的问题。VHE之前的Hypervisor通常需要设计成high-visor和low-visor两部分，前者运行在EL1，后者运行在EL2。分层设计在系统运行时会造成很多不必要的上下文切换，带来不少设计上的复杂性和性能开销。为了解决这个问题，虚拟化主机扩展 （Virtualization Host Extensions, VHE）应运而生。该特性由Armv8.1-A引入，可以让寄主操作系统的内核部分直接运行在EL2上。

<br>
### 将主机操作系统运行在EL2
<br>

VHE由系统寄存器 **HCR_EL2**中的两个比特位控制
<br>
- **E2H**：VHE使能位
- **TGE**：当VHE使能时，控制EL0是Guest还是Host
<br>

```
|         Running in        | E2H | TGE |
|---------------------------|-----|-----|
|Guest kernel (EL1)         |  1  |  0  |
|Guest application (EL0)    |  1  |  0  | 
|Host kernel (EL2)          |  1  |  1* |
|Host application (EL0)     |  1  |  1  |
```
**\*** 当发生异常从VM退出到Hypervisor时，TGE将会初始化为0，软件需要先设置这一比特，再继续运行host kernel的主代码

<br>
一个典型的配置如下图
<br>
<div align="center"><img src="/assets/images/armv8_virtualization/23 E2H and TGE combinations.png"/></div>
<p align="center">图23：E2H与TGE配置</p>
<br>

### 虚拟地址空间
<br>
在VHE引入之前，EL0/1的虚拟地址空间看起来如下。EL0/1分两块区域，上面是内核空间，下面是用户空间。EL2只有一个空间，Hypervisor通常不需要运行应用，因此没有必要划分内核与用户空间。同理，EL0/1虚拟地址空间支持ASID，但EL2不需要支持。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/24 The EL01 and EL2 virtual address spaces pre VHE.png"/></div>
<p align="center">图24：VHE之前的虚拟地址空间</p>
<br>

当VHE引入之后，EL2可以直接运行操作系统代码。因此需要将地址空间划分和ASID的支持添加进来。同样，通过设置 **HCR_EL2.E2H**来解决。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/25 EL02 virtual address space when E2H 1.png"/></div>
<p align="center">图25：开启E2H时的EL2虚拟地址空间</p>
<br>

当运行在EL0时，HCR_EL2.TGE控制使用EL1还是EL2空间，当应用运行在Guest OS (TGE==0)为前者，运行在Host OS（TGE==1）为后者。

<br>
### 重定向寄存器访问
<br>

除了会使用不同的地址空间映射，VHE还有一个问题需要解决，那就寄存器访问。运行在EL2的内核仍然会尝试访问*_EL1的寄存器。为了运行无需修改的内核，我们需要将EL1的寄存器重定向到EL2。当你设置E2H后，这一切就会由硬件实现。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/26 The effect of E2H on system registers access at EL2.png"/></div>
<p align="center">图26：E2H对系统寄存器访问的影响</p>
<br>

但是，重定向又会带来一个新的问题，那就是Hypervisor完全可能在某些情况下，例如当执行任务切换时， 访问真正EL1的寄存器。为了解决这个问题，Arm架构引入了一种新的别名机制，以_EL12或_EL02结尾。如下例，就可以在ECH==1的EL2访问TTBR0_EL1。

<br>
<br>
<div align="center"><img src="/assets/images/armv8_virtualization/27 Access EL1 registers from EL2 when E2H 1.png"/></div>
<p align="center">图27：从EL2访问EL1寄存器</p>
<br>

### 异常

通常系统寄存器 **HCR_EL2.IMO/FMO/AMO**的这几个比特位可以用来控制物理异常被路由至EL1或EL2。当运行在EL0且TGE==1时，HCR_EL2路由比特将会被忽略，所有物理异常（除了那些由SCR_EL3控制的会被路由至EL3）全部路由到EL2。这是因为Host OS里运行的应用是Host OS的一部分，而Host OS运行在EL2。

<br>
## 2.7 嵌套虚拟化
<br>

Hypervisor可以运行在VM中，这称之为嵌套虚拟化。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/28 Nested virtualization.png"/></div>
<p align="center">图28：嵌套虚拟化</p>
<br>

我们将第一个Hypervisor称为Host Hypervisor，VM中运行的Hypervisor称为Guest Hypervisor。

<br>
在Armv8.3-A之前，Guest Hypervisor可以运行在EL0。但这种设计需要大量软件模拟，不仅软件开发困难，性能也很差。Armv8.3-A增加了一些新的特性，可以让Guest Hypervisor运行在EL1。而Armv8.4-A引入的一些新特性，使得这一过程更有效率，虽然仍然需要Host Hypervisor参与做一些额外的工作。

<br>
### Guest Hypervisor访问虚拟化控制接口
<br>

我们并不希望Guest Hypervisor能直接访问虚拟化控制接口，因为这么做会破坏VM的沙箱机制，使得虚拟机能够看到Host平台的信息。当Guest Hypervisor运行在EL1，并访问虚拟化控制接口时，**HCR_EL2**中新的控制比特位可以使这些操作陷入到Host Hypervisor(EL2)以便模拟。

<br>

- **HCR_EL2.NV**：开启硬件辅助嵌套虚拟化
- **HCR_EL2.NV1**：开启额外需要陷入的操作
- **HCR_EL2.NV2**：开启重定向到内存
- **VNCR_EL2**：当NV2==1时，指向一个内存中的结构体

Armv8.3-A添加了NV和NV1控制比特。在此之前，从EL1访问*_EL2寄存器时的行为是未定义的，通常是会产生一个EL1的异常。而控制比特NV和NV1使得这种访问可以被陷入到EL2。这就使得Guest Hypervisor可以运行在EL1，同时由运行在EL2的Host Hypervisor来模拟这些操作。NV还会导致EL1运行ERET陷入到EL2。

<br>
下图展示了Guest Hypervisor如何创建并启动虚拟机

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/29 Guest Hypervisor setting up and entering a VM.png"/></div>
<p align="center">图29：Guest Hypervisor创建并启动虚拟机</p>
<br>

1. 从EL1访问*_EL2寄存器将导致Guest Hypervisor陷入到EL2。Host Hypervisor记录Guest Hypervisor创建的相关配置。
2. Guest Hypervisor尝试进入其创建的虚拟机，此时ERET指令会陷入到EL2。
3. Host Hypervisor根据Guest Hypervisor的配置，设置相关寄存器以便启动VM，清理掉NV比特位，最后进入Guest Hypervisor创建的Guest运行。

<br>

按上述的方法， 在Guest Hypervisor访问任何一个\*_EL2寄存器时都会发生陷入。切换操作如 任务切换，vCPU切换，VMs切换都会访问大量寄存器，每次陷入都会导致异常的进入与返回，从而带来严重的 **陷入 -- 模拟**性能问题。（回忆前面的内容， **虚拟化性能提升的关键就在优化陷入，减少次数，优化流程**）。Armv8.4-A提供了一个更好的方案，当NV2被设置时，从EL1访问\*_EL2寄存器将会被重定向到一块内存区域。Guest Hypervisor可以多次读写这块寄存器区域而不发生陷入。只有当最后运行ERET时，才会陷入到EL2。而后，Host Hypervisor可以从该内存区域中提取相关配置并代Guest Hypervisor执行相关操作。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/30 ERET still traps to EL2.png"/></div>
<p align="center">图30：Guest Hypervisor创建并启动虚拟机优化</p>
<br>

1.  从EL1访问*_EL2寄存器将会被重定向到一块内存区域，该内存区域的地址由Host Hypervisor在 **VNCR_EL2**中指定。
2.  Guest Hypervisor尝试进入其创建的虚拟机，此时ERET指令会陷入到EL2
3.  Host Hypervisor从内存中提取配置信息，设置相关寄存器，以便启动VM，清理掉NV比特位，最后进入Guest Hypervisor创建的Guest运行。

<br>
这个改进方法相比之前的方法会减少陷入到Host Hypervisor的次数，从而提升了性能。

<br>
## 2.8 安全世界虚拟化
<br>

虚拟化扩展最早是在Armv7-A引入的。在Armv7-A中的Hyp模式等同于AArch32中的EL2，仅仅在非安全世界才存在。作为一个可选特性，Armv8.4-A增加了安全世界下EL2的支持。支持安全世界EL2的处理器，需配置EL3下的SCR_EL3.EEL2比特位来开启这一特性。设置了这一比特位，才允许使用安全状态下的虚拟化功能。

<br>
在安全世界虚拟化之前，EL3通常用于运行安全状态切换软件和平台固件。然而从设计上来说，我们希望EL3中运行的软件越少越好，因为越简单才会更安全。安全状态虚拟化使得我们可以将平台固件移到EL1中运行，由虚拟化来隔离平台固件和可信操作系统内核。下图展示了这一理念

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/31 Secure virtualization.png"/></div>
<p align="center">图31：安全世界的虚拟化</p>
<br>

### 安全EL2与两个IPA空间
<br>

Arm体系结构定义了安全世界和非安全世界两个物理地址空间。在非安全状态下，stage 1转换的的输出总是非安全的，因此只需要一个IPA空间来给stage 2使用。然而，对于安全世界，stage 1的输出可能是安全的也能是非安全的。Stage 1转换表中的NS比特位控制使用安全地址还是非安全地址。这意味着在安全世界，需要两个IPA地址空间。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/32 IPA spaces in Secure state.png"/></div>
<p align="center">图32：安全世界的IPA地址空间</p>
<br>

与stage 1表不同，stage 2转换表中没有NS比特位。因为对于一个特定的IPA空间，要么全都是安全地址，要么全都是非安全的，因此只需要由一个寄存器比特位来确定IPA空间。通常来说，非安全地址经过stage 2转换仍然是非安全地址，安全地址经过stage 2转换仍然是安全地址。

<br>

<br>
# 3 虚拟化的损耗
<br>

虚拟化的损耗主要在于虚拟机和Hypervisor切换需要保存和恢复寄存器。Armv8系统中，最少需要对如下寄存器做处理

- 31 x 64-bit通用寄存器(x0...x30)
- 32 x 128-bit浮点/SIMD寄存器(V0...V31)
- 两个栈寄存器(SP_EL0, SP_EL1)
  

使用LDP和STP指令，Hypervisor需要运行33条指令来存储和恢复这些寄存器。虚拟化最终的损耗不仅取决于硬件还取决于Hypervisor的设计。


<br>


<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-66555622-4"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'UA-66555622-4');
</script>


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-27WH7FZ7KT"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-27WH7FZ7KT');
</script>