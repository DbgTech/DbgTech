---
title: ReactOS启动过程之阶段0初始化
date: 2017-09-03 20:52:23
tags:
- Windbg
- ReactOS
- 启动过程
categories:
- ReactOS
---

上一节中阅读代码到控制权转到`ntoskrnl.exe`模块中了。进入该模块的`KiSystemStartup()`函数，由于ReactOS支持各种架构，这里以`i386`为例进行代码阅读。如下函数源码位于`*\ntoskrnl\ke\i386\kiinit.c`文件中。

```
VOID NTAPI KiSystemStartup(IN PLOADER_PARAMETER_BLOCK LoaderBlock)
{
    ULONG Cpu;
    PKTHREAD InitialThread;
    ULONG InitialStack;
    PKGDTENTRY Gdt;
    PKIDTENTRY Idt;
    KIDTENTRY NmiEntry, DoubleFaultEntry;
    PKTSS Tss;
    PKIPCR Pcr;
    KIRQL DummyIrql;

    /* 获取引导 时间戳 */
    BootCycles = __rdtsc();

    /* 保存加载器 参数块，获取当前的CPU */
    KeLoaderBlock = LoaderBlock;
    Cpu = KeNumberProcessors;
    if (!Cpu) // 对于第一个引导CPU，则设置如下变量，其他CPU不设置
    {
        /* If this is the boot CPU, set FS and the CPU Number*/
        Ke386SetFs(KGDT_R0_PCR);
        __writefsdword(KPCR_PROCESSOR_NUMBER, Cpu);

        /* Set the initial stack and idle thread as well */
        LoaderBlock->KernelStack = (ULONG_PTR)P0BootStack;
        LoaderBlock->Thread = (ULONG_PTR)&KiInitialThread;
    }

    /* 保存初始线程和栈 */
    InitialStack = LoaderBlock->KernelStack;
    InitialThread = (PKTHREAD)LoaderBlock->Thread;

    /* Clean the APC List Head */ // 初始化内核APC链表头
    InitializeListHead(&InitialThread->ApcState.ApcListHead[KernelMode]);

    /* Initialize the machine type *///初始化机器类型
    KiInitializeMachineType();

    /* 如果不是引导CPU跳过如下的一部分设置 */
    if (Cpu) goto AppCpuInit;

    /* 获取GDT, IDT, PCR 和 TSS的内存指针 */
    KiGetMachineBootPointers(&Gdt, &Idt, &Pcr, &Tss);

    /* 设置TSS描述符和条目 */
    Ki386InitializeTss(Tss, Idt, Gdt);

    /* 初始化 PCR */
    RtlZeroMemory(Pcr, PAGE_SIZE);
    KiInitializePcr(Cpu,
                    Pcr,
                    Idt,
                    Gdt,
                    Tss,
                    InitialThread,
                    (PVOID)KiDoubleFaultStack);

    /* Set us as the current process */
    InitialThread->ApcState.Process = &KiInitialProcess.Pcb;

    /* Clear DR6/7 to cleanup bootloader debugging */
    __writefsdword(KPCR_TEB, 0);
    __writefsdword(KPCR_DR6, 0);
    __writefsdword(KPCR_DR7, 0);

    /* 设置IDT表内容 */
    KeInitExceptions();

    /* Load Ring 3 selectors for DS/ES */
    Ke386SetDs(KGDT_R3_DATA | RPL_MASK);
    Ke386SetEs(KGDT_R3_DATA | RPL_MASK);

    /* Save NMI and double fault traps */
    RtlCopyMemory(&NmiEntry, &Idt[2], sizeof(KIDTENTRY));
    RtlCopyMemory(&DoubleFaultEntry, &Idt[8], sizeof(KIDTENTRY));

    /* Copy kernel's trap handlers */
    RtlCopyMemory(Idt,
                  (PVOID)KiIdtDescriptor.Base,
                  KiIdtDescriptor.Limit + 1);

    /* Restore NMI and double fault */
    RtlCopyMemory(&Idt[2], &NmiEntry, sizeof(KIDTENTRY));
    RtlCopyMemory(&Idt[8], &DoubleFaultEntry, sizeof(KIDTENTRY));

AppCpuInit:
    /* Loop until we can release the freeze lock */
    do
    {
        /* Loop until execution can continue */
        while (*(volatile PKSPIN_LOCK*)&KiFreezeExecutionLock == (PVOID)1);
    } while(InterlockedBitTestAndSet((PLONG)&KiFreezeExecutionLock, 0));

    /* 设置PCR结构体中CPU相关的 成员 Setup CPU-related fields */
    __writefsdword(KPCR_NUMBER, Cpu);
    __writefsdword(KPCR_SET_MEMBER, 1 << Cpu);
    __writefsdword(KPCR_SET_MEMBER_COPY, 1 << Cpu);
    __writefsdword(KPCR_PRCB_SET_MEMBER, 1 << Cpu);

    /* 用HAL初始化处理器 Initialize the Processor with HAL */
    HalInitializeProcessor(Cpu, KeLoaderBlock);

    /* 设置活动处理器 Set active processors */
    KeActiveProcessors |= __readfsdword(KPCR_SET_MEMBER);
    KeNumberProcessors++;

    /* Check if this is the boot CPU */
    if (!Cpu) // 只有引导CPU经历如下功能
    {
        /* 初始化调试子系统，引导CPU Initialize debugging system */
        KdInitSystem(0, KeLoaderBlock);

        /* 检查是否断下来 Check for break-in */
        if (KdPollBreakIn()) DbgBreakPointWithStatus(DBG_STATUS_CONTROL_C);

        /* Make the lowest page of the boot and double fault stack read-only */
        KiMarkPageAsReadOnly(P0BootStackData);
        KiMarkPageAsReadOnly(KiDoubleFaultStackData);
    }

    /* 提升 IRQL 值到 HIGH_LEVEL */
    KeRaiseIrql(HIGH_LEVEL,
                &DummyIrql);

    /* 切换新的内核栈，开始内核引导 */
    KiSwitchToBootStack(InitialStack & ~3);
}
```

