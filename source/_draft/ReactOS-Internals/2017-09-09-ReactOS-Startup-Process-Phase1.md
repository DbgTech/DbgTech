---
title: ReactOS启动过程
date: 2017-09-03 20:52:23
tags:
- Windbg
- ReactOS
- 启动过程
categories:
- ReactOS
---


在初始化阶段0初始化完进程和线程管理后，会创建`System`进程及其线程，在线程中执行阶段1的初始化。

System进程中线程启动后执行的函数为`Phase1Initialization()`，它完成内核初始化的阶段1。

```
VOID NTAPI Phase1Initialization(IN PVOID Context)
{
    /* Do the .INIT part of Phase 1 which we can free later */
    Phase1InitializationDiscard(Context);

    /* Jump into zero page thread */
    MmZeroPageThread();
}
```

从源码中可以看到，该线程首先调用了`Phase1InitializationDiscard()`函数，完成后调用` MmZeroPageThread()`，使得它退化为内存页面清空的线程。

这里先看一下这个清空内存的线程函数，在初始化完后，首先将已经加载模块中可以抹去的内存抹掉。然后进入死循环，等待事件`MmZeroingPageEvent`，如有有信号，则遍历`MmFreePageListHead`链表，将其中的内存进行清空处理。

```
VOID NTAPI MmZeroPageThread(VOID)
{
    PKTHREAD Thread = KeGetCurrentThread();
    PVOID StartAddress, EndAddress;
    PVOID WaitObjects[2];
    KIRQL OldIrql;
    PVOID ZeroAddress;
    PFN_NUMBER PageIndex, FreePage;
    PMMPFN Pfn1;

    /* Get the discardable sections to free them */
    MiFindInitializationCode(&StartAddress, &EndAddress);
    if (StartAddress) MiFreeInitializationCode(StartAddress, EndAddress);
    DPRINT("Free non-cache pages: %lx\n", MmAvailablePages + MiMemoryConsumers[MC_CACHE].PagesUsed);

    /* Set our priority to 0 */
    Thread->BasePriority = 0;
    KeSetPriorityThread(Thread, 0);

    /* Setup the wait objects */
    WaitObjects[0] = &MmZeroingPageEvent;
//    WaitObjects[1] = &PoSystemIdleTimer; FIXME: Implement idle timer

    while (TRUE)
    {
        KeWaitForMultipleObjects(1, // 2
                                 WaitObjects,
                                 WaitAny,
                                 WrFreePage,
                                 KernelMode,
                                 FALSE,
                                 NULL,
                                 NULL);
        OldIrql = MiAcquirePfnLock();
        MmZeroingPageThreadActive = TRUE;

        while (TRUE)
        {
            if (!MmFreePageListHead.Total)
            {
                MmZeroingPageThreadActive = FALSE;
                MiReleasePfnLock(OldIrql);
                break;
            }

            PageIndex = MmFreePageListHead.Flink;
            ASSERT(PageIndex != LIST_HEAD);
            Pfn1 = MiGetPfnEntry(PageIndex);
            MI_SET_USAGE(MI_USAGE_ZERO_LOOP);
            MI_SET_PROCESS2("Kernel 0 Loop");
            FreePage = MiRemoveAnyPage(MI_GET_PAGE_COLOR(PageIndex));

            /* The first global free page should also be the first on its own list */
            if (FreePage != PageIndex)
            {
                KeBugCheckEx(PFN_LIST_CORRUPT,
                             0x8F,
                             FreePage,
                             PageIndex,
                             0);
            }

            Pfn1->u1.Flink = LIST_HEAD;
            MiReleasePfnLock(OldIrql);

            ZeroAddress = MiMapPagesInZeroSpace(Pfn1, 1);
            ASSERT(ZeroAddress);
            RtlZeroMemory(ZeroAddress, PAGE_SIZE);
            MiUnmapPagesInZeroSpace(ZeroAddress, 1);

            OldIrql = MiAcquirePfnLock();

            MiInsertPageInList(&MmZeroedPageListHead, PageIndex);
        }
    }
}
```

