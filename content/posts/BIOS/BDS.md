---
title: "UEFI BDS 阶段代码梳理"
date: 2023-08-29T16:37:13+08:00
draft: false
math: true
tags: ["BIOS", "EDK2", "UEFI", "AMI", "X86"]
categories: ["BIOS"]
summary: "UEFI 系列第三篇"
---
### DXE Install BDS Protocol

1. 在DXE阶段的Main中，调用`CoreDispatcher() -> CoreStartImage()`，进入各DXE driver的入口函数:

  ```c
  // MdeModulePkg/Core/Dxe/DxeMain/DxeMain.c

  Image->Status = Image->EntryPoint (ImageHandle, Image->Info.SystemTable);
  ```

  此阶段会load并执行各DXE的驱动程序，而BDS就属于DXE的一个驱动，所以`BdsInitialize()`这个函数会在此时被执行：

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

  入口函数`BdsInitialize()`调用`InstallMultipleProtocolInterfaces()`注册BdsArchProtocol到DXE Core，该Protocol的Guid为`gEfiBdsArchProtocolGuid`，`*gBds`是一个指向BdsArchProtocol的指针，在`MdeModulePkg/Core/Dxe/DxeMain/DxeMain.c`开始时定义：

  ```c
  // `MdeModulePkg/Core/Dxe/DxeMain/DxeMain.c

  EFI_BDS_ARCH_PROTOCOL             *gBds           = NULL;
  ```

  在`BdsEntry.c`中BDS初始化阶段`BdsInitialize`，`*gBds`指针作为`InstallMultipleProtocolInterfaces()`的参数，来将入口函数`BdsEntry`注册到`PROTOCOL_INTERFACE`中：

  ```c
  // MdeModulePkg/Universal/BdsDxe/BdsEntry.c
  EFI_BDS_ARCH_PROTOCOL  gBds = {
  BdsEntry
  };
  ```

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

  需要注意的是：在`BdsEntry.c`中的`gBds`只是为了注册，而在`DxeMain()`最后进入Bds阶段使用的指针是在别的地方赋值的。Dxe中有如下结构体：

  ```c
  EFI_CORE_PROTOCOL_NOTIFY_ENTRY  mArchProtocols[] = {
    { &gEfiSecurityArchProtocolGuid,         (VOID **)&gSecurity,      NULL, NULL, FALSE },
    { &gEfiCpuArchProtocolGuid,              (VOID **)&gCpu,           NULL, NULL, FALSE },
    { &gEfiMetronomeArchProtocolGuid,        (VOID **)&gMetronome,     NULL, NULL, FALSE },
    { &gEfiTimerArchProtocolGuid,            (VOID **)&gTimer,         NULL, NULL, FALSE },
    { &gEfiBdsArchProtocolGuid,              (VOID **)&gBds,           NULL, NULL, FALSE },
    ...
    }
  ```

  首先通过这个数组将`*gBds`和`gEfiBdsArchProtocolGuid`建立起联系，然后在Dxe阶段的Main函数中，通过`CoreNotifyOnProtocolInstallation()->CoreNotifyOnProtocolEntryTable()`创建事件(Event)，这里传入的Entry为`mArchProtocols`，最终由`GenericProtocolNotify()`函数给`mArchProtocols`的每一个成员赋值。这里，`CoreCreatEvent()`和`CoreRegisterProtocolNotify`成对出现，一旦Protocol安装则执行回调函数`GenericProtocolNotify()`

  ```c
  /**
  Creates an event for each entry in a table that is fired everytime a Protocol
  of a specific type is installed.

  @param Entry  Pointer to EFI_CORE_PROTOCOL_NOTIFY_ENTRY.

  **/
  VOID
  CoreNotifyOnProtocolEntryTable (
    EFI_CORE_PROTOCOL_NOTIFY_ENTRY  *Entry
    )
  {
    EFI_STATUS  Status;

    for ( ; Entry->ProtocolGuid != NULL; Entry++) {
      //
      // Create the event
      //
      Status = CoreCreateEvent (
               EVT_NOTIFY_SIGNAL,
               TPL_CALLBACK,
               GenericProtocolNotify,
               Entry,
               &Entry->Event
               );
      ASSERT_EFI_ERROR (Status);

      //
      // Register for protocol notifactions on this event
      //
      Status = CoreRegisterProtocolNotify (
               Entry->ProtocolGuid,
               Entry->Event,
               &Entry->Registration
               );
      ASSERT_EFI_ERROR (Status);
    }
  }
  ```

  在回调函数`GenericProtocolNotify`中，Bds入口函数`BdsEntry()`被locate，然后被赋值给`*gBds`：

  ```c
  //
  // See if the expected protocol is present in the handle database
  //
  Status = CoreLocateProtocol (Entry->ProtocolGuid, Entry->Registration, &Protocol);
  if (EFI_ERROR (Status)) {
    return;
  }
  // Mark the protocol as present
  Entry->Present = TRUE;
  // Update protocol global variable if one exists. Entry->Protocol points to a global variable
  // if one exists in the DXE core for this Architectural Protocol
  if (Entry->Protocol != NULL) {
    *(Entry->Protocol) = Protocol;
  }
  ```

  最后，在Bds的入口函数已经被安装后，`DxeMain()`的最后通过`*gBds`指针进入Bds的入口`BdsEntry()`：

  ```c
  gBds->Entry (gBds);
  ```

  再讨论一下BDS的Protocol，`EFI_BDS_ARCH_PROTOCOL`是在`Bds.h`中定义的：

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

  其中，`EFI_BDS_ENTRY `就是这个protocol的一个服务，对应了一个`PROTOCOL_INTERFACE`。在之前的`InstallMultipleProtocolInterfaces()`函数中，`Entry`函数被指向`EFI_BDS_ARCH_PROTOCOL`这个Protocol的例化`PROTOCOL_INTERFACE`中的`*Interface`指针。所以根据Bds的Guid `EFI_BDS_ARCH_PROTOCOL_GUID`可以找到`EFI_BDS_ARCH_PROTOCOL`这个Protocol，再根据这个Protocol找到`PROTOCOL_INTERFACE`实现，就可以找到具体的被注册的`BdsEntry()`函数了
