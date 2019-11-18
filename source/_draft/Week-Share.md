#堆、栈与运行时#

`C/C++`运行时库其实可以大致分为两部分内容，一部分是对系统调用的封装，它提供统一的操作接口，比如标准输入输出，文件操作等；另一部分是和系统无关的一些功能，比如字符串操作。

> VS2008中，`C/C++`的运行时库代码位于`Microsoft Visual Studio 9.0\VC\crt\src`中。

在编译时，就需要选择使用动态运行时库，还是使用静态运行时库。VS2008中默认的就是使用动态运行时库，简单说两者区别，静态运行时库会把上述的源码编译到最终的模块中，而动态运行时库则不会将这些代码编译进最终模块，它默认导入`MSVCR*.dll`对应函数。从这个简单区别来看，如果一个进程的每个模块都将运行时库以静态方式编译，那么每个模块都将保持一份运行时代码（当然只是模块引用到的内容），这样无疑增加了很多的重复代码。

> 动态运行时库的编译选项  /MDd  /MD  -  Multi-threaded Debug DLL / Multi-threaded DLL
> 静态运行时库的编译选项  /MTd  /MT  -  Multi-thread Debug / Multi-threaded

```
// 动态运行时库
0:000> k
ChildEBP RetAddr
0037f760 50c6151e ntdll!RtlAllocateHeap
0037f77c 50c70206 MSVCR90D!_heap_alloc_base+0x5e
0037f7c4 50c6ffbf MSVCR90D!_nh_malloc_dbg+0x2c6
0037f7e4 50c6ff6c MSVCR90D!_nh_malloc_dbg+0x7f
0037f80c 50c7b5eb MSVCR90D!_nh_malloc_dbg+0x2c
0037f82c 50c5db81 MSVCR90D!malloc+0x1b
0037f848 01081fbe MSVCR90D!operator new+0x11
0037f87c 010817b2 RuntimeTest!operator new+0x3e [f:\dd\vctools\crt_bld\self_x86\crt\src\newopnt.cpp @ 18]
0037f88c 010815af RuntimeTest!operator new[]+0x12 [f:\dd\vctools\crt_bld\self_x86\crt\src\newaopnt.cpp @ 15]
0037fad0 01081d58 RuntimeTest!wmain+0x19f [c:\users\administrator\desktop\runtimetest\runtimetest\runtimetest.cpp @ 49]
0037fb20 01081b9f RuntimeTest!__tmainCRTStartup+0x1a8 [f:\dd\vctools\crt_bld\self_x86\crt\src\crtexe.c @ 583]
0037fb28 758a343d RuntimeTest!wmainCRTStartup+0xf [f:\dd\vctools\crt_bld\self_x86\crt\src\crtexe.c @ 403]
0037fb34 771c9832 kernel32!BaseThreadInitThunk+0xe
0037fb74 771c9805 ntdll!__RtlUserThreadStart+0x70
0037fb8c 00000000 ntdll!_RtlUserThreadStart+0x1b
// 静态运行时库
0:000> k
ChildEBP RetAddr
0037f674 50af5d7e ntdll!RtlAllocateHeap
0037f690 50ae0eb6 RuntimeDll!_heap_alloc_base+0x5e [f:\dd\vctools\crt_bld\self_x86\crt\src\malloc.c @ 105]
0037f6d8 50ae0c3f RuntimeDll!_heap_alloc_dbg_impl+0x1f6 [f:\dd\vctools\crt_bld\self_x86\crt\src\dbgheap.c @ 427]
0037f6f8 50ae0bdc RuntimeDll!_nh_malloc_dbg_impl+0x1f [f:\dd\vctools\crt_bld\self_x86\crt\src\dbgheap.c @ 239]
0037f720 50ae068b RuntimeDll!_nh_malloc_dbg+0x2c [f:\dd\vctools\crt_bld\self_x86\crt\src\dbgheap.c @ 296]
0037f740 50ae93f1 RuntimeDll!malloc+0x1b [f:\dd\vctools\crt_bld\self_x86\crt\src\dbgmalloc.c @ 56]
0037f75c 50adf04e RuntimeDll!operator new+0x11 [f:\dd\vctools\crt_bld\self_x86\crt\src\new.cpp @ 59]
0037f790 50adde32 RuntimeDll!operator new+0x3e [f:\dd\vctools\crt_bld\self_x86\crt\src\newopnt.cpp @ 18]
0037f7a0 50add77a RuntimeDll!operator new[]+0x12 [f:\dd\vctools\crt_bld\self_x86\crt\src\newaopnt.cpp @ 15]
0037f894 0108156c RuntimeDll!AllocateMem+0x2a [c:\users\administrator\desktop\runtimetest\runtimedll\runtimedll.cpp @ 20]
0037fad0 01081d58 RuntimeTest!wmain+0x15c [c:\users\administrator\desktop\runtimetest\runtimetest\runtimetest.cpp @ 40]
0037fb20 01081b9f RuntimeTest!__tmainCRTStartup+0x1a8 [f:\dd\vctools\crt_bld\self_x86\crt\src\crtexe.c @ 583]
0037fb28 758a343d RuntimeTest!wmainCRTStartup+0xf [f:\dd\vctools\crt_bld\self_x86\crt\src\crtexe.c @ 403]
0037fb34 771c9832 kernel32!BaseThreadInitThunk+0xe
0037fb74 771c9805 ntdll!__RtlUserThreadStart+0x70
0037fb8c 00000000 ntdll!_RtlUserThreadStart+0x1b
```