对于引导CPU，除了公共的初始化部分外，还需要获取GDT，IDT，TSS等内存指针，并且初始化其中的普通任务TSS，双误异常的TSS和NMI中断的TSS。还要将它们设置仅GDT，以便可以通过选择子获取到。

`KiInitializePcr`对PCR结构体进行初始化，设置其中的Pcr地址，处理器号，当前线程，GDT，IDT，TSS以及DPC栈等。DPC栈使用和`KiDoubleFaultStack`相同的内栈。

设置当前进程为`KiInitialProcess`，清空PCR中的`DR6/DR77`以及TEB字段。

调用`KeInitExceptions()`设置IDT表，并且在之后用于将元IDT表内容复制进新的IDT表中。

非引导CPU要在`AppCpuInit`处等待锁`KiFreezeExecutionLock`解开后才可继续执行。这个锁是在引导CPU初始化了内核之后才会解开。

接下来引导CPU要初始化内核调试子系统，并且确定是否要断下来（即所谓的轮询断入，Windbg中可以设置）。

** 这里插播一段：**

在函数`KdInitSystem()`中会初始化调试，其中会调用`DbgLoadImageSymbols()`函数尝试加载已经加载模块的符号，并且在加载了符号之后调用`nt!DebugService2`中断到调试器。

```
(ntoskrnl\kd64\kdinit.c:74) -----------------------------------------------------
(ntoskrnl\kd64\kdinit.c:75) ReactOS 0.4.12-dev
(ntoskrnl\kd64\kdinit.c:76) 1 System Processor [512 MB Memory]
(ntoskrnl\kd64\kdinit.c:80) Command Line: DEBUG DEBUGPORT=COM1 BAUDRATE=115200 SOS
(ntoskrnl\kd64\kdinit.c:81) ARC Paths: multi(0)disk(0)rdisk(0)partition(1) \ multi(0)disk(0)rdisk(0)partition(1) \ReactOS\
```

> 这里调用的`KdInitSystem()`函数位于`*\ntoskrnl\kd64\kdinit.c`，而并非想象中的`*\ntoskrnl\kd\kdinit.c`，这个应该是因为我在X64的系统上编译的这个系统吧？？（没找具体原因）。

** 从这里往后可以在调试中查看启动过程啦！！！ **

最后调用`KiSwitchToBootStack()`切换到内核栈上（现在仍然在加载器所用的栈上），代码如下所示。切换到了在`KiSystemStartup()`函数中所设置栈`P0BootStack`上。在切换了栈之后，这里还保留了NPX栈帧，陷入框架栈帧的栈空间，然后跳转到`KiSystemStartupBootStack`函数继续执行。

```
VOID KiSwitchToBootStack(IN ULONG_PTR InitialStack)
{
    INIT_FUNCTION VOID NTAPI KiSystemStartupBootStack(VOID);

    /* 在继续内核初始化之前，首先要切换到新的栈上 */
#ifdef __GNUC__
    __asm__
    (
        "movl %0, %%esp\n\t"
        "subl %1, %%esp\n\t"
        "pushl %2\n\t"
        "jmp _KiSystemStartupBootStack@0"
        :
        : "c"(InitialStack),
          "i"(NPX_FRAME_LENGTH + KTRAP_FRAME_ALIGN + KTRAP_FRAME_LENGTH),
          "i"(CR0_EM | CR0_TS | CR0_MP),
          "p"(KiSystemStartupBootStack)
        : "%esp"
    );
#elif defined(_MSC_VER)
    __asm
    {
        mov esp, InitialStack
        sub esp, (NPX_FRAME_LENGTH + KTRAP_FRAME_ALIGN + KTRAP_FRAME_LENGTH)
        push (CR0_EM | CR0_TS | CR0_MP)
        jmp KiSystemStartupBootStack
    }
#else
#error Unknown Compiler
#endif
}
```

函数`KiSystemStartupBootStack()`的代码如下，调用`KiInitializeKernel()`函数为当前的CPU初始化内核，然后设置线程的优先级，开启中断，降低IRQL级别，设置线程的等待IRQL为`DISPATCH_LEVEL`，然后进入`KiIdleLoop()`，即退化为Idle线程。

```
VOID NTAPI KiSystemStartupBootStack(VOID)
{
    PKTHREAD Thread;

    /* 为当前CPU初始化内核 */
    KiInitializeKernel(&KiInitialProcess.Pcb,
                       (PKTHREAD)KeLoaderBlock->Thread,
                       (PVOID)(KeLoaderBlock->KernelStack & ~3),
                       (PKPRCB)__readfsdword(KPCR_PRCB),
                       KeNumberProcessors - 1,
                       KeLoaderBlock);

    /* 设置当前线程的优先级为 0  */
    Thread = KeGetCurrentThread();
    Thread->Priority = 0;

    /* 强制开启中断，降低IRQL到 DISPATCH_LEVEL */
    _enable();
    KeLowerIrql(DISPATCH_LEVEL);

    /* Set the right wait IRQL */
    Thread->WaitIrql = DISPATCH_LEVEL;

    /* Jump into the idle loop */
    KiIdleLoop();
}
```

这里先说一下引导线程进入到Idle线程循环的内容。如下代码所示，获取当前CPU的KPRCB指针，然后进入死循环。在死循环中做这么一些内容：

* 关中断
* 检查是否有挂起的定时器，DPC，如果有则处理它们。（这些DPC都是非紧急的DPC，或低优先级DPC）
* 然后检查当前处理器上是否挂上了待执行线程，如果挂上了，则切换到新准备好线程上继续执行
* 如果没有线程则调用IdleFunction，即`PopIdle0()`函数，该函数中调用`HalProcessorIdle()`，它其实就是halt指令。

