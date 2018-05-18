---
title: ReactOS 内存管理(上)
date: 2017-06-26 09:15:33
tags:
- ReactOS
- Memory
categories:
- ReactOS
---

上一篇总结了一下用Windbg调试系统调用的过程，暂时总结那么多，后面看到新鲜内容再补充进去。这一篇总结一下内存管理的知识，老规矩依照毛德操老师的书，学习一下基本原理，然后用Windbg调试一下过程，总结一下问题。

之前序言中说到现代操作系统的条件，提供对内核的保护，对不同用户程序之间的隔离，以及允许软件的装入位置浮动。要实现这些一方面要硬件支持，再者要有软件配合，软件的配合就是基于页面映射的“虚拟内存”机制。硬件上完成这个工作的就是MMU，如果没有启用该硬件，那么CPU直接使用物理地址访问内存。如果启用了MMU，那么ALU使用的地址就先要给到MMU，转成物理地址才可以使用。虚拟内存机制为每一个进程准备了相同范围的地址空间，不用的进程各自独立，相同的地址会被MMU映射到不同的物理内存，实现进程间相互隔离；由于虚拟地址与最终的物理地址有映射关系，并且不同进程相互隔离，这样同样的模块可以被映射到相同的地址，而不需要进行重定位；同时CPU的运行被分为用户态和内核态，用户态无法直接操作内核态的内存，这样就将内核保护了起来。基于页面映射的“虚拟内存”机制很自然地就满足了现代操作系统的基本要求。

基于页面映射的“虚拟内存”机制，最基本的一个问题就有一个映射。它就涉及到虚拟地址空间，物理地址“空间”，以及相互之间的关系。引申出来的就是虚拟地址空间管理（保存），物理地址空间管理（保存），虚拟地址空间到物理地址空间的映射关系管理（保存）等问题。按照如下的几个主题总结一下学习的内容：

1. 虚拟内存空间管理
2. 物理内存管理
3. 虚拟地址到物理地址映射
4. 读取其他进程内存的实现方法
5. NtAllocateVirtualMemory分配内存过程
6. 页面异常
7. 页面的换入与换出

> 对于进行过Windows编程，尤其了解过Windows安全相关内容的朋友都知道，PE文件的映射还是比较重要的。毛德操老师分析了映射Data文件的过程，后面专门调试一下PE文件的映射过程，加深认识。

ReactOS的内存管理基本上是基于CPU的MMU的功能展开的，毛老师给总结了MMU的功能，这里抄下来备查：

* 根据虚拟地址计算出该地址所属的页面
* 根据页面映射表其实地址计算出页面映射表所在的物理地址
* 根据物理地址在高速缓存TLB中查找表项内容
* 表项不存在于TLB中，从内存中将其装入TLB中
* 检查表项的PA_PRESENT标志位，如果为1就表示映射的目标为某个物理页面，可以访问这个页面。但是需要进一步检查权限，没有权限访问将引发缺页异常。
* 如果PA_PRESENT标志位为0，说明虚存页面不在内存，当前指令执行会失败，CPU产生页面异常，由异常处理程序进行处理：
	1. 如果页面映射表为0，说明虚存页面还没有映射，检查虚存页面是否落入已分配使用区间。如果是就为其分配物理内存，进行访问。如果不是就属于越界访问，应当引起更高层次出错处理。
	2. 如果整个页面映射表项非0，说明虚存页面映像存在于某个页面倒换文件之中，为其分配物理页面，从倒换文件中读入该页面的映像，页面映射表项修改指向物理页面，然后即可访问。

上面说MMU得到的是虚拟地址，其实不准确。在X86的CPU上，基于页面的映射机制其实是建立在基于段式内存管理基础之上。而在Windows以及Linux上，段机制的段寄存器被设置为指向整个地址空间的GDT表项，结果段式内存管理的线性地址就等于虚拟地址，所以上面就直接说虚拟地址了，以简化语言表达。

<!-- more -->
本篇中会分析NtAllocateVirtualMemory函数，所以自己编写一个程序，用于调试内存分配和释放过程。以AllocMemory.exe进程（VS2008编译通过，ReactOS上可以执行）为例来进行调试。该程序的代码如下代码块所示，具体内容不作解释。

```
#include "stdafx.h"
#include <Windows.h>

int _tmain(int argc, _TCHAR* argv[])
{
	printf("Before VirtualAllocEx!\n");
	system("pause");

	LPVOID pAlloc = VirtualAllocEx(GetCurrentProcess(), NULL, 0x1000, MEM_COMMIT|MEM_RESERVE, PAGE_READWRITE);

	printf("Before Write Content to Alloc Buffer!\n");
	system("pause");
	if (NULL != pAlloc)
	{
		sprintf((CHAR*)pAlloc, "Hello ReactOS!");
	}

	printf("Before Free Buffer!\n");
	system("pause");

	VirtualFreeEx(GetCurrentProcess(), pAlloc, 0x1000, MEM_FREE);

	printf("Before Exit main!\n");
	system("pause");

	return 0;
}
```

#### 虚拟内存空间的管理 ####

在编程中，实际使用的地址都是虚拟内存空间的地址，即虚拟地址。首先简单理一下虚拟内存空间管理的逻辑，这一小节按照毛德操老师书中线索总结一下。

第一个线索是在进程控制块EPROCESS中有一个字段VadRoot，它保存的是进程虚拟地址空间的使用情况。该成员的情况如下，与之前的版本不同的是，VadRoot是一块结构体内存`_MM_AVL_TABLE`，而非一个结构体指针；该结构体内容如下所示，它的第一个成员是BalancedRoot，类型为`MMADDRSS_NODE`结构体。在下面的代码块中给出了这两个结构体具体内容，很显然`MMADDRESS_NODE`结构体就是一个二叉树的节点，分别有左孩子，右孩子节点指针。再后面的两个成员`StartingVpn`和`EndingVpn`现在看并不知道是什么意思，看到后面就知道了，它其实是页目录和页表两个索引拼接，也即将虚拟地址右移12位后剩下的高20位（最高10位用于索引页目录表，中间10位用于索引页表，最低12位为页内偏移）。

```
kd> !process 0 0 AllocMemory.exe
PROCESS b22d6388  SessionId: 0  Cid: 030c    Peb: 7ffdf000  ParentCid: 023c
    DirBase: 3623b000  ObjectTable: e172ad68  HandleCount:   5.
    Image: AllocMemory.exe

kd> dt nt!_EPROCESS b22d6388 -y Vad
   +0x140 VadFreeHint : (null)
   +0x250 VadRoot : _MM_AVL_TABLE

kd> dd b22d6388 +0x250 L1
b22d65d8  b22d65db
kd> dt nt!_MM_AVL_TABLE b22d65d8
   +0x000 BalancedRoot     : _MMADDRESS_NODE 一个结构体，而非指针
   +0x014 DepthOfTree      : 0y00110 (0x6)
   +0x014 Unused           : 0y000
   +0x014 NumberGenericTableElements : 0y000000000000000000011100 (0x1c)
   +0x018 NodeHint         : 0xb22d3448
   +0x01c NodeFreeHint     : (null)
```

```
//
// Memory Manager AVL Table for VADs and other descriptors 用于VAD的内存管理AVL表
//
typedef struct _MM_AVL_TABLE
{
    MMADDRESS_NODE BalancedRoot;				// 二叉树 Root节点，其RightChild才是书根
    ULONG_PTR DepthOfTree:5;					// 二叉树深度
    ULONG_PTR Unused:3;							// 1 内核表，0 用户空间表
#ifdef _WIN64
    ULONG_PTR NumberGenericTableElements:56;
#else
    ULONG_PTR NumberGenericTableElements:24;	// 节点数
#endif
    PVOID NodeHint;
    PVOID NodeFreeHint;
} MM_AVL_TABLE, *PMM_AVL_TABLE;

//
// Node in Memory Manager's AVL Table 内存管理的AVL表中节点定义
//
typedef struct _MMADDRESS_NODE
{
    union
    {
        LONG_PTR Balance:2;
        struct _MMADDRESS_NODE *Parent;
    } u1;
    struct _MMADDRESS_NODE *LeftChild;
    struct _MMADDRESS_NODE *RightChild;
    ULONG_PTR StartingVpn;
    ULONG_PTR EndingVpn;
} MMADDRESS_NODE, *PMMADDRESS_NODE;

```

另外一个使用了`MM_AVL_TABLE`结构体的地方是`MM_AVL_TABLE MiRosKernelVadRoot;`全局变量的定义。MiRosKernelVadRoot是保存了内核中虚拟地址空间管理结构（AVL树根节点）的根节点，与用户空间的VadRoot没什么本质区别。

其实在EPROCESS结构体中还有另外一个成员Vm，它是一个`nt!_MMSUPPORT`结构体。从下面调试器中打印出的内容可以看到这个结构体中记录了当前进程内存使用的基本情况，包括上一次内存修剪时间，页面错误次数，工作机峰值，最大工作集，最小工作集等等（编译为不同的目标平台，以及设置不同的系统版本，结构体内的参数不同，其他的信息不再列出）。

```
kd> dt nt!_EPROCESS b230a710 -y vm
   +0x1e8 Vm : _MMSUPPORT
   +0x240 VmDeleted : 0y0
   +0x240 VmTopDown : 0y0
```

```
kd> dt nt!_MMSUPPORT
   +0x000 WorkingSetExpansionLinks : _LIST_ENTRY
   +0x008 LastTrimTime     : _LARGE_INTEGER
   +0x010 Flags            : _MMSUPPORT_FLAGS
   +0x014 PageFaultCount   : Uint4B
   +0x018 PeakWorkingSetSize : Uint4B
   +0x01c GrowthSinceLastEstimate : Uint4B
   +0x020 MinimumWorkingSetSize : Uint4B
   +0x024 MaximumWorkingSetSize : Uint4B
   +0x028 VmWorkingSetList : Ptr32 _MMWSL
   +0x02c Claim            : Uint4B
   +0x030 NextEstimationSlot : Uint4B
   +0x034 NextAgingSlot    : Uint4B
   +0x038 EstimatedAvailable : Uint4B
   +0x03c WorkingSetSize   : Uint4B
   +0x040 WorkingSetMutex  : _EX_PUSH_LOCK
```

与`MM_AVL_TABLE MiRosKernelVadRoot;`类似，在内核地址中也定义了一个全局的变量`PMMSUPPORT MmKernelAddressSpace;`，它表示了内核的地址空间，当然了它的内容与用户进程的EPRCESS中的Vm一样，表示内容的意义也类似。

实际上在新的ReactOS源码中，对于地址空间的引用是通过Vm成员作为标识的，对于内核地址空间的引用是用MmKernelAddressSpace作为参数。根据源码中对`PMMSUPPORT`类型的引用，可以看到有如下几个对于虚拟内存进行操作的函数，如`MmLocateMemoryAreaByAddress`，`MmLocateMemoryAreaByRegion`，`MmInsertMemoryArea`，`MmFindGap`，`MmFreeMemoryArea`，`MmCreateMemoryArea`等。

其中与书中对应的两个函数是`MmLocateMemoryAreaByAddress`和`MmLocateMemoryAreaByRegion`。从两个函数原型中可以看到两个函数的参数发生了变化，MmLocateMemoryAreaByAddress函数的第一个参数`PMMSUPPORT AddressSpace`表示虚拟地址空间信息的结构体指针，第二个参数`Address_`就是要定位的地址值了。

```
PMEMORY_AREA NTAPI
MmLocateMemoryAreaByAddress(PMMSUPPORT AddressSpace,
							PVOID Address_)

PMEMORY_AREA NTAPI
MmLocateMemoryAreaByRegion(PMMSUPPORT AddressSpace,
                           PVOID Address_,
                           ULONG_PTR Length)
```