从这个两种运行时库的内存分配可以发现，对于静态运行时库则在目标模块中就调用了`ntdll!RtlAllocateHeap`从堆上分配了内存。而动态运行时库，`new`的调用会进入到`MSVCR90D.dll`模块中，再调用的`ntdll!RtlAllocateHeap`从堆上分配的内存。从这个区别就可知，两种运行时库分配内存时所用的堆不是一个。

这回引出一个问题，不同模块中分配的内存，能否在不同的模块中释放呢？看了上面的分析就会发现，这个问题的答案依赖于各个模块的编译方式，是链接的静态运行时库还是动态运行时库。如果所有模块都是使用动态运行时库，那么各个模块分配内存都是动态运行时库模块来完成，也即它们来自同一个运行时堆，模块间就可以相互释放非自己分配内存。

但是，如果在一个模块中出现了静态运行时库编译的模块，那么它分配内存与其他模块分配内存并非一个堆，如果将内存传给其他模块释放，必然引起堆错误，造成崩溃。那么同理推之，如果两个模块都是静态链接CRT方式，那么它们动态分配内存也无法相互释放。所以模块间最好不要相互传递自己通过CRT库函数动态分配的内存。其次是模块最好不要返回`std::string`等具有内存申请与释放的类对象，这容易导致错误。

在Windows的进程中还有一个模块`msvcrt.dll`，这个模块在编译链接时是无法链接到的，它和`msvcr90.dll`区别，前者是为操作系统提供，由微软提供和编译，只供操作系统的模块使用。而后者通常位于运行时支持程序包中（编译器也提供该模块），用于支持用户程序。

> 如果使用动态运行时库编译程序，在有编译器的机器上可以直接运行，而在没有安装过编译器的机器上可能就无法运行，报错的原因就是不存在`MSVCR*.dll`运行时库。

```
GlobalAlloc     // 在32位的Windows上已经简化为从进程默认堆上分配内存
HeapAlloc       // 从指定堆上分配内存，有堆句柄HANDLE hHeap指定堆
LocalAlloc      // 基本等价于GlobalAlloc，16位系统上则是另外一回事
malloc          // 从运行时堆上分配内存
calloc          // 从运行时堆上分配内存，分配的内存被清空
realloc         // 调整分配的内存大小
new             // 同malloc
RtlAllocateHeap // 从指定堆上分配内存
VirtualAlloc    // 系统调用，并非从堆上分配内存，而是直接从地址空间划出内存空间
```

> alloca() 也是C运行时的一个函数，与上述的函数有所区别，它是在线程栈上分配内存，一旦函数返回，分配内存也就被释放了。


### 堆和栈 ###

**栈**

进程的第一个线程是在创建进程时进行创建，其栈分配是在`CreateProcessInternal()`中调用`BaseCreateStack`完成的。

