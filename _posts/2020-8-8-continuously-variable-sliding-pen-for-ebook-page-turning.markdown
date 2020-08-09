---
layout: post
title:  "Continuously Variable Sliding Pencil for E-book Page Turning"
categories: Technology
tags: human-machine-interface HMI control-device pen pencil apple iPad
author: Calinyara
description:
---

### **1. Background**

<br>

When I read an e-book on a large screen, I found that turning the pages of the book was a nightmare for me. In order to turn pages, I have to stand very close to the screen and touch the screen with my fingers. Or use either keyboard or mouse and add an extra desk to place them. Or use some button control device based on Bluetooth by pressing the button once turning one whole page without smooth and continuous turning experience. So, I think it's necessary to invent a new tool to help me read books more easily and smoothly on the large screen.

<br>

### **2. Introduction**

<br>

This article elaborates a new screen contactless page turning pencil for E-book reading. It is a screen contactless, portable, easy-to-use, smooth and continuous control device for e-book page turning which can help the customers to control their device more conveniently and smoothly. The innovative technology ( the method of **sensor data acquisition** and **sensor data processing** ) designed for smooth and continuous control can be widely used in various embedded control devices for different purposes ( e.g. as a pencil to control the Safari browser on an iPad ).

<br>

Compared with the existing solutions, it has the following advantages:

- Screen contactless. No need to touch the screen.
- Wide application scenarios. It can be also used on devices without touch screen.
- Better portability than mouse and keyboard. Just put the pencil in your pocket.
- Better ease of use than mouse and keyboard, especially in mobile scenes.
- Better smooth and continuous control ability than button control device based on Bluetooth. 

<br>

<p align="center">Table 1: Comparison of Different Solutions</p>
<div align="center"><img src="/assets/images/20200808-sliding-controlling-pen/fig0-comparison-of-different-solutions.png"/></div>
<br>

### **3. Details**

<br>

The core innovative technology designed to create this innovative product “**Continuously Variable Sliding Pencil for E-book Page Turning**” are the method of **sensor data acquisition** and the method of **sensor data processing** which are described in section in 3.1 and 3.3 correspondingly.

This innovative product “**Continuously Variable Sliding Pencil for E-book Page Turning**” uses some **standard** short distance wireless communication protocols to exchange data with the host screen device. In a specific implementation, it can be **standard** Bluetooth or Zigbee or Wi-Fi or other standard short distance wireless communication protocols, as described in 3.2.

<br>

#### **3.1 Sensor Data Acquisition**

<br>
<div align="center"><img src="/assets/images/20200808-sliding-controlling-pen/fig1.png"/></div>
<p align="center">Figure 1: Pressure Area on The Pencil</p>
<br>

There is a pressure sensitive area on the pencil for collecting pressure data. The pressure sensitive area is divided into three blocks.

<br>
<div align="center"><img src="/assets/images/20200808-sliding-controlling-pen/fig2.png"/></div>
<p align="center">Figure 2: Blocks in Pressure Sensitive Area</p>
<br>

Pressure data will be sampled periodically. So the data is a time series, D0, D1, D2, D3...

<br>

<div align="center"><img src="/assets/images/20200808-sliding-controlling-pen/fig3.png"/></div>
<p align="center">Figure 3: Sensor Data Series</p>
<br>

#### **3.2 Sensor Data Transmission**
<br>

The pencil connects to the host device by using any standard short distance wireless communication protocol (e.g. Bluetooth, Zigbee or Wi-Fi). The data will be sent to the host device with period T.
<br>

<div align="center"><img src="/assets/images/20200808-sliding-controlling-pen/fig4.png"/></div>
<p align="center">Figure 4: Sensor Data Transmission</p>
<br>

#### **3.3 Sensor Data Processing**
<br>
#### **3.3.1 One-hot encoding the Pressure data**
<br>

<div align="left"><img src="/assets/images/20200808-sliding-controlling-pen/text1.png"/></div>

<br>
#### **3.3.2 State Machine for controlling page turning**
<br>
In order to control the book page turning, we need to calculate the direction and speed of fingers sliding on the pencil. Following is a state machine for this purpose.
<br>
<br>
<div align="left"><img src="/assets/images/20200808-sliding-controlling-pen/text2.png"/></div>
<br>
<div align="center"><img src="/assets/images/20200808-sliding-controlling-pen/fig5.png"/></div>
<p align="center">Figure 5: State Machine for Controlling Page Turning</p>
<br>

### **4. Page Turning Effect**
<br>
The Fig.6 shows the page turning effect.
<br>
<div align="center"><img src="/assets/images/20200808-sliding-controlling-pen/fig6.png"/></div>
<p align="center">Figure 6: Design Sketch to Show The Page Turning Effect</p>
<br>

<div align="center">
<video class="center" src="/assets/images/20200808-sliding-controlling-pen/demo.mp4" controls poster="/assets/images/20200808-sliding-controlling-pen/fig6.png" width="600">⁪</video>
</div>
<p align="center">Demo</p>

<br>
<!-- Global site tag (gtag.js) - Google Analytics -->

<script async src="https://www.googletagmanager.com/gtag/js?id=UA-66555622-4"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'UA-66555622-4');
</script>