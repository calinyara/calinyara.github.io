---
layout: post
title:  "ARM通用中断控制器GICv3与GICv4虚拟化"
categories: Technology
tags: arm zh arm64 gic gicv3 gicv4 中断 中断控制器 ITS LPI SPI PPI MSI SGI 虚拟化 virtualization 中断虚拟化
author: Calinyara
description:
---

<br>

# 1 简介

<br>

本文介绍GICv3和GICv4的中断虚拟化支持。包括Hypervisor用来生成和管理虚拟中断的控制接口。

<br>

Armv8-A 包含了虚拟化支持。为了配合这一功能，GICv3 也支持虚拟化。GICv3 为虚拟化添加了以下功能：

- CPU 接口寄存器的硬件虚拟化。
- 生成和发出虚拟中断的能力。
- Maintenance interrupts，用于通知管理软件（如 hypervisor）有关虚拟机内特定事件的信息。

<br>

***注：GIC 架构并未提供用于虚拟化 GICD、GICR 以及 ITS 的功能。这些接口的虚拟化必须由软件来处理。***

<br>

## 1.1 术语

<br>

Hypervisor 创建、控制和调度虚拟机（VM）。虚拟机在功能上等同于物理系统，其中包含一个或多个虚拟处理器。每个虚拟处理器包含一个或多个虚拟 PEs（vPEs）。

<br>

![Untitled](/assets/images/20240322-arm-gic-virt/0.png)

<br>

GICv3.x 和 GICv4.1 中的虚拟化支持工作在 vPEs层面上。例如，当创建一个虚拟中断时，它是针对特定的 vPE，而不是一个虚拟机。通常情况下，GIC 不知道不同的 vPE 与虚拟机之间的关系。

本文使用术语 Hypervisor 指代任何在 EL2 运行的软件，其负责管理 vPE。我们将忽略虚拟化软件之间可能存在的差异，而专注于 GIC 中的功能。请注意，并非所有的虚拟化解决方案都会使用 GIC 中提供的所有功能。

一个给定的 vPE 可以被描述为“scheduled”（已调度）或“not-scheduled”（未调度）。一个已调度的 vPE 指由 Hypervisor 调度到物理 PE（pPE）并正在运行。系统中可能包含多于 pPE个数的vPE 。未被 Hypervisor 调度的 vPE 不在运行状态，因此当前无法接收中断。

<br>

# 2 GICv3 虚拟化

<br>

本节概述了 GICv3 中对虚拟化的支持。 GICv3虚拟化与 GICv2 中首次引入的虚拟化支持类似，主要涉及 CPU 接口。它允许向 正在pPE 上调度的 vPE 发送虚拟中断信号。

<br>

## 2.1 接口

<br>

CPU 接口寄存器分为三组：

- `ICC`：物理CPU 接口寄存器
- `ICH`：虚拟化控制寄存器
- `ICV`：虚拟CPU 接口寄存器

<br>

下图显示了三组CPU接口寄存器：

<br>

![Untitled](/assets/images/20240322-arm-gic-virt/1.png)

<br>

### 2.1.1 ICC 物理CPU接口寄存器

<br>

这些寄存器的名称格式为`ICC_*_ELx`。在 EL2 上执行的Hypervisor使用常规 `ICC_*_ELx` 寄存器来处理物理中断。

<br>

### 2.1.2 ICH 虚拟化控制寄存器

<br>

这些寄存器的命名格式为 `ICH_*_EL2`。Hypervisor 可以访问这些寄存器来控制体系结构提供的虚拟化功能。这些功能包括：

- 启用和禁用虚拟 CPU 接口。
- 访问虚拟寄存器状态以启用上下文切换。
- 配置maintenance中断。
- 控制当前已调度的 vPE 的虚拟中断。

<br>

这些寄存器控制它们对应物理 PE 的虚拟化功能，但无法访问另一个 PE 的状态。即PE X上的软件无法控制 PE Y的状态。

<br>

### 2.1.3 ICV 虚拟 CPU接口寄存器

<br>

这些寄存器的命名格式为 `ICV_*_*EL1`。*在虚拟化环境中执行的软件使用 `ICV_*_EL1` 寄存器来处理虚拟中断。这些寄存器与相应的 `ICC_*_EL1`物理寄存器具有相同的格式和功能。ICV 和 ICC 寄存器具有相同的指令编码。在 EL2 和 EL3，总是访问 ICC 寄存器。在 EL1，`HCR_EL2`中的路由比特位决定是访问 ICC 还是 ICV 寄存器。

<br>

ICV 寄存器分为三组：

**Group 0**

- 用于处理Group 0中断的寄存器，例如 `ICC_IAR0_EL1` 和 `ICV_IAR0_EL1`。当 `HCR_EL2.FMO==1` 时，在 EL1 访问 ICV 寄存器而不是 ICC 寄存器。

**Group 1**

- 用于处理Group 1中断的寄存器，例如 `ICC_IAR1_EL1` 和 `ICV_IAR1_EL1`。当 `HCR_EL2.IMO==1` 时，在 EL1 访问 ICV 寄存器而不是 ICC 寄存器。

