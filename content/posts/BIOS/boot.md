---
title: "Boot 过程分析"
date: 2023-08-04T14:00:29+08:00
draft: false
math: true
tags: ["BIOS","UEFI","X86","Boot"]
categories: ["BIOS"]
summary: "BIOS has now evolved into UEFI"
---
***申明: 本文严禁任何组织或个人在CSDN上进行转载,其他平台转载需经作者授权***

本篇文章详细完整的讨论现代处理器 boot 的过程，主要面向对象为 Intel、AMD 的 X86 架构和大部分 ARM 处理器架构

![][1]

 **前置概念**

- CPU 的引脚宝贵，只能用来连接高速设备（包括 Memory 和 PCIE 设备），原本通过北桥芯片来连接。但是目前，北桥芯片（用来连接内存和高速 PCIE）已经被集成进 CPU 中

- 低速的 IO（USB、SATA、eSPI 等）和其余的 PCIE 设备，不会直连 CPU 的，而是通过南桥芯片（PCH）来连接。PCH 是一块 IO 密集型芯片，BMC 芯片和 BIOS 芯片也是连接在 PCH 上的。然后 PCH 通过高速串行的总线 DMI 来和 CPU 连接，这样的目的是节约 CPU 的引脚（本质上是引脚复用）

## After CPU 加电或 Reset 后，到 BIOS 执行之前的阶段

### 1. CPU 自检

- CPU 进行自身硬件初始化。初始化完成后，CPU 被设置为实地址模式，地址无分页。所有寄存器被初始化为特定的值， Cache、TLB（Translation Lookup Table）、BLB（Branch Target Buffer）这三个部件的内容被清空（Invalidate）

### 2. 设置 CPU 寄存器

- 寄存器 EIP（Instruction Pointer）、CS（Code Segment）被设置为 0x0000FFF0 和 0xFFFF0000。在实地址模式下（寄存器字长为 16 位），指令的物理地址是 CS << 4 + EIP。CPU 根据硬件设计，计算出第一条指令的地址：0xFFFF0000+0xFFF0 = 0xFFFFFFF0。随后，CPU 会从这个地址取指令并执行，需要在这个地址存放 BIOS 的代码

### 3. Load BIOS

- 现在的问题是：在上电启动后，CPU 外围设备包括内存初始化还没有进行，没有内存可供使用，虽然有可以直接使用 Cache 替代内存的方法(Cache As RAM，CAR)，但总是没有直接用起来方便

- 但是，BIOS 代码是存放在一块 NOR Flash 中的，这块 Flash 在主板上通过 SPI 总线与 PCH 相连。NOR Flash 和我们用在 SSD 里面的 Flash 一个显著的不同就是它是字节寻址的，而不是块寻址。这就意味着它可以 XIP（eXecute in place），直接执行代码而不需要先 copy 到内存中

- 随后，CPU 去寻址 0xFFFFFFF0，PCH 的 SPI 总线默认 decode 该地址，从 Flash 芯片取指令；SPI 控制器响应 Flash 并返回内容给 DMI 总线（连接 PCH 和 CPU 的总线）；DMI 总线将指令给 CPU，开始解码执行。随后，通过这种方式一点一点 decode 运行，这个过程通常又称为 shadow

---

待续

[1]: https://pic.imgdb.cn/item/64ccbcdf1ddac507cc907466.png
