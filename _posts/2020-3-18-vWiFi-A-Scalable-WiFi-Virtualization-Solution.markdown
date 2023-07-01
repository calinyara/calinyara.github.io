---
layout: post
title:  "vWiFi - A Scalable and Software Defined WiFi Virtualization Solution"
categories: Technology
tags: Virtualization WiFi hypervisor datacenter 802.11 scalability firmware vWiFi software-defined en
author: Calinyara
description:
---

<br>

### **1. Introduction**

<br>

A Scalable WiFi Virtualization Solution called **Wireless** **Mediated Pass-Though with** **WiFi** **SoC Chip Virtualization** (**vWiFi**) is proposed to create many virtual WiFis on one physical WiFi SoC chip.  Different **vWiFis** can be configured as **AP**s or **STA**s and  run in different WiFi channels independently without  influencing each other.

Intel lost the embedded devices in the competition with ARM. However, x86 has its advantage that is better single core performance, which is especially suitable for **workload consolidation**. Therefore, to utilize the  computing power by **workload consolidation** becomes one of intel strategy in IoT. While Arm CPUs have lower energy consumption but lower single core performance. So multiple-core processors is commonly used to increase computing power. With the development of semiconductor technology, we will have more and more redundant computing power in single device. We need a way to make full use of these power.

Virtualization which is a technology widely used  in cloud computing data center servers can also be applied to edge devices. Imagine the following scenarios:

- *Virtual Machines in the same physical machine can share the same WiFi device. Each virtual machine can have full wireless capability just like working on a physical WiFi device and independently configure their working WiFi channels without influencing each other.*
- *Dozens of WiFi access points can be hosted in one WiFi router without influencing each other.*

<br>

Let's see some cases in Figure 1:

<div align="center"><img src="/assets/images/20200318-vWiFi/example1.png"/></div>
<br>
<p align="center">Figure 1a: Software Defined vWiFi on  Virtualization WiFi Chip</p>
<br>
<div align="center"><img src="/assets/images/20200318-vWiFi/example2.png"/></div>
<br>
<p align="center">Figure 1b: Software Defined vWiFi on  Virtualization WiFi Chip</p>
<br>

To sum up, we may need a **software-definable virtualization WiFi chip** to instead of traditional WiFi chip for better scalability, which can help to build a modern **wireless data center** and save hardware costs.

<br>

### **2. Existing Solutions**

<br>

### **2.1 Traditional WiFi Stack without Virtualization**

<br>

<div align="center"><img src="/assets/images/20200318-vWiFi/Picture1.png"/></div>
<p align="center">Figure 2: Traditional WiFi Stack without Virtualization</p>
<br>

**Characteristics:**

- A bare-mental firmware (usually, a while (1) event handler implementing WiFi protocol) running in the WiFi SOC chip. 
- The SOC uses serial, SDIO, PCI or PCIe interfaces to connect to the host machine.
- No device sharing at all. 

<br>

### **2.2 Ethernet Emulation**
<br>

<div align="center"><img src="/assets/images/20200318-vWiFi/Picture2.png"/></div>
<p align="center">Figure 3: Ethernet Emulation</p>
<br>

**Characteristics:**

- Guests share the same physical WiFi device.
- Guests see Ethernet devices without wireless capability.

<br>

### **2.3 Wireless Emulation**

<br>

<div align="center"><img src="/assets/images/20200318-vWiFi/Picture3.png"/></div>
<p align="center">Figure 4: Wireless Emulation</p>
<br>

**Characteristics:**

- Guests share the same physical WiFi device.
- Guests see WiFi devices but share one physical WiFi channel and the same access point (AP). Configuration in one Guest will influence the others.

<br>

### **2.4 Wireless Direct Pass-Though**

<br>

<div align="center"><img src="/assets/images/20200318-vWiFi/Picture4.png"/></div>
<p align="center">Figure 5: Wireless Direct Pass-Though</p>
<br>

**Characteristics:**

- No device sharing among guests. Each guest has its own dedicated physical WiFi.
- Guests see WiFi devices and can operate in different WiFi Channels without influencing each other.

<br>

### **3. Wireless Mediated Pass-Through with WiFi SoC Chip Virtualization**

<br>

### **3.1 Overview**

<br>

<div align="center"><img src="/assets/images/20200318-vWiFi/Picture5.png"/></div>
<p align="center">Figure 6: Wireless Mediated Pass-Through with WiFi SoC Chip Virtualization</p>
<br>

**Characteristics:**

- Guests share the same SoC Physical WiFi Device (WiFi SoC Chip).
- Guests see WiFi devices. One Guest provides one vWiFi service. Guests can operate in different WiFi Channels without influencing each other.

<br>

### **3.2 WiFi SoC Chip Virtualization**

<br>

<div align="center"><img src="/assets/images/20200318-vWiFi/Picture6.png"/></div>
<p align="center">Figure 7: WiFi SoC Chip Virtualization</p>
<br>

- A tiny hypervisor (virtual machine monitor, VMM) runs in the physical CPUs of the WiFi SoC.
- The WiFi firmware runs in a VCPU (virtual CPU) as a virtual machine.
- Each VCPU simulates a SoC Logical WiFi Device and the hypervisor is responsible to schedule the SoC Logical WiFi Devices to run on the SoC Hardware.

<br>

<div align="center"><img src="/assets/images/20200318-vWiFi/Picture7.png"/></div>
<p align="center">Figure 8: Wireless Mediated Pass-Through</p>
<br>

- All Guest OS share the same SoC Physical WiFi Device (WiFi SoC Chip).
- The Host OS WiFi Driver communicates with the virtualized WiFi SoC Chip. It installs SoC Logical WiFi Devices into Host OS as Host Logical WiFi Devices.
- The Device Model pass through the Host Logical WiFi Devices into the guest OS as Guest Virtual WiFi Devices.
- The Guest driver runs on the Guest Virtual WiFi Device to provide vWiFi service. Many Guests one-one mapping to many vWiFis.

<br>

**Definitions:**

- **SoC Physical** **WiFi** **Device**: The physical WiFi device from the perspective of WiFi SoC Chip.
- **SoC Logical** **WiFi** **Device:** The WiFi device simulated by the WiFi SoC hypervisor.
- **Host** **Logical** **WiFi** **Device:** The SoC Logical WiFi device from the perspective of the Host OS.
- **Guest** **Virtual** **WiFi** **Device**: The WiFi device from the perspective of the Guest OS.
- **vWiFi**: The WiFi service provided by the Guest OS.

<br>

### **3.3 Comparison**

<br>
<div align="center"><img src="/assets/images/20200318-vWiFi/Picture8.png"/></div>
<p align="center">Table 1: Comparison of The Existing Solutions and vWiFi</p>
<br>

### **4. Summary**

<br>

**vWiFi** which is based on the technology named **Wireless** **Mediated Pass-Though with** **WiFi** **SoC Chip Virtualization** is a **a Scalable WiFi Virtualization Solution** that utilizes virtualization technology to build many independent WiFis on a SoC Physical WiFi Device.  It can be used to save hardware costs and build wireless data center. 

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