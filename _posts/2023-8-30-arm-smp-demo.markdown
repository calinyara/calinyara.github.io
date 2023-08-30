---
layout: post
title:  "ARM架构bare-metal多核程序示例"
categories: Technology
tags: 树莓派 raspberry pi 多核 arm zh smp qemu aarch64 体系结构 en bare-metal 裸金属
author: Calinyara
description:
---

<br>

- ***boot.S***

```c
.global _start
.section .text.boot, "ax"

_start:
    mrs x0, mpidr_el1
    and x0, x0, #3

    mov x6, x0
    mov x1, #0x0
    add x2, x1, #0x400000
    lsl x6, x6, #18
    sub x3, x2, x6
    mov sp, x3

    bl uart_init

    mrs x0, mpidr_el1
    and x0, x0, #3
    cbz x0, core0
    cmp x0, #1
    b.eq core1
    cmp x0, #2
    b.eq core2
    cmp x0, #3
    b.eq core3

core0:
    bl print0
    b hang

core1:
    bl print1
    b hang

core2:
    bl print2
    b hang

core3:
    bl print3
    b hang

hang:
    wfi
    b hang
```

<br>

- ***main.c***

```c
extern void delay(unsigned long);
extern void put32(unsigned long, unsigned int);
extern unsigned int get32(unsigned long);
extern int get_el(void);

#define PBASE 0x3F000000
#define GPFSEL1	  (PBASE + 0x00200004)
#define GPSET0	  (PBASE + 0x0020001C)
#define GPCLR0	  (PBASE + 0x00200028)
#define GPPUD	  (PBASE + 0x00200094)
#define GPPUDCLK0 (PBASE + 0x00200098)

#define AUX_ENABLES	(PBASE + 0x00215004)
#define AUX_MU_IO_REG	(PBASE + 0x00215040)
#define AUX_MU_IER_REG	(PBASE + 0x00215044)
#define AUX_MU_IIR_REG	(PBASE + 0x00215048)
#define AUX_MU_LCR_REG	(PBASE + 0x0021504C)
#define AUX_MU_MCR_REG	(PBASE + 0x00215050)
#define AUX_MU_LSR_REG	(PBASE + 0x00215054)
#define AUX_MU_MSR_REG	(PBASE + 0x00215058)
#define AUX_MU_SCRATCH	(PBASE + 0x0021505C)
#define AUX_MU_CNTL_REG (PBASE + 0x00215060)
#define AUX_MU_STAT_REG (PBASE + 0x00215064)
#define AUX_MU_BAUD_REG (PBASE + 0x00215068)

void uart_send_char(char c)
{
	while (1) {
		if (get32(AUX_MU_LSR_REG) & 0x20)
			break;
	}
	put32(AUX_MU_IO_REG, c);
}

void uart_init(void)
{
	unsigned int selector;

	selector = get32(GPFSEL1);
	selector &= ~(7 << 12);
	selector |= 2 << 12;
	selector &= ~(7 << 15);
	selector |= 2 << 15;
	put32(GPFSEL1, selector);

	put32(GPPUD, 0);
	delay(150);
	put32(GPPUDCLK0, (1 << 14) | (1 << 15));
	delay(150);
	put32(GPPUDCLK0, 0);

	put32(AUX_ENABLES, 1);
	put32(AUX_MU_CNTL_REG, 0);
	put32(AUX_MU_IER_REG, 0);
	put32(AUX_MU_LCR_REG, 3);
	put32(AUX_MU_MCR_REG, 0);
	put32(AUX_MU_BAUD_REG, 270);

	put32(AUX_MU_CNTL_REG, 3);
}

void print0() {
	for(int j = 0; j <= 3; j++) {
		for(int i = 0; i <= 200000000; i++) {
			if (i == 1)
				uart_send_char('0');
		}
	}
}

void print1() {
	for(int j = 0; j <= 3; j++) {
		for(int i = 0; i <= 200000000; i++) {
			if (i == 10000000)
				uart_send_char('1');
		}
	}		
}

void print2() {
	for(int j = 0; j <= 3; j++) {
		for(int i = 0; i <= 200000000; i++) {
			if (i == 90000000)
				uart_send_char('2');
		}
	}		
}

void print3() {
	for(int j = 0; j <= 3; j++) {
		for(int i = 0; i <= 200000000; i++) {
			if (i == 80000000)
				uart_send_char('3');
		}
	}		
}
```

<br>

- ***utils.S***

```c
.globl get_el
get_el:
	mrs x0, CurrentEL
	lsr x0, x0, #2
	ret

.globl put32
put32:
	str w1,[x0]
	ret

.globl get32
get32:
	ldr w0,[x0]
	ret

.globl delay
delay:
	subs x0, x0, #1
	bne delay
	ret
```

<br>

- ***Makefile***

```makefile
ARMGNU ?= aarch64-linux-gnu

COPS = -Wall -nostdlib -nostartfiles -ffreestanding -Iinclude -mgeneral-regs-only
ASMOPS = -Iinclude

BUILD_DIR = build
SRC_DIR = .

all : arm_smp

run : arm_smp
	qemu-system-aarch64 -M raspi3b -nographic -serial null -serial mon:stdio -kernel kernel8.elf

clean :
	rm -rf $(BUILD_DIR) *.img *.bin *.elf

$(BUILD_DIR)/%_c.o: $(SRC_DIR)/%.c
	mkdir -p $(@D)
	$(ARMGNU)-gcc $(COPS) -MMD -c $< -o $@

$(BUILD_DIR)/%_s.o: $(SRC_DIR)/%.S
	$(ARMGNU)-gcc $(ASMOPS) -MMD -c $< -o $@

C_FILES = $(wildcard $(SRC_DIR)/*.c)
ASM_FILES = $(wildcard $(SRC_DIR)/*.S)
OBJ_FILES = $(C_FILES:$(SRC_DIR)/%.c=$(BUILD_DIR)/%_c.o)
OBJ_FILES += $(ASM_FILES:$(SRC_DIR)/%.S=$(BUILD_DIR)/%_s.o)

DEP_FILES = $(OBJ_FILES:%.o=%.d)
-include $(DEP_FILES)

arm_smp: $(SRC_DIR)/linker.ld $(OBJ_FILES)
	 $(ARMGNU)-ld -T $(SRC_DIR)/linker.ld -o kernel8.elf  $(OBJ_FILES)
```

<br>

- ***linker.ld***

```c
ENTRY(_start)

SECTIONS {
    . = 0x80000;

    .text : {
        *(.text.boot)
        *(.text)
    }

    .data : {
        *(.data)
    }
}
```

<br>

```shell
$ make run
0132013201320132
```

<br>

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-69PP8GKYST"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'G-69PP8GKYST');
</script>



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