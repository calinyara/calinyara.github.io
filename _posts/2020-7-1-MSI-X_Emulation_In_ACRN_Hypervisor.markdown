---
layout: post
title:  "MSI-X Emulation In ACRN Hypervisor"
categories: Technology
tags: PCI PCIe MSI MSIX interrupt ACRN hypervisor virtualization kvm iommu en
author: Calinyara
description:
---

### 1. Background
<br>

- There are  some platforms
    - Multiple MSI Vectors
    - No MSI-X support
    
- OS kernel like Linux doesn’t support continuous vector allocation.

<br>

### 2. Solutions
<br>

- **Native platforms**:
    - IOMMU can mitigate the issue by using interrupt remapping feature (continuous interrupt remapping Entries).
- **KVM**:
    - vIOMMU
- **ACRN Hypervisor**:
    - vIOMMU is not good since a small TCB is preferred.
    - MSI-X emulation on MSI could be a solution.

<br>

### 3. MSI-X Emulation on MSI Overview
<br>

- ACRN MSI-x Emulation on MSI
    - Hide MSI Capability
    - Present MSI-X Capability
    - Trigger EPT Violation when guest accesses  MSI-X table and do MSI-X table virtualization
    - Support multiple MSI vector via IOMMU  interrupt remapping (continuous IRTEs)
    - No PBA emulation, return 0 if read by guest
- Device Requirements
    - Common PCI device passthrough  requirements
    - The device should support Per-Vector Masking
    - One free BAR available
- Guest
    - Device driver requests MSI-X vectors

<br>
<div align="center"><img src="/assets/images/20200701-msix-emulation-in-acrn/overview.png"/></div>
<br>
<p align="center">Fig 1: MSI-X Emulation on MSI Overview</p>
<br>

### 4. MSI-X Table BAR
<br>

- 4KB, 32bit MMIO BAR
    - MSI-x table offset 0
    - PBA table offset 0x800 (not emulated, return 0 on read)
    
- Base Address
    - For Service VM: when initialization, it is programmed as 0, then OS will program  the value later and the value is stored in vdev->vbars[MSI-X_BAR_ID]. base_gpa.  When the device is assigned to UOS and then assigned back to SOS, the stored  base.
    - For Post-launched VM: The GPA is assigned by device model.
    - For Pre-launched VM: The GPA is defined in VM configuration.
    
- EPT mapping is remove for the corresponding GPA range to trigger EPT violation VM-Exit.

<br>

### 5. Code Changes in The Guest VM
<br>

- Device driver should requests MSI-X vectors with minor change.

  <div align="center"><img src="/assets/images/20200701-msix-emulation-in-acrn/code1.png"/></div>
  
- Such kind of change will not impact the functionality on native platform, easy to upstream.

<br>

###  6. Summary
<br>

- MSI-X is emulated on MSI in ACRN.
- The MSI-x emulation can be enabled on demand.
- No change or minor change in guest driver.
- Interrupt handling can be distributed to different cores.

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