两个函数源码的具体含义根据函数名也差不多了解。`MmLocateMemoryAreaByAddress`函数就是通过一个地址，确定地址所在的内存区间（Area）；`MmLocateMemoryAreaByRegion`函数是根据一个区块，确定区块所在的区间。`MmLocateMemoryAreaByAddress`函数的源码也与书中的源码有不小的区别，AddressSpace参数不再直接是平衡二叉树头结构了，而是一个PMMSUPPORT类型指针。第一步就是先确定当前函数调用到底位于那个进程中，即通过调用`MmGetAddressSpaceOwner`函数确定，从它的源码可以看出指向一个内核全局变量`MmKernelAddressSpace`的时候，返回NULL，其实意思是位于内核“进程”，而其他情况则是在正常进程中调用，通过`CONTAINING_RECORD`宏根据Vm偏移计算得到EPROCESS结构体指针。

接下来根据所在进程Process指针，确定要遍历的地址空间。如果是普通进程，则取EPROCESS的VadRoot成员地址，否则取全局变量MiRosKernelVadRoot的地址。从这里可以看出，进程的虚拟地址空间和内核的虚拟地址空间并非在同一个数据结构中。它们两个分开也是有原因的，所有的进程共享一份内核地址空间，而又有各自的用户地址空间；共享一份内核地址空间，使得每一个进程对内核地址空间的更新，可以共享给所有的进程。

接下来调用`MiCheckForConflictingNode`函数，查找地址对应的二叉树的节点，输出参数类型`PMMADDRESS_NODE`前面已经说了，它就是二叉平衡树的节点。函数中首先找到二叉树的根（注意BanlancedRoot本身也是一个二叉树节点，它的右子节点保存的才是二叉树的根），然后根据传入的地址区间（地址值）的Vpn判断遍历方向，直到找到某一个节点的StartVpn和EndVpn包含了给定的值，或直到遍历完也没有找到。

如果找到了节点了，会分为三种情况处理：节点直接就是`MEMORY_AREA`，`MEMORY_AREA`指针保存于`MMVAD_LONG`结构体的FirstProtoTypePtr字段，还有一种Section将MAREA保存在u4.Banked字段中。结构体`MMVAD_LONG`结构体的定义如下，可知`MMVAD_LONG`结构体是在`MMADDRESS_NODE`结构体基础上进行了扩展，增加了一些成员。另一个结构体`MMVAD`，它的定义也在后面给出了。

```
PMEMORY_AREA NTAPI
MmLocateMemoryAreaByAddress(
    PMMSUPPORT AddressSpace,
    PVOID Address_)
{
    ULONG_PTR StartVpn = (ULONG_PTR)Address_ / PAGE_SIZE;	// PAGE_SIZE = 0x1000 4K，即获取高20位的值
    PEPROCESS Process;
    PMM_AVL_TABLE Table;
    PMMADDRESS_NODE Node;
    PMEMORY_AREA MemoryArea;
    TABLE_SEARCH_RESULT Result;
    PMMVAD_LONG Vad;

    Process = MmGetAddressSpaceOwner(AddressSpace);
    Table = (Process != NULL) ? &Process->VadRoot : &MiRosKernelVadRoot;

    Result = MiCheckForConflictingNode(StartVpn, StartVpn, Table, &Node);	// 查找表中的节点（说是表，其实是一个二叉树）
    if (Result != TableFoundNode)	// 没有找到
    {
        return NULL;
    }

    Vad = (PMMVAD_LONG)Node;
    if (Vad->u.VadFlags.Spare == 0)		// 不等于0，表示是old-style 内存区间
    {
        /* Check if this is VM VAD 是否VM的VAD，不为空说明有Segment信息，即进行了分段*/
        if (Vad->ControlArea == NULL)
        {
            /* We store the reactos MEMORY_AREA here */
            MemoryArea = (PMEMORY_AREA)Vad->FirstPrototypePte;
        }
        else
        {
            /* This is a section VAD. Store the MAREA here for now 一个section Vad，存在Banked字段中*/
            MemoryArea = (PMEMORY_AREA)Vad->u4.Banked;
        }
    }
    else
    {
        MemoryArea = (PMEMORY_AREA)Node;		// 老式风格的 MEMORY_AREA直接将节点返回
    }

    return MemoryArea;
}

FORCEINLINE
PEPROCESS
MmGetAddressSpaceOwner(IN PMMSUPPORT AddressSpace)
{
    if (AddressSpace == MmKernelAddressSpace) return NULL;
    return CONTAINING_RECORD(AddressSpace, EPROCESS, Vm);	// 根据成员获取结构体起始地址
}

TABLE_SEARCH_RESULT
NTAPI
MiCheckForConflictingNode(IN ULONG_PTR StartVpn,
                          IN ULONG_PTR EndVpn,
                          IN PMM_AVL_TABLE Table,
                          OUT PMMADDRESS_NODE *NodeOrParent)
{
    PMMADDRESS_NODE ParentNode, CurrentNode;

    /* If the tree is empty, there is no conflict */
    if (Table->NumberGenericTableElements == 0) return TableEmptyTree;

    /* Start looping from the root node 从根节点开始查找 Root也是一个MMADDRESS_NODE，右节点是有数据的开始*/
    CurrentNode = RtlRightChildAvl(&Table->BalancedRoot);
    ASSERT(CurrentNode != NULL);
    while (CurrentNode)
    {
        ParentNode = CurrentNode;

        /* This address comes after 地址大于当前节点的地址范围，继续向右子节点找*/
        if (StartVpn > CurrentNode->EndingVpn)
        {
            /* Keep searching on the right */
            CurrentNode = RtlRightChildAvl(CurrentNode);
        }
        else if (EndVpn < CurrentNode->StartingVpn)	// 地址小于当前节点的地址范围，继续向左子节点找
        {
            /* This address ends before the node starts, search on the left */
            CurrentNode = RtlLeftChildAvl(CurrentNode);
        }
        else
        {
            /* This address is part of this node, return it 地址位于节点的地址范围之内，直接将节点赋值给输出参数 */
            *NodeOrParent = ParentNode;
            return TableFoundNode;
        }
    }

    /* There is no more child, save the current node as parent 查到结束没有找到，说明还没有分配，返回节点为插入位置 */
    *NodeOrParent = ParentNode;
    if (StartVpn > ParentNode->EndingVpn)
    {
        return TableInsertAsRight;
    }
    else
    {
        return TableInsertAsLeft;
    }
}
```

从下面几个结构体的定义可以看出，`MMADDRESS_NODE`是AVL的节点；`MMVAD`是虚拟地址描述符，描述基本信息；而`MMVAD_LONG`则是对`MMVAD`结构体的扩展，用于描述Section和私有地址分配。三个结构体起始部分相同，也即他们都位于AVL树上，只是不同的时候使用不同的结构体。

```
//
// Node in Memory Manager's AVL Table	内存管理器的AVL表的节点
//
typedef struct _MMADDRESS_NODE
{
    union
    {
        LONG_PTR Balance:2;
        struct _MMADDRESS_NODE *Parent;
    } u1;
    struct _MMADDRESS_NODE *LeftChild;
    struct _MMADDRESS_NODE *RightChild;
    ULONG_PTR StartingVpn;
    ULONG_PTR EndingVpn;
} MMADDRESS_NODE, *PMMADDRESS_NODE;

//
// Virtual Address Descriptor (VAD) Structure	虚拟地址描述符 结构体
//
typedef struct _MMVAD
{
    union
    {
        LONG_PTR Balance:2;
        struct _MMVAD *Parent;
    } u1;
    struct _MMVAD *LeftChild;
    struct _MMVAD *RightChild;
    ULONG_PTR StartingVpn;
    ULONG_PTR EndingVpn;
    union
    {
        ULONG_PTR LongFlags;
        MMVAD_FLAGS VadFlags;
    } u;
    PCONTROL_AREA ControlArea;
    PMMPTE FirstPrototypePte;
    PMMPTE LastContiguousPte;
    union
    {
        ULONG LongFlags2;
        MMVAD_FLAGS2 VadFlags2;
    } u2;
} MMVAD, *PMMVAD;

//
// Long VAD used in section and private allocations 长VAD，用于section（内存映像）和私有分配
//
typedef struct _MMVAD_LONG
{
    union
    {
        LONG_PTR Balance:2;
        PMMVAD Parent;
    } u1;
    PMMVAD LeftChild;
    PMMVAD RightChild;
    ULONG_PTR StartingVpn;		// 区间的起始VPN，Virtual Frame Number？？
    ULONG_PTR EndingVpn;		// 区间的结束VPN
    union
    {
        ULONG_PTR LongFlags;
        MMVAD_FLAGS VadFlags;	// 用于确定节点保存的虚拟内存信息的类型，老式的数据，VM的，section的
    } u;
    PCONTROL_AREA ControlArea;	//
    PMMPTE FirstPrototypePte;	//
    PMMPTE LastContiguousPte;	//
    union
    {
        ULONG LongFlags2;
        MMVAD_FLAGS2 VadFlags2;
    } u2;
    union
    {
        LIST_ENTRY List;
        MMADDRESS_LIST Secured;
    } u3;
    union
    {
        PVOID Banked;			//
        PMMEXTEND_INFO ExtendedInfo;
    } u4;
} MMVAD_LONG, *PMMVAD_LONG;

//
// Flags used in the VAD VAD中使用的标记
//
typedef struct _MMVAD_FLAGS
{
#ifdef _WIN64
    ULONG_PTR CommitCharge:51;
#else
    ULONG_PTR CommitCharge:19;
#endif
    ULONG_PTR NoChange:1;
    ULONG_PTR VadType:3;			// VAD类型  MI_VAD_TYPE 枚举变量值
    ULONG_PTR MemCommit:1;			// 是否已经交割
    ULONG_PTR Protection:5;			// 保护类型
    ULONG_PTR Spare:2;				//
    ULONG_PTR PrivateMemory:1;		// 分配私有内存，非文件映射
} MMVAD_FLAGS, *PMMVAD_FLAGS;

//
// Kinds of VADs
//
typedef enum _MI_VAD_TYPE
{
    VadNone,
    VadDevicePhysicalMemory,
    VadImageMap,
    VadAwe,
    VadWriteWatch,
    VadLargePages,
    VadRotatePhysical,
    VadLargePageSection
} MI_VAD_TYPE, *PMI_VAD_TYPE;

//
// Control Area Structures
//
typedef struct _CONTROL_AREA
{
    PSEGMENT Segment;
    LIST_ENTRY DereferenceList;
    ULONG NumberOfSectionReferences;
    ULONG NumberOfPfnReferences;
    ULONG NumberOfMappedViews;
    ULONG NumberOfSystemCacheViews;
    ULONG NumberOfUserReferences;
    union
    {
        ULONG LongFlags;
        MMSECTION_FLAGS Flags;
    } u;
    PFILE_OBJECT FilePointer;
    PEVENT_COUNTER WaitingForDeletion;
    USHORT ModifiedWriteCount;
    USHORT FlushInProgressCount;
    ULONG WritableUserReferences;
    ULONG QuadwordPad;
} CONTROL_AREA, *PCONTROL_AREA;

```

对于两个函数的返回值`MEMORY_AREA`结构体指针，它是一个内存块的基本描述信息。其中包含了MMVAD节点信息，内存的保护属性，类型，标记位等等内容。