> 一旦调用了 halt 指令后，CPU则进入节能状态，能够唤醒它的只有Reset信号，NMI，如果开启了中断标记位，则可屏蔽中断也可以将CPU重新唤醒。进入Halt状态后，通常是中断唤醒的CPU，其中最可能的就是时钟中断。

```
VOID FASTCALL KiIdleLoop(VOID)
{
    PKPRCB Prcb = KeGetCurrentPrcb();
    PKTHREAD OldThread, NewThread;

    /* Now loop forever */
    while (TRUE)
    {
        /* 开始空闲循环：关闭中断 */
        _enable();
        YieldProcessor();
        YieldProcessor();
        _disable();

        /* 检查挂起的定时器，DPC或挂起的准备好线程 */
        if ((Prcb->DpcData[0].DpcQueueDepth) ||
            (Prcb->TimerRequest) ||
            (Prcb->DeferredReadyListHead.Next))
        {
            /* Quiesce the DPC software interrupt */
            HalClearSoftwareInterrupt(DISPATCH_LEVEL);

            /* Handle it */
            KiRetireDpcList(Prcb);
        }

        /* 检查是否一个新的线程被调度可以执行 */
        if (Prcb->NextThread)
        {
            /* 开启中断 */
            _enable();

            /* 获取当前线程数据 */
            OldThread = Prcb->CurrentThread;
            NewThread = Prcb->NextThread;

            /* 设置新线程数据 */
            Prcb->NextThread = NULL;
            Prcb->CurrentThread = NewThread;

            /* 设置线程状态为正在运行 */
            NewThread->State = Running;

            /* 从Idle线程上切换到目标线程中执行 */
            KiSwapContext(APC_LEVEL, OldThread);
        }
        else
        {
            /* 继续停止在空闲状态，注意，从HAL中的停机指令返回时时处于开中断的状态 */
            Prcb->PowerState.IdleFunction(&Prcb->PowerState);
        }
    }
}

VOID FASTCALL PopIdle0(IN PPROCESSOR_POWER_STATE PowerState)
{
    /* FIXME: Extremly naive implementation */
    HalProcessorIdle();
}

VOID NTAPI PoInitializePrcb(IN PKPRCB Prcb)
{
    /* Initialize the Power State */
    RtlZeroMemory(&Prcb->PowerState, sizeof(Prcb->PowerState));
    Prcb->PowerState.Idle0KernelTimeLimit = 0xFFFFFFFF;
    Prcb->PowerState.CurrentThrottle = 100;
    Prcb->PowerState.CurrentThrottleIndex = 0;
    Prcb->PowerState.IdleFunction = PopIdle0;

    /* Initialize the Perf DPC and Timer */
    KeInitializeDpc(&Prcb->PowerState.PerfDpc, PopPerfIdleDpc, Prcb);
    KeSetTargetProcessorDpc(&Prcb->PowerState.PerfDpc, Prcb->Number);
    KeInitializeTimerEx(&Prcb->PowerState.PerfTimer, SynchronizationTimer);
}
```

###初始化内核###

前面看到在切换了栈之后，马上调用函数`KiInitializeKernel()`为CPU初始化内核。

