---
layout: post
title:  "aVisor: A Tiny Hypervisor for Raspberry Pi"
categories: Technology
tags: 内核 kernel Linux 操作系统 树莓派 raspberry pi buildroot hypervisor 虚拟机 虚拟化 virtualization OS 调度 arm
author: Calinyara
description:
---

<br>

**[aVisor](https://github.com/calinyara/avisor)**  is a bare-metal hypervisor that runs on the Raspberry Pi 3.  It can be used to learn about the basic concepts of ARM virtualization and the principles of hypervisors and operating systems.

<br>

### **DEMO**

<br>
<div align="center"><img src="/assets/images/20230225-aVisor/en1.png"/></div>
<p align="center">fig.1：aVisor Demo</p>

<br>

### **Compilation and QEMU Simulation**

```
./scripts/demo.sh		// Compile and run the demo
./scripts/clean.sh		// Clean the project
```

<br>

### **Operation In the Console**

The above demo will run 4 Guest VMs on the Hypervisor. After startup, press ***Enter*** to go to the hypervisor's console.
- **[echo](https://github.com/calinyara/avisor/tree/main/guests/echo)**:  A baremetal binary that echoes keyboard input.
- **[lrtos](https://github.com/calinyara/avisor/tree/main/guests/lrtos)**:  A miniature operating system that runs two user mode processes after startup, one prints "12345" and the other prints "abcde". The lrtos kernel supports the process scheduling.
- **[uboot](https://github.com/u-boot/u-boot)**: The standard Das U-Boot Bootloader.
- **[FreeRTOS](https://github.com/hacker-jie/freertos-raspi3)**: The FreeRTOS VM runs two tasks, one prints "12345" and the other prints "ABCDE". The tasks are scheduled by FreeRTOS.

<br>

```
help			// Print the help
vml			// Display the current Guest VMs info
vmc <vm id>		// Switch from the hypervisor's console to a Guest VM's console
@+0			// Switch back to the hypervisor's console from a Guest VM's console 
```

<br>

<div align="center"><img src="/assets/images/20230225-aVisor/en2.png"/></div>
<div align="center"><img src="/assets/images/20230225-aVisor/en3.png"/></div>
<div align="center"><img src="/assets/images/20230225-aVisor/en4.png"/></div>

<br>

### **Running On The Physical Board**

<br>

<div align="center"><img src="/assets/images/20230225-aVisor/phy_board.png"/></div>
<br>
<p align="center">Fig.2：GPIO Pin connection</p>

<br>

- Red：5V
- black：GND
- white：TXD
- green：RXD

<br>

Copy ***config.txt, bootcode.bin, start.elf, lrtos.bin, echo.bin, kernel8.img*** to the SD card, connect the USB serial port to the computer, and then you can access aVisor through the serial port.

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