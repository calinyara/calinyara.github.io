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

| ACRN | Crosvm | Device          | Description                                                  |
| :--: | :----: | :-------------- | ------------------------------------------------------------ |
|  Y   |   Y    | virtio-block    | Basic read/write block device.                               |
|  Y   |   Y    | virtio-net      | Device to interface the host and guest networks.             |
|  Y   |   Y    | virtio-rng      | Entropy source used to seed guest OS's entropy pool.         |
|  N   |   Y    | virtio-vsock    | Enabled VSOCKs for host/guest communication.                 |
|  N   |   Y    | virtio-wayland  | Allowed guest to use host Wayland socket.                    |
|  N   |   Y    | virtio-balloon  | Dynamic guest RAM allocation.                                |
|  N   |   Y    | virtio-9p       | Sharing host files with the guest as remote network filesystem.     |
|  N   |   Y    | virtio-gpu      | A display virtualization solution.                           |
|  N   |   Y    | virtio-tpm      | Trusted Platform Module.                                     |
|  N   |   Y    | virtio-fs       | Sharing host files with the guest as local filesystem.       |
|  N   |   Y    | virtio-pmem     | A fake persistent memory(nvdimm) in guest which allows to bypass  the guest page cache. |
|  Y   |   Y    | virtio-input    | Input device.                                                |
|  Y   |   N    | virtio-console  | Console over virtio transport.                               |
|  Y   |   N    | virtio-audio    | Audio device.                                                |
|  Y   |   N    | virtio-gpio     | gpio sharing over virtio.                                    |
|  Y   |   N    | virtio-i2c      | i2c devices.                                                 |
|  Y   |   N    | virito-rpmb     | Replay Protected Memory Block.                               |
|  Y   |   N    | virtio-mei      | Intel Management Engine Interface.                           |
|  Y   |   N    | virtio-ipu      | Imaging Processing Unit.                                     |
|  Y   |   N    | virtio-hyperdma | Hyper DMA buffer.                                            |
|  Y   |   N    | virtio-hdpc     | High-bandwidth Digital Content Protection.                   |
|  Y   |   N    | virtio-coreu    | PAVP ( Protected Audio Video Path) session management.       |

<br>


<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-66555622-4"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'UA-66555622-4');
</script>