```
VOID NTAPI KiInitializeKernel(IN PKPROCESS InitProcess,
                   IN PKTHREAD InitThread,
                   IN PVOID IdleStack,
                   IN PKPRCB Prcb,
                   IN CCHAR Number,
                   IN PLOADER_PARAMETER_BLOCK LoaderBlock)
{
    BOOLEAN NpxPresent;
    ULONG FeatureBits;
    ULONG PageDirectory[2];
    PVOID DpcStack;
    ULONG Vendor[3];
    KIRQL DummyIrql;

    /* 检测和设置CPU类型 */
    KiSetProcessorType();

    /* 检查是否有FPU存在，如果有则 设置开启 */
    NpxPresent = KiIsNpxPresent();

    /* 初始化PRCB中的能源管理支持，初始化 PRCB中的 PowerState结构体 */
    PoInitializePrcb(Prcb); // Idle线程调用的函数就在此函数中初始化

    /* Bugcheck if this is a 386 CPU */
    if (Prcb->CpuType == 3) KeBugCheckEx(UNSUPPORTED_PROCESSOR, 0x386, 0, 0, 0);

    /* 使用CPU获取当前CPU支持的功能，并转换为系统的功能位 */
    FeatureBits = KiGetFeatureBits();

    /* 设置默认的 NX 策略 (opt-in) */
    SharedUserData->NXSupportPolicy = NX_SUPPORT_POLICY_OPTIN;

    /* Check if NPX is always on */
    if (strstr(KeLoaderBlock->LoadOptions, "NOEXECUTE=ALWAYSON"))
    {
        /* Set it always on */
        SharedUserData->NXSupportPolicy = NX_SUPPORT_POLICY_ALWAYSON;
        FeatureBits |= KF_NX_ENABLED;
    }
    else if (strstr(KeLoaderBlock->LoadOptions, "NOEXECUTE=OPTOUT"))
    {
        /* Set it in opt-out mode */
        SharedUserData->NXSupportPolicy = NX_SUPPORT_POLICY_OPTOUT;
        FeatureBits |= KF_NX_ENABLED;
    }
    else if ((strstr(KeLoaderBlock->LoadOptions, "NOEXECUTE=OPTIN")) ||
             (strstr(KeLoaderBlock->LoadOptions, "NOEXECUTE")))
    {
        /* Set the feature bits */
        FeatureBits |= KF_NX_ENABLED;
    }
    else if ((strstr(KeLoaderBlock->LoadOptions, "NOEXECUTE=ALWAYSOFF")) ||
             (strstr(KeLoaderBlock->LoadOptions, "EXECUTE")))
    {
        /* Set disabled mode */
        SharedUserData->NXSupportPolicy = NX_SUPPORT_POLICY_ALWAYSOFF;
        FeatureBits |= KF_NX_DISABLED;
    }

    /* Save feature bits */
    Prcb->FeatureBits = FeatureBits;

    /* 保存当前 CPU 的状态，保存到 ProcessorState 中 */
    KiSaveProcessorControlState(&Prcb->ProcessorState);

    /* 获取CPU的 一级，二级缓存信息 */
    KiGetCacheInformation();

    /* 初始化自旋锁和 DPC 数据*/
    KiInitSpinLocks(Prcb, Number);

    /* 检查是否引导CPU */
    if (!Number)
    {
        /* Set Node Data */
        KeNodeBlock[0] = &KiNode0;
        Prcb->ParentNode = KeNodeBlock[0];
        KeNodeBlock[0]->ProcessorMask = Prcb->SetMember;

        /* 设置 引导级别 标记 Set boot-level flags */
        KeI386NpxPresent = NpxPresent;
        KeI386CpuType = Prcb->CpuType;
        KeI386CpuStep = Prcb->CpuStep;
        KeProcessorArchitecture = PROCESSOR_ARCHITECTURE_INTEL;
        KeProcessorLevel = (USHORT)Prcb->CpuType;
        if (Prcb->CpuID) KeProcessorRevision = Prcb->CpuStep;
        KeFeatureBits = FeatureBits;
        KeI386FxsrPresent = (KeFeatureBits & KF_FXSR) ? TRUE : FALSE;
        KeI386XMMIPresent = (KeFeatureBits & KF_XMMI) ? TRUE : FALSE;

        /* 检查 8字节的 对比交换指令支持 Detect 8-byte compare exchange support */
        if (!(KeFeatureBits & KF_CMPXCHG8B))
        {
            /* Copy the vendor string */
            RtlCopyMemory(Vendor, Prcb->VendorString, sizeof(Vendor));

            /* Bugcheck the system. Windows *requires* this */
            KeBugCheckEx(UNSUPPORTED_PROCESSOR,
                         (1 << 24 ) | (Prcb->CpuType << 16) | Prcb->CpuStep,
                         Vendor[0],
                         Vendor[1],
                         Vendor[2]);
        }

        /* Set the current MP Master KPRCB to the Boot PRCB */
        Prcb->MultiThreadSetMaster = Prcb;

        /* Lower to APC_LEVEL */
        KeLowerIrql(APC_LEVEL);

        /* 初始化 自旋锁，这里开启其他CPU的执行 Initialize some spinlocks */
        KeInitializeSpinLock(&KiFreezeExecutionLock);
        KeInitializeSpinLock(&Ki486CompatibilityLock);

        /* 初始化系统可移植部分 Initialize portable parts of the OS */
        KiInitSystem();

        /* 初始化空闲进程和进程链表头 Initialize the Idle Process and the Process Listhead */
        InitializeListHead(&KiProcessListHead);
        PageDirectory[0] = 0;
        PageDirectory[1] = 0;
        KeInitializeProcess(InitProcess,
                            0,
                            0xFFFFFFFF,
                            PageDirectory,
                            FALSE);
        InitProcess->QuantumReset = MAXCHAR;
    }
    else
    {
        /* FIXME */
        DPRINT1("SMP Boot support not yet present\n");
    }

    /* 配置 空闲线程 Setup the Idle Thread */
    KeInitializeThread(InitProcess,
                       InitThread,
                       NULL,
                       NULL,
                       NULL,
                       NULL,
                       NULL,
                       IdleStack);
    InitThread->NextProcessor = Number;
    InitThread->Priority = HIGH_PRIORITY;
    InitThread->State = Running;
    InitThread->Affinity = 1 << Number;
    InitThread->WaitIrql = DISPATCH_LEVEL;
    InitProcess->ActiveProcessors = 1 << Number;

    /* HACK for MmUpdatePageDir */
    ((PETHREAD)InitThread)->ThreadsProcess = (PEPROCESS)InitProcess;

    /* 设置用户模式可读的 CPU功能  Set basic CPU Features that user mode can read */
    SharedUserData->ProcessorFeatures[PF_MMX_INSTRUCTIONS_AVAILABLE] =
        (KeFeatureBits & KF_MMX) ? TRUE: FALSE;
    SharedUserData->ProcessorFeatures[PF_COMPARE_EXCHANGE_DOUBLE] =
        (KeFeatureBits & KF_CMPXCHG8B) ? TRUE: FALSE;
    SharedUserData->ProcessorFeatures[PF_XMMI_INSTRUCTIONS_AVAILABLE] =
        ((KeFeatureBits & KF_FXSR) && (KeFeatureBits & KF_XMMI)) ? TRUE: FALSE;
    SharedUserData->ProcessorFeatures[PF_XMMI64_INSTRUCTIONS_AVAILABLE] =
        ((KeFeatureBits & KF_FXSR) && (KeFeatureBits & KF_XMMI64)) ? TRUE: FALSE;
    SharedUserData->ProcessorFeatures[PF_3DNOW_INSTRUCTIONS_AVAILABLE] =
        (KeFeatureBits & KF_3DNOW) ? TRUE: FALSE;
    SharedUserData->ProcessorFeatures[PF_RDTSC_INSTRUCTION_AVAILABLE] =
        (KeFeatureBits & KF_RDTSC) ? TRUE: FALSE;

    /* 限制 PRCB中线程相关的域 Set up the thread-related fields in the PRCB */
    Prcb->CurrentThread = InitThread;
    Prcb->NextThread = NULL;
    Prcb->IdleThread = InitThread;

    /* 初始化内核执行体 Initialize the Kernel Executive */
    ExpInitializeExecutive(Number, LoaderBlock);

    /* Only do this on the boot CPU */
    if (!Number)
    {
        /* Calculate the time reciprocal */
        KiTimeIncrementReciprocal =
            KiComputeReciprocal(KeMaximumIncrement,
                                &KiTimeIncrementShiftCount);

        /* Update DPC Values in case they got updated by the executive */
        Prcb->MaximumDpcQueueDepth = KiMaximumDpcQueueDepth;
        Prcb->MinimumDpcRate = KiMinimumDpcRate;
        Prcb->AdjustDpcThreshold = KiAdjustDpcThreshold;

        /* Allocate the DPC Stack */
        DpcStack = MmCreateKernelStack(FALSE, 0);
        if (!DpcStack) KeBugCheckEx(NO_PAGES_AVAILABLE, 1, 0, 0, 0);
        Prcb->DpcStack = DpcStack;

        /* Allocate the IOPM save area. */
        Ki386IopmSaveArea = ExAllocatePoolWithTag(PagedPool,
                                                  PAGE_SIZE * 2,
                                                  '  eK');
        if (!Ki386IopmSaveArea)
        {
            /* Bugcheck. We need this for V86/VDM support. */
            KeBugCheckEx(NO_PAGES_AVAILABLE, 2, PAGE_SIZE * 2, 0, 0);
        }
    }

    /* Raise to Dispatch */
    KeRaiseIrql(DISPATCH_LEVEL,
                &DummyIrql);

    /* Set the Idle Priority to 0. This will jump into Phase 1 */
    KeSetPriorityThread(InitThread, 0);

    /* If there's no thread scheduled, put this CPU in the Idle summary */
    KiAcquirePrcbLock(Prcb);
    if (!Prcb->NextThread) KiIdleSummary |= 1 << Number;
    KiReleasePrcbLock(Prcb);

    /* Raise back to HIGH_LEVEL and clear the PRCB for the loader block */
    KeRaiseIrql(HIGH_LEVEL,
                &DummyIrql);
    LoaderBlock->Prcb = 0;
}
```

