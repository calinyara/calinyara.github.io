---
layout: post
title:  "轻量级虚拟化解决方案"
categories: Technology
tags: Virtualization crosvm ChromeOS kata rust nemu gvisor container hypervisor firecracker zh
author: Calinyara
description: 
---

<br>

### **简介**

<br>

虚拟化技术经过多年的发展已经相当成熟，从早期的**二进制翻译**到**半虚拟**化再到现如今主流体系结构都支持的**硬件辅助虚拟化**，**容器技术**等。 硬件和软件在这一过程中相互促进协同发展。虚拟化技术围绕的三个核心议题是**成本**，**安全**，**性能**。通过资源共享来节约成本是虚拟化的出发点，但这通常意味着性能的下降和安全性的降低。通过隔离带来安全是虚拟化技术规模化应用必须要解决的问题，因为很多时候只有在安全的前提下节约的成本才具有实际意义，而这通常又会打破成本和性能的原则。因此虚拟化也需要针对不同的场景在这些指标中进行权衡。**XEN**, **KVM**, **QEMU**, **docker**是比较著名的开源虚拟化软件，这些软件为虚拟化的应用提供了软件基础。随着云计算的发展，出现了一些轻量级的虚拟化解决方案，它们从新的角度和应用场景解决着虚拟化三大议题。

- [**crosvm**](https://chromium.googlesource.com/chromiumos/platform/crosvm/)
- **[Firecracker](https://github.com/firecracker-microvm/firecracker)**
- [**rust-vmm**](https://github.com/rust-vmm)
- [**cloud-hypervisor**](https://github.com/cloud-hypervisor/cloud-hypervisor)
- [**NEMU**](https://github.com/intel/nemu)
- [**gVisor**](https://github.com/google/gvisor)
- [**Kata Containers**](https://github.com/kata-containers)

<br>

**[Rust](https://zh.wikipedia.org/wiki/Rust)**是由[Mozilla](https://zh.wikipedia.org/wiki/Mozilla)主导开发的通用、编译型编程语言。设计准则为“安全、并发、实用”，支持[函数式](https://zh.wikipedia.org/wiki/函數程式語言)、[并发式](https://zh.wikipedia.org/wiki/參與者模式)、[过程式](https://zh.wikipedia.org/wiki/程序編程)以及[面向对象](https://zh.wikipedia.org/wiki/面向对象程序设计)的编程风格。

[**crosvm**](https://chromium.googlesource.com/chromiumos/platform/crosvm/)是谷歌公司基于Rust语言开发的虚拟机软件，作为**ChromeOS**的一部分，结合**KVM**模块为**ChromeOS**的用户提供完整的虚拟化解决方案。**[Firecracker](https://github.com/firecracker-microvm/firecracker)**由Amazon开发， 最初是基于[**crosvm**](https://chromium.googlesource.com/chromiumos/platform/crosvm/)，但针对的是**无服务器运算([Serverless computing](https://en.wikipedia.org/wiki/Serverless_computing))**，又被称为 **功能即服务(Function-as-a-Service, FaaS)**的场景。由于二者的使用场景不同，走向了不同的发展道路。

为了避免代码冗余，方便人们日后开发类似[**crosvm**](https://chromium.googlesource.com/chromiumos/platform/crosvm/), **[Firecracker](https://github.com/firecracker-microvm/firecracker)**基于**[Rust](https://zh.wikipedia.org/wiki/Rust)**语言的虚拟机软件，[**rust-vmm**](https://github.com/rust-vmm)提供了一系基于**[Rust](https://zh.wikipedia.org/wiki/Rust)**语言开发的用于列构建虚拟机的组件。[**cloud-hypervisor**](https://github.com/cloud-hypervisor/cloud-hypervisor)就是基于[**rust-vmm**](https://github.com/rust-vmm)提供的组件开发的虚拟机软件。

基于**[Rust](https://zh.wikipedia.org/wiki/Rust)**的这类虚拟机软件，遵循了简洁设计原则，避免了QEMU大量复杂的在云场景中用处不大的设备模拟，同时方便定制易于扩展，并天然具备**[Rust](https://zh.wikipedia.org/wiki/Rust)**语言带来的安全性。

[**NEMU**](https://github.com/intel/nemu)的设计初衷与[**cloud-hypervisor**](https://github.com/cloud-hypervisor/cloud-hypervisor)相同，只不过实现手段不同，其通过基于**QEMU**做减法来实现。

基于容器的虚拟化技术属于操作系统层面的虚拟化，相较于上述硬件层面的虚拟化技术，其优势在于性能非常好，劣势也很明显，隔离性较低，安全性差，容易被攻击。[**gVisor**](https://github.com/google/gvisor) 和[**Kata Containers**](https://github.com/kata-containers)瞄准解决的正是容器安全性这一问题。虽然二者在具体的技术实现上有些不同（前者通过隔离系统调用，后者通过增加一个轻量级虚拟机隔离层），但本质是一样的，通过两层隔离增加安全性，然后在此基础上想办法通过一些手段优化性能。

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