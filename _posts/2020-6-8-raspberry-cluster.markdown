---
layout: post
title:  "基于树莓派搭建小型云计算集群"
categories: Technology
tags: 树莓派 raspberry pi cloud cluster kubernetes k8s k3s 边缘计算
author: Calinyara
description:
---

<br>

拥有一个私人的云计算平台是一件很酷的事情。随着技术的发展，实现这一愿望已经变得相当容易。接下来就来说明如何利用树莓派硬件和相关软件搭建一个用于边缘计算的小型云计算集群。

<br>

### **1 硬件准备**

<br>

硬件优先考虑树莓派。选择ARM而不是x86架构硬件，主要是考虑到该云计算平台主要用于私人，家庭以及边缘计算等应用场景。一方面，ARM硬件相对便宜，功耗低，性价比更高；另一方面树莓派拥有成熟的社区生态，可用的软件也比较丰富。

<br>

#### **选择1. 树莓派3B及其之前的版本**

<br>

树莓派3B及其之前的版本由于不支持以太网口供电(PoE), 因此需要额外的USB供电插头。所有树莓派板子都连接到一个交换机/路由器，如下图所示

<br>
<div align="center"><img src="/assets/images/raspberrypi-cluster/picluster3b.jpg"/></div>
<p align="center">图1: 基于树莓派3B搭建的集群</p>
<br>

#### **选择2. 树莓派3B+，树莓派4B**

<br>
树莓派3B+/4B拥有以太网口供电(PoE)功能, 因此可省去USB供电插头。所有树莓派板子都连接到一个支持PoE功能的交换机/路由器，如下图所示

<br>
<div align="center"><img src="/assets/images/raspberrypi-cluster/picluster3bplus.jpg"/></div>
<p align="center">图2: 基于树莓派3B+/4B搭建的集群</p>
<br>

#### **选择3. Turing Pi主板  +  树莓派计算模块**

<br>

**关于树莓派计算模块**

<br>

上面介绍的树莓派3B, 3B+, 4B等板子其实可以拆解成如下两部分，即: 计算模块和计算模块IO扩展板.

<br>
<div align="center"><img src="/assets/images/raspberrypi-cluster/compute_module_poe3b.jpg"/></div>
<div align="center"><img src="/assets/images/raspberrypi-cluster/compute_module_poe3bplus.jpg"/></div>
<p align="center">图3: 树莓派计算模块</p>
<br>

<div align="center"><img src="/assets/images/raspberrypi-cluster/compute_module_poe_board1.png"/></div>
<div align="center"><img src="/assets/images/raspberrypi-cluster/compute_module_poe_board2.png"/></div>
<p align="center">图4: 树莓派计算模块IO扩展板</p>
<br>
将计算模块和计算模块IO扩展板结合起来功能就和上述的树莓派3B, 3B+, 4B 一致。
<br>

<div align="center"><img src="/assets/images/raspberrypi-cluster/compute_module_poe_board3.png"/></div>
<p align="center">图5: 树莓派计算模块 + IO扩展板</p>
<br>

**关于Turing Pi主板**

<br>

利用Turing Pi主板加可扩展树莓派计算模块的方式搭建集群十分的方便。该板最大支持7块树莓派计算模块，并可进行动态扩展，类似于数据中心的**刀片服务器 (blade server)**。

<br>

<div align="center"><img src="/assets/images/raspberrypi-cluster/turing_pi.jpg"/></div>
<p align="center">图6: 树莓派计算模块 + Turing Pi 主板</p>
<br>

Turing Pi同时支持带eMMC的计算模块和不带eMMC的计算模块，其第一个槽可用于烧写操作系统镜像到计算模块eMMC。对于不带eMMC的计算模块可以通过传统的插SD卡的方式启动。

<br>

### **2 烧写树莓派系统**

<br>

#### **2.1 系统选择**

<br>

推荐如下两个系统

- [Raspbian](https://www.raspberrypi.org/downloads/raspberry-pi-os/): 官方系统
- [HypriotOS](https://blog.hypriot.com/): 容器操作系统

<br>

#### **2.2 烧写系统**

<br>

**选择1. 烧写系统到SD卡**

具体步骤参考如下页面

https://www.raspberrypi.org/documentation/installation/installing-images/README.md

<br>

**选择2. 烧写系统到树莓派计算模块的eMMC**

具体步骤参考如下页面 (需要借助树莓派计算模块IO扩展板或者Turing Pi主板)

http://raspberrypiwiki.com/How_to_Burning_System_for_the_eMMC_of_Raspberry_Pi_Compute_Module

<br>

所有单板都烧写好后按照硬件准备中的描述连接好，并将每个板子配置好ssh连接，将公钥放置在***~/.ssh/authorized_keys***里面，以方便连接。确保可以通过***ssh <user_name>@<ip_address>***能连上集群中每一个树莓派节点。

<br>

### 3 安装Kubernetes并连接集群

<br>

#### 3.1 安装Kubernetes

<br>

[Lightweight Kubernetes (K3S)](https://k3s.io/) 是一个面向IoT及边缘计算的Kubernetes版本，比较适合树莓派等资源有限的硬件。

<br>

**方式1：在每个树莓派板子上单独安装**

<br>

在server节点上运行

```bash
curl -sfL https://get.k3s.io | sh -
```

在每个worker节点上运行

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<server_ip>:6443 K3S_TOKEN=<token> sh -
```

PS: **K3S_TOKEN**: <token>存在server节点的 **/var/lib/rancher/k3s/server/node-token**.

<br>

**方式2：使用Ansible自动化安装**

<br>

先在控制机上安装[Ansible](https://github.com/ansible/ansible)
<br>

```bash
sudo apt update
sudo apt install software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

<br>

下载[k3s-ansible](https://github.com/rancher/k3s-ansible)

```bash
git clone https://github.com/rancher/k3s-ansible.git
```
<br>

然后按照 https://github.com/rancher/k3s-ansible 的步骤，在***inventory/my-cluster/hosts.ini***里配置好server (master) 节点和worker(node)节点的IP地址。

<br>

运行如下命令，Ansible会将K3S自动安装在集群的server节点和每个worker节点上
```bash
ansible-playbook site.yml -i inventory/my-cluster/hosts.ini --ask-become-pass
```

<br>

**3.2 连接集群**

<br>

在控制机上安装kubectl

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https gnupg2
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```

<br>

将配置文件从server节点拷贝至控制机并配置环境变量

```bash
scp <user_name>@<server_ip>:~/.kube/config ~/.kube/rasp-config
export KUBECONFIG=~/.kube/rasp-config
```

<br>

连接查看集群

```bash
kubectl get nodes
```

<br>

大功告成，接下来就可以部署服务到集群了。

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