---
layout: post
title:  "tiny_raspi: 用buildroot为树莓派制作最小内核"
categories: Technology
tags: 内核编译 内核 kernel Linux 文件系统 rootfs initramfs 树莓派 raspberry pi buildroot ramfs fs zh
author: Calinyara
description:
---

<br>

使用*buildroot*默认配置*raspberrypi3_64_defconfig*直接编译的出的镜像文件大小居然有153M...

使用**[tiny_raspi](https://github.com/calinyara/tiny_raspi)**编译生成一个使用RAM文件系统的最小内核吧.

<br>

以RaspberryPi 3为例，使用*buildroot*的**EXTERNAL**机制来定制化。我们需要添加3个文件用以描述外部*buildroot*树：

- **external.desc** 
- **external.mk**
- **Config.in**

<br>

**编译**

```shell
git clone https://github.com/calinyara/tiny_raspi.git
cd tiny_raspi
git clone git://git.buildroot.net/buildroot
mkdir build
mkdir buildroot_dl
cd build
make BR2_EXTERNAL=../tiny_raspi_buildroot/ O=$PWD -C ../buildroot/ tiny_raspberrypi3_64_defconfig
make -j4
```

编译完成后在*build/images*目录会生成：**Image**，**rootfs.cpio.gz**，**bcm2710-rpi-3-b.dtb**.

<br>

*buildroot*会自动编译生成*rootfs*文件系统，亦可参照 [**编译一个最小可用Linux内核**](https://calinyara.github.io/technology/2022/10/16/tiny-linux-kernel.html) 的描述，使用**[kernel-utils](https://github.com/hacker-jie/kernel-utils/tree/bd9da8780850a63ee0cfcfd7168d8fad45223850)**工具来生成RAM文件系统.

<br>

**QEMU运行**

```shell
qemu-system-aarch64 -M raspi3b -kernel ./images/Image -dtb ./images/bcm2710-rpi-3-b.dtb -initrd ./images/rootfs.cpio.gz -m 1024 -nographic -append "init=/bin/sh console=ttyAMA0,115200"
```

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