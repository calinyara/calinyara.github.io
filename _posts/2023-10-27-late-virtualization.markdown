---
layout: post
title:  "Late Virtualization and Type 1.5 Hypervisor"
categories: Technology
tags: hypervisor 虚拟机 虚拟化 virtualization zh vbh
author: Calinyara
description:
---

<br>

Hypervisor通常被分成两种类型，独立类型Type 1和寄生类型 Type 2。Type 1类型的Hypervisor没有宿主操作系统，其直接运行在物理硬件之上，直接管理各种物理资源，同时管理并运行客户机操作系统。Type 2类型的Hypervisor，其位于一个寄生的宿主 (Host) 操作系统当中，该操作系统拥有对硬件平台和资源的全部控制权。

<br>

<div align="center"><img src="/assets/images/20231027-late-virtualization/1.png"/></div>
<p align="center">图1：虚拟机类型 TYPE 1 vs TYPE 2</p>

<br>

如果我们在Linux内核中安装一个实现了虚拟化功能的内核模块，当系统启动完成后再insmod该模块，使其在原内核与硬件之间构建一个虚拟化层，将原来的宿主系统往上抬升变成客户机系统。我们称该技术为**Late Virtualization**, 对应**Type 1.5** Hypervisor. [Virtualization Based Hardening (VBH)](https://github.com/intel/vbh) 是一个例子。

<div align="center"><img src="/assets/images/20231027-late-virtualization/2.png"/></div>
<br>
<p align="center">图2：Late Virtualization原理</p>



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