函数中，首先初始化PRCB中CPU相关的内容，其次是PRCB中DPC相关的内容，以及一些用到的自旋锁与互斥量等。调用`KiInitSystem()`函数来初始化OS中可移植部分的内容，其实还是 PRCB 中的内容，代码如下面代码块所示；系统调用表就是从该函数中进行的初始化。

如果是引导CPU，则初始化空闲进程。将当前线程进行进一步初始化，然后挂到空闲进程中去。调用`ExpInitializeExecutive()`函数初始化执行体，这个函数是下一阶段初始化的中断，放到后面说。

如果是引导CPU进一步，分配一个DPC栈，分配IOPM保存区等。

最后将IRQL升到`DISPATCH_LEVEL`，那么就可以执行在执行体初始化时挂入当前CPU的DPC。最后将IRQL升到`HIGH_LEVEL`，返回上一级函数后，随后即进入到空闲先线程的逻辑。

```
VOID
NTAPI
KiInitSystem(VOID)
{
    ULONG i;

    /* Initialize Bugcheck Callback data */
    InitializeListHead(&KeBugcheckCallbackListHead);
    InitializeListHead(&KeBugcheckReasonCallbackListHead);
    KeInitializeSpinLock(&BugCheckCallbackLock);

    /* Initialize the Timer Expiration DPC */
    KeInitializeDpc(&KiTimerExpireDpc, KiTimerExpiration, NULL);
    KeSetTargetProcessorDpc(&KiTimerExpireDpc, 0);

    /* Initialize Profiling data */
    KeInitializeSpinLock(&KiProfileLock);
    InitializeListHead(&KiProfileListHead);
    InitializeListHead(&KiProfileSourceListHead);

    /* Loop the timer table */
    for (i = 0; i < TIMER_TABLE_SIZE; i++)
    {
        /* Initialize the list and entries */
        InitializeListHead(&KiTimerTableListHead[i].Entry);
        KiTimerTableListHead[i].Time.HighPart = 0xFFFFFFFF;
        KiTimerTableListHead[i].Time.LowPart = 0;
    }

    /* Initialize the Swap event and all swap lists */
    KeInitializeEvent(&KiSwapEvent, SynchronizationEvent, FALSE);
    InitializeListHead(&KiProcessInSwapListHead);
    InitializeListHead(&KiProcessOutSwapListHead);
    InitializeListHead(&KiStackInSwapListHead);

    /* Initialize the mutex for generic DPC calls */
    ExInitializeFastMutex(&KiGenericCallDpcMutex);

    /* Initialize the syscall table */
    KeServiceDescriptorTable[0].Base = MainSSDT;
    KeServiceDescriptorTable[0].Count = NULL;
    KeServiceDescriptorTable[0].Limit = KiServiceLimit;
    KeServiceDescriptorTable[1].Limit = 0;
    KeServiceDescriptorTable[0].Number = MainSSPT;

    /* Copy the the current table into the shadow table for win32k */
    RtlCopyMemory(KeServiceDescriptorTableShadow,
                  KeServiceDescriptorTable,
                  sizeof(KeServiceDescriptorTable));
}
```

###阶段0初始化###

上一节的函数中会调用`ExpInitializeExecutive()`来对执行体进行初始化，其代码如下所示。

首先检查加载数据块，校验其版本与大小，初始化旁路查找的指针。

这里有一个特殊的路径，对于非引导CPU，则只需要调用`HalInitSystem()`在阶段0进行Hal的CPU初始化即可，这里先略过。

下面设置全局变量`ExpInitializationPhase`为0，表明进入执行体的阶段0初始化。阶段0主要做如下的几个工作：

