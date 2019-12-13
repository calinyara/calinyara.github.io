---
layout: post
title:  "Play with Crosvm"
categories: Technology
tags: Virtualization crosvm buildroot ChromeOS linux kernel rootfs
author: Calinyara
description: 
---

<br>

### **Background**

<br>

[**Crosvm**](https://chromium.googlesource.com/chromiumos/platform/crosvm/) is a device model based on [**Rust**](https://www.rust-lang.org/) language. [**Chrome OS**](https://www.chromium.org/chromium-os) uses **Crosvm** along with **KVM** to provide virtualization solution.

<br>

### **Install Rust and Dependencies**

```shell
sudo apt-get install -y libcap-dev libfdt-dev
curl https://sh.rustup.rs -sSf | sh
source $HOME/.cargo/env 
```

<br>

### **Build** **Crosvm**

```shell
git clone https://github.com/calinyara/crosvm.git
cd crosvm
./build.sh
```

 Compiled **crosvm** will be located in crosvm/target/release/.

<br>

### **Install** **Crosvm**

```shell
sudo cp minijail/libminijail.so /usr/lib
sudo cp crosvm/target/release/crosvm /usr/bin
```

<br>

### **Build Kernel and Rootfs with Buildroot**

```shell
git clone https://github.com/buildroot/buildroot.git
cd buildroot
wget https://raw.githubusercontent.com/calinyara/crosvm/master/configs/buildroot.config
mv buildroot.config .config
cd board/pc
wget https://raw.githubusercontent.com/calinyara/crosvm/master/configs/linux.config-5.3.x-x86_64
cd ../..
make
```

Compiled **bzImage** and **rootfs.ext4** will be located in output/images.

<br>

### **Try Crosvm**

*// Start a VM, login with root without password*

```shell
crosvm run --disable-sandbox --rwroot rootfs.ext4 -s crosvm.sock bzImage
```

*// Stop the VM, in another shell window run following command*

```shell
crosvm stop crosvm.sock
```
<br>

<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-66555622-4"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'UA-66555622-4');
</script>