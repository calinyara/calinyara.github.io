---
layout: post
title:  "一分钟理解隐私计算(Confidential Computing)"
categories: Technology
tags: Confidential Computing 隐私计算 安全 secure
author: Calinyara
description:
---

<br>

### **TL;DR**

<br>

**隐私计算(Confidential Computing)**是一种保护**正在使用中的数据**的方法。

<br>

### **为什么需要隐私计算**

<br>

数据的生命周期有三种状态：

- 存储着的数据
- 传输中的数据
- 使用中的数据

一直以来，人们对于存储在硬盘或非易失性存储器中的数据，以及传输中的数据都可以进行有效的加密保护。但是，对于在处理器和内存中正在使用的数据却缺乏有效的保护手段，使得这些数据在使用时有被看到、被破坏或被窃取的风险。利用隐私计算，系统现在可以保护数据生命周期中的全部状态。理论上，被保护数据在任何时候都不会有泄露被窃取的风险。

<br>

<div align="center"><img src="/assets/images/20230302-confidential_computing/数据生命周期.png"/></div>
<p align="center">图1：数据的生命周期</p>

<br>

在过去，数据安全意味着保护用户所拥有的物理系统上的数据，比如一台个人PC，一台企业的服务器。在这种情况下，系统软件仍然可以查看用户层的数据。

然而，随着云计算和边缘计算的兴起，用户的工作负载通常运行在CSP的虚拟机当中，而用户本人并非物理系统的实际拥有者。 因此，需要将焦点转移到保护用户的数据，使其不受物理机器所有者的影响。**这是一种新的安全范式，即把安全放在数据层面，而不必担心基础设施的细节**。这样用户可以放心地将工作负载部署到CSP的云环境中而不必担心数据泄露。

使用隐私计算，运行在云或边缘计算机上的软件，如操作系统或Hypervisor，仍然承担着管理工作。例如，为用户程序分配内存，但这些系统软件永远无法读取或改变用户内存中的数据。

<br>

<div align="center"><img src="/assets/images/20230302-confidential_computing/隐私计算原理.png"/></div>
<p align="center">图2：隐私计算</p>

<br>

### **相关技术**

- [Intel **SGX**](https://www.intel.cn/content/www/cn/zh/architecture-and-technology/software-guard-extensions.html)
- [Intel **TDX** (Trusted Domain Extensions)](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-trust-domain-extensions.html)
- [AMD **SEV-SNP**](https://www.amd.com/system/files/TechDocs/SEV-SNP-strengthening-vm-isolation-with-integrity-protection-and-more.pdf): 类似Intel TDX将Intel SGX中的进程级保护机制扩展到了虚拟机，用户无需重写他们的应用程序就可以实现隐私计算。
- [ARMv9 **Realms**](https://community.arm.com/arm-community-blogs/b/architectures-and-processors-blog/posts/unlocking-the-power-of-data-with-arm-cca)
- [RISC-V **Keystone**](https://keystone-enclave.org/)

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