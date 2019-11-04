---
layout: post
title:  "Armv8架构虚拟化介绍"
categories: Technology
tags: Arm Virtualization
author: Calinyara
description: 
---

# 综述

<br>
本文描述了Armv8-A AArch64的虚拟化支持。包括stage 2页表转换，虚拟异常，以及陷阱。本文介绍了一些基础的硬件辅助虚拟化理论以及一些Hypervisor如何利用这些虚拟化特性的例子，但文本不会讲述某一具体的Hypervisor软件是如何工作的以及如何开发一款Hypervisor软件。通过阅读本文，你会学到两种类型的Hypervisor以及它们是如何映射到Arm的异常级别。你将能解释陷阱是如何工作的以及其是如何被用来进行各种模拟操作。你将能描述Hypervisor可以产生什么虚拟异常以及产生这些虚拟异常的机制。看懂本文需要一定基础，本文假定你熟悉ARMv8体系结构的异常模型和内存管理。

<br>
# 虚拟化简介
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
## Hypervisor的两种类型
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
## 全虚拟化和半虚拟化
<br>

关于虚拟机，经典定义是：虚拟机是一个独立的隔离的计算环境，这种计算环境让使用者看起来就像在使用真实的物理机器一样。尽管我们可以在基于ARM的硬件平台上模拟真实硬件，但这通常不是有效的做法，因此我们常常不这么做。例如，模拟一个真实的以太网设备是非常慢的，这是因为对任何一个模拟寄存器的访问都会陷入到Hypervisor当中进行模拟。比起直接访问物理寄存器来说，这种操作的代价要昂贵得多。一个替代方案是修改客户操作系统，使之意识到自身运行在虚拟机当中，通过在Hypervisor中模拟一个虚拟设备来给客户机使用。以此来换取更好得I/O性能。严格来说，全虚拟化需要完全模拟真实硬件，性能上会比较差。开源项目Xen推进了半虚拟化，通过修改客户机操作系统的核心部分使其更适合在虚拟环境中运行，以此来提高性能。

另一个使用半虚拟化的原因是早期的体系结构并不是为虚拟化而设计的，存在虚拟化漏洞。因为虚拟化要求所有敏感指令或访问敏感资源的指令都能被截获模拟。对于存在虚拟化漏洞的体系结构，则需要通过半虚拟化的方案来填补漏洞。而今，大多数体系机构都支持硬件辅助虚拟化，包括Arm。这使得操作系统的核心部分无需修改也能获得较好得性能。只有少数存储和网络相关的I/O设备仍然采用半虚拟化的方案来改善性能，这类半虚拟化的方案如，virtio 和 Xen PV Bus。

<br>
## 虚拟机（VM）和虚拟CPU (vCPU)
<br>

非常有必要区分虚拟机（VM）和虚拟CPU(vCPU)。这有利于理解本文的后续部分。例如，一个内存页面可以被分配一个虚拟机，因此所有属于该VM的所有vCPUs都可以访问它。而一个虚拟中断只是针对某个vCPU，因此只有该vCPU可以收到。虚拟机（VM）和虚拟CPU(vCPU)的关系如图3所示。

<br>
<div align="center"><img src="/assets/images/armv8_virtualization/3 Virtual Machine and Virtual CPUs.png"/></div>
<p align="center">图3：VM vs vCPU</p>
<br>

**注意**：ARM体系结构有定义处理单元（Processing Element, PE）一词，现代CPU可能包含多个内核或线程，PE用来指代单一的执行单元。同样的这里的vCPU严格来说应该是vPE。

<br>

## AArch64的虚拟化
<br>
对于ARMv8, Hypervisor运行在EL2异常级别。只有运行在EL2或更高异常级别的软件才可以访问并配置各项虚拟化功能。
- **第二阶段页表转换**
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
## Stage 2 转换
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

...待续


<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-66555622-4"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'UA-66555622-4');
</script>
