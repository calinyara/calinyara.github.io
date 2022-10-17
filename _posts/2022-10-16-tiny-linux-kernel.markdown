---
layout: post
title:  "编译一个最小可用Linux内核"
categories: Technology
tags: 内核编译 内核 kernel Linux 文件系统 rootfs initramfs 树莓派 raspberry pi
author: Calinyara
description:
---

<br>

## **x86系统**

<br>

### **内核**

<br>

**获取内核源码**

```shell
git clone https://github.com/torvalds/linux.git
```

<br>

**生成最小配置**

```shell
make tinyconfig
```

<br>

**修改配置**

```
make menuconfig
```

- 64-bit Kernel (**CONFIG_64BIT=y**)
- General Setup --> Configure standard kernel features (expert users) --> Enable support for printk (**CONFIG_PRINTK=y**)
- General Setup --> Initial RAM filesystem and RAM disk (initramfs/initrd) support (**CONFIG_BLK_DEV_INITRD=y**)
- General Setup -->kernel compression mode (Gzip) (**CONFIG_RD_GZIP=y**)
- Executable file formats --> Kernel support for ELF binaries (**CONFIG_BINFMT_ELF=y**)
- Executable file formats --> Kernel support for scripts starting with #! (**CONFIG_BINFMT_SCRIPT=y**)
- File systems --> Pseudo filesystems --> /proc file system support (**CONFIG_PROC_FS=y**)
- Device Driver --> Character devices --> Enable TTY (**CONFIG_TTY=y**)
- Device Driver --> Character devices --> Serial Drivers  --> 8250/16550 and compatible serial support (**CONFIG_SERIAL_8250=y**)
- Device Driver --> Character devices --> Serial Drivers  --> Console on 8250/16550 and compatible serial port (**CONFIG_SERIAL_8250_CONSOLE=y**)

<br>

**内核编译**

```shell
make -j4
```

<br>

生成内核 *arch/x86/boot/bzImage*，(Size ≈ 800K).

<br>

### **文件系统**

<br>

**编译脚本**

```shell
git clone https://github.com/hacker-jie/kernel-utils.git
```

<br>

**生成initramfs**

```shell
./kernel-utils/mk-initrd
```

弹出选择，都选**n**，生成内核 ./kernel-utils/*initramfs.cpio.gz*，(Size ≈ 1.3M).

<br>

### **QEMU测试**

```shell
qemu-system-x86_64 -kernel [PATH_TO_KERNEL]/arch/x86/boot/bzImage -initrd [PATH_TO_ROOTFS]/initramfs.cpio.gz -m 32M -nographic -append "init=/bin/sh console=ttyS0"
```

<br>

<br>

## **树莓派3 (64位)**

<br>

### **内核**

<br>

**获取内核源码**

```shell
git clone https://github.com/raspberrypi/linux.git
```

<br>

**生成最小配置**

```shell
make tinyconfig
```

<br>

**修改配置**

```
make menuconfig
```

```shell
CONFIG_BINFMT_ELF
CONFIG_BLK_DEV_INITRD
CONFIG_RD_GZIP
CONFIG_PRINTK
CONFIG_TTY
CONFIG_PROC_FS
CONFIG_DEVTMPFS (optional)
CONFIG_DEVTMPFS_MOUNT (optional)

CONFIG_ARCH_BCM2835
CONFIG_MAILBOX
CONFIG_BCM2835_MBOX
CONFIG_RASPBERRYPI_FIRMWARE
CONFIG_SERIAL_AMBA_PL011
CONFIG_SERIAL_AMBA_PL011_CONSOLE
```

<br>

**内核编译**

```shell
make -j4
```

<br>

生成内核 *arch/arm64/boot/Image.*



**注：编译器用 aarch64-linux-gnu, (sudo apt install -y gcc-aarch64-linux-gnu).*

<br>

### **文件系统**

<br>

与x86系统一致

<br>

### **QEMU测试**

```shell
qemu-system-aarch64 -M raspi3b -kernel [PATH_TO_KERNEL]/arch/arm64/boot/Image -dtb [PATH_TO_DTB]/bcm2710-rpi-3-b.dtb -initrd [PATH_TO_ROOTFS]/initramfs.cpio.gz -m 1024M -nographic -append "init=/bin/sh console=ttyAMA0,115200"
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