# 理解 UEFI 中的面向对象

> UEFI 并不是一个面向对象的系统，它是基于 C 语言的，C 语言是一种过程式语言。然而，UEFI 使用了一些设计模式和技术来模拟面向对象编程的某些特性，如封装、抽象和多态

1. 封装：UEFI 使用结构体（struct）来封装数据和操作数据的函数。例如，每个 UEFI 协议都定义为一个结构体，其中包含一组函数指针，这些函数提供了协议的行为。这种方式类似于面向对象编程中的类和方法（Kernel 中也是类似）
2. 抽象：UEFI 使用接口（协议）来定义可以由多个不同的实现提供的行为。这类似于面向对象编程中的接口或抽象类
3. 多态：UEFI 通过使用函数指针和接口（协议）来实现多态。不同的驱动可以提供同一接口的不同实现，然后通过接口来调用这些函数，实现运行时的多态
4. 继承：UEFI 并没有提供类似于面向对象编程中的继承机制。然而，它使用了一种叫做“装饰者”模式的设计模式，通过这种方式，一个驱动可以“装饰”另一个驱动，提供额外的功能，这在某种程度上模拟了继承的行为

## handle 和 protocol 的概念

UEFI 协议把访问设备的方法都抽象成了`Handle`和`Protocol`，Handle 是一个抽象的引用，用于引用一个或多个协议接口。一个设备便可以当成是一个 Handle（也可以当成是一个实例 Instance），而 Protocol 则是一个封装了某些操作方法的类（Class）。与面向对象有点区别的是，一个 Handle 可能是由多个 Protocols 组成的

Protocol 是一个由 struct 定义的结构体，这个结构体通常是由数据和函数指针组成。每个结构体的定义都有一个 GUID 与之对应。自然并不是所有的结构体都称之为 protocol，protocol 正如其名，它是一种规范，或称协议。比如要建立一个基于 UEFI Driver Model 的 Driver，就必须要绑定一个 EFI_DRIVER_BINGING_PROTOCOL 的实例，并且要自定义且实现 Support、Start、Stop 函数以及填充实例中其他的数据成员。再例如，EFI_SIMPLE_TEXT_INPUT_PROTOCOL 是一个协议，它定义了一组函数，这些函数可以用于从键盘读取输入

**Handle 和 Protocol 都是通过双向链表组织的**

定义的三个关键结构体：

```c
// MdeModulePkg/Core/Dxe/Hand/Handle.h

///
/// IHANDLE - contains a list of protocol handles
///
typedef struct {
  UINTN         Signature;
  /// All handles list of IHANDLE
  LIST_ENTRY    AllHandles;
  /// List of PROTOCOL_INTERFACE's for this handle
  LIST_ENTRY    Protocols;
  UINTN         LocateRequest;
  /// The Handle Database Key value when this handle was last created or modified
  UINT64        Key;
} IHANDLE;

///
/// PROTOCOL_ENTRY - each different protocol has 1 entry in the protocol
/// database.  Each handler that supports this protocol is listed, along
/// with a list of registered notifies.
///
typedef struct {
  UINTN         Signature;
  /// Link Entry inserted to mProtocolDatabase
  LIST_ENTRY    AllEntries;
  /// ID of the protocol
  EFI_GUID      ProtocolID;
  /// All protocol interfaces
  LIST_ENTRY    Protocols;
  /// Registerd notification handlers
  LIST_ENTRY    Notify;
} PROTOCOL_ENTRY;

typedef struct {
  UINTN             Signature;
  /// Link on IHANDLE.Protocols
  LIST_ENTRY        Link;
  /// Back pointer
  IHANDLE           *Handle;
  /// Link on PROTOCOL_ENTRY.Protocols
  LIST_ENTRY        ByProtocol;
  /// The protocol ID
  PROTOCOL_ENTRY    *Protocol;
  /// The interface value
  VOID              *Interface;
  /// OPEN_PROTOCOL_DATA list
  LIST_ENTRY        OpenList;
  UINTN             OpenListCount;
} PROTOCOL_INTERFACE;
```

要明白 IHANDLE 这个结构体，就要明白 LIST_ENTRY 是如何被使用的。LIST_ENTRY 定义如下：

```c
// MdeModulePkg/Include/LinkedList.h
///
/// _LIST_ENTRY structure definition.
///
struct _LIST_ENTRY {
  LIST_ENTRY    *ForwardLink;
  LIST_ENTRY    *BackLink;
};

#define _LIST_ENTRY LIST_ENTRY
```

