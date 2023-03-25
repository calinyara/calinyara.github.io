---
layout: post
title:  "CPU Sharing And vCPU Scheduling In ACRN Hypervisor"
categories: Technology
tags: ACRN hypervisor virtualization scheduling scheduler BVT IORR vcpu
author: Calinyara
description:
---

## 1. Background
<br>

CPU sharing allows the virtual CPUs (vCPUs) of different VMs to run on the same physical CPU, just like how multiple processes run concurrently on a single CPU. Internally the hypervisor adopts time slicing scheduling and periodically switches among those vCPUs.

This feature can help improve overall CPU utilization when the VMs are not fully loaded. However, sharing a physical CPU among multiple vCPUs increases the worst-case response latency of them, and thus is not suitable for vCPUs running latency-sensitive workloads.

## 2. ACRN vCPU Scheduling Overview
<br>

<div align="center"><img src="/assets/images/20200801-acrn-scheduling/1.png"/></div>
<br>
<p align="center">Fig 1: ACRN vCPU Scheduling Overview</p>
<br>

## 3. ACRN Scheduling Framework
<br>

**The design principle of ACRN scheduling framework**

- Follow the principle of modularity and Decoupling vCPU layer and specific scheduling algorithms
- Abstract the scheduling objects, new scheduler can be easily extended.
- A scalable context switch architecture
- Maintain the ***thread_object*** state machine

<br>
<div align="center"><img src="/assets/images/20200801-acrn-scheduling/2.png"/></div>
<br>
<p align="center">Fig 2: ACRN Scheduling Framework</p>
<br>

<div align="center"><img src="/assets/images/20200801-acrn-scheduling/3a.png"/></div>
<br>
<div align="center"><img src="/assets/images/20200801-acrn-scheduling/3b.png"/></div>
<br>
<p align="center">Fig 3: Relationships Between Key Data Structures</p>
<br>

<div align="center"><img src="/assets/images/20200801-acrn-scheduling/4.png"/></div>
<br>
<p align="center">Fig.4 Scheduling Object State Transition</p>
<br>

## 4. Scheduling Framework API
<br>

<div align="center"><img src="/assets/images/20200801-acrn-scheduling/5.png"/></div>
<br>
<p align="center">Fig.5 Scheduling Framework API</p>
<br>

- ***vcpu*** derived from ***thread_obj***
- ***thread_obj*** is the scheduling entity from the scheduling framework perspective
- scheduler is abstracted from the scheduling framework, several callbacks need to be implemented by each scheduler.

<br>

## 5. Scheduling Points and Scheduling Request Submitting Points
<br>

<div align="center"><img src="/assets/images/20200801-acrn-scheduling/C5.png"/></div>
<br>

## 6. Registers Save and Restore In Context Switch
<br>

The vCPU layer will register two function ***switch_in*** and ***switch_out*** for platform architecture level context switch handling.

<div align="center"><img src="/assets/images/20200801-acrn-scheduling/C6.png"/></div>
<br>

## 7. The Schedulers
<br>

### **The IORR Scheduler**

<br>
<div align="center"><img src="/assets/images/20200801-acrn-scheduling/C7a.png"/></div>
<br>

### The BVT Scheduler

<br>
<div align="center"><img src="/assets/images/20200801-acrn-scheduling/C7b.png"/></div>
<br>

### The Priority Based Scheduler
<br>

This scheduler supports **vCPU** scheduling based on their static priorities defined in the scenario configuration. A vCPU can be running only if there is no higher-priority vCPU running on the same physical CPU.

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