```
typedef struct _MEMORY_AREA
{
    MMVAD VadNode;
    ULONG_PTR StartingVpn;
    ULONG_PTR EndingVpn;
    struct _MEMORY_AREA *Parent;
    struct _MEMORY_AREA *LeftChild;
    struct _MEMORY_AREA *RightChild;
    ULONG Type;
    ULONG Protect;
    ULONG Flags;
    BOOLEAN DeleteInProgress;
    ULONG Magic;
    PVOID Vad;
    union
    {
        struct
        {
            ROS_SECTION_OBJECT* Section;
            LARGE_INTEGER ViewOffset;
            PMM_SECTION_SEGMENT Segment;
            LIST_ENTRY RegionListHead;
        } SectionData;
        struct
        {
            LIST_ENTRY RegionListHead;
        } VirtualMemoryData;
    } Data;
} MEMORY_AREA, *PMEMORY_AREA;
```

函数`MmLocateMemoryAreaByRegion`和MmLocateMemoryAreaByAddress函数基本类似，所不同的是前者找一个区块所在区间，后者找一个地址所在区间。可以参考ReactOS0.4.5的代码，这里不再过多叙述。

再一个函数就是`MmFindGap`了，它的源码如下代码块所示，它的源码相较书上也有一些变化。以从高地址向低地址查找为例，列举一下代码。对于内核空间，最高地址被设置为-1，及0xFFFFFFFF，而用户空间的最大地址设置为`MM_HIGHEST_VAD_ADDRESS`，它其实就是`0x7FFFFFFF - 64K`，即用户可以访问的最大地址（64K作为防护栅栏）。
```
#define MM_HIGHEST_VAD_ADDRESS          (PVOID)((ULONG_PTR)MM_HIGHEST_USER_ADDRESS - (16 * PAGE_SIZE))
```

函数`MiFindEmptyAddressRangeDownTree`的代码与书上类似，在代码中有注释，代码主要逻辑就是对AVL树的从右向左遍历，找到可以容纳给定长度空间的未使用的间隙（找前一个节点的算法可能会比较绕，可以提前学习一下二叉树遍历），不再详细叙述。注意`MmFindGap`函数是在寻找未分配的空档，用于分配虚拟地址空间。这里切不可理解为从二叉树中找包含给定大小空间的节点。

```
PVOID NTAPI
MmFindGap(
    PMMSUPPORT AddressSpace,
    ULONG_PTR Length,
    ULONG_PTR Granularity,
    BOOLEAN TopDown)
{
    PEPROCESS Process;
    PMM_AVL_TABLE VadRoot;
    TABLE_SEARCH_RESULT Result;
    PMMADDRESS_NODE Parent;
    ULONG_PTR StartingAddress, HighestAddress;

    Process = MmGetAddressSpaceOwner(AddressSpace);	// 根据进程查找 虚拟地址空间 管理表
    VadRoot = Process ? &Process->VadRoot : &MiRosKernelVadRoot;
    if (TopDown)
    {
        /* Find an address top-down 从高地址向低地址 查找 最高地址*/
        HighestAddress = Process ? (ULONG_PTR)MM_HIGHEST_VAD_ADDRESS : (LONG_PTR)-1;
        Result = MiFindEmptyAddressRangeDownTree(Length,
                                                 HighestAddress,
                                                 Granularity,
                                                 VadRoot,
                                                 &StartingAddress,
                                                 &Parent);
    }
    else
    {
        Result = MiFindEmptyAddressRangeInTree(Length,
                                               Granularity,
                                               VadRoot,
                                               &Parent,
                                               &StartingAddress);
    }

    if (Result == TableFoundNode)
    {
        return NULL;
    }

    return (PVOID)StartingAddress;
}

TABLE_SEARCH_RESULT
NTAPI
MiFindEmptyAddressRangeDownTree(IN SIZE_T Length,
                                IN ULONG_PTR BoundaryAddress,
                                IN ULONG_PTR Alignment,
                                IN PMM_AVL_TABLE Table,
                                OUT PULONG_PTR Base,
                                OUT PMMADDRESS_NODE *Parent)
{
    PMMADDRESS_NODE Node, OldNode, Child;
    ULONG_PTR LowVpn, HighVpn, AlignmentVpn;
    PFN_NUMBER PageCount;

    /* Sanity checks */
    ASSERT(BoundaryAddress);
    ASSERT(BoundaryAddress <= ((ULONG_PTR)MM_HIGHEST_VAD_ADDRESS));
    ASSERT((Alignment & (PAGE_SIZE - 1)) == 0);

    /* Calculate page numbers for the length and alignment 计算长度对齐后的所需的页的数量 */
    Length = ROUND_TO_PAGES(Length);
    PageCount = Length >> PAGE_SHIFT;
    AlignmentVpn = Alignment / PAGE_SIZE;		// 要求的对齐长度 占虚拟地址的页框数

    /* Check for kernel mode table (memory areas) 检查内核模式的虚拟地址表，计算最小的 VPN */
    if (Table->Unused == 1)		// 内核虚拟内存管理表
    {
        LowVpn = ALIGN_UP_BY((ULONG_PTR)MmSystemRangeStart >> PAGE_SHIFT, AlignmentVpn);
    }
    else
    {
        LowVpn = ALIGN_UP_BY((ULONG_PTR)MM_LOWEST_USER_ADDRESS, Alignment);
    }

    /* Check if there is enough space below the boundary 最小内存到上限是否大于 指定大小 */
    if ((LowVpn + Length) > (BoundaryAddress + 1))
    {
        return TableFoundNode;
    }

    /* Check if the table is empty 表为空 */
    if (Table->NumberGenericTableElements == 0)
    {
        /* Tree is empty, the candidate address is already the best one */
        // 则从最高地址向下减去大小，并且以对其要求对齐地址
        *Base = ALIGN_DOWN_BY(BoundaryAddress + 1 - Length, Alignment);
        return TableEmptyTree;
    }

    /* Calculate the initial upper margin 计算最高的 Vpn地址 */
    HighVpn = (BoundaryAddress + 1) >> PAGE_SHIFT;

    /* Starting from the root, follow the right children until we found a node
       that ends above the boundary 从root开始，一直向右遍历，直到边界停止 */
    Node = RtlRightChildAvl(&Table->BalancedRoot);
    while ((Node->EndingVpn < HighVpn) &&
           ((Child = RtlRightChildAvl(Node)) != NULL)) Node = Child;

    /* Now loop the Vad nodes 从找到的最右边节点开始计算*/
    while (Node)
    {
        /* Calculate the lower margin 计算结束地址到最高地址之间 */
        LowVpn = ALIGN_UP_BY(Node->EndingVpn + 1, AlignmentVpn);

        /* Check if the current bounds are suitable 校验当前节点到“最高限制”之间是否满足要求 */
        if ((HighVpn > LowVpn) && ((HighVpn - LowVpn) >= PageCount))
        {
            /* There is enough space to add our node 足够空间容纳请求长度 */
            LowVpn = ALIGN_DOWN_BY(HighVpn - PageCount, AlignmentVpn);
            *Base = LowVpn << PAGE_SHIFT;

            /* Can we use the current node as parent? 后面要插入新的节点，这个肯定是在当前节点右侧，如果空直接就是当前节点 */
            if (!RtlRightChildAvl(Node))
            {
                /* Node has no right child, so use it as parent */
                *Parent = Node;
                return TableInsertAsRight;
            }
            else
            {	// 如果不空，则是当前节点右子节点的的最左子节点，之前记录了
                /* Node has a right child, the node we had before is the most
                   left grandchild of that right child, use it as parent. */
                *Parent = OldNode;
                return TableInsertAsLeft;
            }
        }

        /* Update the upper margin if necessary 修改可用的最大值 */
        if (Node->StartingVpn < HighVpn) HighVpn = Node->StartingVpn;

        /* Remember the current node and go to the previous node 记录当前节点，然后向前找它相邻的节点。*/
        OldNode = Node;
        Node = MiGetPreviousNode(Node);
    }

    /* Check if there's enough space before the lowest Vad 校验在最小的Vad节点之前的空间是否满足 */
    LowVpn = ALIGN_UP_BY((ULONG_PTR)MI_LOWEST_VAD_ADDRESS, Alignment) / PAGE_SIZE;
    if ((HighVpn > LowVpn) && ((HighVpn - LowVpn) >= PageCount))
    {
        /* There is enough space to add our address */
        LowVpn = ALIGN_DOWN_BY(HighVpn - PageCount, Alignment >> PAGE_SHIFT);
        *Base = LowVpn << PAGE_SHIFT;
        *Parent = OldNode;
        return TableInsertAsLeft;
    }

    /* No address space left at all 没有足够大的空间容纳请求的大小 */
    *Base = 0;
    *Parent = NULL;
    return TableFoundNode;
}
```

第三个要分析的函数是创建AVL树节点，即分配一块虚拟内存空间。分配虚拟地址空间的函数为MmCreateMemoryArea()，它的源代码如下代码块所示。