```
NTSTATUS WINAPI
BaseCreateStack(HANDLE hProcess,
                 SIZE_T StackCommit,
                 SIZE_T StackReserve,
                 PINITIAL_TEB InitialTeb)
{
	......
    /* Read page size */
    PageSize = BaseStaticServerData->SysInfo.PageSize;
    AllocationGranularity = BaseStaticServerData->SysInfo.AllocationGranularity;

    /* Get the Image Headers */
    Headers = RtlImageNtHeader(NtCurrentPeb()->ImageBaseAddress);

    StackCommitHeader = Headers->OptionalHeader.SizeOfStackCommit;
    StackReserveHeader = Headers->OptionalHeader.SizeOfStackReserve;

	......

    /* 为栈保留内存 */
    Stack = 0;
    Status = NtAllocateVirtualMemory(hProcess,
                                     (PVOID*)&Stack,
                                     0,
                                     &StackReserve,
                                     MEM_RESERVE,
                                     PAGE_READWRITE);

    /* Now set up some basic Initial TEB Parameters */
    InitialTeb->AllocatedStackBase = (PVOID)Stack;
    InitialTeb->StackBase = (PVOID)(Stack + StackReserve);
    InitialTeb->PreviousStackBase = NULL;
    InitialTeb->PreviousStackLimit = NULL;

    /* Update the Stack Position */
    Stack += StackReserve - StackCommit;

    /* 分配内存 */
    Status = NtAllocateVirtualMemory(hProcess,
                                     (PVOID*)&Stack,
                                     0,
                                     &StackCommit,
                                     MEM_COMMIT,
                                     PAGE_READWRITE);

	......
    /* Now set the current Stack Limit */
    InitialTeb->StackLimit = (PVOID)Stack;

    /* Create a guard page */
    if (UseGuard)
    {
        /* 设置防护页 */
        GuardPageSize = PAGE_SIZE;
        Status = NtProtectVirtualMemory(hProcess,
                                        (PVOID*)&Stack,
                                        &GuardPageSize,
                                        PAGE_GUARD | PAGE_READWRITE,
                                        &Dummy);

        /* Update the Stack Limit keeping in mind the Guard Page */
        InitialTeb->StackLimit = (PVOID)((ULONG_PTR)InitialTeb->StackLimit +
                                         GuardPageSize);
    }

    /* We are done! */
    return STATUS_SUCCESS;
}
```

通过上面代码可知，进程的第一个线程栈是在`CreateProcess*`函数中通过跨进程分配内存来设置的栈。通过`CreateThread`创建的线程，它内部会调用`CreateRemoteThread`，该函数中会调用`BaseCreateStack`函数为新创建进程分配栈空间。

> 也即线程的所使用的栈并不是在内核中创建的，它们是通过跨进程（统一进程创建线程则是非跨进程）分配内存的方式来创建的。

**堆**

与栈其实类似，在进程中使用的堆也是R3的操作，与栈区别是在一个进程中可能有好几种堆。在进程创建完成开始调度执行时，第一个线程在返回R3时，会执行一个APC（这个APC是在内核创建完进程内核结构体和第一个线程结构体后，在线程中插入的。），这个APC返回用户空间指定的APC函数为`LdrInitializeThunk()`，这个函数中会调用`LdrpInitializeProcess()`，进而调用`RtlInitializeHeapManager()`创建进程的默认堆，其最后也是调用的`RtlCreateHeap()`函数创建进程的默认堆。

对于静态运行时库，则在加载模块后调用入口函数时会进行CRT初始化时，会调用到`heap_init`函数进行运行时堆的初始化。最终调用`Kernel32!HeapCreate()`进行堆创建。

对于动态运行时库，在执行模块的入口之前，已经加载了运行时库`MSCRT*.dll`，并且调用它的入口时就会进行它的运行时堆的创建。而使用动态运行时库的模块，在模块入口处就不需要再初始化了。

通过函数`kernel32!GetProcessHeap()`获取当前进程的进程默认堆。`kernel32!GetProcessHeaps()`函数获取进程的私有堆，其中包括默认堆和通过HeapCreate创建的其他的堆。

        其在系统中的位置如下所示：
```
0:000> dt ntdll!_PEB
	......
	+0x014 SubSystemData    : (null)
	+0x018 ProcessHeap      : 0x003c0000     // 进程默认堆
	+0x01c FastPebLock      : 0x77292100 _RTL_CRITICAL_SECTION
	......
	+0x088 NumberOfHeaps    : 1
	+0x08c MaximumNumberOfHeaps : 0x10
	+0x090 ProcessHeaps     : 0x77294760  -> 0x003c0000  // 进程中的堆们
	......
```