* 加载 NLS 数据，并进行初始化
* 调用HalInitSystem()进行Hal阶段0初始化，主要是ACPI初始化，Cmos初始化，初始化时钟等。
* 开中断
* 填充NT系统根路径。
* 调用`CmGetSystemControlValues()`进行注册表初始化。
* 调用`ExInitSystem()`进行执行体阶段0初始化，主要是一些列表以及锁的初始化。
* 调用`MmArmInitSystem()`进行内存管理的阶段0初始化。
* 调用`ExpLoadBootSymbols()`加载已经加载模块的符号，如果符号加载则尝试断下来。
* 调用`ExpInitNls()`函数对NLS表进行拷贝
* 检查系统模块版本号，构建版本号字符串
* 调用`ExpInitializeHandleTables()`初始化句柄表和锁
* 调用`ObInitSystem()`进行对象管理器阶段0初始化，初始化引导进程的句柄表，构建基础类型对象，Type，Directory等
* 调用`SeInitSystem()`进行安全相关的阶段0初始化，主要是会话ID，用户ID，令牌等构建
* 调用`PsInitSystem()`进行进程管理的阶段0初始化，创建进程，线程对象的类型对象，创建系统进程
* 调用`PpInitSystem()`进行即插即用管理器初始化
* 最后调用`DbgkInitialize()`创建调试端口对象的类型对象。