```
/**
 * @name MmCreateMemoryArea
 *
 * Create a memory area. 	// 创建一个内存区块
 *
 * @param AddressSpace
 *        Address space to create the area in.
 * @param Type
 *        Type of the memory area.
 * @param BaseAddress
 *        Base address for the memory area we're about the create. On
 *        input it contains either 0 (auto-assign address) or preferred
 *        address. On output it contains the starting address of the
 *        newly created area.
 * @param Length
 *        Length of the area to allocate.
 * @param Attributes
 *        Protection attributes for the memory area.
 * @param Result
 *        Receives a pointer to the memory area on successful exit.
 *
 * @return Status
 *
 * @remarks Lock the address space before calling this function.
 */

NTSTATUS NTAPI
MmCreateMemoryArea(PMMSUPPORT AddressSpace,
                   ULONG Type,
                   PVOID *BaseAddress,
                   ULONG_PTR Length,
                   ULONG Protect,
                   PMEMORY_AREA *Result,
                   ULONG AllocationFlags,
                   ULONG Granularity)
{
    ULONG_PTR tmpLength;
    PMEMORY_AREA MemoryArea;
    ULONG_PTR EndingAddress;

    DPRINT("MmCreateMemoryArea(Type 0x%lx, BaseAddress %p, "
           "*BaseAddress %p, Length %p, AllocationFlags %x, "
           "Result %p)\n",
           Type, BaseAddress, *BaseAddress, Length, AllocationFlags,
           Result);

    /* Is this a static memory area? */
    if (Type & MEMORY_AREA_STATIC)
    {
        /* Use the static array instead of the pool */
        // 是否使用静态数组来分配内存区块结构体
        ASSERT(MiStaticMemoryAreaCount < MI_STATIC_MEMORY_AREAS);
        MemoryArea = &MiStaticMemoryAreas[MiStaticMemoryAreaCount++];
    }
    else
    {
        /* Allocate the memory area from nonpaged pool */
        // 在堆中分配内存，注意此处分配的是非Paged堆，即不可换入换出的内存
        MemoryArea = ExAllocatePoolWithTag(NonPagedPool,
                                           sizeof(MEMORY_AREA),
                                           TAG_MAREA);
    }

    if (!MemoryArea)
    {
        DPRINT1("Not enough memory.\n");
        return STATUS_NO_MEMORY;
    }

    RtlZeroMemory(MemoryArea, sizeof(MEMORY_AREA));
    MemoryArea->Type = Type & ~MEMORY_AREA_STATIC;
    MemoryArea->Protect = Protect;
    MemoryArea->Flags = AllocationFlags;
    MemoryArea->Magic = 'erAM';		// Magic为固定值
    MemoryArea->DeleteInProgress = FALSE;

    if (*BaseAddress == 0)
    {
        tmpLength = (ULONG_PTR)MM_ROUND_UP(Length, PAGE_SIZE);
        // 根据内存页大小，将内存长度向上进行对齐，并查找虚拟地址空间的空闲块
        *BaseAddress = MmFindGap(AddressSpace,
                                 tmpLength,
                                 Granularity,
                                 (AllocationFlags & MEM_TOP_DOWN) == MEM_TOP_DOWN);
        if ((*BaseAddress) == 0)
        {
            DPRINT("No suitable gap\n");
            if (!(Type & MEMORY_AREA_STATIC)) ExFreePoolWithTag(MemoryArea, TAG_MAREA);
            return STATUS_NO_MEMORY;
        }

        MemoryArea->StartingVpn = (ULONG_PTR)*BaseAddress >> PAGE_SHIFT;
        MemoryArea->EndingVpn = ((ULONG_PTR)*BaseAddress + tmpLength - 1) >> PAGE_SHIFT;
        MmInsertMemoryArea(AddressSpace, MemoryArea);	// 将内存区块信息插入到地址空间
    }
    else
    {
        EndingAddress = ((ULONG_PTR)*BaseAddress + Length - 1) | (PAGE_SIZE - 1);
        *BaseAddress = ALIGN_DOWN_POINTER_BY(*BaseAddress, Granularity);
        tmpLength = EndingAddress + 1 - (ULONG_PTR)*BaseAddress;

        if (!MmGetAddressSpaceOwner(AddressSpace) && *BaseAddress < MmSystemRangeStart)
        {
            ASSERT(FALSE);
            if (!(Type & MEMORY_AREA_STATIC)) ExFreePoolWithTag(MemoryArea, TAG_MAREA);
            return STATUS_ACCESS_VIOLATION;
        }

        if (MmGetAddressSpaceOwner(AddressSpace) &&
                (ULONG_PTR)(*BaseAddress) + tmpLength > (ULONG_PTR)MmSystemRangeStart)
        {
            DPRINT("Memory area for user mode address space exceeds MmSystemRangeStart\n");
            if (!(Type & MEMORY_AREA_STATIC)) ExFreePoolWithTag(MemoryArea, TAG_MAREA);
            return STATUS_ACCESS_VIOLATION;
        }

        /* No need to check ARM3 owned memory areas, the range MUST be free */
        if (MemoryArea->Type != MEMORY_AREA_OWNED_BY_ARM3)
        {
            if (MmLocateMemoryAreaByRegion(AddressSpace,
                                           *BaseAddress,
                                           tmpLength) != NULL)
            {
                DPRINT("Memory area already occupied\n");
                if (!(Type & MEMORY_AREA_STATIC)) ExFreePoolWithTag(MemoryArea, TAG_MAREA);
                return STATUS_CONFLICTING_ADDRESSES;
            }
        }

        MemoryArea->StartingVpn = (ULONG_PTR)*BaseAddress >> PAGE_SHIFT;
        MemoryArea->EndingVpn = ((ULONG_PTR)*BaseAddress + tmpLength - 1) >> PAGE_SHIFT;
        MmInsertMemoryArea(AddressSpace, MemoryArea);
    }

    *Result = MemoryArea;

    DPRINT("MmCreateMemoryArea() succeeded (%p)\n", *BaseAddress);
    return STATUS_SUCCESS;
}

static VOID
MmInsertMemoryArea(
    PMMSUPPORT AddressSpace,
    PMEMORY_AREA marea)
{
    PEPROCESS Process = MmGetAddressSpaceOwner(AddressSpace);

    marea->VadNode.StartingVpn = marea->StartingVpn;
    marea->VadNode.EndingVpn = marea->EndingVpn;
    marea->VadNode.u.VadFlags.Spare = 1;
    marea->VadNode.u.VadFlags.Protection = MiMakeProtectionMask(marea->Protect);

    /* Build a lame VAD if this is a user-space allocation */
    if (marea->EndingVpn + 1 < (ULONG_PTR)MmSystemRangeStart >> PAGE_SHIFT)
    {
        ASSERT(Process != NULL);
        if (marea->Type != MEMORY_AREA_OWNED_BY_ARM3)
        {
            ASSERT(marea->Type == MEMORY_AREA_SECTION_VIEW || marea->Type == MEMORY_AREA_CACHE);

            /* Insert the VAD */
            MiLockProcessWorkingSetUnsafe(PsGetCurrentProcess(), PsGetCurrentThread());
            MiInsertVad(&marea->VadNode, &Process->VadRoot);
            MiUnlockProcessWorkingSetUnsafe(PsGetCurrentProcess(), PsGetCurrentThread());
            marea->Vad = &marea->VadNode;
        }
    }
    else
    {
        ASSERT(Process == NULL);

        if (!MiRosKernelVadRootInitialized)
        {
            MiRosKernelVadRoot.BalancedRoot.u1.Parent = &MiRosKernelVadRoot.BalancedRoot;
            MiRosKernelVadRoot.Unused = 1;
            MiRosKernelVadRootInitialized = TRUE;
        }

        /* Insert the VAD */
        MiLockWorkingSet(PsGetCurrentThread(), &MmSystemCacheWs);
        MiInsertVad(&marea->VadNode, &MiRosKernelVadRoot);
        MiUnlockWorkingSet(PsGetCurrentThread(), &MmSystemCacheWs);
        marea->Vad = NULL;
    }
}

```

该函数有多个参数，要分配内存区块的地址空间（内核空间或进程空间），内存块类型，基地址，内存块长度，保护方式，分配的`MEMORY_AREA`结构体在内存中地址，分配标记，内存分配粒度。根据BaseAddress是否为空，确定是否要在指定的地址分配虚拟内存；对于为没有给定BaseAddress的情况，只要找到可用的虚拟地址空间，分配内存信息块即可；而对于给定分配地址的情况，要使用`MmLocateMemoryAreaByRegion`函数判断给定地址处的区块是否被占用了，没有被占用，才可以进行分配。

当确定了可以分配虚拟地址空间块，则将填写完信息的`MEMORY_AREA`结构体指针插入到结构中保存，保存时调用的函数为`MmInsertMemoryArea`。MmInsertMemoryArea函数在设置了VadNode结构体内容之后，调用`MiInsertVad`函数，将Vad节点插入到进程/内核的虚拟地址空间管理AVL树中。

```
VOID
NTAPI
MiInsertVad(IN PMMVAD Vad,
            IN PMM_AVL_TABLE VadRoot)
{
    TABLE_SEARCH_RESULT Result;
    PMMADDRESS_NODE Parent = NULL;

    /* Validate the VAD and set it as the current hint */
    ASSERT(Vad->EndingVpn >= Vad->StartingVpn);
    VadRoot->NodeHint = Vad;

    /* Find the parent VAD and where this child should be inserted */
    Result = RtlpFindAvlTableNodeOrParent(VadRoot, (PVOID)Vad->StartingVpn, &Parent);
    ASSERT(Result != TableFoundNode);
    ASSERT((Parent != NULL) || (Result == TableEmptyTree));

    /* Do the actual insert operation */
    MiInsertNode(VadRoot, (PVOID)Vad, Parent, Result);
}


VOID
NTAPI
MiInsertNode(IN PMM_AVL_TABLE Table,
             IN PMMADDRESS_NODE NewNode,
             IN PMMADDRESS_NODE Parent,
             IN TABLE_SEARCH_RESULT Result)
{
    PMMVAD_LONG Vad;

    /* Insert it into the tree */
    RtlpInsertAvlTreeNode(Table, NewNode, Parent, Result);

    /* Now insert an ARM3 MEMORY_AREA for this node, unless the insert was already from the MEMORY_AREA code */
    Vad = (PMMVAD_LONG)NewNode;
    if (Vad->u.VadFlags.Spare == 0)
    {
        NTSTATUS Status;
        PMEMORY_AREA MemoryArea;
        SIZE_T Size;
        PEPROCESS Process = CONTAINING_RECORD(Table, EPROCESS, VadRoot);
        PVOID AllocatedBase = (PVOID)(Vad->StartingVpn << PAGE_SHIFT);

        Size = ((Vad->EndingVpn + 1) - Vad->StartingVpn) << PAGE_SHIFT;

        if (AllocatedBase == NULL)
        {
            AllocatedBase = (PVOID)(ULONG_PTR)1;
            Size -= 1;
        }

        Status = MmCreateMemoryArea(&Process->Vm,
                                    MEMORY_AREA_OWNED_BY_ARM3,
                                    &AllocatedBase,
                                    Size,
                                    PAGE_READWRITE,
                                    &MemoryArea,
                                    0,
                                    PAGE_SIZE);
        ASSERT(NT_SUCCESS(Status));

        /* Check if this is VM VAD */
        if (Vad->ControlArea == NULL)
        {
            /* We store the reactos MEMORY_AREA here */
            Vad->FirstPrototypePte = (PMMPTE)MemoryArea;
        }
        else
        {
            /* This is a section VAD. Store the MAREA here for now */
            ASSERT(Vad->u4.Banked == (PVOID)0xDEADBABE);
            Vad->u4.Banked = (PVOID)MemoryArea;
        }
    }
}
```

将AVL节点插入到地址空间AVL树的操作也比较简单，MiInsertVad函数查找了节点应该插入位置的父节点，然后调用了`MiInsertNode`函数，函数直接调用RtlpInsertAvlTreeNode函数，进行AVL二叉树的操作，完全是二叉树数据结构的操作，这里不再做过多解析。在将节点插入到AVL树之后，根据Vad节点的VadFlags的Spare成员判断是否需要构建一个内存区块信息结构体。根据Vad的成员值，将新创建的`MEMORY_AREA`结构体挂到当前Vad的对应字段上（u4.Banked或FirstPrototypePte）。

在`MEMORY_AREA`结构体中最后一个成员是`Data`，它的类型如下代码块所示，它是一个联合体。实际使用中，它可能是SectionData，或者是VirtualMemorydata；其中前者是当前的Memory Area是一块映射，分为多块，每一块内存信息存储在一个Region中，保存在RegionListHead中；后者则是对于普通的内存块而言，只需维护一个Region列表即可。

```
union
{
    struct
    {
        ROS_SECTION_OBJECT* Section;
        LARGE_INTEGER ViewOffset;
        PMM_SECTION_SEGMENT Segment;
        LIST_ENTRY RegionListHead;
    } SectionData;
    struct
    {
        LIST_ENTRY RegionListHead;
    } VirtualMemoryData;
} Data;

typedef struct _MM_REGION
{
    ULONG Type;
    ULONG Protect;
    SIZE_T Length;
    LIST_ENTRY RegionListEntry;
} MM_REGION, *PMM_REGION;
```

这样虚拟内存空间就被分为了空间，区间，区块，页面四个单位。空间即内核空间或用户空间（进程用户空间）；区块即前面一直在用的`MEMORY_AREA`，每一个代表一个内存区间，一个内存区间需要内存地址连续；而内存区块就是刚刚介绍的Region，它的类型是`MM_REGION`，它表示的内存块要是地址连续且内存保护属性一致；最后就是页面了，对于i386而言，一个页面就是4K大小。一个内存区块可以有一个或多个页，而一个内存区间是由一个或多个内存区块组成，内存空间被分割为不定个内存内存区间，使用一个AVL树进行管理。

对于映射内存还有两个名词，Section和Segment。在Windows编程中会遇到着两个词相关的API，比如MapViewOfSection函数用于映射一段内存。正常创建一个内存映射，即创建一个Section；对于PE映射也是创建一个Section，PE文件映射会被分为很多段，每一个段是一个保护属性。大致的对应关系就有Section对应于内存区间，而Segment则对应于内存区块。

在实际使用虚拟内存时，不单要定位地址所属的虚拟内存区间，还需要定位所属的内存区块，即Region。查找给定内存所属区块的操作是由`MmFindRegion`函数完成，它的源码如下所示。由于Region是一个内存区间中的一部分且都是连续的，所以在Region中指保存了区块的大小，所以要从内存区间给的基地址开始，加上区块长度，判断给定地址是否位于该区块内。

