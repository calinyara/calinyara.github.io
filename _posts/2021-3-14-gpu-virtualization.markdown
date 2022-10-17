---
layout: post
title:  "GPU/显卡虚拟化"
categories: Technology
tags: GPU 虚拟化 图形虚拟化 显卡 显卡虚拟化 GPGPU GVT-s GVT-d GVT-g crosvm virtualization display virtio virtio-gpu qemu
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

<div align="center"><img src="/assets/images/20210314-gpu-virtualization/virtio-gpu.png"/></div>

<p align="center">图2: 显卡虚拟化软件栈</p>
<br>

*[ACRN](https://projectacrn.org/): is a flexible, lightweight reference hypervisor.*

*[QEMU](https://www.qemu.org/): is a free and open-source emulator and virtualizer.*

*[CROSVM](https://chromium.googlesource.com/chromiumos/platform/crosvm/): is the Chrome OS Virtual Machine Monitor.*
<br>

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