```
VOID NTAPI
ExpInitializeExecutive(IN ULONG Cpu,
                       IN PLOADER_PARAMETER_BLOCK LoaderBlock)
{
    PNLS_DATA_BLOCK NlsData;
    CHAR Buffer[256];
    ANSI_STRING AnsiPath;
    NTSTATUS Status;
    PCHAR CommandLine, PerfMem;
    ULONG PerfMemUsed;
    PLDR_DATA_TABLE_ENTRY NtosEntry;
    PMESSAGE_RESOURCE_ENTRY MsgEntry;
    ANSI_STRING CSDString;
    size_t Remaining = 0;
    PCHAR RcEnd = NULL;
    CHAR VersionBuffer[65];

    /* 首先验证 加载块，版本与大小不对即蓝屏 Validate Loader */
    if (!ExpIsLoaderValid(LoaderBlock))
    {
        /* Invalid loader version */
        KeBugCheckEx(MISMATCHED_HAL,
                     3,
                     LoaderBlock->Extension->Size,
                     LoaderBlock->Extension->MajorVersion,
                     LoaderBlock->Extension->MinorVersion);
    }

    /* 初始化 PRCB 池的旁路查找 指针 pool lookaside pointers */
    ExInitPoolLookasidePointers();

    /* 检查是否非引导 CPU Check if this is an application CPU */
    if (Cpu)
    {
        /* 非引导CPU执行到这里完成CPU的HAL初始化即可Then simply initialize it with HAL */
        if (!HalInitSystem(ExpInitializationPhase, LoaderBlock))
        {
            /* Initialization failed */
            KeBugCheck(HAL_INITIALIZATION_FAILED);
        }

        /* We're done */
        return;
    }

    /* Assume no text-mode or remote boot */
    ExpInTextModeSetup = FALSE;
    IoRemoteBootClient = FALSE;

    /* Check if we have a setup loader block */
    if (LoaderBlock->SetupLdrBlock)
    {
        /* Check if this is text-mode setup */
        if (LoaderBlock->SetupLdrBlock->Flags & SETUPLDR_TEXT_MODE)
            ExpInTextModeSetup = TRUE;

        /* Check if this is network boot */
        if (LoaderBlock->SetupLdrBlock->Flags & SETUPLDR_REMOTE_BOOT)
        {
            /* Set variable */
            IoRemoteBootClient = TRUE;

            /* Make sure we're actually booting off the network */
            ASSERT(!_memicmp(LoaderBlock->ArcBootDeviceName, "net(0)", 6));
        }
    }

    /* Set phase to 0 */
    ExpInitializationPhase = 0;

    /* Get boot command line */
    CommandLine = LoaderBlock->LoadOptions;
    if (CommandLine)
    {
        /* Upcase it for comparison and check if we're in performance mode */
        _strupr(CommandLine);
        PerfMem = strstr(CommandLine, "PERFMEM");
        if (PerfMem)
        {
            /* Check if the user gave a number of bytes to use */
            PerfMem = strstr(PerfMem, "=");
            if (PerfMem)
            {
                /* Read the number of pages we'll use */
                PerfMemUsed = atol(PerfMem + 1) * (1024 * 1024 / PAGE_SIZE);
                if (PerfMem)
                {
                    /* FIXME: TODO */
                    DPRINT1("BBT performance mode not yet supported."
                            "/PERFMEM option ignored.\n");
                }
            }
        }

        /* Check if we're burning memory */
        PerfMem = strstr(CommandLine, "BURNMEMORY");
        if (PerfMem)
        {
            /* Check if the user gave a number of bytes to use */
            PerfMem = strstr(PerfMem, "=");
            if (PerfMem)
            {
                /* Read the number of pages we'll use */
                PerfMemUsed = atol(PerfMem + 1) * (1024 * 1024 / PAGE_SIZE);
                if (PerfMemUsed) ExBurnMemory(LoaderBlock, PerfMemUsed, LoaderBad);
            }
        }
    }

    /* Setup NLS Base and offsets */
    NlsData = LoaderBlock->NlsData;
    ExpNlsTableBase = NlsData->AnsiCodePageData;
    ExpAnsiCodePageDataOffset = 0;
    ExpOemCodePageDataOffset = (ULONG)((ULONG_PTR)NlsData->OemCodePageData -
                                       (ULONG_PTR)NlsData->AnsiCodePageData);
    ExpUnicodeCaseTableDataOffset = (ULONG)((ULONG_PTR)NlsData->UnicodeCodePageData -
                                            (ULONG_PTR)NlsData->AnsiCodePageData);

    /* Initialize the NLS Tables */
    RtlInitNlsTables((PVOID)((ULONG_PTR)ExpNlsTableBase +
                             ExpAnsiCodePageDataOffset),
                     (PVOID)((ULONG_PTR)ExpNlsTableBase +
                             ExpOemCodePageDataOffset),
                     (PVOID)((ULONG_PTR)ExpNlsTableBase +
                             ExpUnicodeCaseTableDataOffset),
                     &ExpNlsTableInfo);
    RtlResetRtlTranslations(&ExpNlsTableInfo);

    /* Now initialize the HAL */
    if (!HalInitSystem(ExpInitializationPhase, LoaderBlock))
    {
        /* HAL failed to initialize, bugcheck */
        KeBugCheck(HAL_INITIALIZATION_FAILED);
    }

    /* Make sure interrupts are active now */
    _enable();

    /* Clear the crypto exponent */
    SharedUserData->CryptoExponent = 0;

    /* Set global flags for the checked build */
#if DBG
    NtGlobalFlag |= FLG_ENABLE_CLOSE_EXCEPTIONS |
                    FLG_ENABLE_KDEBUG_SYMBOL_LOAD;
#endif

    /* Setup NT System Root Path */
    sprintf(Buffer, "C:%s", LoaderBlock->NtBootPathName);

    /* Convert to ANSI_STRING and null-terminate it */
    RtlInitString(&AnsiPath, Buffer);
    Buffer[--AnsiPath.Length] = ANSI_NULL;

    /* Get the string from KUSER_SHARED_DATA's buffer */
    RtlInitEmptyUnicodeString(&NtSystemRoot,
                              SharedUserData->NtSystemRoot,
                              sizeof(SharedUserData->NtSystemRoot));

    /* Now fill it in */
    Status = RtlAnsiStringToUnicodeString(&NtSystemRoot, &AnsiPath, FALSE);
    if (!NT_SUCCESS(Status)) KeBugCheck(SESSION3_INITIALIZATION_FAILED);

    /* Setup bugcheck messages */
    KiInitializeBugCheck();

    /* Setup initial system settings */
    CmGetSystemControlValues(LoaderBlock->RegistryBase, CmControlVector);

    /* Set the Service Pack Number and add it to the CSD Version number if needed */
    CmNtSpBuildNumber = VER_PRODUCTBUILD_QFE;
    if (((CmNtCSDVersion & 0xFFFF0000) == 0) && (CmNtCSDReleaseType == 1))
    {
        CmNtCSDVersion |= (VER_PRODUCTBUILD_QFE << 16);
    }

    /* Add loaded CmNtGlobalFlag value */
    NtGlobalFlag |= CmNtGlobalFlag;

    /* Initialize the executive at phase 0 */
    if (!ExInitSystem()) KeBugCheck(PHASE0_INITIALIZATION_FAILED);

    /* Initialize the memory manager at phase 0 */
    if (!MmArmInitSystem(0, LoaderBlock)) KeBugCheck(PHASE0_INITIALIZATION_FAILED);

    /* Load boot symbols */
    ExpLoadBootSymbols(LoaderBlock);

    /* Check if we should break after symbol load */
    if (KdBreakAfterSymbolLoad) DbgBreakPointWithStatus(DBG_STATUS_CONTROL_C);

    /* Check if this loader is compatible with NT 5.2 */
    if (LoaderBlock->Extension->Size >= sizeof(LOADER_PARAMETER_EXTENSION))
    {
        /* Setup headless terminal settings */
        HeadlessInit(LoaderBlock);
    }

    /* Set system ranges */
#ifdef _M_AMD64
    SharedUserData->Reserved1 = MM_HIGHEST_USER_ADDRESS_WOW64;
    SharedUserData->Reserved3 = MM_SYSTEM_RANGE_START_WOW64;
#else
    SharedUserData->Reserved1 = (ULONG_PTR)MmHighestUserAddress;
    SharedUserData->Reserved3 = (ULONG_PTR)MmSystemRangeStart;
#endif

    /* Make a copy of the NLS Tables */
    ExpInitNls(LoaderBlock);

    /* Get the kernel's load entry */
    NtosEntry = CONTAINING_RECORD(LoaderBlock->LoadOrderListHead.Flink,
                                  LDR_DATA_TABLE_ENTRY,
                                  InLoadOrderLinks);

    /* Check if this is a service pack */
    if (CmNtCSDVersion & 0xFFFF)
    {
        /* Get the service pack string */
        Status = RtlFindMessage(NtosEntry->DllBase,
                                11,
                                0,
                                WINDOWS_NT_CSD_STRING,
                                &MsgEntry);
        if (NT_SUCCESS(Status))
        {
            /* Setup the string */
            RtlInitAnsiString(&CSDString, (PCHAR)MsgEntry->Text);

            /* Remove trailing newline */
            while ((CSDString.Length > 0) &&
                   ((CSDString.Buffer[CSDString.Length - 1] == '\r') ||
                    (CSDString.Buffer[CSDString.Length - 1] == '\n')))
            {
                /* Skip the trailing character */
                CSDString.Length--;
            }

            /* Fill the buffer with version information */
            Status = RtlStringCbPrintfA(Buffer,
                                        sizeof(Buffer),
                                        "%Z %u%c",
                                        &CSDString,
                                        (CmNtCSDVersion & 0xFF00) >> 8,
                                        (CmNtCSDVersion & 0xFF) ?
                                        'A' + (CmNtCSDVersion & 0xFF) - 1 :
                                        ANSI_NULL);
        }
        else
        {
            /* Build default string */
            Status = RtlStringCbPrintfA(Buffer,
                                        sizeof(Buffer),
                                        "CSD %04x",
                                        CmNtCSDVersion);
        }

        /* Check for success */
        if (!NT_SUCCESS(Status))
        {
            /* Fail */
            KeBugCheckEx(PHASE0_INITIALIZATION_FAILED, Status, 0, 0, 0);
        }
    }
    else
    {
        /* Then this is a beta */
        Status = RtlStringCbCopyExA(Buffer,
                                    sizeof(Buffer),
                                    VER_PRODUCTBETA_STR,
                                    NULL,
                                    &Remaining,
                                    0);
        if (!NT_SUCCESS(Status))
        {
            /* Fail */
            KeBugCheckEx(PHASE0_INITIALIZATION_FAILED, Status, 0, 0, 0);
        }

        /* Update length */
        CmCSDVersionString.MaximumLength = sizeof(Buffer) - (USHORT)Remaining;
    }

    /* Check if we have an RC number */
    if ((CmNtCSDVersion & 0xFFFF0000) && (CmNtCSDReleaseType == 1))
    {
        /* Check if we have no version data yet */
        if (!(*Buffer))
        {
            /* Set defaults */
            Remaining = sizeof(Buffer);
            RcEnd = Buffer;
        }
        else
        {
            /* Add comma and space */
            Status = RtlStringCbCatExA(Buffer,
                                       sizeof(Buffer),
                                       ", ",
                                       &RcEnd,
                                       &Remaining,
                                       0);
            if (!NT_SUCCESS(Status))
            {
                /* Fail */
                KeBugCheckEx(PHASE0_INITIALIZATION_FAILED, Status, 0, 0, 0);
            }
        }

        /* Add the version format string */
        Status = RtlStringCbPrintfA(RcEnd,
                                    Remaining,
                                    "v.%u",
                                    (CmNtCSDVersion & 0xFFFF0000) >> 16);
        if (!NT_SUCCESS(Status))
        {
            /* Fail */
            KeBugCheckEx(PHASE0_INITIALIZATION_FAILED, Status, 0, 0, 0);
        }
    }

    /* Now setup the final string */
    RtlInitAnsiString(&CSDString, Buffer);
    Status = RtlAnsiStringToUnicodeString(&CmCSDVersionString,
                                          &CSDString,
                                          TRUE);
    if (!NT_SUCCESS(Status))
    {
        /* Fail */
        KeBugCheckEx(PHASE0_INITIALIZATION_FAILED, Status, 0, 0, 0);
    }

    /* Add our version */
    Status = RtlStringCbPrintfA(VersionBuffer,
                                sizeof(VersionBuffer),
                                "%u.%u",
                                VER_PRODUCTMAJORVERSION,
                                VER_PRODUCTMINORVERSION);
    if (!NT_SUCCESS(Status))
    {
        /* Fail */
        KeBugCheckEx(PHASE0_INITIALIZATION_FAILED, Status, 0, 0, 0);
    }

    /* Build the final version string */
    RtlCreateUnicodeStringFromAsciiz(&CmVersionString, VersionBuffer);

    /* Check if the user wants a kernel stack trace database */
    if (NtGlobalFlag & FLG_KERNEL_STACK_TRACE_DB)
    {
        /* FIXME: TODO */
        DPRINT1("Kernel-mode stack trace support not yet present."
                "FLG_KERNEL_STACK_TRACE_DB flag ignored.\n");
    }

    /* Check if he wanted exception logging */
    if (NtGlobalFlag & FLG_ENABLE_EXCEPTION_LOGGING)
    {
        /* FIXME: TODO */
        DPRINT1("Kernel-mode exception logging support not yet present."
                "FLG_ENABLE_EXCEPTION_LOGGING flag ignored.\n");
    }

    /* Initialize the Handle Table */
    ExpInitializeHandleTables();

#if DBG
    /* On checked builds, allocate the system call count table */
    KeServiceDescriptorTable[0].Count =
        ExAllocatePoolWithTag(NonPagedPool,
                              KiServiceLimit * sizeof(ULONG),
                              'llaC');

    /* Use it for the shadow table too */
    KeServiceDescriptorTableShadow[0].Count = KeServiceDescriptorTable[0].Count;

    /* Make sure allocation succeeded */
    if (KeServiceDescriptorTable[0].Count)
    {
        /* Zero the call counts to 0 */
        RtlZeroMemory(KeServiceDescriptorTable[0].Count,
                      KiServiceLimit * sizeof(ULONG));
    }
#endif

    /* Create the Basic Object Manager Types to allow new Object Types */
    if (!ObInitSystem()) KeBugCheck(OBJECT_INITIALIZATION_FAILED);

    /* Load basic Security for other Managers */
    if (!SeInitSystem()) KeBugCheck(SECURITY_INITIALIZATION_FAILED);

    /* Initialize the Process Manager */
    if (!PsInitSystem(LoaderBlock)) KeBugCheck(PROCESS_INITIALIZATION_FAILED);

    /* Initialize the PnP Manager */
    if (!PpInitSystem()) KeBugCheck(PP0_INITIALIZATION_FAILED);

    /* Initialize the User-Mode Debugging Subsystem */
    DbgkInitialize();

    /* Calculate the tick count multiplier */
    ExpTickCountMultiplier = ExComputeTickCountMultiplier(KeMaximumIncrement);
    SharedUserData->TickCountMultiplier = ExpTickCountMultiplier;

    /* Set the OS Version */
    SharedUserData->NtMajorVersion = NtMajorVersion;
    SharedUserData->NtMinorVersion = NtMinorVersion;

    /* Set the machine type */
    SharedUserData->ImageNumberLow = IMAGE_FILE_MACHINE_NATIVE;
    SharedUserData->ImageNumberHigh = IMAGE_FILE_MACHINE_NATIVE;
}
```

由于阶段0涉及的东西较多，这里只是简单列举一下过程，到了需要详细研究再仔细分析对应部分。

**未总结内容**

1. 无

**参考文章**

* WoW64 Wiki	[https://zh.wikipedia.org/wiki/WoW64](https://zh.wikipedia.org/wiki/WoW64)

**修订历史**

* 2017-09-04 11:21:23		完成文章

By Andy @2017-09-04 11:21:23