```
PMM_REGION
NTAPI
MmFindRegion(PVOID BaseAddress, PLIST_ENTRY RegionListHead, PVOID Address,
             PVOID* RegionBaseAddress)
{
    PLIST_ENTRY current_entry;
    PMM_REGION current;
    PVOID StartAddress = BaseAddress;

    current_entry = RegionListHead->Flink;
    while (current_entry != RegionListHead)
    {
        current = CONTAINING_RECORD(current_entry, MM_REGION, RegionListEntry);

        if (StartAddress <= Address &&
                ((char*)StartAddress + current->Length) > (char*)Address)
        {
            if (RegionBaseAddress != NULL)
            {
                *RegionBaseAddress = StartAddress;
            }
            return(current);
        }

        current_entry = current_entry->Flink;

        StartAddress = (PVOID)((ULONG_PTR)StartAddress + current->Length);

    }
    return(NULL);
}
```

区块是一个或多个页面组成，对于多个页面组成的区块，它是可以被进一步分割的，完成区块分割的就是MmSplitRegion函数。同时区块的保护属性也是可以被修改的，该功能是MmAlterRegion函数完成。相关函数在如下的代码块给出，代码中给出注释，这里不再解释。

```
static PMM_REGION
MmSplitRegion(PMM_REGION InitialRegion, PVOID InitialBaseAddress,
              PVOID StartAddress, SIZE_T Length, ULONG NewType,
              ULONG NewProtect, PMMSUPPORT AddressSpace,
              PMM_ALTER_REGION_FUNC AlterFunc)
{
    PMM_REGION NewRegion1;
    PMM_REGION NewRegion2;
    SIZE_T InternalLength;

    /* Allocate this in front otherwise the failure case is too difficult. */
    // 首先分配两个MM_REGION区块，用于存储分割后的区块
    NewRegion2 = ExAllocatePoolWithTag(NonPagedPool, sizeof(MM_REGION),
                                       TAG_MM_REGION);
    if (NewRegion2 == NULL)
    {
        return(NULL);
    }

    /* Create the new region. */ // NewRegion1 指向新分割出来的区块
    NewRegion1 = ExAllocatePoolWithTag(NonPagedPool, sizeof(MM_REGION),
                                       TAG_MM_REGION);
    if (NewRegion1 == NULL)
    {
        ExFreePoolWithTag(NewRegion2, TAG_MM_REGION);
        return(NULL);
    }
    NewRegion1->Type = NewType;
    NewRegion1->Protect = NewProtect;
    InternalLength = ((char*)InitialBaseAddress + InitialRegion->Length) - (char*)StartAddress;
    InternalLength = min(InternalLength, Length);
    NewRegion1->Length = InternalLength;
    InsertAfterEntry(&InitialRegion->RegionListEntry,
                     &NewRegion1->RegionListEntry);

    /*
     * Call our helper function to do the changes on the addresses contained
     * in the initial region.
     */
    AlterFunc(AddressSpace, StartAddress, InternalLength, InitialRegion->Type,
              InitialRegion->Protect, NewType, NewProtect);

    /*
     * If necessary create a new region for the portion of the initial region
     * beyond the range of addresses to alter.
     */ // 对于老区块被分割为3部分的情况，用NewRegion2指向第三部分。（第一部分由老区块结构体指向）
    if (((char*)InitialBaseAddress + InitialRegion->Length) > ((char*)StartAddress + Length))
    {
        NewRegion2->Type = InitialRegion->Type;
        NewRegion2->Protect = InitialRegion->Protect;
        NewRegion2->Length = ((char*)InitialBaseAddress + InitialRegion->Length) -
                             ((char*)StartAddress + Length);
        InsertAfterEntry(&NewRegion1->RegionListEntry,
                         &NewRegion2->RegionListEntry);
    }
    else
    {	// 只分成了两部分，NewRegion1所指向直接到了原始区块结尾，NewRegion2就没用了
        ExFreePoolWithTag(NewRegion2, TAG_MM_REGION);
    }

    /* Either remove or shrink the initial region. */
    if (InitialBaseAddress == StartAddress)
    {	// 新分配的区块是从原始区块起始位置开始，则原始区块的信息结构体就无用了，要删除
        RemoveEntryList(&InitialRegion->RegionListEntry);
        ExFreePoolWithTag(InitialRegion, TAG_MM_REGION);
    }
    else
    {
    	// 对于非从原始区块起始地址开始，则需要重设原始区块的大小。
        InitialRegion->Length = (char*)StartAddress - (char*)InitialBaseAddress;
    }

    return(NewRegion1);
}

NTSTATUS
NTAPI
MmAlterRegion(PMMSUPPORT AddressSpace, PVOID BaseAddress,
              PLIST_ENTRY RegionListHead, PVOID StartAddress, SIZE_T Length,
              ULONG NewType, ULONG NewProtect, PMM_ALTER_REGION_FUNC AlterFunc)
{
    PMM_REGION InitialRegion;
    PVOID InitialBaseAddress = NULL;
    PMM_REGION NewRegion;
    PLIST_ENTRY CurrentEntry;
    PMM_REGION CurrentRegion = NULL;
    PVOID CurrentBaseAddress;
    SIZE_T RemainingLength;

    /*
     * Find the first region containing part of the range of addresses to
     * be altered. 首先要查找包含了要内存段所在区块
     */
    InitialRegion = MmFindRegion(BaseAddress, RegionListHead, StartAddress,
                                 &InitialBaseAddress);
    /*
     * If necessary then split the region into the affected and unaffected parts.
     */ // 找到的区块与目标类型和保护属性不同，则要对原区块进行分割
    if (InitialRegion->Type != NewType || InitialRegion->Protect != NewProtect)
    {
        NewRegion = MmSplitRegion(InitialRegion, InitialBaseAddress,
                                  StartAddress, Length, NewType, NewProtect,
                                  AddressSpace, AlterFunc);
        if (NewRegion == NULL)
        {
            return(STATUS_NO_MEMORY);
        }
        if(NewRegion->Length < Length)
            RemainingLength = Length - NewRegion->Length;
        else
            RemainingLength = 0;
    }
    else
    {
        NewRegion = InitialRegion;
        if(((ULONG_PTR)InitialBaseAddress + NewRegion->Length) <
                ((ULONG_PTR)StartAddress + Length))
            RemainingLength = ((ULONG_PTR)StartAddress + Length) - ((ULONG_PTR)InitialBaseAddress + NewRegion->Length);
        else
            RemainingLength = 0;
    }

    /*
     * Free any complete regions that are containing in the range of addresses
     * and call the helper function to actually do the changes.
     */ // 要修改的内存范围，可能跨过几个区块，对于中间的区块，则要将区块信息结构体释放掉
    CurrentEntry = NewRegion->RegionListEntry.Flink;
    CurrentRegion = CONTAINING_RECORD(CurrentEntry, MM_REGION,
                                      RegionListEntry);
    CurrentBaseAddress = (char*)StartAddress + NewRegion->Length;
    while (RemainingLength > 0 && CurrentRegion->Length <= RemainingLength &&
            CurrentEntry != RegionListHead)
    {
        if (CurrentRegion->Type != NewType ||
                CurrentRegion->Protect != NewProtect)
        {
            AlterFunc(AddressSpace, CurrentBaseAddress, CurrentRegion->Length,
                      CurrentRegion->Type, CurrentRegion->Protect,
                      NewType, NewProtect);
        }

        CurrentBaseAddress = (PVOID)((ULONG_PTR)CurrentBaseAddress + CurrentRegion->Length);
        NewRegion->Length += CurrentRegion->Length;
        RemainingLength -= CurrentRegion->Length;
        CurrentEntry = CurrentEntry->Flink;
        RemoveEntryList(&CurrentRegion->RegionListEntry);
        ExFreePoolWithTag(CurrentRegion, TAG_MM_REGION);
        CurrentRegion = CONTAINING_RECORD(CurrentEntry, MM_REGION,
                                          RegionListEntry);
    }

    /*
     * Split any final region.
     */ // 对于最后一个区块，要进行分割
    if (RemainingLength > 0 && CurrentEntry != RegionListHead)
    {
        CurrentRegion = CONTAINING_RECORD(CurrentEntry, MM_REGION,
                                          RegionListEntry);
        if (CurrentRegion->Type != NewType ||
                CurrentRegion->Protect != NewProtect)
        {
            AlterFunc(AddressSpace, CurrentBaseAddress, RemainingLength,
                      CurrentRegion->Type, CurrentRegion->Protect,
                      NewType, NewProtect);
        }
        NewRegion->Length += RemainingLength;
        CurrentRegion->Length -= RemainingLength;
    }

    /* // 一块内存修改后，它和前面或后面的内存区块属性可能就相同了，则要进行合并（只针对非分割情况）
     * If the region after the new region has the same type then merge them.
     */
    if (NewRegion->RegionListEntry.Flink != RegionListHead)
    {
        CurrentEntry = NewRegion->RegionListEntry.Flink;
        CurrentRegion = CONTAINING_RECORD(CurrentEntry, MM_REGION,
                                          RegionListEntry);
        if (CurrentRegion->Type == NewRegion->Type &&
                CurrentRegion->Protect == NewRegion->Protect)
        {
            NewRegion->Length += CurrentRegion->Length;
            RemoveEntryList(&CurrentRegion->RegionListEntry);
            ExFreePoolWithTag(CurrentRegion, TAG_MM_REGION);
        }
    }

    /*
     * If the region before the new region has the same type then merge them.
     */
    if (NewRegion->RegionListEntry.Blink != RegionListHead)
    {
        CurrentEntry = NewRegion->RegionListEntry.Blink;
        CurrentRegion = CONTAINING_RECORD(CurrentEntry, MM_REGION,
                                          RegionListEntry);
        if (CurrentRegion->Type == NewRegion->Type &&
                CurrentRegion->Protect == NewRegion->Protect)
        {
            NewRegion->Length += CurrentRegion->Length;
            RemoveEntryList(&CurrentRegion->RegionListEntry);
            ExFreePoolWithTag(CurrentRegion, TAG_MM_REGION);
        }
    }

    return(STATUS_SUCCESS);
}
```

这样就将虚拟地址空间管理的几个关键的函数做了简单分析，下一节先看一下物理内存管理。

#### 物理内存管理 ####

虚拟地址只是虚拟的地址空间，真正访问时还是要落到真正的物理内存上。物理内存也需要管理，进行合理分配。物理内存是以页面为单位进行管理，每个页面都对应一个`MMPFN`结构体，用于保存物理页面的信息，ReactOS源码中现在的内容和毛老师书上的内容有一些差别。

```
// Mm internal
typedef struct _MMPFN
{
    union
    {
        PFN_NUMBER Flink;					// 用于列表
        ULONG WsIndex;
        PKEVENT Event;
        NTSTATUS ReadStatus;
        SINGLE_LIST_ENTRY NextStackPfn;

        // HACK for ROSPFN
        SWAPENTRY SwapEntry;
    } u1;
    PMMPTE PteAddress;
    union
    {
        PFN_NUMBER Blink;					// 用于列表
        ULONG_PTR ShareCount;
    } u2;
    union
    {
        struct
        {
            USHORT ReferenceCount;
            MMPFNENTRY e1;
        };
        struct
        {
            USHORT ReferenceCount;
            USHORT ShortFlags;
        } e2;
    } u3;
    union
    {
        MMPTE OriginalPte;					// 不同状态下，意义不同
        LONG AweReferenceCount;

        // HACK for ROSPFN
        PMM_RMAP_ENTRY RmapListHead;
    };
    union
    {
        ULONG_PTR EntireFrame;
        struct
        {
            ULONG_PTR PteFrame:25;			// 不同状态内存块，意义不同
            ULONG_PTR InPageError:1;
            ULONG_PTR VerifierAllocation:1;
            ULONG_PTR AweAllocation:1;
            ULONG_PTR Priority:3;			// 用于优先级队列
            ULONG_PTR MustBeCached:1;
        };
    } u4;
#if MI_TRACE_PFNS
    MI_PFN_USAGES PfnUsage;
    CHAR ProcessName[16];
#endif

    // HACK until WS lists are supported
    MMWSLE Wsle;
} MMPFN, *PMMPFN;
```