首先，上面的**LIST_ENTRY 这个结构体用于实现双向链表**。但是与一般的链表实现方式不一样，它纯粹是 LIST*ENTRY 这个成员的链接，而不用在乎这个成员所在的结构体。一般的链表要求结点之间的类型一致，而这种链表只要求结构体存在 EFI_LIST_ENTRY 这个成员就够了。比如说 `IHANDLE *handle1,_handle2`;初始化后，`handle1->AllHandles->ForwardLink=handle2->AllHandles`; `handle2->AllHandles->BackLink=handle1->AllHandles`。这样 handle1 与 handle2 的 AllHandles 就链接到了一起。但是这样就只能进行 AllHandles 的遍历了，怎么样遍历 IHANLE 实例呢？。这时候就要用到\_CR 宏，\_CR 宏的定义如下：`#define \_CR(Record, TYPE, Field) ((TYPE _) ((CHAR8 _) (Record) - (CHAR8 _) &(((TYPE \_) 0)->Field)))`，这个宏可以通过结构体实例的成员访问到实例本身

**IHANDLE 中的 AllHandles 成员用来链接 IHANDLE 实例**。这个链表的头部是一个空结点，定义为：`EFI_LIST_ENTRY gHandleList`。一开始 `gHandleList->ForwardLink=gHandleList`; `gHandleList->BackLink=gHandleList`。每次 IHANDLE 都从 `gHandleList->BackLink` 插入进来，这个链表是一个环形双向链表。每当 Driver 建立一个新的 EFI_HANDLE 的时候就会插入到这条链表中来，被称之为 handle database

**Driver 会为 handle 添加多个 protocol**，这些实例也是链表的形式存在。PROTOCOL_INTERFACE 的 link 用于连接以 IHANDLE 为空头结点以 PPOTOCOL_INTERFACE 为后续结点的链表

具体而言：IHANDLE 结构体定义中，Protocols 是一个 LIST_ENTRY 类型的成员，它是一个双向链表。这个链表用于链接所有的 PROTOCOL_INTERFACE 实例。在这个链表中，每个节点都包含两个指针：ForwardLink 和 BackLink。ForwardLink 指向链表中的下一个节点，BackLink 指向链表中的上一个节点。这样，通过遍历这个链表，就可以访问到所有的 PROTOCOL_INTERFACE 实例

此外，PROTOCOL_ENTRY 结构体是用来管理和跟踪已注册的协议的。每个协议都有一个对应的 PROTOCOL_ENTRY 实例，这个实例包含了协议的标识符和一个链表，这个链表链接了所有安装了这个协议的协议接口。例如，假设我们有一个协议 EFI_SIMPLE_TEXT_INPUT_PROTOCOL，它的标识符是{0x387477c1, 0x69c7, 0x11d2, {0x8e, 0x39, 0x00, 0xa0, 0xc9, 0x69, 0x72, 0x3b}}。当这个协议被注册时，会创建一个 PROTOCOL_ENTRY 实例，这个实例的 ProtocolID 成员被设置为这个标识符，Protocols 链表被初始化为一个空链表。然后，当一个驱动程序安装了一个 EFI_SIMPLE_TEXT_INPUT_PROTOCOL 协议接口到一个句柄上时，这个协议接口的 PROTOCOL_INTERFACE 实例就会被添加到 Protocols 链表中。这样，通过遍历 Protocols 链表，就可以找到所有安装了 EFI_SIMPLE_TEXT_INPUT_PROTOCOL 的协议接口

还有一个概念，就是 GUID，**每个协议接口都由一个全局唯一标识符（GUID）来标识**。当一个驱动程序或应用程序想要使用一个特定的协议接口时，它需要通过 GUID 来查找这个协议接口。这个过程通常是通过调用 LocateProtocol 或 OpenProtocol 这样的 UEFI 服务来完成的。例如，如果一个驱动程序想要使用 EFI_SIMPLE_TEXT_INPUT_PROTOCOL（这是一个用于从键盘读取输入的协议），它需要先获取这个协议的 GUID，然后调用 LocateProtocol 函数，传入这个 GUID 作为参数。如果成功，LocateProtocol 函数会返回一个指向 EFI_SIMPLE_TEXT_INPUT_PROTOCOL 接口的指针，然后驱动程序就可以通过这个指针来调用协议的函数。这种通过 GUID 来访问协议接口的机制使得 UEFI 可以在运行时动态地添加、删除和查找协议接口，这是一种非常灵活和强大的设计

根据此前的分析：在 UEFI 中，一个句柄（Handle）可以关联多个协议接口（Protocol Interface），这些协议接口被组织成一个链表，这个链表可以通过句柄的 Protocols 成员来访问。然而，在实际的使用中，通常不会直接操作这个链表。相反，UEFI 提供了一组服务，如 LocateProtocol 和 OpenProtocol，这些服务可以根据 GUID 来查找协议接口，这使得查找协议接口变得更加简单和安全。总的来说，虽然可以直接通过句柄的 Protocols 成员来访问所有的协议接口，但在实际的使用中，通常会通过 UEFI 提供的服务来查找和访问协议接口

**以上的概念用下图来描述：**

![][1]

---

待续

[1]: https://pic.imgdb.cn/item/64d206df1ddac507ccefe33b.jpg

