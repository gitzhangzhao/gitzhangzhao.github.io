# U-Boot启动流程分析


**前言**

本文将以 arm 为例结合 u-boot 的`board—>machine—>arch—>cpu`框架，介绍在 aspeed 平台下， uboot-spl 启动双系统的流程

## 前置知识

u-boot 的启动流程大体可以分为两个部分：**平台无关部分**和**平台相关部分**。平台无关部分主要包括 u-boot 的主初始化流程，如初始化 BSS 段，设置栈，初始化硬件（如 DDR，时钟，串口等）等。这一部分的流程对于所有平台都是相同的，是 u-boot 的通用启动流程。而平台相关部分则涉及到特定硬件平台的初始化细节，如特定的内存布局，特定的外设初始化等，如下图所示：

![alt text](/U-Boot/structure.gif)

1. u-boot 启动后，会先执行 CPU 初始化代码
2. CPU 相关的代码，会调用 ARCH 的公共代码（如 arch/arm）
3. ARCH 的公共代码，在适当的时候，调用 board 有关的接口。u-boot 的功能逻辑，大多是由 common 代码实现，部分和平台有关的部分，则由公共代码声明，由 board 代码实现
4. board 代码在需要的时候，会调用 machine（arch/arm/mach-xxx）提供的接口，实现特定的功能。因此 machine 的定位是提供一些基础的代码支持，不会直接参与到 u-boot 的功能逻辑中

## 一般平台启动流程

uboot 的启动流程从`arch/arm/cpu/armv7/start.S`中的 reset vector 开始，这是所有 ARM 程序的入口点，完成下面工作：

1. save board registers
2. disable interrupts
3. setup CP15 寄存器（cache、MMU）
4. 跳转到`cpu_init_crit`进行关键初始化，如 memory
5. 跳转到`_main`，设置 C 运行环境，准备调用`board_init_f`
6. board_init_f 函数，完成一些前期的初始化工作，包括点灯、init DDR 等
7. 跳转到`board_init_r`，进入更高级别的初始化，包括设备树的处理，驱动的初始化，命令行的处理等（spl 调用不同的 board_init_r 实现）

![alt text](/U-Boot/uboot.jpg)

uboot 和 uboot-spl 代码是共用的，在 spl 的实现中，`board_init_f` 函数完成之后，流程才会继续执行 u-boot 映像。因此，SPL 需要完成的所有工作都应该放在 `board_init_f` 函数中执行。这包括启动 RTOS，它也应当在 `board_init_f` 中进行初始化，以确保顺利过渡到引导流程的后续阶段。

## uboot-spl 双系统启动流程

### 1. \_start(arch/arm/lib/vector.S)

\_start 是整个 spl 的入口，首先执行 `u-boot/arch/arm/lib/vector.S`的`_start`，然后跳转到`start.S`的`reset`，然后的流程和 uboot 一致
uboot-spl 在链接脚本`u-boot-spl.lds`中定义了 spl 的入口函数：

```armasm
ENTRY(_start)

_start:
#ifdef CONFIG_ARCH_K3
    ldr    pc, _reset
#end
    b      reset
```

spl 的流程在 reset 中就应该被结束，reset 最后应该转到 BL2，也就是 uboot 中了

### 2. reset(arch/arm/cpu/armv7/start.S)

#### 2.1 关闭中断

```armasm
reset:
...
/*
     * disable interrupts (FIQ and IRQ), also set the cpu to SVC32 mode,
     * except if in HYP mode already
     */
    mrs r0, cpsr
    and r1, r0, #0x1f       @ mask mode bits
    teq r1, #0x1a       @ test for HYP mode
    bicne   r0, r0, #0x1f       @ clear all mode bits
    orrne   r0, r0, #0x13       @ set SVC mode
    orr r0, r0, #0xc0       @ disable FIQ and IRQ
    msr cpsr,r0
```

#### 2.2 cpu_init_cp15

关闭中断后，跳转到`cpu_init_cp15`，关闭 MMU，关闭 caches，关闭 MMU 权限检查，关闭 MMU 异常处理

```armasm
reset:
...
bl cpu_init_cp15
```

#### 2.3 cpu_init_crit

跳转到`cpu_init_crit`函数，进行重要寄存器和内存时序的配置

```armasm
reset:
...
bl cpu_init_crit
```

在`cpu_init_crit`中，跳转到`lowlevel_init`函数，配置 pll、mux 和 memory

```armasm
ENTRY(cpu_init_crit)
    bl lowlevel_init
```

### 3. lowlevel_init(arch/arm/mach-aspeed/ast2600/platform.S)

u-boot 默认的 lowlevel_init 函数是在`arch/arm/cpu/armv7/lowlevel_init.S`中定义的，但是定义为弱符号链接（.weak），该实现由板级代码`arch/arm/mach-aspeed/ast2600/platform.S`中定义的`lowlevel_init`函数替代

完成`lowlevel_init`函数后，return `reset`(arch/arm/cpu/armv7/start.S)并跳转到`_main`函数

```armasm
@ arch/arm/cpu/armv7/start.S
reset:
...
bl _main
```

### 4. main(arch/lib/crt0.S)

#### 4.1 C 运行时初始化

堆栈、GD、early malloc 空间的分配

```armasm
_main:
    ...
    bl board_init_f_alloc_reserve
    ...
    bl board_init_f_init_reserve
    ...
```

#### 4.2 board_init_f

relocate 前的初始化，以弱符号链接定义在`arch/arm/lib/spl.c`中,实现在不同的板级代码中。在 ast2600 上，`board_init_f`实现在`arch/arm/mach-aspeed/ast2600/spl.c`中

```c
void board_init_f(ulong dummy)
{
    spl_early_init();
    timer_init();
    preloader_console_init();
    dram_init();
    aspeed_mmc_init();
#ifdef CONFIG_SPX_FEATURE_FREERTOS_SUPPORT
    rtos_init();
#endif
}
```

rtos 在`board_init_f`中进行初始化

