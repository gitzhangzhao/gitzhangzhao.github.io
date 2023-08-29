---
title: "UEFI BDS 阶段代码梳理"
date: 2023-08-29T16:37:13+08:00
draft: false
math: true
tags: ["BIOS", "EDK2", "UEFI", "AMI", "X86"]
categories: ["BIOS"]
summary: ""
---
### DXE Install BDS Protocol

1. 在 DXE 阶段的 Main 中，调用`CoreDispatcher() -> CoreStartImage()`，进入各 DXE driver 的入口函数:

```c
// MdeModulePkg/Core/Dxe/DxeMain/DxeMain.c

Image->Status = Image->EntryPoint (ImageHandle, Image->Info.SystemTable);
```

此阶段会 load 并执行各 DXE 的驱动程序，而 BDS 就属于 DXE 的一个驱动，所以`BdsInitialize()`这个函数会在此时被执行：

```c
// MdeModulePkg/Universal/BdsDxe/BdsDxe.inf

[Defines]
INF_VERSION                    = 0x00010005
BASE_NAME                      = BdsDxe
MODULE_UNI_FILE                = BdsDxe.uni
FILE_GUID                      = 6D33944A-EC75-4855-A54D-809C75241F6C
MODULE_TYPE                    = DXE_DRIVER                            // BDS是DXE驱动
VERSION_STRING                 = 1.0
ENTRY_POINT                    = BdsInitialize                         // 驱动入口函数
```

入口函数`BdsInitialize()`调用`InstallMultipleProtocolInterfaces()`注册 BdsArchProtocol 到 DXE Core，该 Protocol 的 Guid 为`gEfiBdsArchProtocolGuid`，`*gBds`是一个指向 BdsArchProtocol 的指针，在`MdeModulePkg/Core/Dxe/DxeMain/DxeMain.c`开始时定义：

```c
// `MdeModulePkg/Core/Dxe/DxeMain/DxeMain.c

EFI_BDS_ARCH_PROTOCOL             *gBds           = NULL;
```

在 BDS 初始化阶段`BdsInitialize`，将该指针作为`InstallMultipleProtocolInterfaces()`的参数，目的是通过`gEfiBdsArchProtocolGuid`可以查询到`*gBds`：

```c
// MdeModulePkg/Universal/BdsDxe/BdsEntry.c

// Install protocol interface
Handle = NULL;
Status = gBS->InstallMultipleProtocolInterfaces (
                &Handle,
                &gEfiBdsArchProtocolGuid,
                &gBds,
                NULL
                );

```

在`InstallMultipleProtocolInterfaces()`之前，`*gBds`就已经指向 BDS 的入口函数`BdsEtry`：

```c
// MdeModulePkg/Universal/BdsDxe/BdsEntry.c
EFI_BDS_ARCH_PROTOCOL  gBds = {
BdsEntry
};
```

再讨论一下 BDS 的 Protocol，`EFI_BDS_ARCH_PROTOCOL`是在`Bds.h`中定义的：

```c
// MdePkg/Include/Bds.h
typedef struct _EFI_BDS_ARCH_PROTOCOL EFI_BDS_ARCH_PROTOCOL;

typedef
VOID
(EFIAPI *EFI_BDS_ENTRY)(
  IN EFI_BDS_ARCH_PROTOCOL  *This
);

struct _EFI_BDS_ARCH_PROTOCOL {
  EFI_BDS_ENTRY    Entry;
};
```

其中，`EFI_BDS_ENTRY `就是这个 protocol 的一个服务，对应了一个`PROTOCOL_INTERFACE`。在之前的`InstallMultipleProtocolInterfaces()`函数中，`Entry`函数被指向`EFI_BDS_ARCH_PROTOCOL`这个 Protocol 的例化`PROTOCOL_INTERFACE`中的`*Interface`指针。所以根据 Bds 的 Guid `EFI_BDS_ARCH_PROTOCOL_GUID`可以找到`EFI_BDS_ARCH_PROTOCOL`这个 Protocol，再根据这个 Protocol 找到`PROTOCOL_INTERFACE`实现，就可以找到具体的被注册的`BdsEntry()`函数了

另外，`*gBds`属于全局变量：

```c
// MdeModulePkg/Core/Dxe/DxeMain.h
extern EFI_BDS_ARCH_PROTOCOL             *gBds;
```

最后，在 Bds 的入口函数已经被安装后，可以直接通过`*gBds`指针访问`Entry()`函数了：

```c
gBds->Entry (gBds);
```

4.

---
待续
