---
title: ReactOS 内存管理(下)
date: 2017-06-30 09:15:33
tags:
- ReactOS
- Memory
categories:
- ReactOS
---

上一篇内存管理的总结中只有三个主题，鉴于关于内存的内容众多，将后四个主题在这一篇总结一下。

在上一篇中只是看了一下物理页面如何和虚拟页面映射，这个前提是页面映射表已经构建完成。但是对于新建的进程的页面映射表是如何构建的呢？在继续下面内容之前，先分析一下系统的页面映射表是如何构建的！

在系统启动过程中，内核初始化会有两个阶段，这个阶段中会调用内存的初始化函数MmInitSystem()，

```
VOID
INIT_FUNCTION
NTAPI
MmInitGlobalKernelPageDirectory(VOID)
{
   ULONG i;
   DPRINT("MmInitGlobalKernelPageDirectory()\n");

   if (Ke386Pae)
   {
	  ...
   }
   else
   {
      PULONG CurrentPageDirectory = (PULONG)PAGEDIRECTORY_MAP;
      for (i = ADDR_TO_PDE_OFFSET(MmSystemRangeStart); i < 1024; i++)
      {
         if (i != ADDR_TO_PDE_OFFSET(PTE_BASE) &&
             i != ADDR_TO_PDE_OFFSET(HYPERSPACE) &&
             0 == MmGlobalKernelPageDirectory[i] && 0 != CurrentPageDirectory[i])
         {
            MmGlobalKernelPageDirectory[i] = CurrentPageDirectory[i];
            if (Ke386GlobalPagesEnabled)
            {
               MmGlobalKernelPageDirectory[i] |= PA_GLOBAL;
               CurrentPageDirectory[i] |= PA_GLOBAL;
            }
         }
      }
   }
}
```
页面映射表初始化



#### 读取其他进程内存 ###

在编程中访问内存用到的都是虚拟地址，对于本进程的内存进行访问直接使用进程的内存映射表即可找到最终访问的物理内存了。但是在Windows中可以访问其他进程的内存，并且可以跨进程进行内存分配。那么这是如何做到的呢？在上篇的内容中提到了对Hyperspace的使用，Hyperspace就是这么一块虚拟内存，用于将一些特定的物理页面映射进来以便访问。那么要访问其他进程的内存，只需要将其他进程的页面映射表映射到Hyperspace（一部分），找到对应要访问的内存的物理页面，然后将内存物理页面映射到这个Hyperspace即可访问了。这一小节就学习一下Hyperspace是如何做到这么牛X的！！

ReactOS给Hyperspace划出来的这块虚拟地址空间是由HYPERSPACE宏定义确定，定义如下所示。

```
#define HYPERSPACE              (Ke386Pae ? 0xc0800000 : 0xc0400000)
#define IS_HYPERSPACE(v)        (((ULONG)(v) >= HYPERSPACE && (ULONG)(v) < HYPERSPACE + 0x400000))
```

对于普通的系统（未开启内存扩展），超级空间的起始地址为0xC0400000。下面直接看如何在“超级空间”中建立映射，完成这个操作的函数是MmCreateHyperspaceMapping()，它的源代码如下代码块所示。函数比较简单，获取了EPROCESS之后，直接调用`MmCreateHyperspaceMapping`函数。

PointerPte等于本进程的Hyperspace第一个Pte，其中

```
/* Convert an address to a corresponding PTE */
// PTE_BASE = 0xC0000000
#define MiAddressToPte(x) \
    ((PMMPTE)(PTE_BASE + (((ULONG)(x) >> 12) << 2)))
//
// Setup the mapping PTEs
// 设置映射PTE
MmFirstReservedMappingPte = MiAddressToPte(MI_MAPPING_RANGE_START);		// HYPER_SPACE = 0xC0400000
MmLastReservedMappingPte = MiAddressToPte(MI_MAPPING_RANGE_END);
MmFirstReservedMappingPte->u.Hard.PageFrameNumber = MI_HYPERSPACE_PTES;
```

