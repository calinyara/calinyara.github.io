---
layout: post
title:  "Virtio Devices in Crosvm vs ACRN"
categories: Technology
tags: Virtualization crosvm ACRN virtio ChromeOS
author: Calinyara
description: 
---

# Background

<br>
[**Crosvm**](https://chromium.googlesource.com/chromiumos/platform/crosvm/) is a device model based on [**Rust**](https://www.rust-lang.org/) language. [**Chrome OS**](https://www.chromium.org/chromium-os) uses **Crosvm** along with **KVM** to provide virtualization solution.

[**ACRN-DM**](https://github.com/projectacrn/acrn-hypervisor/tree/master/devicemodel) is a device model based on **C** language, which is part of [**ACRN project**](https://projectacrn.org/). **ACRN-DM** along with **ACRN hypervisor** provides a flexible, lightweight reference virtualization solution for embedded development. 

<br>

| Device          | ACRN | Crosvm | Frontend  Upstream          | Description                                                  |
| --------------- | :----: | :------: | --------------------------- | ------------------------------------------------------------ |
| virtio-block    | Y    | Y      | Yes                         | Basic read/write block device.                               |
| virtio-net      | Y    | Y      | Yes                         | Device to interface the host and guest networks.             |
| virtio-rng      | Y    | Y      | Yes                         | Entropy source used to seed guest OS's entropy pool.         |
| virtio-vsock    | N    | Y      | Yes                         | Enabled VSOCKs for host/guest communication.                 |
| virtio-wayland  | N    | Y      | No (Chrome OS kernel)       | Allowed guest to use host Wayland socket.                    |
| virtio-balloon  | N    | Y      | Yes                         | Dynamic guest RAM allocation.                                |
| virtio-9p       | N    | Y      | Yes                         | Sharing host files with the guest as network filesystem.     |
| virtio-gpu      | N    | Y      | Yes                         | A display virtualization solution.                           |
| virtio-tpm      | N    | Y      | No (Chrome OS kernel)       | Trusted Platform Module.                                     |
| virtio-fs       | N    | Y      | Yes (merged in v5.4)        | Sharing host files with the guest   as local filesystem.     |
| virtio-pmem     | N    | Y      | Yes                         | A fake persistent memory(nvdimm) in guest which allows to bypass  the guest page cache. |
| virtio-input    | Y    | Y      | Yes                         | Input device.                                                |
| virtio-console  | Y    | N      | Yes                         | Console over virtio transport.                               |
| virtio-audio    | Y    | N      | No (acrn-kernel)            | Audio device.                                                |
| virtio-gpio     | Y    | N      | No (acrn-kernel)            | gpio sharing over virtio.                                    |
| virtio-i2c      | Y    | N      | No (acrn-kernel)            | i2c devices.                                                 |
| virito-rpmb     | Y    | N      | No (acrn-kernel)            | Replay Protected Memory Block.                               |
| virtio-mei      | Y    | N      | No (acrn-kernel)            | Intel Management Engine Interface.                           |
| virtio-ipu      | Y    | N      | No (acrn-kernel)            | Imaging Processing Unit.                                     |
| virtio-hyperdma | Y    | N      | No (acrn-kernel)            | Hyper DMA buffer.                                            |
| virtio-hdpc     | Y    | N      | No												  | High-bandwidth Digital Content Protection.                   |
| virtio-coreu    | Y    | N      | No                          | PAVP ( Protected Audio Video Path) session management.       |

<br>


<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-66555622-4"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'UA-66555622-4');
</script>