**Common**

- 处理 Group 0和 Group 1中断的公共寄存器，例如 `ICC_DIR_EL1` 和 `ICV_DIR_EL1`。当 `HCR_EL2.IMO==1` 或 `HCR_EL2.FMO==1` 时，在 EL1 访问 ICV 寄存器而不是 ICC 寄存器。

*注：ICV 寄存器是否在安全的 EL1 中使用取决于是否启用了安全虚拟化。*

<br>

下图显示了相同指令如何根据 `HCR_EL2` 路由控制来访问 ICC 或 ICV 寄存器。

<br>

![Untitled](/assets/images/20240322-arm-gic-virt/2.png)

<br>

## 2.2 管理虚拟中断

<br>

Hypervisor可以使用列表寄存器 `ICH_LR<n>_EL2` 为当前调度的 vPE 生成虚拟中断。每个寄存器表示一个虚拟中断，并记录以下信息：

<br>

- **vINTID（虚拟 INTID）**
  
    在虚拟环境中报告的 `INTID`。
    
- **State**
  
    虚拟中断的状态（Pending、Active、Active and Pending 或 Inactive）。状态机会随着虚拟环境中的软件与 GIC 交互而自动更新。例如，Hypervisor可能创建一个新的虚拟中断，最初将状态设置为 Pending。当 vPE 上的软件读取 `ICV_IARn_EL1` 时，状态将更新为 Active。
    
- **中断Group**
  
    在非安全状态下，虚拟环境始终表现为`GICD_CTLR.DS==1`。在安全状态下，虚拟环境表现为 `GICD_CTLR.DS==0`，FIQs 被路由到 EL1。因此，在这两种情况下，虚拟中断可以是Group 0 或Group 1。Group 0 中断作为 vFIQs 传递。Group 1 中断作为 vIRQs 传递。
    
- **pINTID（物理 INTID）**
  
    虚拟中断可以选择性附加物理中断的 `INTID`。当 `vINTID` 的状态机更新时，`pINTID` 的状态机也会更新。
    

<br>

`ICH_LR<n>_EL2`寄存器不记录目标 vPE，总是将目标定位到当前调度的 vPE，软件的责任是在更改调度的 vPE 时切换列表寄存器的上下文。

<br>

## 2.3 物理中断被转发到vPE的示例

<br>

下图表显示了一个将物理中断转发到vPE的例子：

<br>

![Untitled](/assets/images/20240322-arm-gic-virt/3.png)

<br>

1. 物理中断从`GICR`被转发到物理 CPU 接口。
2. 物理 CPU 接口检查物理中断是否可以转发到对应PE。在此例，检查通过，并且物理异常被发送。
3. 中断被发送到 EL2。Hypervisor读取 `IAR`，返回 `pINTID`。此时 `pINTID` 处于active状态。Hypervisor确定中断将被转发到当前运行的虚拟处理器。Hypervisor将 `pINTID` 写入 `ICC_EOIR1_EL1`。由于 `ICC_CTLR_EL1.EOImode==1`，这只执行Priority drop 而不会deactivating物理中断。
4. Hypervisor写入一个 List 寄存器来注册一个Pending的虚拟中断。List 寄存器条目指定要发送的 `vINTID`和原始的 `pINTID`。然后，Hypervisor执行异常返回，将执行返回到vPE。
5. 虚拟 CPU 接口检查虚拟中断是否可以转发到vPE。这些检查与物理中断的检查相同，只不过它们使用的是 ICV 寄存器。在此例，检查通过，虚拟异常被发送。
6. 虚拟异常被带到 EL1。当软件读取 `IAR` 时，返回 `vINTID`，虚拟中断现在处于active状态。
7. Guest OS处理中断。当它完成中断处理后，它写入 `EOIR` 来执行Priority drop和中断deactivation。由于 List 寄存器记录了 `pINTID`，这会deactivates `vINTID` 和 `pINTID`。

<br>

这个例子展示了一个物理中断被转发到vPE作为虚拟中断。例如，这可能来自Hypervisor为虚拟机分配的外围设备。并非所有的虚拟中断都必须由物理中断引起。虚拟化软件可以随时在 List 寄存器中创建虚拟中断。

<br>

## 2.4 Maintenance中断

<br>

CPU接口可以在虚拟CPU接口满足特定条件时生成物理中断。这些中断被报告为`PPI`，具有`INTID 25`。这个中断通常配置为非安全Group 1，并由EL2的Hypervisor序软件处理。Maintenance中断的生成受`ICH_HCR_EL2`控制，当前正发送的Maintenance中断在`ICH_MISR_EL2`中报告。

<br>

一个Maintenance中断的例子：如果vPE清除了虚拟CPU接口中的任何一个Group使能位，这时就可以生成Maintenance中断。当监视到这一情况时，Hypervisor可以删除属于已禁用Group的任何Pending的虚拟中断的List 寄存器中的条目。

<br>

## 2.5 上下文切换

<br>

在vPE之间进行上下文切换时，Hypervisor软件保存一个vPE的状态并加载另一个vPE的上下文。虚拟CPU接口的状态是vPE上下文的一部分，其中包括以下信息：

