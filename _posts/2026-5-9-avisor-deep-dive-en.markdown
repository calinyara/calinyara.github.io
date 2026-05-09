---
layout: post
title:  "A Technical Deep Dive into Building an AArch64 Type-1 Hypervisor from Scratch"
categories: Technology
tags: 内核 kernel Linux 操作系统 树莓派 raspberry pi buildroot hypervisor 虚拟机 虚拟化 virtualization OS 调度 arm en avisor
author: Calinyara
description:
---


> This article provides an in-depth analysis of aVisor — a lightweight AArch64 Type-1 Hypervisor designed for IoT scenarios. Starting from the fundamentals of ARM virtualization, it progressively unfolds the technical architecture and implementation details, covering the boot sequence, exception handling, memory virtualization, device emulation, scheduler, console system, and other core modules.

---

## Table of Contents

1. [ARM Virtualization Fundamentals](#1-arm-virtualization-fundamentals)
2. [aVisor Overall Architecture](#2-avisor-overall-architecture)
3. [Boot Sequence: From Power-On to Guest OS Execution](#3-boot-sequence-from-power-on-to-guest-os-execution)
4. [Exception Handling and Trap Mechanism](#4-exception-handling-and-trap-mechanism)
5. [Memory Virtualization: Stage-2 Address Translation](#5-memory-virtualization-stage-2-address-translation)
6. [Device Emulation and MMIO Interception](#6-device-emulation-and-mmio-interception)
7. [Interrupt Virtualization and Timers](#7-interrupt-virtualization-and-timers)
8. [Scheduler and Context Switching](#8-scheduler-and-context-switching)
9. [Multi-Core Support (SMP)](#9-multi-core-support-smp)
10. [Console and Shell System](#10-console-and-shell-system)
11. [Filesystem and VM Loading](#11-filesystem-and-vm-loading)
12. [Source Code Structure Overview](#12-source-code-structure-overview)

---

## 1. ARM Virtualization Fundamentals

### 1.1 ARM Exception Levels

The AArch64 architecture defines four exception levels, with privilege increasing from bottom to top:

```
┌──────────────────────────────────────┐
│  EL3  Secure Monitor (Firmware/ATF)  │  ← Highest privilege, secure world switch
├──────────────────────────────────────┤
│  EL2  Hypervisor                     │  ← aVisor runs here
├──────────────────────────────────────┤
│  EL1  Guest OS Kernel (Linux)        │  ← Guest kernel
├──────────────────────────────────────┤
│  EL0  User Applications              │  ← Guest userspace programs
└──────────────────────────────────────┘
```

As a Type-1 (bare-metal) Hypervisor, aVisor runs directly at EL2 without a host operating system. The Guest OS runs at EL1/EL0, and its privileged operations are automatically trapped by hardware to EL2 for aVisor to handle.

### 1.2 Key System Registers

| Register | Function |
|----------|----------|
| **HCR_EL2** | Hypervisor Configuration Register — controls trap behavior and Stage-2 translation |
| **VTTBR_EL2** | Stage-2 page table base address + VMID |
| **VTCR_EL2** | Stage-2 translation control (page size, address width, etc.) |
| **ELR_EL2** | Exception Link Register (saves Guest PC on trap) |
| **SPSR_EL2** | Saved Program Status Register (saves Guest PSTATE on trap) |
| **ESR_EL2** | Exception Syndrome Register (encodes trap reason) |
| **FAR_EL2** | Fault Address Register |
| **HPFAR_EL2** | Hypervisor IPA Fault Address Register (for Stage-2 faults) |

### 1.3 Stage-2 Address Translation

ARM virtualization extensions provide two levels of address translation:

```
Guest VA  ──Stage-1(Guest-controlled)──►  IPA  ──Stage-2(Hypervisor-controlled)──►  PA
             (EL1 MMU, TTBR0/1_EL1)            (EL2 MMU, VTTBR_EL2)
```

- **Stage-1**: Managed by the Guest, translating Virtual Addresses (VA) to Intermediate Physical Addresses (IPA)
- **Stage-2**: Managed by the Hypervisor, translating IPA to real Physical Addresses (PA), achieving memory isolation

### 1.4 HVC/SMC Calls

- **HVC (Hypervisor Call)**: Guest proactively calls Hypervisor services (e.g., PSCI power management)
- **SMC (Secure Monitor Call)**: When `HCR_EL2.TSC=1`, SMC is also trapped to EL2

---

## 2. aVisor Overall Architecture

### 2.1 Design Goals

aVisor is designed for the Raspberry Pi 3 (BCM2837, 4-core Cortex-A53), aiming to run a full Linux Guest with minimal overhead on an embedded platform. Core features:

- **Type-1 bare-metal**: Runs directly on hardware EL2, no host OS needed
- **1:1 vCPU pinning**: Each vCPU is pinned to a specific physical CPU core
- **Full virtualization**: Guest runs unmodified, standard AArch64 Linux kernel
- **Demand paging**: Guest memory is dynamically allocated on page-fault
- **Complete device emulation**: Emulates BCM2837 UART, interrupt controller, timer, Mailbox, and other peripherals

### 2.2 Architecture Overview

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
│  │  │Exception │  │  Memory  │  │ Device  │ │  │
│  │  │ Handling │  │  Mgmt    │  │Emulation│ │  │
│  │  │(sync_exc)│  │ (mm.c)   │  │(bcm2837)│ │  │
│  │  ├──────────┤  ├──────────┤  ├─────────┤ │  │
│  │  │Scheduler │  │Stage-2 PT│  │ Console │ │  │
│  │  │ (sched)  │  │  (vm.c)  │  │(console)│ │  │
│  │  └──────────┘  └──────────┘  └─────────┘ │  │
│  └───────────────────────────────────────────┘  │
├─────────────────────────────────────────────────┤
│          BCM2837 Hardware (Raspberry Pi 3)        │
│  CPU0  CPU1  CPU2  CPU3  UART  Timer  EMMC ...   │
└─────────────────────────────────────────────────┘
```

### 2.3 Memory Layout

aVisor uses the following layout within the Raspberry Pi 3's 1GB physical memory:

| Address Range | Purpose |
|---------------|---------|
| `0x00000000 - 0x3EFFFFFF` | Normal memory (Guest RAM + Hypervisor) |
| `0x3F000000 - 0x3FFFFFFF` | BCM2837 GPU peripherals (MMIO) |
| `0x40000000 - 0x40FFFFFF` | BCM2836 local peripherals (timers, interrupts, Mailbox) |

The Hypervisor itself occupies the low address region. The Guest kernel image is loaded at IPA `0x08000000`, the DTB at `0x3B000000`, and the initramfs at `0x02200000`.

---

## 3. Boot Sequence: From Power-On to Guest OS Execution

### 3.1 EL3 → EL2 Transition

The aVisor boot code resides in `boot.S`, with the entry point `_start` in the `.text.boot` section. On QEMU raspi3b, the CPU starts at EL3:

```
_start:
    mrs x0, mpidr_el1       // Read CPU ID
    and x0, x0, #3
    ...
    mrs x0, CurrentEL       // Check current exception level
    cmp x0, #3
    beq el3                 // If EL3, configure secure registers

el3:
    msr hcr_el2, HCR_VALUE  // Configure Hypervisor control
    msr scr_el3, SCR_VALUE  // Non-secure + HVC enabled + AArch64
    msr spsr_el3, SPSR_VALUE// Target: EL2h, all interrupts masked
    adr x0, el2_entry
    msr elr_el3, x0
    eret                    // Exception return, enter EL2
```

`SCR_EL3` sets `NS=1` (Non-secure world), `HCE=1` (enable HVC), and `RW=1` (EL2 uses AArch64). `SPSR_EL3` targets `EL2h` mode with all interrupts masked. `eret` transfers execution to EL2.

### 3.2 EL2 Page Tables and MMU Initialization

After entering EL2, the BSP (CPU0) executes:

```
el2_entry:
    // 1. Zero out BSS section
    adr x0, bss_begin
    adr x1, bss_end
    sub x1, x1, x0
    bl memzero

    // 2. Create EL2 page tables
    bl __create_page_tables

    // 3. Configure MMU control registers
    adrp x0, pg_dir
    msr ttbr0_el2, x0           // Page table base address
    msr tcr_el2, TCR_VALUE      // Translation control
    msr vtcr_el2, VTCR_VALUE    // Stage-2 translation control
    msr mair_el2, MAIR_VALUE    // Memory attributes

    // 4. Enable MMU
    mov x0, #SCTLR_MMU_ENABLED
    msr sctlr_el2, x0
    isb

    // 5. Jump to C entry point
    br hypervisor_main
```

**`__create_page_tables`** creates three-level page tables (PGD → PUD → PMD) using 2MB block mappings:

- **Normal memory** (`0x00000000 - 0x3EFFFFFF`): `MMU_FLAGS = 0x705` (Normal, Inner Shareable, AF)
- **Device memory** (`0x3F000000 - 0x3FFFFFFF`): `MMU_DEVICE_FLAGS = 0x701` (Device-nGnRnE, AF)
- **Local peripherals** (`0x40000000`): A single 2MB block with device memory attributes

The QEMU linker script `linker_qemu.ld` sets the code base address to `0x80000` (QEMU's kernel load address), so the page tables are effectively identity-mapped (VA = PA).

### 3.3 `hypervisor_main` Initialization Sequence

```c
void hypervisor_main(void)
{
    // 1. Initialize per-CPU data structures
    init_per_cpu_data();

    // 2. Initialize physical UART (Mini UART, 115200 baud)
    uart_init();

    // 3. Print logo and initialize Shell
    printf(logo);
    shell_init();
    init_hv();

    // 4. Install exception vector table
    irq_vector_init();

    // 5. Configure timers
    init_misc_timer();      // BCM2835 system timer
    init_hv_timer(0);       // CPU0's Hypervisor physical timer

    // 6. Enable interrupt controller
    enable_interrupt_controller();

    // 7. Mount SD card FAT32 filesystem
    f_mount(&fatfs, "/", 1);

    // 8. Create VM and load Guest kernel
    for (int i = 0; i < get_avisor_config_amount(); i++) {
        create_vm(i, get_avisor_config(i));
    }

    // 9. Start remaining physical CPU cores
    start_secondary_cores(1, secondary_main);
    start_secondary_cores(2, secondary_main);
    start_secondary_cores(3, secondary_main);

    // 10. Enter scheduling loop
    enable_irq();
    while (1)
        schedule();
}
```

### 3.4 VM Creation and Kernel Loading

`create_vm()` is the starting point of the VM lifecycle:

1. **Allocate VM structure**: Obtain a slot from the global `vm_array[]`, assign VMID
2. **Initialize Stage-2 page tables**: Mark device regions (`0x3F000000+`) as "inaccessible" to trigger MMIO traps on access
3. **Initialize console**: Allocate `in_fifo` and `out_fifo`
4. **Create vCPUs**: Create one vCPU for each physical CPU core

**vCPU creation** allocates a `THREAD_SIZE` (4KB) kernel stack for each vCPU and initializes:

- **EL1 system register shadows**: `SCTLR_EL1 = 0` (MMU/Cache off), `MPIDR_EL1 = vcpu_id`
- **Board interface**: Binds `bcm2837_board_ops` (BCM2837 peripheral emulation)
- **Entry point**: The primary vCPU enters via `switch_from_kthread` → `prepare_vcpu` → `raw_binary_loader` to load the kernel

**`raw_binary_loader`** loads files from the SD card FAT32 filesystem:
- `Image` (Linux kernel) → IPA `0x08000000`
- `rasp3b.dtb` (Device Tree) → IPA `0x3B000000`
- `rootfs.gz` (initramfs) → IPA `0x02200000`

After loading, it sets the AArch64 Linux boot protocol registers: `x0 = DTB address`, `PC = kernel entry point`.

---

## 4. Exception Handling and Trap Mechanism

### 4.1 Exception Vector Table

aVisor defines a standard AArch64 EL2 exception vector table in `entry.S`, with each vector entry aligned to 128 bytes:

```
              ┌────────────────────────────┐
VBAR_EL2 + 0x000 │ Current EL, SP_EL0, Sync  │  (unused)
         + 0x080 │ Current EL, SP_EL0, IRQ   │
         + 0x100 │ Current EL, SP_EL0, FIQ   │
         + 0x180 │ Current EL, SP_EL0, SError │
         + 0x200 │ Current EL, SP_ELx, Sync  │  (Hypervisor's own exceptions)
         + 0x280 │ Current EL, SP_ELx, IRQ   │
         + 0x300 │ Current EL, SP_ELx, FIQ   │
         + 0x380 │ Current EL, SP_ELx, SError │
         + 0x400 │ Lower EL, AArch64, Sync   │  ← Guest trap entry
         + 0x480 │ Lower EL, AArch64, IRQ    │  ← Guest IRQ entry
         + 0x500 │ Lower EL, AArch64, FIQ    │
         + 0x580 │ Lower EL, AArch64, SError │
              └────────────────────────────┘
```

All Guest synchronous exceptions (HVC, SMC, memory faults, system register accesses, etc.) enter through `VBAR_EL2 + 0x400`; IRQs enter through `+ 0x480`.

### 4.2 `kernel_entry` / `kernel_exit` Macros

Context save/restore on each trap:

```
kernel_entry:
    // 1. Save Guest general-purpose registers x0-x29 to EL2 stack
    stp x0, x1, [sp, #-288]!
    stp x2, x3, [sp, #16]
    ...
    // 2. Save ELR_EL2 (Guest return address) and SPSR_EL2
    mrs x22, elr_el2
    mrs x23, spsr_el2
    stp x22, x23, [sp, #256]

    // 3. Call vm_leaving_work(): save EL1 sysreg shadows, flush console
    bl vm_leaving_work

kernel_exit:
    // 1. Call vm_entering_work(): restore sysregs, inject virtual interrupts
    bl vm_entering_work

    // 2. Restore ELR_EL2 and SPSR_EL2
    ldp x22, x23, [sp, #256]
    msr elr_el2, x22
    msr spsr_el2, x23

    // 3. Restore Guest general-purpose registers x0-x29
    ldp x0, x1, [sp], #288
    ...
    // 4. Return to Guest
    eret
```

### 4.3 Synchronous Exception Dispatch

`handle_sync_exception()` dispatches based on the Exception Class (EC) field of `ESR_EL2`:

```c
void handle_sync_exception(unsigned long esr, struct pt_regs *regs)
{
    int ec = (esr >> 26) & 0x3f;

    switch (ec) {
    case 0x16:  // HVC (AArch64)
        handle_system_call(esr, regs);   // PSCI and other services
        break;
    case 0x17:  // SMC (AArch64), trapped when HCR_EL2.TSC=1
        handle_system_call(esr, regs);
        break;
    case 0x18:  // MSR/MRS system register access
        handle_trap_system(esr, regs);
        break;
    case 0x01:  // WFI/WFE
        handle_trap_wfx(esr, regs);
        break;
    case 0x20:  // IABT (Lower EL instruction abort)
    case 0x24:  // DABT (Lower EL data abort)
        handle_mem_abort(esr, regs);     // Memory fault / MMIO
        break;
    }
}
```

### 4.4 PSCI Emulation

Guest Linux uses HVC to call PSCI (Power State Coordination Interface) for CPU power management:

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
        asm volatile("sev");     // Wake the WFE-waiting vCPU
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

This enables Guest Linux to boot multiple cores and query CPU status through the standard PSCI interface.

---

## 5. Memory Virtualization: Stage-2 Address Translation

### 5.1 Stage-2 Page Table Structure

aVisor uses a **38-bit IPA space** (`VTCR_EL2.T0SZ = 26`), 4KB pages, and three-level page tables:

```
IPA[37:30]          IPA[29:21]          IPA[20:12]         IPA[11:0]
    │                   │                   │                  │
    ▼                   ▼                   ▼                  ▼
┌────────┐         ┌────────┐         ┌────────┐
│ Level 1│────────►│ Level 2│────────►│ Level 3│────────► 4KB Physical Page
│ (PGD)  │ 512 ent │ (PMD)  │ 512 ent │ (PTE)  │ 512 ent
└────────┘         └────────┘         └────────┘
    ↑
VTTBR_EL2
```

Attribute bits in each PTE control Stage-2 permissions:

| Attribute | Normal Memory Page | MMIO Device Page |
|-----------|-------------------|------------------|
| AP (Access Permission) | `(3<<6)` EL1 Read/Write | `0` No access |
| SH (Shareability) | Inner Shareable | — |
| MemAttr | `(0x5<<2)` WB Cacheable | `0` Device-nGnRnE |

### 5.2 Demand Paging

aVisor does not pre-allocate all Guest memory at VM creation time. Initially, the Stage-2 page tables are nearly empty, and Guest memory accesses trigger **Stage-2 Translation Faults**, which the Hypervisor catches and handles by dynamically allocating physical pages:

```c
void handle_mem_abort(unsigned long esr, struct pt_regs *regs)
{
    // Obtain faulting IPA from HPFAR_EL2
    unsigned long ipa = (hpfar << 8) | (far & 0xFFF);
    int dfsc = esr & 0x3f;

    if ((dfsc >> 2) == 0x1) {
        // Translation Fault: allocate physical page and map it
        unsigned long page = allocate_page();
        map_stage2_page(vm, ipa, page, MMU_STAGE2_PAGE_FLAGS);
    }
    else if ((dfsc >> 2) == 0x3) {
        if (ipa >= DEVICE_BASE) {
            // Device region access → MMIO emulation
            int wnr = (esr >> 6) & 1;
            int rt = (esr >> 16) & 0x1f;
            if (wnr)
                board_ops->mmio_write(vcpu, ipa, regs->regs[rt]);
            else
                regs->regs[rt] = board_ops->mmio_read(vcpu, ipa);
            increment_current_pc(4);  // Skip the trapped instruction
        } else {
            // Lazy mapping for normal memory
            unsigned long page = allocate_page();
            map_stage2_page(vm, ipa, page, MMU_STAGE2_PAGE_FLAGS);
        }
    }
}
```

### 5.3 MMIO Interception Principle

Device MMIO regions are marked as **AP=0 (inaccessible)** in the Stage-2 page tables. When the Guest accesses these addresses:

1. Stage-2 generates a **Permission Fault** (DFSC class `0x3`)
2. The Hypervisor catches the exception, decodes the target register and read/write direction from `ESR_EL2`
3. Calls the corresponding MMIO handler to emulate device behavior
4. Advances PC by 4 bytes to skip the handled instruction
5. `eret` returns to the Guest to continue execution

---

## 6. Device Emulation and MMIO Interception

### 6.1 BCM2837 Peripheral Emulation Overview

aVisor emulates the major peripherals of the Raspberry Pi 3 in `bcm2837.c`:

```
┌──────────────────────────────────────────────────────┐
│                bcm2837_mmio_read/write                │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────┐ │
│  │  PL011   │  │ Mini UART│  │ Interrupt           │ │
│  │  UART    │  │  (AUX)   │  │ Controller          │ │
│  │0x3f201xxx│  │0x3f215xxx│  │ IRQ_PENDING_1/2     │ │
│  └──────────┘  └──────────┘  └────────────────────┘ │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────┐ │
│  │  System  │  │   GPIO   │  │ Local Interrupt     │ │
│  │  Timer   │  │  GPFSEL  │  │ Controller          │ │
│  │  CS/CLO  │  │0x3f200xxx│  │ IRQ_PENDING /       │ │
│  │  C0-C3   │  └──────────┘  │ Mailbox IPI         │ │
│  │0x3f003xxx│                │ 0x40000xxx          │ │
│  └──────────┘                └────────────────────┘ │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │         VideoCore Mailbox (vmbox.c)           │   │
│  │         ARM memory size / serial / power      │   │
│  └──────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

Each vCPU holds an independent `bcm2837_state` structure containing interrupt enable registers, AUX UART state, PL011 IMSC, system timer, and other peripheral shadow state.

### 6.2 PL011 UART Emulation

PL011 is the Guest Linux main console (`ttyAMA0`). The emulation is FIFO-based:

**Output path** (Guest → Physical UART):
```
Guest writes PL011_DR → handle_pl011_write → enqueue(out_fifo)
                                                ↓
vm_entering_work → flush_console → _putchar → Physical Mini UART
```

**Input path** (Physical UART → Guest):
```
Physical UART IRQ → handle_uart_irq → enqueue(in_fifo)
                                           ↓
Guest reads PL011_DR → handle_pl011_read → dequeue(in_fifo)
```

PL011 interrupt emulation tracks IMSC (interrupt mask) and RIS (raw interrupt status):
- When `in_fifo` is non-empty, RIS bit RXRIS (bit 4) is set
- When `out_fifo` is not full, RIS bit TXRIS (bit 5) is set
- MIS = RIS & IMSC; when MIS is non-zero, a vIRQ is injected through the interrupt controller chain

### 6.3 Interrupt Controller Emulation

BCM2837 has two levels of interrupt controllers:

**GPU Interrupt Controller** (`0x3F00B200`):
- `IRQ_PENDING_1`: System timer match interrupts (bit 1, 3), AUX/Mini UART interrupt (bit 29)
- `IRQ_PENDING_2`: PL011 UART interrupt (bit 25, corresponding to IRQ 57)
- `IRQ_BASIC_PENDING`: Aggregates the status of PENDING_1 and PENDING_2

**Local Interrupt Controller** (`0x40000060+`) — independent per CPU core:
- bit 3: CNTV (Guest virtual timer) interrupt
- bit 4-7: Mailbox 0-3 interrupts (used for IPI inter-core communication)
- bit 8: GPU interrupt (from the upper-level GPU interrupt controller)

### 6.4 IPI Mailbox Emulation

Linux SMP uses BCM2836 Mailboxes for inter-processor interrupts. aVisor emulates this with a global array `volatile uint32_t ipi_mbox[4][4]`:

```c
// Write to Mailbox SET register: atomic OR
handle_local_intc_write(MBOX_SET):
    ipi_mbox[target_core][mbox] |= val;
    asm volatile("dsb ish");

// Read from Mailbox RDCLR register: return value and clear
handle_local_intc_read(MBOX_RDCLR):
    result = ipi_mbox[core][mbox];

handle_local_intc_write(MBOX_RDCLR):
    ipi_mbox[core][mbox] &= ~val;
```

### 6.5 System Timer Virtualization

The BCM2835 system timer (`0x3F003000`) is virtualized through time offsetting:

- `bcm2837_state.systimer.offset` records the difference between virtual and physical time
- When Guest reads `CLO/CHI`, the emulator returns `physical_count - offset`
- When Guest writes compare registers `C0-C3`, `entering_vm` programs the nearest deadline into the physical `TIMER_C3`
- When the timer expires, the IRQ is injected into the Guest through the interrupt controller emulation chain

---

## 7. Interrupt Virtualization and Timers

### 7.1 Hypervisor Timer (Scheduling Tick)

Each physical CPU core uses **CNTHP** (Hypervisor Physical Timer) to generate scheduling ticks:

```c
void init_hv_timer(int core)
{
    uint64_t cntfrq;
    asm volatile("mrs %0, cntfrq_el0" : "=r"(cntfrq));

    uint64_t ticks = cntfrq / TICK_RATE_HZ;  // TICK_RATE_HZ = 10
    write_cnthp_tval(ticks);                  // 100ms per tick
    enable_cnthp();                           // CNTHP_CTL_EL2 = 1

    // Route to local interrupt controller
    put32(COREn_TIMER_IRQCNTL(core), 1 << 2); // HPtimer → core IRQ
}
```

### 7.2 Virtual Interrupt Injection

`set_cpu_virtual_interrupt()` evaluates before each Guest entry (in `vm_entering_work`) whether a virtual interrupt should be injected:

```c
void set_cpu_virtual_interrupt(struct avisor_vcpu *vcpu)
{
    int virq = 0;

    // 1. Check board-level IRQ (GPU interrupt controller has pending)
    if (board_ops->is_irq_asserted(vcpu))
        virq = 1;

    // 2. Check Guest virtual timer interrupt (CNTV)
    if (is_cntv_irq_pending())  // CNTV_CTL: ENABLE && !IMASK && ISTATUS
        virq = 1;

    // 3. Check IPI Mailbox
    for (int m = 0; m < 4; m++)
        if (ipi_mbox[cpu][m]) { virq = 1; break; }

    // 4. Inject via HCR_EL2 VI/VF bits
    if (virq) assert_virq();   // Set HCR_EL2.VI
    else      clear_virq();    // Clear HCR_EL2.VI
}
```

`assert_virq()` sets the `VI` bit (bit 7) of `HCR_EL2`, causing the hardware to automatically trigger a virtual IRQ exception after `eret` returns to the Guest. The Guest's interrupt handler sees an IRQ indistinguishable from real hardware.

---

## 8. Scheduler and Context Switching

### 8.1 Scheduling Algorithm

aVisor uses a **Priority-Decay Round-Robin** algorithm:

```c
void _schedule(void)
{
    while (1) {
        // Select the vCPU with the highest counter among RUNNING ones
        int c = -1, next = 0;
        for (int i = 0; i < MAX_VCPUS; i++) {
            if (vcpu[i] && vcpu[i]->state == VCPU_RUNNING
                && vcpu[i]->counter > c) {
                c = vcpu[i]->counter;
                next = i;
            }
        }
        if (c) break;  // Found a runnable vCPU

        // Recharge all counters when all are depleted
        for (int i = 0; i < MAX_VCPUS; i++)
            if (vcpu[i])
                vcpu[i]->counter = (vcpu[i]->counter >> 1) + vcpu[i]->priority;
    }
    switch_to(cpu_data->vcpu[next]);
}
```

- Each vCPU has a `counter` (remaining time slice) and `priority` (base priority)
- Timer ticks decrement `counter`; scheduling triggers when it reaches zero
- The recharge formula `counter = counter/2 + priority` implements an aging effect

### 8.2 Context Switch

`cpu_switch_to` implements a classic symmetric stack switch in `sched.S`:

```asm
cpu_switch_to:
    // Save prev's callee-saved registers
    stp x19, x20, [x0, #THREAD_CPU_CONTEXT + 0]
    stp x21, x22, [x0, #THREAD_CPU_CONTEXT + 16]
    ...
    mov x9, sp
    str x9, [x0, #THREAD_CPU_CONTEXT + 96]   // Save SP
    str x30, [x0, #THREAD_CPU_CONTEXT + 104]  // Save LR (return address)

    // Restore next's callee-saved registers
    ldp x19, x20, [x1, #THREAD_CPU_CONTEXT + 0]
    ...
    ldr x9, [x1, #THREAD_CPU_CONTEXT + 96]
    mov sp, x9                                 // Restore SP
    ldr x30, [x1, #THREAD_CPU_CONTEXT + 104]  // Restore LR
    ret                                        // "Return" into next's execution flow
```

Guest GPRs are not saved/restored during the switch — that is the job of `kernel_entry`/`kernel_exit`. `cpu_switch_to` only switches the Hypervisor's own C call stack.

### 8.3 Complete Trap-Schedule-Return Flow

```
Guest executes ──► Trap (IRQ/HVC/Fault) ──► kernel_entry
                                                │
                                         vm_leaving_work()
                                         ├─ save_sysregs()
                                         ├─ leaving_vm()  (board hook)
                                         └─ flush_console()
                                                │
                                         C exception handler
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
                                         kernel_exit ──► eret ──► Guest continues
```

---

## 9. Multi-Core Support (SMP)

### 9.1 Physical Core Startup

The 4 CPU cores of the Raspberry Pi 3 are started using a **spin-table** mechanism:

1. The BSP (CPU0) writes the `_start` address into spin-table addresses (`0xE0/E8/F0`) in `boot.S`
2. Sends `SEV` (Send Event) to wake Application Processors (APs)
3. APs wake from `secondary_cpu_entry`, configure EL2 MMU, and jump to the BSP-specified entry point
4. BSP calls `start_secondary_cores(n, secondary_main)` to write `secondary_main` into `smp_cores[n]`

```c
void secondary_main(void)
{
    irq_vector_init();
    init_hv_timer(core_id);
    disable_irq();

    // Wait for BSP to finish creating VMs
    while (hv->nr_vm_ready < get_avisor_config_amount())
        ;

    enable_irq();
    while (1)
        schedule();
}
```

### 9.2 Guest vCPU SMP Startup

Guest Linux declares multi-core startup via `enable-method = "psci"` in the Device Tree:

1. Guest CPU0 issues `HVC #0` with `x0 = PSCI_CPU_ON`, `x1 = target_cpu`, `x2 = entry_point`
2. aVisor catches the HVC, sets the target vCPU state to `VCPU_RUNNING`, and sets its PC
3. Sends `SEV` to wake the physical AP waiting in `switch_to_secondary_vcpu` via `WFE`
4. The AP detects `regs->pc != 0` and enters the Guest's secondary CPU entry via `kernel_exit` → `eret`

---

## 10. Console and Shell System

### 10.1 Dual-Mode Console Architecture

aVisor uses the physical Mini UART (`0x3F215040`) as the sole physical console, implementing **multiplexing** through software switching:

```
┌────────────────────┐
│  Physical Mini UART│
│  (Serial Terminal) │
└────────┬───────────┘
         │
    ┌────▼─────────────────┐
    │  handle_uart_irq()   │
    │                      │
    │  uart_forwarded_hv?  │
    │    ├─ true:  shell   │ ← Hypervisor Shell mode
    │    └─ false: VM fifo │ ← Guest console mode
    └──────────────────────┘
```

- **Hypervisor mode** (default): Input goes to the Shell command processor
- **VM mode** (after `vmc <id>`): Input goes to the specified VM's `in_fifo`

### 10.2 Shell Commands

| Command | Function |
|---------|----------|
| `help` | Display all available commands |
| `vml` | List all VM/vCPU status (PC, counters, trap statistics, etc.) |
| `vmc <id>` | Switch to the specified VM's console |
| `vmld <file> [entry] [core]` | Dynamically load and start a new VM |
| `ls` | List SD card files |

### 10.3 Escape Sequences

In VM console mode, the `@` key triggers an escape sequence:

| Sequence | Function |
|----------|----------|
| `@c` | Return to Hypervisor Shell |
| `@0`-`@9` | Switch to specified VM |
| `@l` | Display vCPU list |
| `@@` | Input literal `@` character |

---

## 11. Filesystem and VM Loading

### 11.1 SD Card Driver

aVisor implements a complete BCM2835 EMMC controller driver (`sd.c`) supporting:
- SD card initialization (CMD0/CMD8/ACMD41/CMD2/CMD3/CMD7 sequence)
- 4-bit data bus mode
- Block reads (`sd_readblock`)

### 11.2 FAT32 Filesystem

By integrating the **FatFs** generic FAT filesystem library, aVisor can read standard FAT32-formatted SD card images. The disk I/O layer (`diskio.c`) bridges FatFs and the SD card driver.

File layout on the SD card:

```
/
├── kernel8.img    ← aVisor Hypervisor itself
├── rasp3b.dtb     ← Guest Linux Device Tree
├── rootfs.gz      ← Guest Linux initramfs
└── Image          ← Guest Linux kernel image
```

### 11.3 Loading Process

`raw_binary_loader` loads files page-by-page into the Guest IPA space:

```c
void load_file_to_memory(vm, filename, ipa, max_size)
{
    f_open(&file, filename, FA_READ);
    while (bytes_read > 0) {
        page = allocate_vcpu_page(vm, ipa);  // Allocate physical page + Stage-2 mapping
        f_read(&file, page_va, PAGE_SIZE, &bytes_read);
        dcache_clean_invalidate_range(page_va, PAGE_SIZE);
        ipa += PAGE_SIZE;
    }
    f_close(&file);
}
```

For each page loaded, an IPA → PA mapping is established via `map_stage2_page`, and the DCache is flushed to ensure coherency.

---

## 12. Source Code Structure Overview

```
avisor/
├── hypervisor/
│   ├── arch/aarch64/
│   │   ├── boot.S          # Boot code: EL3→EL2, page tables, MMU
│   │   ├── entry.S         # Exception vector table, kernel_entry/exit
│   │   ├── sched.S         # cpu_switch_to context switch
│   │   ├── utils.S         # save/restore_sysregs, set_stage2_pgd
│   │   ├── irq.S           # IRQ enable/disable
│   │   ├── sync_exc.c      # Sync exception dispatch: HVC/SMC/PSCI/sysreg/fault
│   │   ├── timer.c         # Hypervisor timer initialization
│   │   ├── vcpu.c          # vCPU creation and management
│   │   └── vm.c            # VM creation, Stage-2 page table initialization
│   ├── boards/raspi/
│   │   ├── mini_uart.c     # Physical UART driver, console forwarding
│   │   ├── irq.c           # IRQ dispatch
│   │   ├── timer.c         # BCM2835 system timer
│   │   └── sd.c            # SD card EMMC driver
│   ├── common/
│   │   ├── main.c          # hypervisor_main entry point
│   │   ├── sched.c         # Scheduler, virtual interrupt injection
│   │   ├── mm.c            # Physical page allocation, Stage-2 page table ops
│   │   ├── shell.c         # Hypervisor Shell (vml/vmc/vmld/ls)
│   │   ├── console.c       # Console FIFO flushing
│   │   ├── fifo.c          # Ring buffer implementation
│   │   ├── loader.c        # Guest kernel loader
│   │   ├── smp.c           # Multi-core startup
│   │   └── spinlock.c      # LL/SC spinlock
│   ├── emulator/raspi/
│   │   ├── bcm2837.c       # BCM2837 full peripheral MMIO emulation
│   │   └── vmbox.c         # VideoCore Mailbox emulation
│   ├── fs/
│   │   ├── ff.c            # FatFs FAT32 filesystem
│   │   └── diskio.c        # Disk I/O glue layer
│   └── config.c            # VM static configuration
├── include/
│   ├── arch/aarch64/
│   │   ├── sysregs.h       # HCR_EL2/SCR/SPSR/VTCR register definitions
│   │   ├── mmu.h           # MMU constants (page size, attribute flags)
│   │   └── vm.h            # avisor_vm / cpu_sysregs structures
│   ├── common/
│   │   ├── sched.h         # avisor_vcpu / per_cpu_data_t
│   │   ├── mm.h            # VA_START / PHYS_MEMORY_SIZE
│   │   └── board.h         # board_ops interface
│   └── boards/raspi/
│       ├── base.h          # DEVICE_BASE / PBASE
│       ├── irq.h           # IRQ register addresses
│       └── timer.h         # Timer register addresses
└── scripts/
    └── mksd3.py            # FAT32 SD card image build tool
```

---

## Conclusion

aVisor implements a fully functional Type-1 Hypervisor in fewer than 10,000 lines of C and assembly code, covering all core virtualization technologies:

| Technology Dimension | aVisor Implementation |
|---------------------|----------------------|
| CPU Virtualization | EL2 Trap-and-Emulate + HCR_EL2 control |
| Memory Virtualization | Stage-2 address translation + demand paging |
| I/O Virtualization | MMIO trap + software device emulation |
| Interrupt Virtualization | HCR_EL2.VI/VF virtual interrupt injection |
| Timer Virtualization | CNTV passthrough + system timer offsetting |
| Multi-Core | PSCI emulation + spin-table physical core startup |

It is an excellent learning resource for understanding ARM virtualization principles — small enough to read through all the code, yet complete enough to run a real Linux kernel.

---

## References

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
