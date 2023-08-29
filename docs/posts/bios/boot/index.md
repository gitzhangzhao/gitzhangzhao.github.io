# Boot 过程分析

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

## After BIOS 被启动

> 在此之前，需要先明确什么是 BIOS？以及目前的 BIOS 是什么架构？

**需要明确的是，启动 OS 并不一定需要使用 BIOS**，我们常见的嵌入式设备（arm），它们使用 BootLoader 来引导 OS（一般是指 Linux），BootLoader 相当于 BIOS 的角色，负责初始化内存，加载 OS 内核等。由于嵌入式设备的硬件环境各不相同，没有统一的标准，所以在不同嵌入式硬件上运行 OS 每次都得修改配置 BootLoader 和内核，比较繁琐

**BIOS 存在的条件是统一的硬件，像嵌入式这种就完全没有必要了**。传统的 PC 设备和服务器则不一样，整个硬件系统架构都是有标准的，早期由 Intel，IBM，AMD 等大厂制定，发展至今已成为一套成熟且统一的系统架构。同样的，既然硬件能统一，OS 也可以做到一致。此时，要让 OS 可以在硬件工作起来，还需要一段引导程序，负责检初始化硬件（如初始化内存），检测硬件资源（如可否正常分配资源）等，前面这些都正常了，说明 OS 正常运行的条件满足了，此时做最后一步就是引导启动 OS 了，这一阶段的程序就称为 BIOS。BIOS 就是分级的配置各种硬件寄存器，Load Kernel, 启动 Boot Manager。这个过程是很复杂的，OS Loader 然后把控制权交接给 Kernel。最后，退居幕后给 Kernel 提供运行时服务

**目前的 BIOS 采用 UEFI 标准设计**，是一个简易的小操作系统，在完成主线任务的同时，也提供 UI、Shell 等方便交互的程序

> 早期以汇编为主的 BIOS 经过发展被目前的 UEFI BIOS 所替代。要了解 UEFI，还需要明确下面的概念：

首先，UEFI 只是一个规范，没有具体的实现。**就目前而言，BIOS 就是指 UEFI，UEFI 就是 BIOS，不需要再进行区分**。UEFI 是 Intel 及其小弟联合的规范，后来分为 UEFI 和 PI 两个部分，PI 后面再说。UEFI 侧重于和操作系统的接口，纯粹地是一个接口规范，它不会具体涉及平台固件是如何实现的。对于一台计算机，UEFI 固件提供服务，grub 这类 OS Loader 依赖于 UEFI 固件提供的接口来启动 Linux 内核。UEFI 是一套规范，阐述了 UEFI 需要实现哪些功能，这些功能该怎么实现，实现的时候需要使用什么名称。比如规范中有一个 USB 相关的定义，有一个 USB Host Controller Protocol，用来控制 USB 控制器，这个 Protocol 需要由 13 个函数组，每个函数都有详细注明了它的使用方法，传入传出参数，返回值等，但是具体函数内部具体如何实现是不做要求的，只要能实现对应的功能即可:

```c
typedef struct _EFI_USB2_HC_PROTOCOL {
  EFI_USB2_HC_PROTOCOL_GET_CAPABILITY              GetCapability;
  EFI_USB2_HC_PROTOCOL_RESET                       Reset;
  EFI_USB2_HC_PROTOCOL_GET_STATE                   GetState;
  EFI_USB2_HC_PROTOCOL_SET_STATE                   SetState;
  EFI_USB2_HC_PROTOCOL_CONTROL_TRANSFER            ControlTransfer;
  EFI_USB2_HC_PROTOCOL_BULK_TRANSFER               BulkTransfer;
  EFI_USB2_HC_PROTOCOL_ASYNC_INTERRUPT_TRANSFER    AsyncInterruptTransfer;
  EFI_USB2_HC_PROTOCOL_ASYNC_INTERRUPT_TRANSFER    SyncInterruptTransfer;
  EFI_USB2_HC_PROTOCOL_ISOCHRONOUS_TRANSFER        IsochronousTransfer;
  EFI_USB2_HC_PROTOCOL_ASYNC_ISOCHRONOUS_TRANSFER  AsyncIsochronousTransfer;
  EFI_USB2_HC_PROTOCOL_GET_ROOTHUB_PORT_STATUS     GetRootHubPortStatus;
  EFI_USB2_HC_PROTOCOL_SET_ROOTHUB_PORT_FEATURE    SetRootHubPortFeature;
  EFI_USB2_HC_PROTOCOL_CLEAR_ROOTHUB_PORT_FEATURE  ClearRootHubPortFeature
  UINT16                                           MajorRevision;
  UINT16                                           MinorRevision;
} EFI_USB2_HC_PROTOCOL;
```

**UEFI 是标准，是给别人用的**。UEFI 的使用者包括但不仅限于操作系统加载器（OS Loader），安装程序（installers），来自引导设备的 ROM（adapter ROMS），操作系统的预诊断程序（pre-OS diagnostics），工具（utilities）以及操作系统运行时服务（OS runtimes-services）。通常，UEFI 是关于如何进行引导过程的。引导就是一个将控制权限连续，逐级地移交，从而启动整个 OS 的过程，这就是 OS Loader 所肩负的职责

**UEFI 的规范可以当做是一套库函数，是用来被调用的**，那要怎么去使用这些库呢？整套引导的代码该从哪里写起？这时候一份称为《Platform Initialization Specification》（简称 PI Spec）的手册派上用场了

**PI （Platform Initialization）规定了如何实现一个 UEFI 环境**，是另一个规范，从 UEFI 分离出来的。PI 描述了从平台重启直到构建出 UEFI 兼容环境所需的完整过程。“如何实现”这一内容是 PI 要解决的问题。与 UEFI 的开放不同，PI 对 pre-OS 引导程序，OS，以及他们的加载器，在极大程度上是无关的，因为 PI 中有太多与这些 UEFI 使用者无关的平台构造方面的程序。PI 描述了从平台重启直到构建出 UEFI 兼容环境所需的完整过程。PI 手册定义了 UEFI 代码的各个启动阶段（除了 SEC 阶段）的流程，每个阶段需要做哪些操作。此外，还定义了启动过程中使用到的一些机制，如 HOB，SMM，S3 Resume 等

> 基于以上的规范，UEFI 代码 EDK 便可以实现了，目前最新的代码已经更名为 EDK2，是目前所有 UEFI 固件的基础代码

**EDK2 主要是实现手册所定义的各项基本功能，是一套通用的代码**，并不能直接拿来使用，还需要根据各个平台的差异做修改或对功能进行扩充，此时就有了做这项工作 UEFI 代码厂商，他们会整合优化 EDK2 的基础代码和芯片厂商（Intel、AMD 还有中国的龙芯，兆芯等）的核心代码，有些还会加进一些自家的特色功能，让代码使用起来更加便捷。AMI 就是比较著名的厂家，他们的代码都进行了深度的定制，很多功能几乎只要改改 enable 跟 disable 就可以了

**在了解了基本概念后，开始分析 BIOS 的执行**

### BIOS 进入

---

待续

[1]: https://pic.imgdb.cn/item/64ccbcdf1ddac507cc907466.png