- ICV寄存器的状态。
- active虚拟优先级。
- 任何Pending、active或 Active-and-pending 的虚拟中断。

<br>

ICV寄存器的状态可以使用ICH寄存器从EL2访问。例如，以下示例说明了`ICH_VMCR_EL2`中的字段如何映射到ICV寄存器状态。

<br>

![Untitled](/assets/images/20240322-arm-gic-virt/4.png)

<br>

在切换vPE时，必须保存并恢复vPE的active的虚拟优先级。可以使用`ICH_APxRn_EL2`寄存器访问当前vPE的active优先级。

<br>

如在2.2节管理虚拟中断中所述，使用List寄存器管理虚拟中断。这些寄存器的状态特定于当前vPE。因此，在上下文切换时必须保存和恢复这些寄存器。

<br>

# 3 GICv3.1的安全世界虚拟化

<br>

Armv8.4-A在安全世界中引入了虚拟化支持，如下图所示：

<br>

![Untitled](/assets/images/20240322-arm-gic-virt/5.png)

<br>

如果硬件支持，可以使用`SCR_EL3.EEL2`来启用或禁用安全虚拟化支持。

<br>

GICv3.1将对虚拟化的支持扩展到了安全状态，以与Armv8.4-A对齐。当`SCR_EL3.EEL2==1`时，前面章节描述的所有功能也适用于安全状态。

<br>

安全状态和非安全状态的虚拟化之间有一些细微差别。在非安全状态下，`GICD_CTLR.DS==1`。在安全状态下，`GICD_CTLR.DS==0`，FIQ被路由到EL1。对于大多数寄存器访问，这种区别没有实际的影响。然而，当写入`ICV_BPR1_EL1`时，它改变了最小允许的值。

<br>

## 3.1 共享maintenance中断

<br>

只有一个GIC maintenance 中断，由安全状态和非安全状态下的不同虚拟化软件共享。处理这个问题的一种方法是在改变安全状态时保存和恢复该中断的配置。

<br>

# 4 GICv4.1 虚拟中断直通

<br>

GICv4继承了前面章节介绍的所有中断虚拟化支持功能。它增加了对虚拟中断直通的支持。此功能允许软件提前向`ITS`描述物理事件如何映射到虚拟中断。如果虚拟中断所对应的vPE正在运行，则可以直接转发该虚拟中断，而无需首先进入Hypervisor。这可以通过减少进入Hypervisor的次数来减少虚拟化中断的开销。

<br>

GICv4.0支持虚拟LPIs（vLPIs）直通，GICv4.1支持扩展到包括虚拟SGIs（vSGIs）的直通。GICv4.0和GICv4.1之间存在几个变化，使它们彼此不兼容。本文描述GICv4.1的编程模型。

<br>

在GICv4.0和GICv4.1中，虚拟中断直通仅限于非安全状态，安全状态的虚拟中断直通不支持。

<br>

## 4.1 总述

<br>

本节首先简要介绍GICv4.1中虚拟中断直通的工作原理。接下来的几节将提供更详细的信息。

<br>

GICv4.1允许软件定义多个vPE，并将物理中断映射到这些vPE。vPE由`vPEID`（虚拟处理单元标识符）标识。`vPEID`是一个全局标识符，由系统中所有的`GICR`和`ITS`共享。

<br>

vPE的配置和状态存储在内存中的Table中。这与管理物理LPI的配置和状态方式类似。`GICR`用于管理虚拟中断的三种Table类型如下：

**`Virtual LPI Pending Table`**

- 每个vPE都有一个虚拟LPI Pending表。它存储针对该vPE的虚拟中断的Pending状态。

**`Virtual LPI Configuration Table`** 

- 虚拟LPI配置表存储vLPI的配置（如：启用和优先级）。一个虚拟配置表可以被多个vPE共享。例如，一个VM中的所有vPE可能共享一个虚拟LPI配置表。

**`vPE Configuration Table`** 

- vPE配置表存储所有vPE的设置。表中每个vPE都有一个条目，存储该vPE的虚拟LPI Pending表和LPI Configuration表的指针。vPE配置表条目还存储有关vPE的其他信息，例如`vINTID`空间的大小。一个vPE配置表在多个`GICR`间共享，通常每个SoC有一个表。

<br>

下图显示了这些表之间的关系：

<br>

![Untitled](/assets/images/20240322-arm-gic-virt/6.png)

<br>

vPE 配置表的位置和大小由软件使用 `GICR_VPROPBASER` 寄存器指定。该表中的条目是通过发出 `ITS`命令来填充的。这将在后面进行讨论。

GIC 需要知道在物理 PE 上当前是否有任何 vPE 被调度。对于 GICv4.1，当前调度的 vPE 被指定在 `GICR_VPENDBASER` 中。

*注：这是 GICv3 和 GICv4 之间的一个重大区别。在 GICv3 和 GICv4 中，虚拟中断都只能传递给当前调度的 vPE。在 GICv3 中，硬件并不知道当前调度的 vPE 的 ID，而是由软件在上下文切换时负责管理。而在 GICv4 中，由于中断可能随时到达，中断直通会绕过Hypervisor, 硬件需要知道当前调度的是哪个 vPE。*

