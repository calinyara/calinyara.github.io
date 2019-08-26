---
layout: post
title:  "虚拟网络设备简介"
categories: Network
tags: Network
author: Calinyara
description: 
---

# 概述

<br>
Linux提供了许多虚拟网络设备用于运行VMs和containers。下面对这些网络虚拟化设备做相应介绍。

<br>
## Bridge

**Bridge**类似于一个网络交换机，用于交换数据包，连接不同的VMs, containers以及Host.

<br>
<div align="center"><img src="/assets/images/vnet_interface/f0001_bridge.png"/></div>
<p align="center">图1：Bridge</p>

<br>
```
# ip link add br0 type bridge
# ip link set eth0 master br0
# ip link set tap1 master br0
# ip link set tap2 master br0
# ip link set veth1 master br0
```

<br>
## Bonded Interface

**Bonded Interface**用于将多个网络接口聚合成一个逻辑上的"bonded"接口。可用于故障备份或负载均衡等场景。

<br>
<div align="center"><img src="/assets/images/vnet_interface/f0002_bond.png"/></div>
<p align="center">图2：Bonded Interface</p>

<br>
## Team Device

与**Bonded Interface**类似，将多个网络接口聚合成一个逻辑接口。参见 [Bonded Interface vs Team Device](https://github.com/jpirko/libteam/wiki/Bonding-vs.-Team-features) 

<br>
<div align="center"><img src="/assets/images/vnet_interface/f0003_team.png"/></div>
<p align="center">图3：Team Device</p>

<br>
Linux中还有一种叫[**net_failover**](https://www.kernel.org/doc/html/latest/networking/net_failover.html)的虚拟设备，如下图所示，将半虚拟化的网卡方案和passthru的方案聚合成一个界面。增加了抵抗设备出错的风险。

<br>
<div align="center"><img src="/assets/images/vnet_interface/f0004_net_failover.png"/></div>
<p align="center">图4：net_failover</p>

<br>
## VLAN

**VLAN**被用来划分子网，可以减少广播包对网络的压力。其通过往以太网数据包头添加一个TAG来实现过滤。下图是**VLAN**数据包格式:

<br>
<div align="center"><img src="/assets/images/vnet_interface/f0005_vlan_01.png"/></div>
<p align="center">图5：VLAN数据包格式</p>

<br>

可以使用VLAN来分割VMs, Containers以及host中的网络。以下是一个例子

<br>
```
# ip link add link eth0 name eth0.2 type vlan id 2
# ip link add link eth0 name eth0.3 type vlan id 3
```
<br>
<div align="center"><img src="/assets/images/vnet_interface/f0006_vlan.png"/></div>
<p align="center">图6：VLAN</p>

<br>
## VXLAN

**VXLAN** (Virtual eXtensible Local Area Network),见[IETF RFC 7348](https://tools.ietf.org/html/rfc7348)，是一种隧道协议。其解决了在划分大规模网络VLAN IDs不足（只有4096）的缺点。VXLAN有一个24-bit的 ID, 一共支持2^24(16777216)个virtual LANs。 其将L2的数据包添加上 **VXLAN**头，然后整个数据包作为一个UDP IP包在网络上传输。

**VXLAN**数据包格式如下图所示

<div align="center"><img src="/assets/images/vnet_interface/f0007_vxlan_01.png"/></div>
<p align="center">图7：VXLAN数据包格式</p>

<br>
VXLAN可以跨本地网络建立子网。

<div align="center"><img src="/assets/images/vnet_interface/f0008_vxlan.png"/></div>
<p align="center">图8：VXLAN</p>

<br>
```
# ip link add vx0 type vxlan id 100 local 1.1.1.1 remote 2.2.2.2 dev eth0 dstport 4789
```

<br>
参考 [Introduction to Cloud Overlay Networks - VXLAN](https://www.youtube.com/watch?v=Jqm_4TMmQz8&t=362s)

<br>
## VETH

**VETH**（virtual Ethernet）是一种本地以太网隧道。该设备成对出现，用于连接不同的namespace。如下图所示, 其一端收到的数据会直接发送给其peer端。

<div align="center"><img src="/assets/images/vnet_interface/f0021_veth.png"/></div>
<p align="center">图9：VETH</p>

<br>
```
# ip netns add net1
# ip netns add net2
# ip link add veth1 netns net1 type veth peer name veth2 netns net2
```

<br>
## MACVLAN

使用**VLAN**， 你可以在一个网络接口上建立多个虚拟网络接口，并通过VLAN tag来过滤数据包。使用**MACVLAN**你可以在一个网络接口上建立多个拥有不同MAC地址的界面。

在**MACVLAN**之前，如果你想连接两个不同的namespace，需要时用 **Bridge + VETH**, 如下图所示

<div align="center"><img src="/assets/images/vnet_interface/f0009_br_ns.png"/></div>
<p align="center">图10：VETH连接namespces</p>

而使用**MACVLAN**取代 **Bridge + VETH**如下图所示

<div align="center"><img src="/assets/images/vnet_interface/f0010_macvlan.png"/></div>
<p align="center">图11：MACVLAN</p>

**MACVLAN** 有5中不同的模式，在隔离性上有所不同。
<br>

**1 Private Mode**: 该模式下，各个端点只能与外界通信。MACVLAN端点之间不能通信。

<div align="center"><img src="/assets/images/vnet_interface/f0011_macvlan_01.png"/></div>
<p align="center">图12：MACVLAN Private Mode</p>

**2 VEPA Mode**: 该模式下，各个端点可以与外界通信。MACVLAN端点之间通信需要外部Switch支持一个叫发夹弯（hairpin）的功能才可以。

<div align="center"><img src="/assets/images/vnet_interface/f0012_macvlan_02.png"/></div>
<p align="center">图13：MACVLAN VEPA Mode</p>

**3 Bridge Mode**: 该模式下，各个端点之间可以通信，与外界也可以通信。功能和**Bridge + VETH**一致。

<div align="center"><img src="/assets/images/vnet_interface/f0013_macvlan_03.png"/></div>
<p align="center">图14：MACVLAN Bridge Mode</p>

**4 Passthru Mode**: 该模式可以让Container直接连接外部Switch。

<div align="center"><img src="/assets/images/vnet_interface/f0014_macvlan_04.png"/></div>
<p align="center">图15：MACVLAN Passthru Mode</p>

**5 Source Mode**: 该模式主要用于traffic过滤，是一种基于MAC的VLAN。可参考 [源代码](https://git.kernel.org/pub/scm/linux/kernel/git/davem/net.git/commit/?id=79cf79abce71)

<br>
综上模式，Bridge Mode是最常用的模式。

```
# ip link add macvlan1 link eth0 type macvlan mode bridge
# ip link add macvlan2 link eth0 type macvlan mode bridge
# ip netns add net1
# ip netns add net2
# ip link set macvlan1 netns net1
# ip link set macvlan2 netns net2
```

<br>
## IPVLAN

**IPVLAN**与 **MACVLAN**类似， 区别在于 **IPVLAN**各端点的MAC地址是相同的，与其parent设备一致。

<div align="center"><img src="/assets/images/vnet_interface/f0015_ipvlan.png"/></div>
<p align="center">图16：IPVLAN</p>

<br>
**IPVLAN**可以工作在L2或L3模式。如果工作在L2模式，其parent设备的行为可以看成是Bridge或Switch，与 **MACVLAN**的区别在于其通过IP地址来过滤数据包，而后者通过MAC地址过滤。如果工作在L3模式，其parent设备的行为可以看成是一个Router。

<br>
<div align="center"><img src="/assets/images/vnet_interface/f0016_ipvlan_01.png"/></div>
<p align="center">图17：IPVLAN working in L2 Mode</p>

<br>
<div align="center"><img src="/assets/images/vnet_interface/f0017_ipvlan_02.png"/></div>
<p align="center">图18：IPVLAN working in L3 Mode</p>

<br>
**MACVLAN**与 **IPVLAN**在很多方面相似。下面介绍几需使用 **IPVLAN** 而非 **MACVLAN**的场景。

- 1) 如果host连接的外部交换机只允许一个MAC地址。
- 2) 如果出于性能考虑，需要关闭网卡杂项模式（promiscuous mode)， **MACVLAN**需要网卡运行在杂项模式。
- 3) 创建的virtual device 超过了parent的MAC地址容量。
- 4) 如果工作在不安全的L2网络

