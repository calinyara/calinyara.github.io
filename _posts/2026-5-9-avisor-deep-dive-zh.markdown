---
layout: post
title:  "从零构建 AArch64 Type-1 Hypervisor 的技术深度解析"
categories: Technology
tags: 内核 kernel Linux 操作系统 树莓派 raspberry pi buildroot hypervisor 虚拟机 虚拟化 virtualization OS 调度 arm zh avisor
author: Calinyara
description:
---


> 本文深入剖析 aVisor —— 一个面向 IoT 场景的轻量级 AArch64 Type-1 Hypervisor。从 ARM 虚拟化基础原理出发，逐层展开其技术架构与实现细节，涵盖启动流程、异常处理、内存虚拟化、设备模拟、调度器、控制台系统等核心模块。

---

## 目录

1. [ARM 虚拟化基础原理](#1-arm-虚拟化基础原理)
2. [aVisor 总体架构](#2-avisor-总体架构)
3. [启动流程：从上电到 Guest OS 运行](#3-启动流程从上电到-guest-os-运行)
4. [异常处理与 Trap 机制](#4-异常处理与-trap-机制)
5. [内存虚拟化：Stage-2 地址翻译](#5-内存虚拟化stage-2-地址翻译)
6. [设备模拟与 MMIO 拦截](#6-设备模拟与-mmio-拦截)
7. [中断虚拟化与定时器](#7-中断虚拟化与定时器)
8. [调度器与上下文切换](#8-调度器与上下文切换)
9. [多核支持（SMP）](#9-多核支持smp)
10. [控制台与 Shell 系统](#10-控制台与-shell-系统)
11. [文件系统与 VM 加载](#11-文件系统与-vm-加载)
12. [源码结构总览](#12-源码结构总览)

---

## 1. ARM 虚拟化基础原理

### 1.1 ARM 异常级别（Exception Levels）

AArch64 架构定义了四个异常级别，权限由低到高：

```
┌──────────────────────────────────────┐
│  EL3  Secure Monitor (固件/ATF)       │  ← 最高权限，安全世界切换
├──────────────────────────────────────┤
│  EL2  Hypervisor                     │  ← aVisor 运行在这里
├──────────────────────────────────────┤
│  EL1  Guest OS Kernel (Linux)        │  ← Guest 内核
├──────────────────────────────────────┤
│  EL0  User Applications              │  ← Guest 用户态程序
└──────────────────────────────────────┘
```

aVisor 作为 Type-1（裸金属）Hypervisor，直接运行在 EL2，无需宿主操作系统。Guest OS 运行在 EL1/EL0，其特权操作被硬件自动 Trap 到 EL2 由 aVisor 处理。

### 1.2 关键系统寄存器

| 寄存器 | 功能 |
|--------|------|
| **HCR_EL2** | Hypervisor 配置寄存器，控制 Trap 行为和 Stage-2 翻译 |
| **VTTBR_EL2** | Stage-2 页表基地址 + VMID |
| **VTCR_EL2** | Stage-2 翻译控制（页大小、地址宽度等）|
| **ELR_EL2** | 异常返回地址（Trap 时保存 Guest PC）|
| **SPSR_EL2** | 保存的处理器状态（Trap 时保存 Guest PSTATE）|
| **ESR_EL2** | 异常综合信息寄存器（Trap 原因编码）|
| **FAR_EL2** | 故障地址寄存器 |
| **HPFAR_EL2** | Hypervisor IPA 故障地址（用于 Stage-2 故障）|

### 1.3 Stage-2 地址翻译

ARM 虚拟化扩展提供两级地址翻译：

```
Guest VA  ──Stage-1(Guest控制)──►  IPA  ──Stage-2(Hypervisor控制)──►  PA
           (EL1 MMU, TTBR0/1_EL1)        (EL2 MMU, VTTBR_EL2)
```

- **Stage-1**：Guest 自行管理，将虚拟地址（VA）翻译为中间物理地址（IPA）
- **Stage-2**：Hypervisor 管理，将 IPA 翻译为真实物理地址（PA），实现内存隔离

### 1.4 HVC/SMC 调用

- **HVC（Hypervisor Call）**：Guest 主动调用 Hypervisor 服务（如 PSCI 电源管理）
- **SMC（Secure Monitor Call）**：当 `HCR_EL2.TSC=1` 时，SMC 也被 Trap 到 EL2

---

## 2. aVisor 总体架构

### 2.1 设计目标

aVisor 面向 Raspberry Pi 3（BCM2837，4 核 Cortex-A53）设计，目标是在嵌入式平台上以最小开销运行完整的 Linux Guest。核心特征：

- **Type-1 裸金属**：直接运行在硬件 EL2，无宿主 OS
- **1:1 vCPU 绑定**：每个 vCPU 固定绑定到一个物理 CPU 核心
- **全虚拟化**：Guest 无需修改，直接运行标准 AArch64 Linux 内核
- **按需分页**：Guest 内存按访问触发的缺页异常动态分配
- **完整设备模拟**：模拟 BCM2837 的 UART、中断控制器、定时器、Mailbox 等外设

### 2.2 架构全景

```
┌─────────────────────────────────────────────────┐
│                   Guest Linux                    │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│  │ vCPU 0  │  │ vCPU 1  │  │ vCPU 2  │  vCPU 3 │  ← EL1/EL0
│  └────┬────┘  └────┬────┘  └────┬────┘         │
├───────┼────────────┼────────────┼───────────────┤
│       │  HVC/Trap  │            │               │
│  ┌────▼────────────▼────────────▼────────────┐  │
│  │              aVisor Hypervisor             │  │  ← EL2
│  │  ┌──────────┐  ┌──────────┐  ┌─────────┐ │  │
│  │  │ 异常处理  │  │ 内存管理  │  │ 设备模拟 │ │  │
│  │  │(sync_exc) │  │ (mm.c)   │  │(bcm2837)│ │  │
│  │  ├──────────┤  ├──────────┤  ├─────────┤ │  │
│  │  │  调度器   │  │Stage-2 PT│  │ 控制台   │ │  │
│  │  │ (sched)  │  │  (vm.c)  │  │(console)│ │  │
│  │  └──────────┘  └──────────┘  └─────────┘ │  │
│  └───────────────────────────────────────────┘  │
├─────────────────────────────────────────────────┤
│          BCM2837 硬件 (Raspberry Pi 3)           │
│  CPU0  CPU1  CPU2  CPU3  UART  Timer  EMMC ...  │
└─────────────────────────────────────────────────┘
```

### 2.3 内存布局

aVisor 在 Raspberry Pi 3 的 1GB 物理内存中规划了以下布局：

| 地址范围 | 用途 |
|----------|------|
| `0x00000000 - 0x3EFFFFFF` | 普通内存（Guest RAM + Hypervisor） |
| `0x3F000000 - 0x3FFFFFFF` | BCM2837 GPU 外设（MMIO） |
| `0x40000000 - 0x40FFFFFF` | BCM2836 本地外设（定时器、中断、Mailbox） |

Hypervisor 自身使用低地址区域，Guest 的内核镜像加载到 IPA `0x08000000`，DTB 在 `0x3B000000`，initramfs 在 `0x02200000`。

---

## 3. 启动流程：从上电到 Guest OS 运行

### 3.1 EL3 → EL2 转换

aVisor 启动代码位于 `boot.S`，入口 `_start` 在 `.text.boot` 段。在 QEMU raspi3b 上，CPU 从 EL3 启动：

```
_start:
    mrs x0, mpidr_el1       // 读取 CPU ID
    and x0, x0, #3
    ...
    mrs x0, CurrentEL       // 检查当前异常级别
    cmp x0, #3
    beq el3                 // 如果 EL3，配置安全寄存器

el3:
    msr hcr_el2, HCR_VALUE  // 配置 Hypervisor 控制
    msr scr_el3, SCR_VALUE  // 非安全态 + HVC 启用 + AArch64
    msr spsr_el3, SPSR_VALUE// 目标：EL2h，中断全屏蔽
    adr x0, el2_entry
    msr elr_el3, x0
    eret                    // 异常返回，进入 EL2
```

`SCR_EL3` 设置了 `NS=1`（非安全世界）、`HCE=1`（使能 HVC）、`RW=1`（EL2 使用 AArch64）。`SPSR_EL3` 设为 `EL2h` 模式并屏蔽所有中断。`eret` 将执行流切换到 EL2。

### 3.2 EL2 页表与 MMU 初始化

进入 EL2 后，BSP（CPU0）执行：

```
el2_entry:
    // 1. 清零 BSS 段
    adr x0, bss_begin
    adr x1, bss_end
    sub x1, x1, x0
    bl memzero

    // 2. 创建 EL2 页表
    bl __create_page_tables

    // 3. 配置 MMU 控制寄存器
    adrp x0, pg_dir
    msr ttbr0_el2, x0           // 页表基地址
    msr tcr_el2, TCR_VALUE      // 翻译控制
    msr vtcr_el2, VTCR_VALUE    // Stage-2 翻译控制
    msr mair_el2, MAIR_VALUE    // 内存属性

    // 4. 启用 MMU
    mov x0, #SCTLR_MMU_ENABLED
    msr sctlr_el2, x0
    isb

    // 5. 跳转到 C 入口
    br hypervisor_main
```

**`__create_page_tables`** 创建三级页表（PGD → PUD → PMD），使用 2MB 块映射：

- **普通内存**（`0x00000000 - 0x3EFFFFFF`）：`MMU_FLAGS = 0x705`（Normal, Inner Shareable, AF）
- **设备内存**（`0x3F000000 - 0x3FFFFFFF`）：`MMU_DEVICE_FLAGS = 0x701`（Device-nGnRnE, AF）
- **本地外设**（`0x40000000`）：单独一个 2MB 块，设备内存属性

QEMU 版本的链接脚本 `linker_qemu.ld` 将代码起始地址设为 `0x80000`（QEMU 的内核加载地址），因此页表实际是恒等映射（VA = PA）。

### 3.3 `hypervisor_main` 初始化序列

```c
void hypervisor_main(void)
{
    // 1. 初始化每核数据结构
    init_per_cpu_data();

    // 2. 初始化物理 UART（Mini UART, 115200 波特率）
    uart_init();

    // 3. 打印 Logo 和初始化 Shell
    printf(logo);
    shell_init();
    init_hv();

    // 4. 安装异常向量表
    irq_vector_init();

    // 5. 配置定时器
    init_misc_timer();      // BCM2835 系统定时器
    init_hv_timer(0);       // CPU0 的 Hypervisor 物理定时器

    // 6. 启用中断控制器
    enable_interrupt_controller();

    // 7. 挂载 SD 卡 FAT32 文件系统
    f_mount(&fatfs, "/", 1);

    // 8. 创建 VM 并加载 Guest 内核
    for (int i = 0; i < get_avisor_config_amount(); i++) {
        create_vm(i, get_avisor_config(i));
    }

    // 9. 启动其余物理 CPU 核心
    start_secondary_cores(1, secondary_main);
    start_secondary_cores(2, secondary_main);
    start_secondary_cores(3, secondary_main);

    // 10. 进入调度循环
    enable_irq();
    while (1)
        schedule();
}
```

### 3.4 VM 创建与内核加载

`create_vm()` 是 VM 生命周期的起点：

1. **分配 VM 结构**：从全局 `vm_array[]` 获取 slot，设置 VMID
2. **初始化 Stage-2 页表**：将设备区域（`0x3F000000+`）标记为"不可访问"，访问时触发 MMIO Trap
3. **初始化控制台**：分配 `in_fifo` 和 `out_fifo`
4. **创建 vCPU**：为每个物理 CPU 核心创建一个 vCPU

**vCPU 创建**为每个 vCPU 分配一个 `THREAD_SIZE`（4KB）的内核栈，初始化：

- **EL1 系统寄存器影子**：`SCTLR_EL1 = 0`（MMU/Cache 关闭）、`MPIDR_EL1 = vcpu_id`
- **Board 接口**：绑定 `bcm2837_board_ops`（BCM2837 外设模拟）
- **入口点**：主 vCPU 通过 `switch_from_kthread` → `prepare_vcpu` → `raw_binary_loader` 加载内核

**`raw_binary_loader`** 从 SD 卡 FAT32 文件系统加载：
- `Image`（Linux 内核）→ IPA `0x08000000`
- `rasp3b.dtb`（设备树）→ IPA `0x3B000000`
- `rootfs.gz`（initramfs）→ IPA `0x02200000`

加载完成后设置 AArch64 Linux 启动协议寄存器：`x0 = DTB 地址`，`PC = 内核入口`。

---

## 4. 异常处理与 Trap 机制

### 4.1 异常向量表

aVisor 在 `entry.S` 中定义了标准的 AArch64 EL2 异常向量表，每个向量入口 128 字节对齐：

```
              ┌────────────────────────────┐
VBAR_EL2 + 0x000 │ Current EL, SP_EL0, Sync  │  (未使用)
         + 0x080 │ Current EL, SP_EL0, IRQ   │
         + 0x100 │ Current EL, SP_EL0, FIQ   │
         + 0x180 │ Current EL, SP_EL0, SError │
         + 0x200 │ Current EL, SP_ELx, Sync  │  (Hypervisor 自身异常)
         + 0x280 │ Current EL, SP_ELx, IRQ   │
         + 0x300 │ Current EL, SP_ELx, FIQ   │
         + 0x380 │ Current EL, SP_ELx, SError │
         + 0x400 │ Lower EL, AArch64, Sync   │  ← Guest Trap 入口
         + 0x480 │ Lower EL, AArch64, IRQ    │  ← Guest IRQ 入口
         + 0x500 │ Lower EL, AArch64, FIQ    │
         + 0x580 │ Lower EL, AArch64, SError │
              └────────────────────────────┘
```

Guest 的所有同步异常（HVC、SMC、内存故障、系统寄存器访问等）通过 `VBAR_EL2 + 0x400` 进入，IRQ 通过 `+ 0x480` 进入。

### 4.2 `kernel_entry` / `kernel_exit` 宏

每次 Trap 时的上下文保存/恢复：

```
kernel_entry:
    // 1. 保存 Guest 通用寄存器 x0-x29 到 EL2 栈
    stp x0, x1, [sp, #-288]!
    stp x2, x3, [sp, #16]
    ...
    // 2. 保存 ELR_EL2（Guest 返回地址）和 SPSR_EL2
    mrs x22, elr_el2
    mrs x23, spsr_el2
    stp x22, x23, [sp, #256]

    // 3. 调用 vm_leaving_work()：保存 EL1 系统寄存器影子，刷新控制台
    bl vm_leaving_work

kernel_exit:
    // 1. 调用 vm_entering_work()：恢复系统寄存器，注入虚拟中断
    bl vm_entering_work

    // 2. 恢复 ELR_EL2 和 SPSR_EL2
    ldp x22, x23, [sp, #256]
    msr elr_el2, x22
    msr spsr_el2, x23

    // 3. 恢复 Guest 通用寄存器 x0-x29
    ldp x0, x1, [sp], #288
    ...
    // 4. 返回 Guest
    eret
```

### 4.3 同步异常分发

`handle_sync_exception()` 根据 `ESR_EL2` 的 Exception Class（EC）字段分发：

```c
void handle_sync_exception(unsigned long esr, struct pt_regs *regs)
{
    int ec = (esr >> 26) & 0x3f;

    switch (ec) {
    case 0x16:  // HVC (AArch64)
        handle_system_call(esr, regs);   // PSCI 等服务
        break;
    case 0x17:  // SMC (AArch64), HCR_EL2.TSC=1 时 Trap
        handle_system_call(esr, regs);
        break;
    case 0x18:  // MSR/MRS 系统寄存器访问
        handle_trap_system(esr, regs);
        break;
    case 0x01:  // WFI/WFE
        handle_trap_wfx(esr, regs);
        break;
    case 0x20:  // IABT (Lower EL 指令中止)
    case 0x24:  // DABT (Lower EL 数据中止)
        handle_mem_abort(esr, regs);     // 内存故障/MMIO
        break;
    }
}
```

### 4.4 PSCI 模拟

Guest Linux 通过 HVC 调用 PSCI（Power State Coordination Interface）管理 CPU 电源状态：

```c
void handle_system_call(unsigned long esr, struct pt_regs *regs)
{
    uint32_t fid = regs->regs[0];  // Function ID

    switch (fid) {
    case PSCI_VERSION:           // 0x84000000
        regs->regs[0] = 0x00010000;  // v1.0
        break;

    case PSCI_CPU_ON_64:         // 0xC4000003
        // target = regs[1], entry = regs[2], context = regs[3]
        target_vcpu->state = VCPU_RUNNING;
        vcpu_pt_regs(target_vcpu)->pc = regs->regs[2];
        asm volatile("sev");     // 唤醒 WFE 等待的 vCPU
        regs->regs[0] = PSCI_SUCCESS;
        break;

    case PSCI_AFFINITY_INFO_64:  // 0xC4000004
        regs->regs[0] = (target_vcpu->state == VCPU_RUNNING) ? 0 : 1;
        break;

    case PSCI_SYSTEM_OFF:        // 0x84000008
        stop_vcpu();
        break;
    }
}
```

这使得 Guest Linux 可以通过标准 PSCI 接口启动多核、查询 CPU 状态。

---

## 5. 内存虚拟化：Stage-2 地址翻译

### 5.1 Stage-2 页表结构

aVisor 使用 **38 位 IPA 空间**（`VTCR_EL2.T0SZ = 26`），4KB 页，三级页表：

```
IPA[37:30]          IPA[29:21]          IPA[20:12]         IPA[11:0]
    │                   │                   │                  │
    ▼                   ▼                   ▼                  ▼
┌────────┐         ┌────────┐         ┌────────┐
│ Level 1│────────►│ Level 2│────────►│ Level 3│────────► 4KB 物理页
│ (PGD)  │ 512项   │ (PMD)  │ 512项   │ (PTE)  │ 512项
└────────┘         └────────┘         └────────┘
    ↑
VTTBR_EL2
```

每个 PTE 中的属性位控制 Stage-2 权限：

| 属性 | 正常内存页 | MMIO 设备页 |
|------|-----------|-------------|
| AP (访问权限) | `(3<<6)` EL1 读写 | `0` 无访问权限 |
| SH (共享性) | Inner Shareable | - |
| MemAttr | `(0x5<<2)` WB Cacheable | `0` Device-nGnRnE |

### 5.2 按需分页（Demand Paging）

aVisor 不会在 VM 创建时预分配所有 Guest 内存。初始时 Stage-2 页表几乎为空，Guest 的内存访问触发 **Stage-2 Translation Fault**，Hypervisor 捕获后动态分配物理页：

```c
void handle_mem_abort(unsigned long esr, struct pt_regs *regs)
{
    // 从 HPFAR_EL2 获取故障的 IPA
    unsigned long ipa = (hpfar << 8) | (far & 0xFFF);
    int dfsc = esr & 0x3f;

    if ((dfsc >> 2) == 0x1) {
        // Translation Fault：分配物理页并映射
        unsigned long page = allocate_page();
        map_stage2_page(vm, ipa, page, MMU_STAGE2_PAGE_FLAGS);
    }
    else if ((dfsc >> 2) == 0x3) {
        if (ipa >= DEVICE_BASE) {
            // 设备区域访问 → MMIO 模拟
            int wnr = (esr >> 6) & 1;
            int rt = (esr >> 16) & 0x1f;
            if (wnr)
                board_ops->mmio_write(vcpu, ipa, regs->regs[rt]);
            else
                regs->regs[rt] = board_ops->mmio_read(vcpu, ipa);
            increment_current_pc(4);  // 跳过触发 Trap 的指令
        } else {
            // 普通内存的延迟映射
            unsigned long page = allocate_page();
            map_stage2_page(vm, ipa, page, MMU_STAGE2_PAGE_FLAGS);
        }
    }
}
```

### 5.3 MMIO 拦截原理

设备 MMIO 区域在 Stage-2 页表中被标记为 **AP=0（不可访问）**。当 Guest 访问这些地址时：

1. Stage-2 产生 **Permission Fault**（DFSC 类别 `0x3`）
2. Hypervisor 捕获异常，从 `ESR_EL2` 解码出访问指令的目标寄存器和读写方向
3. 调用对应的 MMIO 处理函数模拟设备行为
4. 将 PC 前进 4 字节，跳过已处理的指令
5. `eret` 回到 Guest 继续执行

---

## 6. 设备模拟与 MMIO 拦截

### 6.1 BCM2837 外设模拟总览

aVisor 在 `bcm2837.c` 中模拟了 Raspberry Pi 3 的主要外设：

```
┌──────────────────────────────────────────────────────┐
│                bcm2837_mmio_read/write                │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────┐ │
│  │  PL011   │  │ Mini UART│  │  中断控制器         │ │
│  │  UART    │  │  (AUX)   │  │  IRQ_PENDING_1/2   │ │
│  │0x3f201xxx│  │0x3f215xxx│  │  ENABLE/DISABLE     │ │
│  └──────────┘  └──────────┘  └────────────────────┘ │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────┐ │
│  │ 系统定时器│  │   GPIO   │  │  本地中断控制器     │ │
│  │  CS/CLO  │  │  GPFSEL  │  │  IRQ_PENDING/       │ │
│  │  C0-C3   │  │0x3f200xxx│  │  Mailbox IPI        │ │
│  │0x3f003xxx│  └──────────┘  │  0x40000xxx         │ │
│  └──────────┘                └────────────────────┘ │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │         VideoCore Mailbox (vmbox.c)           │   │
│  │         ARM 内存大小 / 序列号 / 电源          │   │
│  └──────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

每个 vCPU 持有独立的 `bcm2837_state` 结构，包含中断使能寄存器、AUX UART 状态、PL011 IMSC、系统定时器等外设影子状态。

### 6.2 PL011 UART 模拟

PL011 是 Guest Linux 的主控制台（`ttyAMA0`）。模拟实现基于 FIFO：

**输出路径**（Guest → 物理 UART）：
```
Guest 写 PL011_DR → handle_pl011_write → enqueue(out_fifo)
                                            ↓
vm_entering_work → flush_console → _putchar → 物理 Mini UART
```

**输入路径**（物理 UART → Guest）：
```
物理 UART IRQ → handle_uart_irq → enqueue(in_fifo)
                                       ↓
Guest 读 PL011_DR → handle_pl011_read → dequeue(in_fifo)
```

PL011 中断模拟追踪 IMSC（中断掩码）和 RIS（原始中断状态）：
- 当 `in_fifo` 非空时，RIS 的 RXRIS（bit 4）置位
- 当 `out_fifo` 未满时，RIS 的 TXRIS（bit 5）置位
- MIS = RIS & IMSC，当 MIS 非零时通过中断控制器链注入 vIRQ

### 6.3 中断控制器模拟

BCM2837 有两层中断控制器：

**GPU 中断控制器**（`0x3F00B200`）：
- `IRQ_PENDING_1`：系统定时器匹配中断（bit 1, 3）、AUX/Mini UART 中断（bit 29）
- `IRQ_PENDING_2`：PL011 UART 中断（bit 25，对应 IRQ 57）
- `IRQ_BASIC_PENDING`：聚合 PENDING_1 和 PENDING_2 的状态

**本地中断控制器**（`0x40000060+`）—— 每个 CPU 核心独立：
- bit 3：CNTV（Guest 虚拟定时器）中断
- bit 4-7：Mailbox 0-3 中断（用于 IPI 核间通信）
- bit 8：GPU 中断（来自上层 GPU 中断控制器）

### 6.4 IPI Mailbox 模拟

Linux SMP 使用 BCM2836 Mailbox 实现核间中断。aVisor 用全局数组 `volatile uint32_t ipi_mbox[4][4]` 模拟：

```c
// 写入 Mailbox SET 寄存器：原子 OR
handle_local_intc_write(MBOX_SET):
    ipi_mbox[target_core][mbox] |= val;
    asm volatile("dsb ish");

// 读取 Mailbox RDCLR 寄存器：返回值并清零
handle_local_intc_read(MBOX_RDCLR):
    result = ipi_mbox[core][mbox];

handle_local_intc_write(MBOX_RDCLR):
    ipi_mbox[core][mbox] &= ~val;
```

### 6.5 系统定时器虚拟化

BCM2835 系统定时器（`0x3F003000`）通过时间偏移实现虚拟化：

- `bcm2837_state.systimer.offset` 记录虚拟时间与物理时间的差值
- Guest 读 `CLO/CHI` 时返回 `physical_count - offset`
- Guest 写比较寄存器 `C0-C3` 时，`entering_vm` 将最近的到期时间编程到物理 `TIMER_C3`
- 定时器到期时触发 IRQ，通过中断控制器模拟链注入到 Guest

---

## 7. 中断虚拟化与定时器

### 7.1 Hypervisor 定时器（调度 Tick）

每个物理 CPU 核心使用 **CNTHP**（Hypervisor Physical Timer）产生调度 tick：

```c
void init_hv_timer(int core)
{
    uint64_t cntfrq;
    asm volatile("mrs %0, cntfrq_el0" : "=r"(cntfrq));

    uint64_t ticks = cntfrq / TICK_RATE_HZ;  // TICK_RATE_HZ = 10
    write_cnthp_tval(ticks);                  // 100ms 一次 tick
    enable_cnthp();                           // CNTHP_CTL_EL2 = 1

    // 路由到本地中断控制器
    put32(COREn_TIMER_IRQCNTL(core), 1 << 2); // HPtimer → core IRQ
}
```

### 7.2 虚拟中断注入

`set_cpu_virtual_interrupt()` 在每次 Guest 进入前（`vm_entering_work`）评估是否需要注入虚拟中断：

```c
void set_cpu_virtual_interrupt(struct avisor_vcpu *vcpu)
{
    int virq = 0;

    // 1. 检查 Board 级 IRQ（GPU 中断控制器有 pending）
    if (board_ops->is_irq_asserted(vcpu))
        virq = 1;

    // 2. 检查 Guest 虚拟定时器中断（CNTV）
    if (is_cntv_irq_pending())  // CNTV_CTL: ENABLE && !IMASK && ISTATUS
        virq = 1;

    // 3. 检查 IPI Mailbox
    for (int m = 0; m < 4; m++)
        if (ipi_mbox[cpu][m]) { virq = 1; break; }

    // 4. 通过 HCR_EL2 的 VI/VF 位注入
    if (virq) assert_virq();   // 设置 HCR_EL2.VI
    else      clear_virq();    // 清除 HCR_EL2.VI
}
```

`assert_virq()` 通过设置 `HCR_EL2` 的 `VI` 位（bit 7），使硬件在 `eret` 返回 Guest 后自动触发虚拟 IRQ 异常。Guest 的中断处理程序看到的 IRQ 与真实硬件无异。

---

## 8. 调度器与上下文切换

### 8.1 调度算法

aVisor 使用**优先级衰减轮转**（Priority-Decay Round-Robin）算法：

```c
void _schedule(void)
{
    while (1) {
        // 从所有 RUNNING 的 vCPU 中选择 counter 最大的
        int c = -1, next = 0;
        for (int i = 0; i < MAX_VCPUS; i++) {
            if (vcpu[i] && vcpu[i]->state == VCPU_RUNNING
                && vcpu[i]->counter > c) {
                c = vcpu[i]->counter;
                next = i;
            }
        }
        if (c) break;  // 找到可运行的 vCPU

        // 所有 counter 耗尽时重新充值
        for (int i = 0; i < MAX_VCPUS; i++)
            if (vcpu[i])
                vcpu[i]->counter = (vcpu[i]->counter >> 1) + vcpu[i]->priority;
    }
    switch_to(cpu_data->vcpu[next]);
}
```

- 每个 vCPU 有 `counter`（剩余时间片）和 `priority`（基础优先级）
- 定时器 tick 递减 `counter`，耗尽时触发调度
- 重新充值公式 `counter = counter/2 + priority` 实现了老化效果

### 8.2 上下文切换

`cpu_switch_to` 在 `sched.S` 中实现经典的对称栈切换：

```asm
cpu_switch_to:
    // 保存 prev 的 callee-saved 寄存器
    stp x19, x20, [x0, #THREAD_CPU_CONTEXT + 0]
    stp x21, x22, [x0, #THREAD_CPU_CONTEXT + 16]
    ...
    mov x9, sp
    str x9, [x0, #THREAD_CPU_CONTEXT + 96]   // 保存 SP
    str x30, [x0, #THREAD_CPU_CONTEXT + 104]  // 保存 LR（返回地址）

    // 恢复 next 的 callee-saved 寄存器
    ldp x19, x20, [x1, #THREAD_CPU_CONTEXT + 0]
    ...
    ldr x9, [x1, #THREAD_CPU_CONTEXT + 96]
    mov sp, x9                                 // 恢复 SP
    ldr x30, [x1, #THREAD_CPU_CONTEXT + 104]  // 恢复 LR
    ret                                        // "返回"到 next 的执行流
```

切换时不保存/恢复 Guest GPR——那是 `kernel_entry`/`kernel_exit` 的工作。`cpu_switch_to` 只切换 Hypervisor 自身的 C 调用栈。

### 8.3 完整的 Trap-Schedule-Return 流程

```
Guest 执行 ──► Trap (IRQ/HVC/Fault) ──► kernel_entry
                                            │
                                     vm_leaving_work()
                                     ├─ save_sysregs()
                                     ├─ leaving_vm()  (board hook)
                                     └─ flush_console()
                                            │
                                     C 异常处理器
                                     ├─ handle_sync_exception()
                                     ├─ handle_irq()
                                     └─ timer_tick() → _schedule()
                                            │
                                     vm_entering_work()
                                     ├─ entering_vm()  (board hook)
                                     ├─ flush_console()
                                     ├─ set_cpu_sysregs()  (Stage-2 + EL1 regs)
                                     └─ set_cpu_virtual_interrupt()
                                            │
                                     kernel_exit ──► eret ──► Guest 继续
```

---

## 9. 多核支持（SMP）

### 9.1 物理核启动

Raspberry Pi 3 的 4 个 CPU 核心通过 **spin-table** 机制启动：

1. BSP（CPU0）在 `boot.S` 中将 `_start` 地址写入 spin-table 地址（`0xE0/E8/F0`）
2. 发送 `SEV`（Send Event）唤醒 AP
3. AP 从 `secondary_cpu_entry` 醒来，配置 EL2 MMU，跳转到 BSP 指定的入口
4. BSP 调用 `start_secondary_cores(n, secondary_main)` 将 `secondary_main` 写入 `smp_cores[n]`

```c
void secondary_main(void)
{
    irq_vector_init();
    init_hv_timer(core_id);
    disable_irq();

    // 等待 BSP 创建完 VM
    while (hv->nr_vm_ready < get_avisor_config_amount())
        ;

    enable_irq();
    while (1)
        schedule();
}
```

### 9.2 Guest vCPU 的 SMP 启动

Guest Linux 通过设备树中的 `enable-method = "psci"` 声明使用 PSCI 启动多核：

1. Guest CPU0 发出 `HVC #0`，`x0 = PSCI_CPU_ON`，`x1 = target_cpu`，`x2 = entry_point`
2. aVisor 捕获 HVC，将目标 vCPU 状态设为 `VCPU_RUNNING`，设置其 PC
3. 发送 `SEV` 唤醒在 `switch_to_secondary_vcpu` 中 `WFE` 等待的物理 AP
4. AP 检测到 `regs->pc != 0`，通过 `kernel_exit` → `eret` 进入 Guest 的二级 CPU 入口

---

## 10. 控制台与 Shell 系统

### 10.1 双模式控制台架构

aVisor 使用物理 Mini UART（`0x3F215040`）作为唯一的物理控制台，通过软件切换实现**多路复用**：

```
┌────────────────────┐
│   物理 Mini UART   │
│   (串口终端)       │
└────────┬───────────┘
         │
    ┌────▼─────────────────┐
    │  handle_uart_irq()   │
    │                      │
    │  uart_forwarded_hv?  │
    │    ├─ true:  shell   │ ← Hypervisor Shell 模式
    │    └─ false: VM fifo │ ← Guest 控制台模式
    └──────────────────────┘
```

- **Hypervisor 模式**（默认）：输入送到 Shell 命令处理器
- **VM 模式**（`vmc <id>` 后）：输入送到指定 VM 的 `in_fifo`

### 10.2 Shell 命令

| 命令 | 功能 |
|------|------|
| `help` | 显示所有可用命令 |
| `vml` | 列出所有 VM/vCPU 状态（PC、计数器、Trap 统计等）|
| `vmc <id>` | 切换到指定 VM 的控制台 |
| `vmld <file> [entry] [core]` | 动态加载并启动新 VM |
| `ls` | 列出 SD 卡文件 |

### 10.3 转义序列

在 VM 控制台模式下，`@` 键触发转义序列：

| 序列 | 功能 |
|------|------|
| `@c` | 返回 Hypervisor Shell |
| `@0`-`@9` | 切换到指定 VM |
| `@l` | 显示 vCPU 列表 |
| `@@` | 输入字面 `@` 字符 |

---

## 11. 文件系统与 VM 加载

### 11.1 SD 卡驱动

aVisor 实现了完整的 BCM2835 EMMC 控制器驱动（`sd.c`），支持：
- SD 卡初始化（CMD0/CMD8/ACMD41/CMD2/CMD3/CMD7 序列）
- 4-bit 数据总线模式
- 块读取（`sd_readblock`）

### 11.2 FAT32 文件系统

通过集成 **FatFs** 通用 FAT 文件系统库，aVisor 可以读取标准 FAT32 格式的 SD 卡镜像。磁盘 I/O 层（`diskio.c`）桥接 FatFs 和 SD 卡驱动。

SD 卡上的文件布局：

```
/
├── kernel8.img    ← aVisor Hypervisor 自身
├── rasp3b.dtb     ← Guest Linux 的设备树
├── rootfs.gz      ← Guest Linux 的 initramfs
└── Image          ← Guest Linux 内核镜像
```

### 11.3 加载过程

`raw_binary_loader` 按页加载文件到 Guest IPA 空间：

```c
void load_file_to_memory(vm, filename, ipa, max_size)
{
    f_open(&file, filename, FA_READ);
    while (bytes_read > 0) {
        page = allocate_vcpu_page(vm, ipa);  // 分配物理页 + Stage-2 映射
        f_read(&file, page_va, PAGE_SIZE, &bytes_read);
        dcache_clean_invalidate_range(page_va, PAGE_SIZE);
        ipa += PAGE_SIZE;
    }
    f_close(&file);
}
```

每加载一页数据就通过 `map_stage2_page` 建立 IPA → PA 映射，并刷新 DCache 确保一致性。

---

## 12. 源码结构总览

```
avisor/
├── hypervisor/
│   ├── arch/aarch64/
│   │   ├── boot.S          # 启动代码：EL3→EL2，页表，MMU
│   │   ├── entry.S         # 异常向量表，kernel_entry/exit
│   │   ├── sched.S         # cpu_switch_to 上下文切换
│   │   ├── utils.S         # save/restore_sysregs，set_stage2_pgd
│   │   ├── irq.S           # IRQ 使能/禁止
│   │   ├── sync_exc.c      # 同步异常分发：HVC/SMC/PSCI/sysreg/fault
│   │   ├── timer.c         # Hypervisor 定时器初始化
│   │   ├── vcpu.c          # vCPU 创建与管理
│   │   └── vm.c            # VM 创建，Stage-2 页表初始化
│   ├── boards/raspi/
│   │   ├── mini_uart.c     # 物理 UART 驱动，控制台转发
│   │   ├── irq.c           # IRQ 分发
│   │   ├── timer.c         # BCM2835 系统定时器
│   │   └── sd.c            # SD 卡 EMMC 驱动
│   ├── common/
│   │   ├── main.c          # hypervisor_main 入口
│   │   ├── sched.c         # 调度器，虚拟中断注入
│   │   ├── mm.c            # 物理页分配，Stage-2 页表操作
│   │   ├── shell.c         # Hypervisor Shell（vml/vmc/vmld/ls）
│   │   ├── console.c       # 控制台 FIFO 刷新
│   │   ├── fifo.c          # 环形缓冲区实现
│   │   ├── loader.c        # Guest 内核加载器
│   │   ├── smp.c           # 多核启动
│   │   └── spinlock.c      # LL/SC 自旋锁
│   ├── emulator/raspi/
│   │   ├── bcm2837.c       # BCM2837 全外设 MMIO 模拟
│   │   └── vmbox.c         # VideoCore Mailbox 模拟
│   ├── fs/
│   │   ├── ff.c            # FatFs FAT32 文件系统
│   │   └── diskio.c        # 磁盘 I/O 胶水层
│   └── config.c            # VM 静态配置
├── include/
│   ├── arch/aarch64/
│   │   ├── sysregs.h       # HCR_EL2/SCR/SPSR/VTCR 等寄存器定义
│   │   ├── mmu.h           # MMU 常量（页大小、属性标志）
│   │   └── vm.h            # avisor_vm / cpu_sysregs 结构体
│   ├── common/
│   │   ├── sched.h         # avisor_vcpu / per_cpu_data_t
│   │   ├── mm.h            # VA_START / PHYS_MEMORY_SIZE
│   │   └── board.h         # board_ops 接口
│   └── boards/raspi/
│       ├── base.h          # DEVICE_BASE / PBASE
│       ├── irq.h           # IRQ 寄存器地址
│       └── timer.h         # 定时器寄存器地址
└── scripts/
    └── mksd3.py            # FAT32 SD 卡镜像构建工具
```

---

## 总结

aVisor 以不到 1 万行 C/汇编代码实现了一个功能完整的 Type-1 Hypervisor，涵盖了虚拟化的所有核心技术：

| 技术维度 | aVisor 实现方式 |
|----------|----------------|
| CPU 虚拟化 | EL2 Trap-and-Emulate + HCR_EL2 控制 |
| 内存虚拟化 | Stage-2 地址翻译 + 按需分页 |
| I/O 虚拟化 | MMIO Trap + 软件设备模拟 |
| 中断虚拟化 | HCR_EL2.VI/VF 虚拟中断注入 |
| 定时器虚拟化 | CNTV 直通 + 系统定时器偏移 |
| 多核 | PSCI 模拟 + spin-table 物理核启动 |

它是理解 ARM 虚拟化原理的优秀学习材料——足够小以便通读全部代码，又足够完整可以运行真实的 Linux 内核。

---

## 参考

```
git clone -b boot_linux --single-branch https://github.com/calinyara/avisor.git
cd avisor
./scripts/linux.sh
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


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-27WH7FZ7KT"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'G-27WH7FZ7KT');
</script>