```
//
// ReactOS Compatibility Layer
//
FORCEINLINE
PVOID
MmCreateHyperspaceMapping(IN PFN_NUMBER Page)
{
    HyperProcess = (PEPROCESS)KeGetCurrentThread()->ApcState.Process;
    return MiMapPageInHyperSpace(HyperProcess, Page, &HyperIrql);
}

#define MmDeleteHyperspaceMapping(x) MiUnmapPageInHyperSpace(HyperProcess, x, HyperIrql);

PVOID
NTAPI
MiMapPageInHyperSpace(IN PEPROCESS Process,
                      IN PFN_NUMBER Page,
                      IN PKIRQL OldIrql)
{
    MMPTE TempPte;
    PMMPTE PointerPte;
    PFN_NUMBER Offset;

    //
    // Never accept page 0 or non-physical pages
    // 判断页帧号有效性
    ASSERT(Page != 0);
    ASSERT(MiGetPfnEntry(Page) != NULL);

    //
    // Build the PTE
    // 构建PTE，预构建的PTE为ValidKernelPteLocal
    TempPte = ValidKernelPteLocal;
    TempPte.u.Hard.PageFrameNumber = Page;

    //
    // Pick the first hyperspace PTE
    // 获取第一个超级空间的 PTE，MmFirstReservedMappingPte指向超级空间的页表
    PointerPte = MmFirstReservedMappingPte;

    //
    // Acquire the hyperlock
    // 获取锁
    ASSERT(Process == PsGetCurrentProcess());
    KeAcquireSpinLock(&Process->HyperSpaceLock, OldIrql);

    //
    // Now get the first free PTE
    // 获取第一个空闲的PTE
    Offset = PFN_FROM_PTE(PointerPte);
    if (!Offset)
    {
        //
        // Reset the PTEs
        //
        Offset = MI_HYPERSPACE_PTES;
        KeFlushProcessTb();
    }

    //
    // Prepare the next PTE
    //
    PointerPte->u.Hard.PageFrameNumber = Offset - 1;

    //
    // Write the current PTE
    //
    PointerPte += Offset;
    MI_WRITE_VALID_PTE(PointerPte, TempPte);

    //
    // Return the address
    //
    return MiPteToAddress(PointerPte);
}

VOID
NTAPI
MiUnmapPageInHyperSpace(IN PEPROCESS Process,
                        IN PVOID Address,
                        IN KIRQL OldIrql)
{
    ASSERT(Process == PsGetCurrentProcess());

    //
    // Blow away the mapping
    //
    MiAddressToPte(Address)->u.Long = 0;

    //
    // Release the hyperlock
    //
    ASSERT(KeGetCurrentIrql() == DISPATCH_LEVEL);
    KeReleaseSpinLock(&Process->HyperSpaceLock, OldIrql);
}
```

#### NtAllocateVirtualMemory函数调用过程 ####

编程中用到`VirtualAlloc`和`VirtualAllocEx`等内存分配函数都会进入NtAllocateVirtualMemory系统调用中，最终进入内核进行内存的分配。这一小节简单分析一下这个函数的代码，以对普通的内存分配有个了解。



#### 页面异常 ####

在前面看到的NtAllocateVirtualMemory函数调用过程中，其实并没有给分配的内存映射物理页面。鉴于资源的有效利用，尽量延后物理内存的分配，所以物理内存都是在访问内存时如果没有对应物理页面映射，引发缺页异常来给要访问的内存配备物理内存。


#### 页面的换入与换出 ####



** 修订记录 **

1. 文章完成于 2017-06-30 09:24:23

** 参考资料 **

1. 《Windows内核情景分析-采用开源代码ReactOS》
2. ReactOS源码


By Andy @2017-06-30 09:24:34