### 阶段1初始化 ###

阶段1的初始化主要完成的是驱动的`.INIT`部分的调用，然后在后面的清空内存线程中就可以将该部分所占内存释放。调用`Phase1InitializationDiscard()`函数完成阶段1初始化，源码如下所示：

```
VOID NTAPI Phase1InitializationDiscard(IN PVOID Context)
{
    PLOADER_PARAMETER_BLOCK LoaderBlock = Context;
    NTSTATUS Status, MsgStatus;
    TIME_FIELDS TimeFields;
    LARGE_INTEGER SystemBootTime, UniversalBootTime, OldTime, Timeout;
    BOOLEAN SosEnabled, NoGuiBoot, ResetBias = FALSE, AlternateShell = FALSE;
    PLDR_DATA_TABLE_ENTRY NtosEntry;
    PMESSAGE_RESOURCE_ENTRY MsgEntry;
    PCHAR CommandLine, Y2KHackRequired, SafeBoot, Environment;
    PCHAR StringBuffer, EndBuffer, BeginBuffer, MpString = "";
    PINIT_BUFFER InitBuffer;
    ANSI_STRING TempString;
    ULONG LastTzBias, Length, YearHack = 0, Disposition, MessageCode = 0;
    SIZE_T Size;
    size_t Remaining;
    PRTL_USER_PROCESS_INFORMATION ProcessInfo;
    KEY_VALUE_PARTIAL_INFORMATION KeyPartialInfo;
    UNICODE_STRING KeyName;
    OBJECT_ATTRIBUTES ObjectAttributes;
    HANDLE KeyHandle, OptionHandle;
    PRTL_USER_PROCESS_PARAMETERS ProcessParameters = NULL;

    /* Allocate the initialization buffer */
    InitBuffer = ExAllocatePoolWithTag(NonPagedPool,
                                       sizeof(INIT_BUFFER),
                                       TAG_INIT);
    if (!InitBuffer)
    {
        /* Bugcheck */
        KeBugCheckEx(PHASE1_INITIALIZATION_FAILED, STATUS_NO_MEMORY, 8, 0, 0);
    }

    /* Set to phase 1 */
    ExpInitializationPhase = 1;

    /* Set us at maximum priority */
    KeSetPriorityThread(KeGetCurrentThread(), HIGH_PRIORITY);

    /* Do Phase 1 HAL Initialization */
    if (!HalInitSystem(1, LoaderBlock)) KeBugCheck(HAL1_INITIALIZATION_FAILED);

    /* Get the command line and upcase it */
    CommandLine = (LoaderBlock->LoadOptions ? _strupr(LoaderBlock->LoadOptions) : NULL);

    /* Check if GUI Boot is enabled */
    NoGuiBoot = (CommandLine && strstr(CommandLine, "NOGUIBOOT") != NULL);

    /* Get the SOS setting */
    SosEnabled = (CommandLine && strstr(CommandLine, "SOS") != NULL);

    /* Setup the boot driver */
    InbvEnableBootDriver(!NoGuiBoot);
    InbvDriverInitialize(LoaderBlock, IDB_MAX_RESOURCE);

    /* Check if GUI boot is enabled */
    if (!NoGuiBoot)
    {
        /* It is, display the boot logo and enable printing strings */
        InbvEnableDisplayString(SosEnabled);
        DisplayBootBitmap(SosEnabled);
    }
    else
    {
        /* Release display ownership if not using GUI boot */
        InbvNotifyDisplayOwnershipLost(NULL);

        /* Don't allow boot-time strings */
        InbvEnableDisplayString(FALSE);
    }

    /* Check if this is LiveCD (WinPE) mode */
    if (CommandLine && strstr(CommandLine, "MININT") != NULL)
    {
        /* Setup WinPE Settings */
        InitIsWinPEMode = TRUE;
        InitWinPEModeType |= (strstr(CommandLine, "INRAM") != NULL) ? 0x80000000 : 0x00000001;
    }

    /* Get the kernel's load entry */
    NtosEntry = CONTAINING_RECORD(LoaderBlock->LoadOrderListHead.Flink,
                                  LDR_DATA_TABLE_ENTRY,
                                  InLoadOrderLinks);

    /* Find the banner message */
    MsgStatus = RtlFindMessage(NtosEntry->DllBase,
                               11,
                               0,
                               WINDOWS_NT_BANNER,
                               &MsgEntry);

    /* Setup defaults and check if we have a version string */
    StringBuffer = InitBuffer->VersionBuffer;
    BeginBuffer = StringBuffer;
    EndBuffer = StringBuffer;
    Remaining = sizeof(InitBuffer->VersionBuffer);
    if (CmCSDVersionString.Length)
    {
        /* Print the version string */
        Status = RtlStringCbPrintfExA(StringBuffer,
                                      Remaining,
                                      &EndBuffer,
                                      &Remaining,
                                      0,
                                      ": %wZ",
                                      &CmCSDVersionString);
        if (!NT_SUCCESS(Status))
        {
            /* Bugcheck */
            KeBugCheckEx(PHASE1_INITIALIZATION_FAILED, Status, 7, 0, 0);
        }
    }
    else
    {
        /* No version */
        *EndBuffer = ANSI_NULL; /* Null-terminate the string */
    }

    /* Skip over the null-terminator to start a new string */
    ++EndBuffer;
    --Remaining;

    /* Build the version number */
    StringBuffer = InitBuffer->VersionNumber;
    Status = RtlStringCbPrintfA(StringBuffer,
                                sizeof(InitBuffer->VersionNumber),
                                "%u.%u",
                                VER_PRODUCTMAJORVERSION,
                                VER_PRODUCTMINORVERSION);
    if (!NT_SUCCESS(Status))
    {
        /* Bugcheck */
        KeBugCheckEx(PHASE1_INITIALIZATION_FAILED, Status, 7, 0, 0);
    }

    /* Check if we had found a banner message */
    if (NT_SUCCESS(MsgStatus))
    {
        /* Create the banner message */
        /* ReactOS specific: Report ReactOS version, NtBuildLab information and reported NT kernel version */
        Status = RtlStringCbPrintfA(EndBuffer,
                                    Remaining,
                                    (PCHAR)MsgEntry->Text,
                                    KERNEL_VERSION_STR,
                                    NtBuildLab,
                                    StringBuffer,
                                    NtBuildNumber & 0xFFFF,
                                    BeginBuffer);
        if (!NT_SUCCESS(Status))
        {
            /* Bugcheck */
            KeBugCheckEx(PHASE1_INITIALIZATION_FAILED, Status, 7, 0, 0);
        }
    }
    else
    {
        /* Use hard-coded banner message */
        Status = RtlStringCbCopyA(EndBuffer, Remaining, "REACTOS (R)\r\n");
        if (!NT_SUCCESS(Status))
        {
            /* Bugcheck */
            KeBugCheckEx(PHASE1_INITIALIZATION_FAILED, Status, 7, 0, 0);
        }
    }

    /* Display the version string on-screen */
    InbvDisplayString(EndBuffer);

    /* Initialize Power Subsystem in Phase 0 */
    if (!PoInitSystem(0)) KeBugCheck(INTERNAL_POWER_ERROR);

    /* Check for Y2K hack */
    Y2KHackRequired = CommandLine ? strstr(CommandLine, "YEAR") : NULL;
    if (Y2KHackRequired) Y2KHackRequired = strstr(Y2KHackRequired, "=");
    if (Y2KHackRequired) YearHack = atol(Y2KHackRequired + 1);

    /* Query the clock */
    if ((ExCmosClockIsSane) && (HalQueryRealTimeClock(&TimeFields)))
    {
        /* Check if we're using the Y2K hack */
        if (Y2KHackRequired) TimeFields.Year = (CSHORT)YearHack;

        /* Convert to time fields */
        RtlTimeFieldsToTime(&TimeFields, &SystemBootTime);
        UniversalBootTime = SystemBootTime;

        /* Check if real time is GMT */
        if (!ExpRealTimeIsUniversal)
        {
            /* Check if we don't have a valid bias */
            if (ExpLastTimeZoneBias == MAXULONG)
            {
                /* Reset */
                ResetBias = TRUE;
                ExpLastTimeZoneBias = ExpAltTimeZoneBias;
            }

            /* Calculate the bias in seconds */
            ExpTimeZoneBias.QuadPart = Int32x32To64(ExpLastTimeZoneBias * 60,
                                                    10000000);

            /* Set the boot time-zone bias */
            SharedUserData->TimeZoneBias.High2Time = ExpTimeZoneBias.HighPart;
            SharedUserData->TimeZoneBias.LowPart = ExpTimeZoneBias.LowPart;
            SharedUserData->TimeZoneBias.High1Time = ExpTimeZoneBias.HighPart;

            /* Convert the boot time to local time, and set it */
            UniversalBootTime.QuadPart = SystemBootTime.QuadPart +
                                         ExpTimeZoneBias.QuadPart;
        }

        /* Update the system time */
        KeSetSystemTime(&UniversalBootTime, &OldTime, FALSE, NULL);

        /* Do system callback */
        PoNotifySystemTimeSet();

        /* Remember this as the boot time */
        KeBootTime = UniversalBootTime;
        KeBootTimeBias = 0;
    }

    /* Initialize all processors */
    if (!HalAllProcessorsStarted()) KeBugCheck(HAL1_INITIALIZATION_FAILED);

#ifdef CONFIG_SMP
    /* HACK: We should use RtlFindMessage and not only fallback to this */
    MpString = "MultiProcessor Kernel\r\n";
#endif

    /* Setup the "MP" String */
    RtlInitAnsiString(&TempString, MpString);

    /* Make sure to remove the \r\n if we actually have a string */
    while ((TempString.Length > 0) &&
           ((TempString.Buffer[TempString.Length - 1] == '\r') ||
            (TempString.Buffer[TempString.Length - 1] == '\n')))
    {
        /* Skip the trailing character */
        TempString.Length--;
    }

    /* Get the information string from our resource file */
    MsgStatus = RtlFindMessage(NtosEntry->DllBase,
                               11,
                               0,
                               KeNumberProcessors > 1 ?
                               WINDOWS_NT_INFO_STRING_PLURAL :
                               WINDOWS_NT_INFO_STRING,
                               &MsgEntry);

    /* Get total RAM size, in MiB */
    /* Round size up. Assumed to better match actual physical RAM size */
    Size = ALIGN_UP_BY(MmNumberOfPhysicalPages * PAGE_SIZE, 1024 * 1024) / (1024 * 1024);

    /* Create the string */
    StringBuffer = InitBuffer->VersionBuffer;
    Status = RtlStringCbPrintfA(StringBuffer,
                                sizeof(InitBuffer->VersionBuffer),
                                NT_SUCCESS(MsgStatus) ?
                                (PCHAR)MsgEntry->Text :
                                "%u System Processor [%Iu MB Memory] %Z\r\n",
                                KeNumberProcessors,
                                Size,
                                &TempString);
    if (!NT_SUCCESS(Status))
    {
        /* Bugcheck */
        KeBugCheckEx(PHASE1_INITIALIZATION_FAILED, Status, 4, 0, 0);
    }

    /* Display RAM and CPU count */
    InbvDisplayString(StringBuffer);

    /* Update the progress bar */
    InbvUpdateProgressBar(5);

    /* Call OB initialization again */
    if (!ObInitSystem()) KeBugCheck(OBJECT1_INITIALIZATION_FAILED);

    /* Initialize Basic System Objects and Worker Threads */
    if (!ExInitSystem()) KeBugCheckEx(PHASE1_INITIALIZATION_FAILED, 0, 0, 1, 0);

    /* Initialize the later stages of the kernel */
    if (!KeInitSystem()) KeBugCheckEx(PHASE1_INITIALIZATION_FAILED, 0, 0, 2, 0);

    /* Call KD Providers at Phase 1 */
    if (!KdInitSystem(ExpInitializationPhase, KeLoaderBlock))
    {
        /* Failed, bugcheck */
        KeBugCheckEx(PHASE1_INITIALIZATION_FAILED, 0, 0, 3, 0);
    }

    /* Initialize the SRM in Phase 1 */
    if (!SeInitSystem()) KeBugCheck(SECURITY1_INITIALIZATION_FAILED);

    /* Update the progress bar */
    InbvUpdateProgressBar(10);

    /* Create SystemRoot Link */
    Status = ExpCreateSystemRootLink(LoaderBlock);
    if (!NT_SUCCESS(Status))
    {
        /* Failed to create the system root link */
        KeBugCheckEx(SYMBOLIC_INITIALIZATION_FAILED, Status, 0, 0, 0);
    }

    /* Set up Region Maps, Sections and the Paging File */
    if (!MmInitSystem(1, LoaderBlock)) KeBugCheck(MEMORY1_INITIALIZATION_FAILED);

    /* Create NLS section */
    ExpInitNls(LoaderBlock);

    /* Initialize Cache Views */
    if (!CcInitializeCacheManager()) KeBugCheck(CACHE_INITIALIZATION_FAILED);

    /* Initialize the Registry */
    if (!CmInitSystem1()) KeBugCheck(CONFIG_INITIALIZATION_FAILED);

    /* Initialize Prefetcher */
    CcPfInitializePrefetcher();

    /* Update progress bar */
    InbvUpdateProgressBar(15);

    /* Update timezone information */
    LastTzBias = ExpLastTimeZoneBias;
    ExRefreshTimeZoneInformation(&SystemBootTime);

    /* Check if we're resetting timezone data */
    if (ResetBias)
    {
        /* Convert the local time to system time */
        ExLocalTimeToSystemTime(&SystemBootTime, &UniversalBootTime);
        KeBootTime = UniversalBootTime;
        KeBootTimeBias = 0;

        /* Set the new time */
        KeSetSystemTime(&UniversalBootTime, &OldTime, FALSE, NULL);
    }
    else
    {
        /* Check if the timezone switched and update the time */
        if (LastTzBias != ExpLastTimeZoneBias) ZwSetSystemTime(NULL, NULL);
    }

    /* Initialize the File System Runtime Library */
    if (!FsRtlInitSystem()) KeBugCheck(FILE_INITIALIZATION_FAILED);

    /* Initialize range lists */
    RtlInitializeRangeListPackage();

    /* Report all resources used by HAL */
    HalReportResourceUsage();

    /* Call the debugger DLL */
    KdDebuggerInitialize1(LoaderBlock);

    /* Setup PnP Manager in phase 1 */
    if (!PpInitSystem()) KeBugCheck(PP1_INITIALIZATION_FAILED);

    /* Update progress bar */
    InbvUpdateProgressBar(20);

    /* Initialize LPC */
    if (!LpcInitSystem()) KeBugCheck(LPC_INITIALIZATION_FAILED);

    /* Make sure we have a command line */
    if (CommandLine)
    {
        /* Check if this is a safe mode boot */
        SafeBoot = strstr(CommandLine, "SAFEBOOT:");
        if (SafeBoot)
        {
            /* Check what kind of boot this is */
            SafeBoot += 9;
            if (!strncmp(SafeBoot, "MINIMAL", 7))
            {
                /* Minimal mode */
                InitSafeBootMode = 1;
                SafeBoot += 7;
                MessageCode = BOOTING_IN_SAFEMODE_MINIMAL;
            }
            else if (!strncmp(SafeBoot, "NETWORK", 7))
            {
                /* With Networking */
                InitSafeBootMode = 2;
                SafeBoot += 7;
                MessageCode = BOOTING_IN_SAFEMODE_NETWORK;
            }
            else if (!strncmp(SafeBoot, "DSREPAIR", 8))
            {
                /* Domain Server Repair */
                InitSafeBootMode = 3;
                SafeBoot += 8;
                MessageCode = BOOTING_IN_SAFEMODE_DSREPAIR;

            }
            else
            {
                /* Invalid */
                InitSafeBootMode = 0;
            }

            /* Check if there's any settings left */
            if (*SafeBoot)
            {
                /* Check if an alternate shell was requested */
                if (!strncmp(SafeBoot, "(ALTERNATESHELL)", 16))
                {
                    /* Remember this for later */
                    AlternateShell = TRUE;
                }
            }

            /* Find the message to print out */
            Status = RtlFindMessage(NtosEntry->DllBase,
                                    11,
                                    0,
                                    MessageCode,
                                    &MsgEntry);
            if (NT_SUCCESS(Status))
            {
                /* Display it */
                InbvDisplayString((PCHAR)MsgEntry->Text);
            }
        }
    }

    /* Make sure we have a command line */
    if (CommandLine)
    {
        /* Check if bootlogging is enabled */
        if (strstr(CommandLine, "BOOTLOG"))
        {
            /* Find the message to print out */
            Status = RtlFindMessage(NtosEntry->DllBase,
                                    11,
                                    0,
                                    BOOTLOG_ENABLED,
                                    &MsgEntry);
            if (NT_SUCCESS(Status))
            {
                /* Display it */
                InbvDisplayString((PCHAR)MsgEntry->Text);
            }

            /* Setup boot logging */
            //IopInitializeBootLogging(LoaderBlock, InitBuffer->BootlogHeader);
        }
    }

    /* Setup the Executive in Phase 2 */
    //ExInitSystemPhase2();

    /* Update progress bar */
    InbvUpdateProgressBar(25);

#ifdef _WINKD_
    /* No KD Time Slip is pending */
    KdpTimeSlipPending = 0;
#endif

    /* Initialize in-place execution support */
    XIPInit(LoaderBlock);

    /* Set maximum update to 75% */
    InbvSetProgressBarSubset(25, 75);

    /* Initialize the I/O Subsystem */
    if (!IoInitSystem(LoaderBlock)) KeBugCheck(IO1_INITIALIZATION_FAILED);

    /* Set maximum update to 100% */
    InbvSetProgressBarSubset(0, 100);

    /* Are we in safe mode? */
    if (InitSafeBootMode)
    {
        /* Open the safe boot key */
        RtlInitUnicodeString(&KeyName,
                             L"\\REGISTRY\\MACHINE\\SYSTEM\\CURRENTCONTROLSET"
                             L"\\CONTROL\\SAFEBOOT");
        InitializeObjectAttributes(&ObjectAttributes,
                                   &KeyName,
                                   OBJ_CASE_INSENSITIVE,
                                   NULL,
                                   NULL);
        Status = ZwOpenKey(&KeyHandle, KEY_ALL_ACCESS, &ObjectAttributes);
        if (NT_SUCCESS(Status))
        {
            /* First check if we have an alternate shell */
            if (AlternateShell)
            {
                /* Make sure that the registry has one setup */
                RtlInitUnicodeString(&KeyName, L"AlternateShell");
                Status = NtQueryValueKey(KeyHandle,
                                         &KeyName,
                                         KeyValuePartialInformation,
                                         &KeyPartialInfo,
                                         sizeof(KeyPartialInfo),
                                         &Length);
                if (!(NT_SUCCESS(Status) || Status == STATUS_BUFFER_OVERFLOW))
                {
                    AlternateShell = FALSE;
                }
            }

            /* Create the option key */
            RtlInitUnicodeString(&KeyName, L"Option");
            InitializeObjectAttributes(&ObjectAttributes,
                                       &KeyName,
                                       OBJ_CASE_INSENSITIVE,
                                       KeyHandle,
                                       NULL);
            Status = ZwCreateKey(&OptionHandle,
                                 KEY_ALL_ACCESS,
                                 &ObjectAttributes,
                                 0,
                                 NULL,
                                 REG_OPTION_VOLATILE,
                                 &Disposition);
            NtClose(KeyHandle);

            /* Check if the key create worked */
            if (NT_SUCCESS(Status))
            {
                /* Write the safe boot type */
                RtlInitUnicodeString(&KeyName, L"OptionValue");
                NtSetValueKey(OptionHandle,
                              &KeyName,
                              0,
                              REG_DWORD,
                              &InitSafeBootMode,
                              sizeof(InitSafeBootMode));

                /* Check if we have to use an alternate shell */
                if (AlternateShell)
                {
                    /* Remember this for later */
                    Disposition = TRUE;
                    RtlInitUnicodeString(&KeyName, L"UseAlternateShell");
                    NtSetValueKey(OptionHandle,
                                  &KeyName,
                                  0,
                                  REG_DWORD,
                                  &Disposition,
                                  sizeof(Disposition));
                }

                /* Close the options key handle */
                NtClose(OptionHandle);
            }
        }
    }

    /* Are we in Win PE mode? */
    if (InitIsWinPEMode)
    {
        /* Open the safe control key */
        RtlInitUnicodeString(&KeyName,
                             L"\\REGISTRY\\MACHINE\\SYSTEM\\CURRENTCONTROLSET"
                             L"\\CONTROL");
        InitializeObjectAttributes(&ObjectAttributes,
                                   &KeyName,
                                   OBJ_CASE_INSENSITIVE,
                                   NULL,
                                   NULL);
        Status = ZwOpenKey(&KeyHandle, KEY_ALL_ACCESS, &ObjectAttributes);
        if (!NT_SUCCESS(Status))
        {
            /* Bugcheck */
            KeBugCheckEx(PHASE1_INITIALIZATION_FAILED, Status, 6, 0, 0);
        }

        /* Create the MiniNT key */
        RtlInitUnicodeString(&KeyName, L"MiniNT");
        InitializeObjectAttributes(&ObjectAttributes,
                                   &KeyName,
                                   OBJ_CASE_INSENSITIVE,
                                   KeyHandle,
                                   NULL);
        Status = ZwCreateKey(&OptionHandle,
                             KEY_ALL_ACCESS,
                             &ObjectAttributes,
                             0,
                             NULL,
                             REG_OPTION_VOLATILE,
                             &Disposition);
        if (!NT_SUCCESS(Status))
        {
            /* Bugcheck */
            KeBugCheckEx(PHASE1_INITIALIZATION_FAILED, Status, 6, 0, 0);
        }

        /* Close the handles */
        NtClose(KeyHandle);
        NtClose(OptionHandle);
    }

    /* FIXME: This doesn't do anything for now */
    MmArmInitSystem(2, LoaderBlock);

    /* Update progress bar */
    InbvUpdateProgressBar(80);

    /* Initialize VDM support */
#if defined(_M_IX86)
    KeI386VdmInitialize();
#endif

    /* Initialize Power Subsystem in Phase 1*/
    if (!PoInitSystem(1)) KeBugCheck(INTERNAL_POWER_ERROR);

    /* Update progress bar */
    InbvUpdateProgressBar(90);

    /* Initialize the Process Manager at Phase 1 */
    if (!PsInitSystem(LoaderBlock)) KeBugCheck(PROCESS1_INITIALIZATION_FAILED);

    /* Make sure nobody touches the loader block again */
    if (LoaderBlock == KeLoaderBlock) KeLoaderBlock = NULL;
    MmFreeLoaderBlock(LoaderBlock);
    LoaderBlock = Context = NULL;

    /* Initialize the SRM in phase 1 */
    if (!SeRmInitPhase1()) KeBugCheck(PROCESS1_INITIALIZATION_FAILED);

    /* Update progress bar */
    InbvUpdateProgressBar(100);

    /* Clear the screen */
    if (InbvBootDriverInstalled) FinalizeBootLogo();

    /* Allow strings to be displayed */
    InbvEnableDisplayString(TRUE);

    /* Launch initial process */
    DPRINT("Free non-cache pages: %lx\n", MmAvailablePages + MiMemoryConsumers[MC_CACHE].PagesUsed);
    ProcessInfo = &InitBuffer->ProcessInfo;
    ExpLoadInitialProcess(InitBuffer, &ProcessParameters, &Environment);

    /* Wait 5 seconds for initial process to initialize */
    Timeout.QuadPart = Int32x32To64(5, -10000000);
    Status = ZwWaitForSingleObject(ProcessInfo->ProcessHandle, FALSE, &Timeout);
    if (Status == STATUS_SUCCESS)
    {
        /* Failed, display error */
        DPRINT1("INIT: Session Manager terminated.\n");

        /* Bugcheck the system if SMSS couldn't initialize */
        KeBugCheck(SESSION5_INITIALIZATION_FAILED);
    }

    /* Close process handles */
    ZwClose(ProcessInfo->ThreadHandle);
    ZwClose(ProcessInfo->ProcessHandle);

    /* Free the initial process environment */
    Size = 0;
    ZwFreeVirtualMemory(NtCurrentProcess(),
                        (PVOID*)&Environment,
                        &Size,
                        MEM_RELEASE);

    /* Free the initial process parameters */
    Size = 0;
    ZwFreeVirtualMemory(NtCurrentProcess(),
                        (PVOID*)&ProcessParameters,
                        &Size,
                        MEM_RELEASE);

    /* Increase init phase */
    ExpInitializationPhase++;

    /* Free the boot buffer */
    ExFreePoolWithTag(InitBuffer, TAG_INIT);
    DPRINT("Free non-cache pages: %lx\n", MmAvailablePages + MiMemoryConsumers[MC_CACHE].PagesUsed);
}
```