<br>

软件使用`ITS` 命令来创建和管理 vPEs。`VMAPP`命令用于定义一个新的 vPE，指定其配置以及Pending表和Configuration表的位置。这些信息被存储在 vPE 配置表中。`VMAPI` 和 `VMAPTI` 将物理中断映射到特定 vPE 的虚拟中断。

<br>

下图显示了当针对 vPE 的中断到达时会发生什么：

<br>

![Untitled](/assets/images/20240322-arm-gic-virt/7.png)

<br>

1. 外设向 `ITS` 发送 `MSI`。
2. `ITS` 对 `MSI` 中的 `EventID/DeviceID` 进行转换。返回的映射指示该中断被映射到一个 vPE，而不是一个物理 LPI。
3. `ITS` 将中断转发给目标 `GICR`，并发送中断的 `vINTID` 和 `vPEID`。
4. `GICR`从 vPE 配置表中检索 vPE 和 `vINTID`的配置。它还使用 `GICR_VPENDBASER`检查 vPE 是否正在被调度。
5. 如果 vPE 正在被调度运行，则将中断转发到虚拟 CPU 接口。否则，中断将被记录为Pending状态，并在下次调度 vPE 时直接发送。

<br>

`ITS` 和 `GICR`可以缓存来自不同表的信息。因此，在具体实践中，并非所有中断都需要从内存来检索表内容。

<br>

## 4.2 GICR

<br>

`GICR`从 vPE 配置表中检索 vPE 和 `vINTID`的配置。

<br>

### 4.2.1 CommonLPIAff groups

<br>

`GICRs`被分组在一起，可由 `GICR_TYPER.CommonLPIAff`和 `GICR_TYPER.Affinity` 来配置。 `CommonLPIAff`在 `Affinity`值上充当掩码，应用掩码后具有相同亲和力值的 GICR属于同一组。

<br>

例如，`CommonLPIAff==2`，则所有具有相同 `Aff3.Aff2` 值的 `GICR`属于同一组。考虑一个具有四个 `GICR`的系统，其具有以下`Affinity`：

- 0.0.0.0
- 0.0.0.1
- 0.1.0.0
- 0.1.0.1

当应用掩码后

- 0.0.x.x
- 0.0.x.x
- 0.1.x.x
- 0.1.x.x

我们有了两组`GICR`, 即 `0.1.x.x` 和 `0.0.x.x`

<br>

`CommonLPIAff` 组最好是包含物理上彼此接近的 `GICR`。例如，在多芯片设计中，可能会将每个芯片的`GICRs`设置为一个组。

<br>

`CommonLPIAff`值很重要，因为它决定了软件必须分配多少内存结构，以及一个 vPE 可以被调度到哪些 `GICR`上。我们将在接下来的部分讨论这一点。

<br>

### 4.2.2 The vPE Configuration Table

<br>

vPE 配置表存储了所有 vPE 的详细信息。表的大小决定了软件可以创建多少个 vPE。

<br>

vPE 配置表是由 `GIC`在执行 `ITS`命令时填充和维护的。软件在将内存交给 `GIC`后不应再读取或写入该表。如这样做，可能会导致 GIC 的行为异常。

软件必须为每个 `CommonLPIAff`组分配一个vPE配置表。也就是说，如果有两个 `CommonLPIAff`组，则软件必须为两个表分配足够的内存。属于不同 `CommonLPIAff` 组的`GICR`不可以共享同一个vPE配置表。

<br>

在创建 vPE 映射之前，软件必须为每个活动的 `GICR`分配所需数量的表，并填充其`GICR_VPROPBASER`。

<br>

### 4.2.3 控制调度哪一个vPE

<br>

当前运行在物理 PE 上的 vPE 由 `GICR_VPENDBASER`来确定。要更改正在调度的 vPE，软件必须：

<br>

1. **清除 `GICR_VPENDBASER.Valid`**
清除 `Valid` 位通知 `GICR`正在进行上下文切换。`GICR`从虚拟 CPU 接口检索任何待处理的虚拟中断，并确保内存中的虚拟 LPI Pending表是正确的。
2. **轮询 `GICR_VPENDBASER.Dirty` 直到读取为 0**
`Dirty` 位报告了 `GICR`已经完成更新其内部状态。这包括从虚拟 CPU 接口检索任何旧 vPE 的待处理虚拟中断。在 `Dirty` 读取为 0 之前，不能调度新的 vPE。ARM 建议虚拟化软件在观察到 `Dirty` 为 0 之前不要切换 ICH 寄存器来更改上下文。
3. **更新 `GICR_VPENDBASER`，在过程中设置 `Valid==1`**
将 `Valid` 位设置为 1 通知 `GICR`新的 `vPEID`现在有效，该 vPE 的虚拟中断现在可以被转发。`GICR_VPENDBASER`还包含了虚拟`GICD`的Group 使能位，它控制着哪些Group的虚拟中断组可以被转发到 CPU 接口。
4. **轮询 `GICR_VPENDBASER.Dirty` 直到读取为 0 （此步为 可选）**
当 `Valid` 被写入为 1 时，`GIC`寻找新调度的 vPE 的中断。在 `GIC`找到可以发送的中断，或者完成遍历Pending表并发现没有挂起的中断时，`Dirty`值为 1。

