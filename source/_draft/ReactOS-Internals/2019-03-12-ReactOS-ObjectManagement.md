
#ReactOS 对象管理#

对于所有的对象，Windows内核提供了统一的操作模式，先通过系统调用“打开”或创建目标对象，让当前进程与目标对象之前建立连接，然后在通过的别的系统调用进行操作，最后通过系统调用“关闭”对象，实际上是关闭当前进程与目标对象的联系。

在打开对象的时候，一方面要说明对象的类别，另一方面要说明怎样找到具体的目标对象，即提供目标对象的“路径”。显然“打开”对象也是定义在对象上的操作之一，但有时一种特殊的操作，其实就是让当前进程与目标对象之间建立连接。

Windows内核中的对象究竟是什么呢？事实上Windows内核中的对象和面向对象的程序设计中所说的对象有很大不同，Windows内核中对象并没有继承，多态等基本要素，所以做多只能称为“基于对象的程序设计”。在Windows内核中对象的操作与数据结构体的结合很松散，所以Windows内核中的对象实质上就是数据结构体，一些带有“对象头”的特殊数据结构。

任何对象的数据结构都由对象头和具体对象数据两部分构成。但实际上对象的数据结构体被分为三部分，对象头的结构`OBJECT_HEADER`，对象的数据，比如事件对象的`KEVENT`，文件对象的`FILE_OBJECT`，还有一部分是可选结构，这部分可选的附加信息包括：创建者信息结构体`OBJECT_HEADER_CREATOR_INFO`，对象名和目录节点指针的结构`OBJECT_HEADER_NAME_INFO`，有关句柄的信息`OBJECT_HEADER_HANDLE_INFO`数据结构，关于内存配额的信息`OBJECT_HEADER_QUOTA_INFO`等。


|OBJECT_HEADER_QUOTA_INFO  |
|--------------------------|
|OBJECT_HEADER_HANDLE_INFO |
|OBJECT_HEADER_NAME_INFO   |
|OBJECT_HEADER_CREATOR_INFO|
| 对象头（Object Header）    |
| 对象体（Object Body）      |

在内核中对象指针指向的是对象体，而非对象头。要想找到对象头则需要`CONTAINING_RECORED((o), OBJECT_HEADER， Body)`方式来找到对象头。

```
kd> dt nt!_OBJECT_HEADER
   +0x000 PointerCount     : Int4B
   +0x004 HandleCount      : Int4B	// 句柄基数
   +0x004 NextToFree       : Ptr32 Void
   +0x008 Type             : Ptr32 _OBJECT_TYPE  // 对象类型
   +0x00c NameInfoOffset   : UChar	// OBJECT_HEADER_NAME_INFO 偏移
   +0x00d HandleInfoOffset : UChar	// OBJECT_HEADER_HANDLE_INFO 偏移
   +0x00e QuotaInfoOffset  : UChar	// OBJECT_HEADER_NAME_INFO 偏移
   +0x00f Flags            : UChar
   +0x010 ObjectCreateInfo : Ptr32 _OBJECT_CREATE_INFORMATION  指针
   +0x010 QuotaBlockCharged : Ptr32 Void
   +0x014 SecurityDescriptor : Ptr32 Void
   +0x018 Body             : _QUAD	// 对象体，即对象的实际内容
kd> dt ffff9d891ddc77e0 nt!_FILE_OBJECT
   +0x000 Type             : 0n5
   +0x002 Size             : 0n216
   +0x008 DeviceObject     : 0xffff9d88`faff7840 _DEVICE_OBJECT
   +0x010 Vpb              : 0xffff9d88`faef7ed0 _VPB
   +0x018 FsContext        : 0xffffc405`c2a9f700 Void
   +0x020 FsContext2       : 0xffffc405`c11acf60 Void
   +0x028 SectionObjectPointer : 0xffff9d89`19b8fcb8 _SECTION_OBJECT_POINTERS
   +0x030 PrivateCacheMap  : (null) 
   +0x038 FinalStatus      : 0n0
   +0x040 RelatedFileObject : (null) 
   +0x048 LockOperation    : 0 ''
   +0x049 DeletePending    : 0 ''
   +0x04a ReadAccess       : 0x1 ''
   +0x04b WriteAccess      : 0 ''
   +0x04c DeleteAccess     : 0 ''
   +0x04d SharedRead       : 0x1 ''
   +0x04e SharedWrite      : 0 ''
   +0x04f SharedDelete     : 0 ''
   +0x050 Flags            : 0x40042
   +0x058 FileName         : _UNICODE_STRING "\MyData\abc.txt"
   +0x068 CurrentByteOffset : _LARGE_INTEGER 0x0
   +0x070 Waiters          : 0
   +0x074 Busy             : 0
   +0x078 LastLock         : (null) 
   +0x080 Lock             : _KEVENT
   +0x098 Event            : _KEVENT
   +0x0b0 CompletionContext : (null) 
   +0x0b8 IrpListLock      : 0
   +0x0c0 IrpList          : _LIST_ENTRY [ 0xffff9d89`1ddc78a0 - 0xffff9d89`1ddc78a0 ]
   +0x0d0 FileObjectExtension : (null) 
```


```
kd> !handle 1ec

PROCESS ffff9d8917482080
    SessionId: 1  Cid: 0e8c    Peb: 00f86000  ParentCid: 1b9c
FreezeCount 2
    DirBase: 1f77e002  ObjectTable: ffffc405b8ae2840  HandleCount: 138.
    Image: Test.exe

Handle table at ffffc405b8ae2840 with 138 entries in use

01ec: Object: ffff9d891ddc77e0  GrantedAccess: 00120089 (Protected) (Inherit) Entry: ffffc405bb4e47b0
Object: ffff9d891ddc77e0  Type: (ffff9d88f8af7f00) File
    ObjectHeader: ffff9d891ddc77b0 (new version)
        HandleCount: 1  PointerCount: 32768
        Directory Object: 00000000  Name: \MyData\abc.txt {HarddiskVolume3}
```