首先将初始化阶段设置为1，即全局变量`ExpInitializationPhase`，将当前的线程的权限设置为高权限（HIGH_PRIORITY）。设置引导驱动，主要是显示驱动。

找到内核模块，然后从中找到内核版本信息，将版本信息添加到初始化内存块的`VersionNumber`字段中。

* 调用`PoInitSystem()`完成能源子系统的阶段0初始化
* 获取Cmos时钟信息，并且更新当前的系统时间与引导时间
* 调用`HalAllProcessorsStarted()`进行多CPU的启动，如果是多CPU的系统，则要初始化其他CPU的ACPI
* 调用`ObInitSystem()`函数进行阶段1的对象管理器初始化
* 调用`ExInitSystem()`函数进行执行体的阶段1初始化
* 调用`KeInitSystem()`函数进行内核的阶段1初始化
* 调用`KdInitSystem()`函数，在阶段1调用KD提供者
* 调用`SeInitSystem()`函数进行监视计数器的初始化
* 调用`MmInitSystem()`函数进行内存区块映射，扇区映射和分页文件设置
* 调用`ExpInitNls()`进行NLS分区
* 调用`CcInitializeCacheManager()`初始化缓存的视图
* 调用`CmInitSystem1()`进一步初始化注册表
* 调用`CcPfInitializePrefetcher()`初始化预取器。
* 调用`FsRtlInitSystem()`初始化文件系统运行时库
* 调用`PpInitSystem()`进行阶段1的即插即用管理器初始化
* 调用`LpcInitSystem()`初始化LPC
* 调用`IoInitSystem()`进行I/O子系统的阶段1初始化
* 调用`PoInitSystem(1)`进行能源子系统的阶段1初始化
* 调用`PsInitSystem()`进行进程管理子系统的阶段1初始化。
* 在阶段1调用`SeRmInitPhase1()`，进行安全引用计数器的阶段1初始化
* 调用`ExpLoadInitialProcess()`启动初始进程


在`ExpLoadInitialProcess()`中首先构建进程的环境变量，包括Path，SystemDrive，SystemRoot等环境变量，然后调用函数`RtlCreateUserProcess()`创建SMSS进程。

当创建完了SMSS进程后，页面清零线程要等五秒钟，等待进程运行起来。


**未总结内容**

1. 无

**参考文章**

* WoW64 Wiki	[https://zh.wikipedia.org/wiki/WoW64](https://zh.wikipedia.org/wiki/WoW64)

**修订历史**

* 2017-09-04 11:21:23		完成文章

By Andy @2017-09-04 11:21:23