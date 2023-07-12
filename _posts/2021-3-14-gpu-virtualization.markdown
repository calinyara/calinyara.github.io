---
layout: post
title:  "GPU虚拟化/显示虚拟化"
categories: Technology
tags: GPU 虚拟化 图形虚拟化 显卡 显卡虚拟化 GPGPU GVT-s GVT-d GVT-g crosvm virtualization display virtio virtio-gpu qemu zh automotive
author: Calinyara
description:
---

<br>

近些年，网络视频成指数增长，多媒体视频在网络流量中的占比越来越高。如果这些多媒体资源能得到有效管理与运用，对企业来说是一个获得新收入并降低总体成本的机会。而今，效率意味这使用云计算，但很多时候，由于缺乏显卡虚拟化技术，用户却难以在云计算环境中使用GPU来处理这些多媒体负载并获得最佳性能。

<br>

**INTEL图形虚拟化技术**

<br>

为了应对这些挑战，显卡虚拟化技术不断发展，以允许多媒体负载运行在虚拟化的环境中。Intel的显卡虚拟化技术叫做Intel Graphics Virtualization Technology （Intel GVT）， 其中主要包括三种技术，分别是GVT-s, GVT-d以及GVT-g。

<br>

Intel® Graphics Virtualization Technology -s (Intel® GVT-s)，虚拟共享图形加速，允许多个虚拟机(VM)共享一个物理GPU。该技术也被称为虚拟共享图形适配器（vSGA）。

<br>

Intel® Graphics Virtualization Technology -d (Intel® GVT-d)，虚拟专用图形加速，一个GPU可以直通给一个虚拟(VM)机使用。该技术有时也被称为虚拟直接图形适配器（vDGA）。

<br>

Intel® Graphics Virtualization Technology -g (Intel® GVT-g)，虚拟图形处理单元(vGPU), 同样允许多个虚拟机(VM)共享一个物理GPU。

<br>

对性能，功能与共享的考量存在于每一种图形虚拟化技术当中。这三种技术也都各有特点。GVT-d由于直通给虚拟机使用，能提供最好的性能，特别适合对GPU敏感计算需求量大的负载。GVT-s采用的是API转发技术，理论上该技术可以支持任意多个虚拟机，但由于没有虚拟化完整的GPU，不能给虚拟机展现完整的GPU功能，而这些功能可能是某些负载需要的。GVT-g能够虚拟化完整的GPU功能，性能比GVT-d稍差，可以在多个虚拟机（最多8个）之间共享硬件，可算是一种折中考量。

<br>

<div align="center"><img src="/assets/images/20210314-gpu-virtualization/fig1.png"/></div>
<p align="center">图1: INTEL图形虚拟化技术比较(GVT-s, GVT-d, GVT-g)</p>
<br>

**显卡虚拟化软件栈**
<br>

<br>

<div align="center"><img src="/assets/images/20210314-gpu-virtualization/fig2.png"/></div>
<p align="center">图2: 显卡虚拟化软件栈</p>

<br>

*[ACRN](https://projectacrn.org/): is a flexible, lightweight reference hypervisor.*

*[QEMU](https://www.qemu.org/): is a free and open-source emulator and virtualizer.*

*[CROSVM](https://chromium.googlesource.com/chromiumos/platform/crosvm/): is the Chrome OS Virtual Machine Monitor.*

<br>



**ACRN virtio-gpu 方案**

<div align="center"><img src="/assets/images/20210314-gpu-virtualization/virtio-gpu.png"/></div>
<p align="center">图3: ACRN virtio-gpu</p>

<br>

virtio-gpu 是一个基于 virtio 的图形适配器，支持2D模式。它可以将User VM的帧缓冲区传输到 ServiceVM 的缓冲区用于显示。可与 Intel GPU VF ( SRIOV ) 协同工作并为图形功能提供加速。利用virtio-gpu，User VM 可以受益于 Intel GPU 硬件来加速媒体编码、3D 渲染和计算。

ACRN设备模型会在Service VM 上呈现一个图形界面窗口，它可以显示User VM利用virtio-gpu存储在Service VM缓存区的图形。virtio-gpu通过SDL（OpenGL ES 2.0 后端）与Service VM (HOST)上的显示服务连接， 为User VM提供了一种通用的显示解决方案。 当 ACRN virtio-gpu后端启动时，它会首先尝试与Service VM的图形子系统连接，然后在Service VM上以图形窗口的形式显示User VM的图形界面。

许多操作系统利用VGA来显示系统安装界面，安全模式，以及系统蓝屏。此外，像Windows这样的系统默认都不带virtio-gpu前端驱动。为了解决这些显示需求，ACRN的virtio-gpu后端方案支持传统VGA模式。为了兼容VGA与现代型virtio-gpu设备，ACRN virtio-gpu设备的PCI bar空间定义如下:

```shell
1.	BAR0: VGA Framebuffer memory, 16 MB in size.
2.	BAR2: MMIO Space
3.	  [0x0000~0x03ff] EDID data blob
4.	  [0x0400~0x041f] VGA ioports registers
5.	  [0x0500~0x0516] bochs display interface registers
6.	  [0x1000~0x17ff] virtio common configuration registers
7.	  [0x1800~0x1fff] virtio ISR state registers
8.	  [0x2000~0x2fff] virtio device configuration registers
9.	  [0x3000~0x3fff] virtio notification registers
10.	BAR4: MSI/MSI-X
11.	BAR5: virtio port io
```


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