由于物理内存只会在内核进行操作，所以物理内存也是通过一个全局变量进行管理，即`PMMPFN MmPfnDatabase;`。MmPfnDatabase一个指针，它指向一组`MMPFN`结构体数组，这一组结构体数组的一项就对应物理内存的一个4K页面。至于物理页面的数量，则可以从如下的几个值获取到，其中MmNumberOfPhysicalPages则是实际的内存所具有的页数。

```
kd> x nt!Mm*PhysicalPage*
805ad678          nt!MmNumberOfPhysicalPages = 0x3ff86
805ad674          nt!MmHighestPhysicalPage = 0x3fffe
80577bb4          nt!MmLowestPhysicalPage = 7
```

根据物理内存的用途或所处状态不同，分别使用如下的几种队列对物理内存进行管理。例如MmZeroedPageListHead列表对应的是被清空的物理内存页的列表。物理内存的状态可以分为活动状态（Active，正在被进程或内核使用），备用状态（StandBy，原来被进程或内核使用，但是现在被从工作集中移除了），已修改状态（modified，类似备用，但是页面包含内容已经被修改，如果用作它用，则需要将内容写回磁盘），已修改但是不写出（modified no-write，与已修改类似，只是不需要将内容写回磁盘），转移状态，空闲状态（Free），零化状态（Zerod），坏状态。MMPFN结构体中的成员，对应不同状态下，内容的含义不同。

```
//
// Memory Manager Page Lists 内存管理器页列表
//
typedef enum _MMLISTS
{
   ZeroedPageList = 0,				// 零化页列表
   FreePageList = 1,				// 空闲页列表
   StandbyPageList = 2,				// 备用页列表
   ModifiedPageList = 3,			// 已修改页列表（需要写回磁盘）
   ModifiedNoWritePageList = 4,		// 已修改页列表（不需要写回磁盘）
   BadPageList = 5,					// 坏页面列表
   ActiveAndValid = 6,				// 活动和有效页
   TransitionPage = 7				// 转移状态
} MMLISTS;

MMPFNLIST MmZeroedPageListHead = {0, ZeroedPageList, LIST_HEAD, LIST_HEAD};
MMPFNLIST MmFreePageListHead = {0, FreePageList, LIST_HEAD, LIST_HEAD};
MMPFNLIST MmStandbyPageListHead = {0, StandbyPageList, LIST_HEAD, LIST_HEAD};
MMPFNLIST MmStandbyPageListByPriority[8];
MMPFNLIST MmModifiedPageListHead = {0, ModifiedPageList, LIST_HEAD, LIST_HEAD};
MMPFNLIST MmModifiedPageListByColor[1] = {{0, ModifiedPageList, LIST_HEAD, LIST_HEAD}};
MMPFNLIST MmModifiedNoWritePageListHead = {0, ModifiedNoWritePageList, LIST_HEAD, LIST_HEAD};
MMPFNLIST MmBadPageListHead = {0, BadPageList, LIST_HEAD, LIST_HEAD};
MMPFNLIST MmRomPageListHead = {0, StandbyPageList, LIST_HEAD, LIST_HEAD};

PMMPFNLIST MmPageLocationList[] =
{
    &MmZeroedPageListHead,			// 零化页列表
    &MmFreePageListHead,			// 空闲页列表
    &MmStandbyPageListHead,			// 备用页列表
    &MmModifiedPageListHead,		// 已修改页列表（需要写回磁盘）
    &MmModifiedNoWritePageListHead, // 已修改页列表（不需要写回磁盘）
    &MmBadPageListHead,				// 坏页面列表
    NULL,
    NULL
};

typedef struct _MMPFNLIST
{
    PFN_NUMBER Total;
    MMLISTS ListName;
    PFN_NUMBER Flink;
    PFN_NUMBER Blink;
} MMPFNLIST, *PMMPFNLIST;
```

从MmPageLocationList数组可以看到，活动状态和转移状态没有对应的列表。另外的六个状态有对应的列表，分别用于保存对应状态的物理页面。

那么一个物理页面可能会经历从空闲，到被使用，用完后的页面删除后进入备用或已修改状态，当物理页释放后又重新进入到空闲状态，空闲状态经过零化线程处理，最终进入零化页面列表这么一个过程。对物理内存使用的起点，就是分配物理页面。分配物理页面的函数是MmAllocPage()，函数源码代码如下代码块所示。返回类型是PFN_NUMBER，即LONG_PTR类型，它是所分配的页面页框号（页帧号）。函数比较简单，首先锁定页帧号队列锁，然后调用MiRemoveZeroPage()获取一个空闲页帧的页帧号，获取成功则将物理页信息数组中对应项的内容进行设置，表明对应物理页面已经分配。

```
PFN_NUMBER NTAPI MmAllocPage(ULONG Type)
{
    PFN_NUMBER PfnOffset;
    PMMPFN Pfn1;
    KIRQL OldIrql;

    OldIrql = KeAcquireQueuedSpinLock(LockQueuePfnLock);

    PfnOffset = MiRemoveZeroPage(MI_GET_NEXT_COLOR());
    if (!PfnOffset)
    {
        KeBugCheck(NO_PAGES_AVAILABLE);
    }

    DPRINT("Legacy allocate: %lx\n", PfnOffset);
    Pfn1 = MiGetPfnEntry(PfnOffset);
    Pfn1->u3.e2.ReferenceCount = 1;
    Pfn1->u3.e1.PageLocation = ActiveAndValid;

    /* This marks the PFN as a ReactOS PFN */
    Pfn1->u4.AweAllocation = TRUE;

    /* Allocate the extra ReactOS Data and zero it out */
    Pfn1->u1.SwapEntry = 0;
    Pfn1->RmapListHead = NULL;

    KeReleaseQueuedSpinLock(LockQueuePfnLock, OldIrql);
    return PfnOffset;
}

PFN_NUMBER NTAPI MiRemoveZeroPage(IN ULONG Color)
{
    PFN_NUMBER PageIndex;
    PMMPFN Pfn1;
    BOOLEAN Zero = FALSE;

    /* Make sure PFN lock is held and we have pages */
    // 验证已经获取了PFN锁
    ASSERT(KeGetCurrentIrql() == DISPATCH_LEVEL);
    ASSERT(MmAvailablePages != 0);
    ASSERT(Color < MmSecondaryColors);	// 这里有一个颜色值判断，不能超过最大值 >

    // Check the colored zero list 首先校验染色的清零内存块列表
    PageIndex = MmFreePagesByColor[ZeroedPageList][Color].Flink;
    if (PageIndex == LIST_HEAD)
    {
        // Check the zero list
        ASSERT_LIST_INVARIANT(&MmZeroedPageListHead);
        PageIndex = MmZeroedPageListHead.Flink;
        if (PageIndex == LIST_HEAD)
        {
            /* This means there's no zero pages, we have to look for free ones */
            ASSERT(MmZeroedPageListHead.Total == 0);
            Zero = TRUE;

            /* Check the colored free list */
            PageIndex = MmFreePagesByColor[FreePageList][Color].Flink;
            if (PageIndex == LIST_HEAD)
            {
                /* Check the free list */
                ASSERT_LIST_INVARIANT(&MmFreePageListHead);
                PageIndex = MmFreePageListHead.Flink;
                Color = PageIndex & MmSecondaryColorMask;
                ASSERT(PageIndex != LIST_HEAD);
                if (PageIndex == LIST_HEAD)
                {
                    /* FIXME: Should check the standby list */
                    ASSERT(MmZeroedPageListHead.Total == 0);
                }
            }
        }
        else
        {
            Color = PageIndex & MmSecondaryColorMask;
        }
    }

    /* Sanity checks */
    Pfn1 = MI_PFN_ELEMENT(PageIndex);
    ASSERT((Pfn1->u3.e1.PageLocation == FreePageList) ||
           (Pfn1->u3.e1.PageLocation == ZeroedPageList));

    /* Remove the page from its list */
    PageIndex = MiRemovePageByColor(PageIndex, Color);
    ASSERT(Pfn1 == MI_PFN_ELEMENT(PageIndex));

    /* Zero it, if needed */
    if (Zero) MiZeroPhysicalPage(PageIndex);

    /* Sanity checks */
    ASSERT(Pfn1->u3.e2.ReferenceCount == 0);
    ASSERT(Pfn1->u2.ShareCount == 0);
    ASSERT_LIST_INVARIANT(&MmFreePageListHead);
    ASSERT_LIST_INVARIANT(&MmZeroedPageListHead);

    /* Return the page */
    return PageIndex;
}
```

真正获取物理页面号的是MiRemoveZeroPage()函数。在详细解析这个函数之前，先说一下物理页面管理的一个着色的算法。其实，这个着色算法是一个缓存算法，着色列表是一个缓存列表，Windows试图让页面尽可能遍布缓冲中，有效发挥缓存的作用，这种做法是为了加速MIPS体系将结构和NUMA结构。这里不过多关注着色这个算法。函数中，首先根据颜色值，从对应颜色的零化页面列表中查找，如果查找成功，则直接从列表中删除该页面，返回页面索引。否则查找零化页表，如果零化页表为空，则查找着色的空闲列表对应颜色的列表是否有页面存在，如果依旧为空，则查找空闲列表。如果空闲列表为空，则直接异常了，从注释可以看到，需要检查备用列表。

> 这里有一点需要注意，所有的物理页面是放在上述MmPageLocationList数组中六个列表之一。对于着色列表，仅仅是用于快速检索，使用缓存功能，一个空闲页，会被放到空闲列表，同时也会根据其颜色值，放到对应的颜色列表。
> 关于着色的算法是比较完整过程的，后面有机会专门针对算法进行总结吧，这里不再过多详述。

比MmAllocPage函数高一级的函数是MmRequestPageMemoryConsumer函数，这个函数根据不同的用途，进行物理页面的分配。代码逻辑比较简单，不再复述，详细内容可以参考注释。

