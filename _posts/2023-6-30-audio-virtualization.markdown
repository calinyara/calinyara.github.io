---
layout: post
title:  "Audio Virtualization"
categories: Technology
tags: audio virtualization 虚拟化 ALSA SOF virtio en automotive
author: Calinyara
description:
---

<br>

### **ALSA, ASoC and SOF** 

<br>

**ALSA** (Advanced Linux Sound Architecture) is a general-purpose audio framework that provides a unified interface for accessing audio hardware. It is used by a wide range of applications, including media players, games, and recording software.

**ALSA** provides audio and MIDI functionality to the Linux operating system. **ASoC** (ALSA System-on-Chip) is a subsystem of **ALSA**. **ASoC** provides a modular architecture to share audio codec drivers across different SoC implementations, unify the controls provided to applications, and provide a common infrastructure to manage SoC audio component power and clocks.

<br>

<div align="center"><img src="/assets/images/20230630-audio-virtualization/1.png"/></div>
<p align="center">Fig.1：Audio Driver Architecture</p>

<br>

**SOF** (Sound Open Firmware) is an open source audio Digital Signal Processing (DSP) firmware infrastructure and SDK. SOF provides infrastructure, real-time control pieces, and audio drivers. A generic **SOF** subsystem is implemented in Linux as a subsystem of **ALSA ASoC**.

<br>



<div align="center"><img src="/assets/images/20230630-audio-virtualization/2.png"/></div>
<br>
<p align="center">Fig.2：Native Audio SOF Driver</p>

<br>

- **ASoC machine driver**: a glue driver to integrate codec, platform and DAIs.
- **ASoC PCM Driver**: an abstract layer for PCM  data processing and PCM stream control.
- **Generic IPC Driver**: the communication channel  with DSP.
- **DSP Platform Driver** : platform specific handing. Something like quirk.

<br>

### **Audio Virtualization**

<br>

**Monolithic Hypervisors**, such as **KVM**, are typically built on top of general-purpose monolithic operating systems, such as Linux. They provide hardware access to the host and CPU virtualization in the kernel, and they rely on user-space software, such as QEMU, to emulate guest I/O in the host OS.

<div align="center"><img src="/assets/images/20230630-audio-virtualization/3.png"/></div>
<br>
<p align="center">Fig.3：Monolithic Hypervisor Based Audio Virtualization Architecture</p>

<br>

Micro-kernelized Hypervisors, such as **[Xen](https://xenproject.org/)** and **[ACRN](https://projectacrn.org/)**, are typically lightweight microkernels that provide basic hardware access to the host and CPU virtualization in the kernel. They rely on a management guest, such as Xen's Dom0 and ACRN's Service OS, to provide the rest of the functionality, such as complete hardware access, a management interface, and guest I/O emulation.

<br>

Following is a reference micro-kernelized hypervisor based audio virtualization solution for automotive cases.

<div align="center"><img src="/assets/images/20230630-audio-virtualization/4.png"/></div>
<br>
<p align="center">Fig.4：Micro-kernelized Hypervisor Based Audio Virtualization Architecture</p>

<br>

- Coordinated stream management among audio application in different Guests.
- Audio application in Service OS has the highest privilege level (e.g. Phone calls over audio playback).
- Virtio based platform-agnostic audio drivers.
- Put the backend in Service OS kernel side for better performance. Zero-copy is required.
- Physical audio device is assigned to Service OS by using pass-through techology. 



<br>

<!-- Global site tag (gtag.js) - Google Analytics -->

<script async src="https://www.googletagmanager.com/gtag/js?id=UA-66555622-4"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'UA-66555622-4');
</script>
<br>

<!-- Google tag (gtag.js) -->

<script async src="https://www.googletagmanager.com/gtag/js?id=G-27WH7FZ7KT"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'G-27WH7FZ7KT');
</script>