<br>

在 `ITS`中，一个 vPE 被映射到一个特定的 `GICR`。这种映射会随着时间而改变，但在任何给定的时间点，一个 vPE 有一个单一的目标 `GICR`。然而，一个 vPE 可能被调度到任何一个与目标 `GICR`属于相同的 `CommonLPIAff`组的 `GICR`。将 vPE 调度到不同组的 `GICR`可能会导致 `GIC`行为异常。

<br>

在单芯片设计中，所有的 `GICR`可能都属于同一个 `CommonLPIAff` 组。在这种情况下，可以将 vPE 调度到任何一个 `GICR`上。

<br>

软件不得做如下操作：

- 将 vPE 在 `ITS` 中建立映射之前，不得将一个 `vPEID` 标记为在任何 `GICR`上调度。
- 不得将相同的 `vPEID` 标记为在多个 `GICR`上调度。

<br>

## 4.3 Doorbells

<br>

Hypervisor通常将 vPE 分为三类：

<br>

**Running**

- vPE 当前由Hypervisor调度到一个物理PE。对于 `GIC`，这意味着 vPE 可以接收直通的虚拟中断。

**Runnable or to-be-scheduled** 

- vPE 没有被调度到任何物理PE。Hypervisor知道 vPE 有工作要做，所以在将来的某个时候会将其调度。而在当前，无法通过 `GIC`将虚拟中断传递给此 vPE。

**Idle**

- vPE 没有被调度到任何物理PE。Hypervisor认为 vPE 没有工作要做，因此不会在未来调度它。

<br>

当 vPE 有工作要做时，它将从 `**Idle**`状态转移到**`Runnable`**状态。这种情况可能是到达中断是发送给该 vPE的，而此时其并未被调度，这时需Hypervisor意识到中断已经到达，从而进行必要的vPE调度。此时，通知Hypervisor的机制称为`Doorbell`中断。

<br>

当虚拟中断到达时，如果目标 vPE 已经被调度，那么中断可以被转发到 CPU 接口：

<br>

![Untitled](/assets/images/20240322-arm-gic-virt/8.png)

<br>

若此时， vPE 未被调度时，可以选择发送`Doorbell` 中断：

<br>

![Untitled](/assets/images/20240322-arm-gic-virt/9.png)

<br>

这个`Doorbell` 是一个物理中断，通常会被发送到 EL2 并由Hypervisor处理。它向Hypervisor发出信号，表示有一个Pending中断针对未被调度的 vPE，意味着应将其移动到可运行队列中供接下来调度。

<br>

GICv4.1 支持两种类型的`Doorbell`：

- `Default Doorbells`
- `Individual Doorbells`（在 GICv4.1 中是可选的）

<br>

### 4.3.1 Default doorbells

<br>

每个 vPE 可以被分配一个`default doorbell`。当针对该 vPE 的任何中断变为待处理且该 vPE 未被调度时，将生成`default doorbell`。

<br>

ARM架构为`default doorbell`提供了几项保证：

- 在vPE调度之间，针对 vPE 的`default doorbell`最多设置为一次待处理状态。一旦将 vPE 从空闲状态移至可运行状态，软件就不需要为该 vPE 再次配置`doorbell`通知。vPE 已经准备好被调度了。
- 仅当 vPE在其非调度状态时请求了`default doorbell`时才会发送`default doorbell`。如果在将 vPE 设为非调度状态时，Hypervisor已经打算将 vPE 标记为可运行，那么它就不需要 `doorbell`。这种情况下接收`doorbell`是低效的。
- 仅当待处理中断没有被禁用时才会生成`default doorbell`。如果中断已被禁用，则不需要将 vPE 设置为可运行状态。
- 通过将 vPE 设为已调度状态来清除Pending的`default doorbell`。如果 vPE 调度时仍存在未处理的`default doorbell`，那么该中断将被清除。

<br>

`default doorbell`的生成由 `GICR_VPENDBASER`中的两个比特位控制：

- `GICR_VPENDBASER.PendingLast`：当 vPE 被设为非调度状态时，此位报告是否存在未处理的中断请求。
- `GICR_VPENDBASER.Doorbell`：软件设置此位以指示是否要生成Doorbell。

<br>

当 `PendingLast`报告存在针对 vPE 的未处理中断时，`Doorbell`被视为 0（无`Doorbell`），否则，意味着必须立即生成一个`Doorbell`。

用于 vPE `default doorbell`的 `INTID`是使用 `ITS`命令设置的，我们稍后会讨论。

*注：软件在创建 vPE 时无需注册`default doorbell`。将`default doorbell`的 `INTID`设置为 1023 则表示没有`doorbell`。*

<br>

### 4.3.2 更改 Default doorbells 配置

<br>