```
#define MC_CACHE                            (0)
#define MC_USER                             (1)
#define MC_SYSTEM                           (2)
#define MC_MAXIMUM                          (3)

NTSTATUS
NTAPI MmRequestPageMemoryConsumer(ULONG Consumer, BOOLEAN CanWait, PPFN_NUMBER AllocatedPage)
{
    ULONG PagesUsed;
    PFN_NUMBER Page;

    /*
     * Make sure we don't exceed our individual target.
     * 首先保证没有超过单个物理内存使用的限制
     */
    PagesUsed = InterlockedIncrementUL(&MiMemoryConsumers[Consumer].PagesUsed);
    if (PagesUsed > MiMemoryConsumers[Consumer].PagesTarget &&
            !MiIsBalancerThread())
    {
        MmRebalanceMemoryConsumers();
    }

    /*
     * Allocate always memory for the non paged pool and for the pager thread.
     */
    // 为Non-Paged内存池分配物理页面，它有特殊的优待，不受页面数量限制
    if ((Consumer == MC_SYSTEM) /* || MiIsBalancerThread() */)
    {
        Page = MmAllocPage(Consumer);
        if (Page == 0)
        {
            KeBugCheck(NO_PAGES_AVAILABLE);
        }
        if (Consumer == MC_USER) MmInsertLRULastUserPage(Page);
        *AllocatedPage = Page;
        if (MmAvailablePages  MiMinimumAvailablePages)	// 这里有一个小于号，判断是否比最低限还小
            MmRebalanceMemoryConsumers();
        return(STATUS_SUCCESS);
    }

    /*
     * Make sure we don't exceed global targets.
     * 对于用户请求，要判断是否超过了全局的限制值，如果超过，根据是否等待，分别进行处理
     */
    if (((MmAvailablePages < MiMinimumAvailablePages) && !MiIsBalancerThread())	//
            || (MmAvailablePages < (MiMinimumAvailablePages / 2)))
    {
        MM_ALLOCATION_REQUEST Request;

        if (!CanWait)	// 内存用量超过预定限制数量，如果不能等待物理页面释放，则返回没有内存可用
        {
            (void)InterlockedDecrementUL(&MiMemoryConsumers[Consumer].PagesUsed);
            MmRebalanceMemoryConsumers();
            return(STATUS_NO_MEMORY);
        }

        /* Insert an allocation request. */ //将分配请求添加到等待队列，等待物理页面释放
        Request.Page = 0;
        KeInitializeEvent(&Request.Event, NotificationEvent, FALSE);

        ExInterlockedInsertTailList(&AllocationListHead, &Request.ListEntry, &AllocationListLock);
        MmRebalanceMemoryConsumers();

        KeWaitForSingleObject(&Request.Event,
                              0,
                              KernelMode,
                              FALSE,
                              NULL);

        Page = Request.Page;
        if (Page == 0)
        {
            KeBugCheck(NO_PAGES_AVAILABLE);
        }

        if(Consumer == MC_USER) MmInsertLRULastUserPage(Page);
        *AllocatedPage = Page;

        if (MmAvailablePages < MiMinimumAvailablePages)	// >
        {
            MmRebalanceMemoryConsumers();
        }

        return(STATUS_SUCCESS);
    }

    /*
     * Actually allocate the page. 进行页面分配
     */
    Page = MmAllocPage(Consumer);
    if (Page == 0)
    {
        KeBugCheck(NO_PAGES_AVAILABLE);
    }
    if(Consumer == MC_USER) MmInsertLRULastUserPage(Page);
    *AllocatedPage = Page;

    if (MmAvailablePages < MiMinimumAvailablePages)	//> 小于最小限制，进行内存整合
    {
        MmRebalanceMemoryConsumers();
    }

    return(STATUS_SUCCESS);
}
```

上述为分配的过程，下面给出物理页面释放的过程，与MmRequestPageMemoryConsumer分配函数对应的释放函数为MmReleasePageMemoryConsumer函数，它依次会调用MmDereferencePage()减少物理页的引用，如果引用为0，则正式释放页面。有前面的分配过程的解释，物理页面的释放过程就会简单很多，这里不再做过多解释，一码胜千言！

有一个地方需要补充一下，在释放页面时，同时会将空闲的物理页面放到着色列表中，但是并没有用到备用列表。和分配函数中相对应，分配物理页面时也没有检查备用列表。其实在我目前看的ReactOS源码中，它并没有启用六个状态中的备用列表。

```
NTSTATUS
NTAPI
MmReleasePageMemoryConsumer(ULONG Consumer, PFN_NUMBER Page)
{
    if (Page == 0)
    {
        DPRINT1("Tried to release page zero.\n");
        KeBugCheck(MEMORY_MANAGEMENT);
    }

    if (MmGetReferenceCountPage(Page) == 1)		// 如果物理页面引用为1，则将对应Consumer使用计数进行修改
    {
        if(Consumer == MC_USER) MmRemoveLRUUserPage(Page);
        (void)InterlockedDecrementUL(&MiMemoryConsumers[Consumer].PagesUsed);
    }

    MmDereferencePage(Page);					// 修改物理页面引用

    return(STATUS_SUCCESS);
}

VOID
NTAPI
MmDereferencePage(PFN_NUMBER Pfn)
{
    PMMPFN Pfn1;
    KIRQL OldIrql;
    DPRINT("MmDereferencePage(PhysicalAddress %x)\n", Pfn << PAGE_SHIFT);

    OldIrql = KeAcquireQueuedSpinLock(LockQueuePfnLock);	// 物理页帧数组锁

    Pfn1 = MiGetPfnEntry(Pfn);		// 获取物理页对应的页帧数组元素
    ASSERT(Pfn1);
    ASSERT_IS_ROS_PFN(Pfn1);

    ASSERT(Pfn1->u3.e2.ReferenceCount != 0);
    Pfn1->u3.e2.ReferenceCount--;	// 引用计数
    if (Pfn1->u3.e2.ReferenceCount == 0)
    {
    	// 引用计数降为0，则要将页面释放掉
        /* Mark the page temporarily as valid, we're going to make it free soon */
        Pfn1->u3.e1.PageLocation = ActiveAndValid;	// 页面所处的状态还要设置为活动有效

        /* It's not a ROS PFN anymore */	// 不再是ROS PFN页面
        Pfn1->u4.AweAllocation = FALSE;

        /* Bring it back into the free list */ // 将页面插入空闲列表
        DPRINT("Legacy free: %lx\n", Pfn);
        MiInsertPageInFreeList(Pfn);
    }

    KeReleaseQueuedSpinLock(LockQueuePfnLock, OldIrql);
}

VOID NTAPI MiInsertPageInFreeList(IN PFN_NUMBER PageFrameIndex)
{
    PMMPFNLIST ListHead;
    PFN_NUMBER LastPage;
    PMMPFN Pfn1;
    ULONG Color;
    PMMPFN Blink;
    PMMCOLOR_TABLES ColorTable;

    /* Make sure the page index is valid */ // 确保页面索引有效
    ASSERT(KeGetCurrentIrql() >= DISPATCH_LEVEL);
    ASSERT((PageFrameIndex != 0) &&
           (PageFrameIndex <= MmHighestPhysicalPage) &&
           (PageFrameIndex >= MmLowestPhysicalPage));

    /* Get the PFN entry */ // 获取页帧号对应的页帧数组的项目
    Pfn1 = MI_PFN_ELEMENT(PageFrameIndex);

    /* Sanity checks that a right kind of page is being inserted here */
    ASSERT(Pfn1->u4.MustBeCached == 0);
    ASSERT(Pfn1->u3.e1.Rom != 1);
    ASSERT(Pfn1->u3.e1.RemovalRequested == 0);
    ASSERT(Pfn1->u4.VerifierAllocation == 0);
    ASSERT(Pfn1->u3.e2.ReferenceCount == 0);

    /* HACK HACK HACK : Feed the page to legacy Mm */ // 兼容老的物理页分配等待列表
    if (MmRosNotifyAvailablePage(PageFrameIndex))
    {
        DPRINT1("Legacy Mm eating ARM3 page!.\n");
        return;
    }

    /* Get the free page list and increment its count */ // 将页面插入到空闲列表中
    ListHead = &MmFreePageListHead;
    ASSERT_LIST_INVARIANT(ListHead);
    ListHead->Total++;

    /* Get the last page on the list */
    LastPage = ListHead->Blink;
    if (LastPage != LIST_HEAD)
    {
        /* Link us with the previous page, so we're at the end now */
        MI_PFN_ELEMENT(LastPage)->u1.Flink = PageFrameIndex;
    }
    else
    {
        /* The list is empty, so we are the first page */
        ListHead->Flink = PageFrameIndex;
    }

    /* Now make the list head point back to us (since we go at the end) */
    ListHead->Blink = PageFrameIndex;
    ASSERT_LIST_INVARIANT(ListHead);

    /* And initialize our own list pointers */
    Pfn1->u1.Flink = LIST_HEAD;
    Pfn1->u2.Blink = LastPage;

    /* Set the list name and default priority */ // 修改列表状态，以及优先级
    Pfn1->u3.e1.PageLocation = FreePageList;
    Pfn1->u4.Priority = 3;

    /* Clear some status fields */ // 清空状态域
    Pfn1->u4.InPageError = 0;
    Pfn1->u4.AweAllocation = 0;

    /* Increment number of available pages */
    MiIncrementAvailablePages();	// 修改可用物理页数量

    /* Get the page color */		// 计算页面颜色，下面将物理页面插入到对应颜色列表
    Color = PageFrameIndex & MmSecondaryColorMask;

    /* Get the first page on the color list */
    ColorTable = &MmFreePagesByColor[FreePageList][Color];
    if (ColorTable->Flink == LIST_HEAD)	// 注意，颜色列表的前后向指针和空闲列表用的并不是同一个成员
    {
        /* The list is empty, so we are the first page */
        Pfn1->u4.PteFrame = COLORED_LIST_HEAD;
        ColorTable->Flink = PageFrameIndex;
    }
    else
    {
        /* Get the previous page */
        Blink = (PMMPFN)ColorTable->Blink;

        /* Make it link to us, and link back to it */
        Blink->OriginalPte.u.Long = PageFrameIndex;
        Pfn1->u4.PteFrame = MiGetPfnEntryIndex(Blink);
    }

    /* Now initialize our own list pointers */
    ColorTable->Blink = Pfn1;

    /* This page is now the last */
    Pfn1->OriginalPte.u.Long = LIST_HEAD;

    /* And increase the count in the colored list */
    ColorTable->Count++;	// 修改颜色列表的计数

    /* Notify zero page thread if enough pages are on the free list now */
    // 如果空闲列表元素足够多，则唤醒零化线程，进行清零处理
    if ((ListHead->Total >= 8) && !(MmZeroingPageThreadActive))
    {
        /* Set the event */
        KeSetEvent(&MmZeroingPageEvent, IO_NO_INCREMENT, FALSE);
    }

#if MI_TRACE_PFNS
    Pfn1->PfnUsage = MI_USAGE_FREE_PAGE;
    RtlZeroMemory(Pfn1->ProcessName, 16);
#endif
}
```

#### 虚拟内存到物理内存映射 ####

前面两小节分别总结了虚拟内存管理和物理内存管理的方法和重点函数，这一节则总结一下如何将两者联系起来，即虚拟内存到物理内存的映射。

按照一个进程来看，它整个地址空间有4G，2G用户空间和2G内核空间；物理内存是以4K大小的页面分割后进行管理。要想访问虚拟内存，则需要将虚拟内存映射到实际存在的物理内存上，那么虚拟内存也需要以4K为单位进行划分，以便与物理内存页进行一一对应。最直接的想法则是将4G的内存划分成为`4G/4K=1M个页面`，每个页面要对应一个32位地址，即四个字节，那么要保存这些条目则需要4M连续的内存空间。如果是这么设计，对于早期的计算机来说代价是很大的。

于是提出了二级映射的概念，就是将1M个页面的索引设计为二级目录，通过间接索引的方式获得。那么对于一个虚拟地址，它就被划分成为三部分，第一部分为最低12位，作为4k页面内的内存单元地址索引；第二部分为10位，作为第二级目录的索引，10位可以索引1024个条目，每个条目为4字节，则一个二级目录正好占一页内存；第三部分为10位，同样索引1024个条目，也占用一页。如下图所示，给出一个地址拆分转换的过程。

![图 1 32位地址拆分索引示意](/img/2017-06-26-ReactOS-MemoryManagement-address-mapping.jpg)

图下面使用的就是页面映射表了，CR3寄存器指向当前进程的页面映射表。页面映射表分为页目录和页表，页目录表项简称PDE，页面映射表项简称PTE。页表项表示的是对应的虚拟内存是否映射为物理页面，条目高20位保存了物理页面的页帧号PFN，那么低12位则空下来。低12位中其中较高的4位定义为Avail，由程序员使用；剩余的8位则表示了条目所对应的物理页面的状态或属性。

