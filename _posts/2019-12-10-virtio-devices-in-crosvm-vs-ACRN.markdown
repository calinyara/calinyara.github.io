---
layout: post
title:  "Virtio Devices in Crosvm vs ACRN"
categories: Technology
tags: Virtualization crosvm ACRN virtio ChromeOS en
author: Calinyara
description: 
---

# Background

<br>
[**Crosvm**](https://chromium.googlesource.com/chromiumos/platform/crosvm/) is a device model based on [**Rust**](https://www.rust-lang.org/) language. [**Chrome OS**](https://www.chromium.org/chromium-os) uses **Crosvm** along with **KVM** to provide virtualization solution.

[**ACRN-DM**](https://github.com/projectacrn/acrn-hypervisor/tree/master/devicemodel) is a device model based on **C** language, which is part of [**ACRN project**](https://projectacrn.org/). **ACRN-DM** along with **ACRN hypervisor** provides a flexible, lightweight reference virtualization solution for embedded development. 

<br>

| Device          | ACRN | Crosvm | ID In virtio spec | Frontend  Upstream          | Description                                                  |
| --------------- | :----: | :------: | :-------------------------: | ------------------------------------------------------------ | ------------------------------------------------------------ |
| virtio-block    | Y    | Y      | 2     | Yes                         | Basic read/write block device.                               |
| virtio-net      | Y    | Y      | 1     | Yes                         | Device to interface the host and guest networks.             |
| virtio-rng      | Y    | Y      | 4     | Yes                         | Entropy source used to seed guest OS's entropy pool.         |
| virtio-vsock    | N    | Y      | 19    | Yes                         | Enabled VSOCKs for host/guest communication.                 |
| virtio-wayland  | N    | Y      | N     | No (Chrome OS kernel)       | Allowed guest to use host Wayland socket.                    |
| virtio-balloon  | N    | Y      | 5     | Yes                         | Dynamic guest RAM allocation.                                |
| virtio-9p       | N    | Y      | 9     | Yes                         | Sharing host files with the guest as network filesystem.     |
| virtio-gpu      | N    | Y      | 16    | Yes                         | A display virtualization solution.                           |
| virtio-tpm      | N    | Y      | N     | No (Chrome OS kernel)       | Trusted Platform Module.                                     |
| virtio-fs       | N    | Y      | 26    | Yes (merged in v5.4)        | Sharing host files with the guest   as local filesystem.     |
| virtio-pmem     | N    | Y      | 27    | Yes                         | A fake persistent memory(nvdimm) in guest which allows to bypass  the guest page cache. |
| virtio-input    | Y    | Y      | 18    | Yes                         | Input device.                                                |
| virtio-console  | Y    | N      | 3     | Yes                         | Console over virtio transport.                               |
| virtio-audio    | Y    | N      | 25    | No (acrn-kernel)            | Audio device.                                                |
| virtio-gpio     | Y    | N      | N     | No (acrn-kernel)            | gpio sharing over virtio.                                    |
| virtio-i2c      | Y    | N      | N     | No (acrn-kernel)            | i2c devices.                                                 |
| virito-rpmb     | Y    | N      | 28    | No (acrn-kernel)            | Replay Protected Memory Block.                               |
| virtio-mei      | Y    | N      | N     | No (acrn-kernel)            | Intel Management Engine Interface.                           |
| virtio-ipu      | Y    | N      | N     | No (acrn-kernel)            | Imaging Processing Unit.                                     |
| virtio-hyperdma | Y    | N      | N     | No (acrn-kernel)            | Hyper DMA buffer.                                            |
| virtio-hdpc     | Y    | N      | N     | No												  | High-bandwidth Digital Content Protection.                   |
| virtio-coreu    | Y    | N      | N     | No                          | PAVP ( Protected Audio Video Path) session management.       |

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