`Default doorbells`是物理 LPI，这意味着它们的配置存储在内存中。具体来说，在物理 LPI 配置表中。如在 [ARM LPI 中断 与 ITS](https://calinyara.github.io/technology/2024/03/21/arm-lpi.html)  中所述，如果软件想要更改物理 LPI 的配置，则会写入LPI配置表项，然后使用 `INV`命令使旧配置的任何缓存无效。尽管`Default doorbells`是物理 LPI，但它们在缓存方面与其他 LPI 不同。软件必须为用作`Default doorbells`的 `INTID` 发出 `INVDB`命令。

<br>

### 4.3.3 Individual Doorbells

<br>

`Individual Doorbells`可以选择性地针对每个虚拟中断进行设置，而不是针对每个 vPE 进行设置。这意味着根据导致 vPE 成为Pending状态的目标中断的不同，Hypervisor 可能会采取不同的操作。例如，大多数中断可以使用`Default doorbell`，并且只会导致 vPE 被标记为可运行状态。然后分配一个高优先级中断为`Individual Doorbell`，并导致立即重新调度。

<br>

`Individual Doorbell`没有`Default doorbell`具有的所有保证。特别是：

- 不能保证在vPE调度之间只设置一次`Individual Doorbells`为Pending状态。
- 软件无法在使 vPE 变为非调度状态时注册是否需要`Individual Doorbell`。如果已为虚拟中断提供了一个`Individual Doorbell`，它将在 vPE 处于非调度状态时产生。

<br>

软件可以为多个虚拟中断分配相同的物理 `INTID`，只要所有这些中断都属于同一个 vPE。

`Individual Doorbell`的支持是可选的，是否支持由 `GITS_TYPER.nID`报告。

*注：在映射虚拟中断时，软件不需要注册`Individual Doorbell`。将 `Doorbell` `INTID` 设置为 1023 表示没有 `Doorbell`。*

<br>

# 4.4 ITS

<br>

`ITS`负责将传入的 `MSI`进行翻译并转发为虚拟中断。在 [ARM LPI 中断 与 ITS](https://calinyara.github.io/technology/2024/03/21/arm-lpi.html) 中，我们介绍了基本的转换机制。

<br>

## 4.4.1 vPE Table

<br>

对于虚拟中断，中断转换表（ITTs）记录了中断所指向的 vPE 和`vINTID`。`ITS`还需要记录一个 vPE 被映射到`GICR`的哪个 Group。`ITS`可以通过两种方式来完成这个任务，`GITS_TYPER.SVEPT` 指示了支持的模型：

- `SVEPT==0`：`ITS`使用一个私有表来记录 vPE 的映射。软件必须为这个表分配内存，并设置 `GITS_BASER2`指向分配的内存。
- `SVEPT==1`：`ITS`重用了 `GICR`的 vPE 配置表。软件必须设置 `GITS_BASER2`指向为 `GICR`分配的 vPE 配置表。

*注：与其他 `ITS` 表类似，这些数据结构必须在 `ITS`启用之前分配，这适用于两种模型。*

<br>

### 4.4.2 映射一个vPE

<br>

vPE通过`VMAPP`命令来创建

```bash
VMAPP <vPEID>, <RDADDR>, <VPT size>, <VPT address>, <VCT address>, <doorbell>
```

<br>

在这个命令中

- `vPEID` 是 vPE 的 ID
- `RDADDR` 是目标 `GICR`
- `VPT address` 和 `VCT address` 是虚拟 Pending与Configuration 表的地址。
- `VPT size` 指定用于 vPE 的 `vINTID`的位宽。可以用来确定虚拟 Pending与Configuration 表的大小。
- `doorbell` 是 vPE `default doorbell`的物理 `INTID`。指定为 1023意味着 vPE 没有`doorbell`中断。

在给目标 `GICR`分配和设置 vPE 配置表之前，软件不得使用 VMAPP 创建 vPE。

<br>

### 4.4.3 映射MSI到vINTID

<br>

`EventID/DeviceID` 组合被映射到一个 `vINTID`和 vPE。当 `EventID`和 `vINTID`相同时，使用 `VMAPI` 命令。

```bash
VMAPI <DeviceID>, <EventID>, <pINTID>, <vPEID>
```

<br>

当 `EventID`和 `vINTID`不同时，使用 `VMAPTI` 命令。

```bash
VMAPTI <DeviceID>, <EventID>, <vINTID>, <pINTID>, <vPEID>
```

<br>

在这些命令中：

- `DeviceID`和 `EventID`共同标识被映射的中断。
- `vPEID` 是 vPE 的 ID。
- `vINTID`是虚拟 LPI 的 `INTID`。对于 `VMAPI`，`EventID`和 `vINTID`具有相同的值。
- `pINTID`是如果 vPE 没有被调度时需要产生的`Individual Doorbell`中断。指定为 1023 则表示没有`Individual Doorbell`中断。

在使用 `VMAPP`命令创建 vPE 之前，软件不得将中断映射到 vPE。

<br>

### 4.4.4 例子

<br>

一个外设具有 `DeviceID`5。它生成两个 `EventID`，0 和 1。这两个 `EventID` 都映射到 `vINTIDs`，这些vINTIDs属于`vPEID`为 6 的vPE ：

- EventID 0 – vINTID 8725，没有`Individual Doorbell`中断
- EventID 1 – vINTID 9000，没有`Individual Doorbell`中断

vPE 6 被映射到 `GICR`编号 7，并且使用 8192 作为其`Default doorbell`。

<br>

对应的命令序列如下：

```bash
VMAPP 6, 7, 14, <Pending Table Addr>, <Config Table Addr>, 8192
VMAPTI 5, 0, 8725, 1023, 6
VMAPTI 5, 1, 9000, 1023, 6
VSYNC 6
```

*注： 该示例假设 `GITS_TYPER.PTA==0`，并且之前已经发出了 `MAPD` 命令以映射 `ITT`。*

<br>

### 4.4.5 重映射vPE到另一个GICR

<br>

如果一个 Hypervisor 将一个 vPE 迁移到一个属于不同 `CommonLPIAff`组的 `GICR`上，那么 `ITS`映射必须更新，以便将虚拟中断发送到正确的位置。使用 `VMOVP`命令更新 ITS 映射，然后使用 `VSYNC`同步上下文。

<br>

`Doorbell`中断始终传递到映射的 `GICR`，但是 vPE 可以被调度到同一 `CommonLPIAff`组中的任何 `GICR`。如果软件希望将 vPE 的`Doorbell`中断传递到不同的处理器，则必须发出 `VMOVP`命令。

<br>

系统可以包含多个 `ITS`。当一个 vPE 的映射由多个 `ITS`进行管理时，任何更改都必须应用于包含原始映射的所有 `ITSs`。GICv4 支持两种模型来实现这一点，`GITS_TYPER.VMOVP`指示使用的模型。

- `GITS_TYPER.VMOVP==1`
不管有多少个 ITS 具有该 vPE 的映射， `VMOVP`命令必须仅在一个 `ITS` 上发出。硬件必须传播更改并处理同步。这意味着不需要 `ITS List`和 `SequenceNumber`字段。

```bash
VMOVP <vPE ID>, <RDADDR>
```

这应该是大多数 GIC 实现的模型。

<br>

- `GITS_TYPER.VMOVP==0`
必须在所有具有 vPE 映射的 `ITS`上发出 `VMOVP`命令。

```bash
VMOVP <vPEID>, <RDADDR>, <ITS List>, <Sequence Number> 
```

<br>

在此命令中：

- `vPEID`是 vPE 的 ID。
- `RDADDR`是将 vPE 重新映射到的 `GICR`。
- `ITS List` 是具有 vPE 映射的所有 `ITS`的列表。此字段编码为每个 `ITS`一个比特，其中比特 0 对应于 ITS 0。ITS 的编号由 `GITS_CTLR.ITS_Number`报告。
- `Sequence Number`是同步点。在向不同的 `ITS`发出 `VMOVP`命令时，软件必须使用相同的值。并且只有在所有 `ITS`上的命令完成后，才能重复使用相同的值。

<br>

### 4.4.6 重映射vINTIDs

<br>

`VMOVI`命令将 `EventID`和 `DeviceID`组合重新映射到不同的 `vINTID`和 vPE。

```bash
VMOVI <DeviceID>, <EventID>, <vPEID>, <vINTID>, <pINTID>
```

<br>

在此命令中：

- `DeviceID`和 `EventID`共同标识正在重新映射的中断。
- `vPEID` 是中断要移动到的 vPE 的 ID。
- `vINTID`是中断现在应该使用的虚拟 `INTID`。
- `pINTID`是在 vPE 未被调度时生成的`Individual Doorbell`中断。指定为 1023 表示没有`Individual Doorbell`中断。

<br>

## 4.5 移除映射

<br>

映射命令 `VMAPP`和 `VMAPTI`具有一个 V 字段。当 `V==1`时，它们被视为映射命令。当 `V==0` 时，它们被视为取消映射命令。当移除 vPE 的映射时，软件必须首先移除该 vPE 的所有中断映射。

<br>

## 4.6 更改vLPI配置

<br>

与物理 LPI 一样，`GICR`可以缓存 vLPI 的配置。如果更改了 vLPI 的配置，则必须使缓存副本无效。有两个 `ITS`命令可用于执行此操作。

<br>

`INV`命令通常用于更改单个或少量 vLPI 的配置。对于每个修改的 vLPI，都需要单独的 `INV`命令。

<br>

`VINVALL`命令使属于指定 vPE 的所有 vLPI 的配置无效。这个命令通常用于修改许多 vLPIs。

<br>

在 GICv4.1 中，还可以使用每个 `GICR` 中的 `GICR_INVLPIR`寄存器进行无效操作。ARM 期望软件只使用这些命令或寄存器的一种方法，而不是经常混合使用这两种方法。

<br>

# 5 GICv4.1 -  vSGIs的直接注入

<br>

GICv4.1引入了一项新功能，即直通虚拟 SGI（vSGI）。该功能通过消除一些需要进入 Hypervisor 的情况， 进一步减少了开销了。

<br>

## 5.1 发送vSGI

<br>

软件通过写入`ITS`中的`GITS_SGIR`寄存器来生成vSGI。软件写入目标vPE的`vPEID`和要发送的SGI的`INTID`。`ITS`查找指定vPE的映射，然后将vSGI转发给目标`GICR`。

<br>

下图示显示了发送vSGI的流程：

<br>

![Untitled](/assets/images/20240322-arm-gic-virt/10.png)

<br>

1. 在GuestVM中运行的软件写入其中一个`ICC_SGI`寄存器以生成一个SGI。这个写操作包含正在发送的`INTID`以及目标vPE(s)的关联值。这个寄存器写触发一个EL2级别的陷阱异常。
2. Hypervisor将GuestVM写入的`affinity`值转换为`vPEID`，然后写入`GITS_SGIR`以生成一个虚拟中断。
3. `ITS`查找指定`vPEID`的目标`GICR`，然后将vSGI转发给该`GICR`。
4. 如果目标vPE已经被调度，那么`GICR`会检索中断的配置，并将中断转发到CPU接口。然后CPU接口将vSGI信号作为虚拟中断异常发送。如果vPE没有被调度，那么中断会被记录为`Pending`状态。可以选择生成一个`default doorbell`。这个过程仍然需要一定程度的Hypervisor交互，以将写入`ICC_SGIR`寄存器的虚拟关联值转换为vPEID。但在此之后，GIC的虚拟中断直通机制可以处理剩余的过程。

<br>

## 7.2 SIG 配置

<br>

为了直通vSGI，`GICR`需要知道SGI的配置（启用、优先级和Group）。这些信息记录在虚拟LPI Pending表中。在 [ARM LPI 中断 与 ITS](https://calinyara.github.io/technology/2024/03/21/arm-lpi.html) 中，我们介绍了LPI Pending表，并描述了其前1K空间的用途取决于具体实现。GICv4.1就使用了其中一小部分空间来存储vSGI的配置。

<br>

与vLPI不同，软件不能直接写入表格来更新vSGI的配置。相反，使用新的`ITS`命令设置配置：

```bash
VSGI <vPEID>, <vINTID>, <Enable>, <Group>, <Priority>, <Clear>
```

<br>

其中：

- `vPEID`标识目标vPE。
- `vINTID`是正在更新的vSGI。
- `Enable`、`Group`和`Priority`是该vSGI的配置信息。
- `Clear`可用于清除中断的Pending状态。

<br>

## 7.3 发送Group vs 接收Group

<br>

当软件生成一个SGI时，写入的寄存器指定了发送的Group：

- `ICC_SGI0R_EL1`：发送Group 0中断
- `ICC_SGI1R_EL1`：发送Group 1中断

<br>

接收器还为每个`SGI INTID`指定一个Group。对于物理SGI，这是使用`GICR_IGROUPR0`和`GICR_IGRPMODR0`寄存器指定的。对于vSGI，如7.2节描述，Group配置是使用`VSGI`命令设置的。

<br>

对于物理SGI，GIC会将发送的Group与接收器上配置的Group进行比较。只有它们匹配时，中断才会被设置为Pending。

<br>

对于vSGI，在`GITS_SGIR`中没有Group字段。GIC将始终使用接收器中配置的Group。因此，通常由Hypervisor 来决定检查发送的Group是否与接收器配置的Group匹配。

<br>

## 7.4 vSGIs与pSGIs的不同

<br>

在大多数方面，虚拟vSGI和物理pSGI的行为都是相同的。它们不同之处在于vSGI使用与LPI相同的状态机，如下图所示：

<br>

![Untitled](/assets/images/20240322-arm-gic-virt/11.png)

<br>

这减少了`GIC`硬件的复杂性。在大多数情况下，这种差异对软件来说是不可见的。要看到差异，GuestVM内的软件必须使用`EOImode==1`。这是指使用两次单独的寄存器写操作执行`Priority-drop`和`deactivation`。对于pSGI，同一`INTID`直到`deactivation`之后才能被再次接收。对于vSGI，在`Priority-drop`后，`deactivation`之前就可以被接收。

<br>

## 7.5 查询SGI状态

<br>

Hypervisor有时需要检查vSGI是否处于Pending状态。例如，处理GuestVM对`GICR_ISPENDR0`寄存器的读取。为了实现这一点，GICR添加了两个新寄存器，用于查询vPE的vSGI的当前Pending状态。软件执行以下操作：

1. 将`vPEID`写入`GICR_VSGIR`。
2. 轮询`GICR_VSGIPENDR.Busy`直到读取为0，这时，`GICR_VSGIPENDR.Pending`报告vPE的SGIs的Pending状态。

<br>

没有GICR寄存器来模拟对`GICR_ISPENDR0`和`GICR_ICPENDR0`的写入。这可以通过ITS来模拟：

- 写`GICR_ICPENDR0`可以使用`VSGI`命令进行模拟，其中`Clear==1`。
- 写`GICR_ISPENDR0`可以通过`GITS_SGIR`寄存器来模拟。





<br>

<br>

<!-- Google tag (gtag.js) -->

<script async src="https://www.googletagmanager.com/gtag/js?id=G-69PP8GKYST"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'G-69PP8GKYST');
</script>



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