```
#define PA_BIT_PRESENT   (0)		// 是否有效，即映射为物理内存了
#define PA_BIT_READWRITE (1)		// 可读可写
#define PA_BIT_USER      (2)		// 用户
#define PA_BIT_WT        (3)		// 写穿透
#define PA_BIT_CD        (4)		//
#define PA_BIT_ACCESSED  (5)		// 已经访问过
#define PA_BIT_DIRTY     (6)		// 内存块数据是'脏'数据
#define PA_BIT_GLOBAL    (8)		// 用于全局内存，即内核使用

#define PA_PRESENT   (1 << PA_BIT_PRESENT)
#define PA_READWRITE (1 << PA_BIT_READWRITE)
#define PA_USER      (1 << PA_BIT_USER)
#define PA_DIRTY     (1 << PA_BIT_DIRTY)
#define PA_WT        (1 << PA_BIT_WT)
#define PA_CD        (1 << PA_BIT_CD)
#define PA_ACCESSED  (1 << PA_BIT_ACCESSED)
#define PA_GLOBAL    (1 << PA_BIT_GLOBAL)
```

8位状态位依次对应含义如上代码块所示，其中最重要的是0位，表示当前条目对应的物理内存是否存在。

首先从页映射的建立开始分析映射中的几个比较重要的函数，建立页面映射的即MmCreateVirtualMapping()函数，代码如下代码块所示。代码比较简单，首先要确保给出的物理页面都是可用的，一旦出现错误情况，直接就蓝屏了。然后直接调用了MmCreateVirtualMappingUnsafe()函数。

```
NTSTATUS
NTAPI MmCreateVirtualMapping(PEPROCESS Process, PVOID Address, ULONG flProtect,
					   PPFN_NUMBER Pages, ULONG PageCount)
{
    ULONG i;

    ASSERT((ULONG_PTR)Address % PAGE_SIZE == 0);
    for (i = 0; i < PageCount; i++)	// >创建映射时，要映射的物理页面是否都有效 Pages是个页帧号数组
    {
        if (!MmIsPageInUse(Pages[i]))
        {
            DPRINT1("Page at address %x not in use\n", PFN_TO_PTE(Pages[i]));
            KeBugCheck(MEMORY_MANAGEMENT);
        }
    }

    return(MmCreateVirtualMappingUnsafe(Process, Address, flProtect, Pages, PageCount));
}

NTSTATUS
NTAPI
MmCreateVirtualMappingUnsafe(PEPROCESS Process, PVOID Address, ULONG flProtect, PPFN_NUMBER Pages,
                             ULONG PageCount)
{
    ULONG Attributes;
    PVOID Addr;
    ULONG i;
    ULONG oldPdeOffset, PdeOffset;
    PULONG Pt = NULL;
    ULONG Pte;
    DPRINT("MmCreateVirtualMappingUnsafe(%p, %p, %lu, %p (%x), %lu)\n",
           Process, Address, flProtect, Pages, *Pages, PageCount);

    ASSERT(((ULONG_PTR)Address % PAGE_SIZE) == 0);

    if (Process == NULL)	// 内核空间页面映射
    {
        if (Address < MmSystemRangeStart)	// 校验参数符合条件
        {
            DPRINT1("No process\n");
            KeBugCheck(MEMORY_MANAGEMENT);
        }
        if (PageCount > 0x10000 ||
            (ULONG_PTR) Address / PAGE_SIZE + PageCount > 0x100000)
        {
            DPRINT1("Page count too large\n");
            KeBugCheck(MEMORY_MANAGEMENT);
        }
    }
    else
    {
        if (Address >= MmSystemRangeStart)	// 用户空间页面映射，校验给定参数的合法性
        {
            DPRINT1("Setting kernel address with process context\n");
            KeBugCheck(MEMORY_MANAGEMENT);
        }
        if (PageCount > (ULONG_PTR)MmSystemRangeStart / PAGE_SIZE ||
            (ULONG_PTR) Address / PAGE_SIZE + PageCount >
            (ULONG_PTR)MmSystemRangeStart / PAGE_SIZE)
        {
            DPRINT1("Page Count too large\n");
            KeBugCheck(MEMORY_MANAGEMENT);
        }
    }
	// 将页面的保护属性转化为PDE/PTE的第八位属性值
    Attributes = ProtectToPTE(flProtect);
    Attributes &= 0xfff;
    if (Address >= MmSystemRangeStart)
    {
        Attributes &= ~PA_USER;	// 内核页面映射去掉PA_USER标记
    }
    else
    {
        Attributes |= PA_USER;
    }

    Addr = Address;
    /* MmGetPageTableForProcess should be called on the first run, so
     * let this trigger it */ // 第一次运行时，就应该调用MmGetPageTableForProcess
    oldPdeOffset = ADDR_TO_PDE_OFFSET(Addr) + 1;
    for (i = 0; i < PageCount; i++, Addr = (PVOID)((ULONG_PTR)Addr + PAGE_SIZE)) //>
    {
        if (!(Attributes & PA_PRESENT) && Pages[i] != 0)
        {
            DPRINT1("Setting physical address but not allowing access at address "
                    "0x%p with attributes %x/%x.\n",
                    Addr, Attributes, flProtect);
            KeBugCheck(MEMORY_MANAGEMENT);
        }
        PdeOffset = ADDR_TO_PDE_OFFSET(Addr);	// 获取PDE
        if (oldPdeOffset != PdeOffset)
        {
            if(Pt) MmUnmapPageTable(Pt);
            Pt = MmGetPageTableForProcess(Process, Addr, TRUE);
            if (Pt == NULL)
            {
                KeBugCheck(MEMORY_MANAGEMENT);
            }
        }
        else
        {
            Pt++;
        }
        oldPdeOffset = PdeOffset;
		// Pt为指向当前要映射页面在页表中的条目，填写PTE内容
        Pte = InterlockedExchangePte(Pt, PFN_TO_PTE(Pages[i]) | Attributes);

        /* There should not be anything valid here */
        if (Pte != 0)
        {
            DPRINT1("Bad PTE %lx at %p for %p + %lu\n", Pte, Pt, Address, i);
            KeBugCheck(MEMORY_MANAGEMENT);
        }

        /* We don't need to flush the TLB here because it only caches valid translations
         * and we're moving this PTE from invalid to valid so it can't be cached right now */

		if (Addr < MmSystemRangeStart)
		{
			/* Add PDE reference */
			Process->Vm.VmWorkingSetList->UsedPageTableEntries[MiGetPdeOffset(Addr)]++;
			ASSERT(Process->Vm.VmWorkingSetList->UsedPageTableEntries[MiGetPdeOffset(Addr)] <= PTE_COUNT);
		}
    }

    ASSERT(Addr > Address);
    MmUnmapPageTable(Pt);

    return(STATUS_SUCCESS);
}
```

函数MmCreateVirtualMappingUnsafe()的逻辑也比较清晰，首先判断参数的合法性；然后则根据要映射地址修改对应PTE内容。这里面有一个隐含的问题，要访问PTE（页表项）首先要根据页目录找到页表项的物理内存，访问页目录会存在几种情况。第一种是从一个进程访问另外一个进程的内存，这时要使用Hyperspace进行进程间内存的临时映射；第二种情况是映射进程自己的页面，则只需要获取当前进程的映射表即可；第三种是映射内核的页面，也是直接操作即可。

函数MmGetPageTableForProcess可以获取指定进程的页表，这个对于跨进程的操作，在使用完之后需要进行释放，释放通过MmUnmapPageTable()函数完成。这块涉及到Hyperspace的操作，这里不做过多介绍，了解即可。

创建映射的逆操作就是删除页面映射，这个动作通过MmDeleteVirtualMapping函数完成。首先获取进程的页表项指针，删除映射直接将映射置空即可。

```
/*
 * FUNCTION: Delete a virtual mapping
 */
VOID NTAPI
MmDeleteVirtualMapping(PEPROCESS Process, PVOID Address, BOOLEAN* WasDirty, PPFN_NUMBER Page)
{
    BOOLEAN WasValid = FALSE;
    PFN_NUMBER Pfn;
    ULONG Pte;
    PULONG Pt;

    DPRINT("MmDeleteVirtualMapping(%p, %p, %p, %p)\n",
           Process, Address, WasDirty, Page);
	// 获取进程的页表
    Pt = MmGetPageTableForProcess(Process, Address, FALSE);

    if (Pt == NULL)
    {
        if (WasDirty != NULL)
        {
            *WasDirty = FALSE;
        }
        if (Page != NULL)
        {
            *Page = 0;
        }
        return;
    }

    /*
     * Atomically set the entry to zero and get the old value.
     */ // 直接将页表内容设置为空即可
    Pte = InterlockedExchangePte(Pt, 0);

    /* We count a mapping as valid if it's a present page, or it's a nonzero pfn with
     * the swap bit unset, indicating a valid page protected to PAGE_NOACCESS. */
    WasValid = (Pte & PA_PRESENT) || ((Pte >> PAGE_SHIFT) && !(Pte & 0x800));
    if (WasValid)
    {
        /* Flush the TLB since we transitioned this PTE
         * from valid to invalid so any stale translations
         * are removed from the cache */ // PTE从有效变为无效，要刷新TLB表
        MiFlushTlb(Pt, Address);

		if (Address < MmSystemRangeStart) // 用户空间的地址，则将PDE引用值减小
		{
			/* Remove PDE reference */
			Process->Vm.VmWorkingSetList->UsedPageTableEntries[MiGetPdeOffset(Address)]--;
			ASSERT(Process->Vm.VmWorkingSetList->UsedPageTableEntries[MiGetPdeOffset(Address)] < PTE_COUNT);	// >
		}

        Pfn = PTE_TO_PFN(Pte);
    }
    else
    {
        MmUnmapPageTable(Pt);
        Pfn = 0;
    }

    /*
     * Return some information to the caller
     */
    if (WasDirty != NULL)
    {
        *WasDirty = ((Pte & PA_DIRTY) && (Pte & PA_PRESENT)) ? TRUE : FALSE;
    }
    if (Page != NULL)
    {
        *Page = Pfn;
    }
}
```

删除映射时，对于有物理页对应的，则将物理页号返回，由上层逻辑删除物理内存的使用。而映射无效的情况不需要处理物理页面，则调用MmUnmapPageTable将Hyperspace映射删除即可。

关于添加虚拟页面到物理页面的映射操作很简单，但是里面有几个点是理解起来比较困难的，列举如下。

* 二级映射表如何建立，建立过程是怎样的？这个只能在学习系统启动过程时再回来补充一些内容！
* Hyperspace映射的原理是什么，通过什么方式实现？
	它其实是一个子话题，Windows的进程间内存访问是如何实现的？这个单独整理一片文章，学习一下。
* 页面的换出与换入，这个是不切断页面映射的，但是没有物理页面与之对应了，这个也作为下篇的一小节学习一下。
* 目前的操作都是建立在进程/系统运行起来后的操作，与二级映射表建立类似的问题，物理页面管理和虚拟内存管理AVL树等是何时建立？

关于虚拟页面映射到物理页面还有几个问题需要注意下：

> 一级映射和二级映射管理4G内存空间，他们其实消耗是相同的，都只需要1024个页面即可。很多人考虑到二级映射中除了1024个页表外，还需要一个页目录，那就应该是1025个页面！但其实细想的话会发现，页目录本身也是一个页表，也即在页目录中有一项是指向自己的，另外只需要1023个页表就可以指向剩余的4G-4M的内存块了，

> 接上一条，其实页面映射表占用的也是虚拟地址空间，他们自身也需要占据一个页目录；再者页目录表和页面表都是需要物理内存才能访问，这个在创建页映射的函数中有所体现。

这样就将三个小话题简单过了一下，只是按照毛老师书上的介绍顺序过了一遍，里面其实涉及很多知识没有加进去，容以后学习更多后，慢慢补充。

** 更新记录 **

* 文章完成于 2017-06-26 09:24:23

** 参考资料 **

1. 《Windows内核情景分析-采用开源代码ReactOS》
2. ReactOS源码
3. 《WINDOWS内核原理与实现》


By Andy @2017-06-26 09:24:47