<br>
```
# ip netns add ns0
# ip link add name ipvl1 link eth0 type ipvlan mode l2
# ip link set dev ipvl0 netns ns0
```

<br>
## MACVTAP/IPVTAP

类似 **MACVLAN**用以取代 **Bridge + VETH** 来连接不同的namespace. **MACVTAP** 用以取代 **Bridge + TUN/TAP**来连接不同的VMs。**MACVLAN / IPVLAN** 连接不同的namespace, 旨在将Guest namespace和Host的网络接口直接程序给外部Switch。**MACVTAP / IPVTAP** 连接不同VMs, 内核为其创建了设备文件/dev/tapX，可以直接被虚拟化软件QEMU使用。**MACVTAP**与 **IPVTAP**的区别 和 **MACVLAN** 与 **IPVLAN**的区别相类似。

<br>
<div align="center"><img src="/assets/images/vnet_interface/f0018_macvtap.png"/></div>
<p align="center">图19：MACVTAP</p>

<br>
```
# ip link add link eth0 name macvtap0 type macvtap
```

<br>
## MACsec

**MACsec** (Media Access Control Security)， 是一个IEEE制定的以太网安全标准。其与IPsec类似，但其不仅可以保护IP数据包，还可以保护其他IP层之外的协议包，如ARP, 邻居发现协议包（neighbor discovery），DHCP包。其封包结构如下图所示:

<div align="center"><img src="/assets/images/vnet_interface/f0019_macsec_01.png"/></div>
<p align="center">图20：MACsec封包</p>

<br>
其主要也是用在保护APR，NS, DHCP等以太网数据包

<div align="center"><img src="/assets/images/vnet_interface/f0020_macsec.png"/></div>
<p align="center">图21：MACsec</p>

<br>
```
# ip link add macsec0 link eth1 type macsec
```

<br>
## VCAN

与网络loopback设备类似，VCAN(virtual CAN)驱动提供一种虚拟CAN接口叫VCAN。具体参见[内核CAN文档](https://www.kernel.org/doc/Documentation/networking/can.txt)

<br>
## VXCAN

VXCAN可以作为一种本地隧道连接两个VCAN网络设备。与VETH类似，你会创建一个VXCAN对， 用于连接不同namespaces的VCAN。

```
# ip netns add net1
# ip netns add net2
# ip link add vxcan1 netns net1 type vxcan peer name vxcan2 netns net2
```

<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-66555622-4"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'UA-66555622-4');
</script>
