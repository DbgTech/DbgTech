---
title: ReactOS启动过程之系统进程
date: 2017-09-03 20:52:23
tags:
- Windbg
- ReactOS
- 启动过程
categories:
- ReactOS
---


在阶段1 的初始化最后调用`ExpLoadInitialProcess()`函数，创建SMSS进程，这一篇将几个系统进程的过程简单学习学习。可执行文件`SMSS.exe`的源码`\base\system`子目录中，其入口函数位于文件`*\base\system\smss\smss.c`中。

```
NTSTATUS __cdecl _main(IN INT argc,
      IN PCHAR argv[],
      IN PCHAR envp[],
      IN ULONG DebugFlag)
{
    NTSTATUS Status;
    KPRIORITY SetBasePriority;
    ULONG_PTR Parameters[4];
    HANDLE Handles[2];
    PVOID State;
    ULONG Flags;
    PROCESS_BASIC_INFORMATION ProcessInfo;
    UNICODE_STRING DbgString, InitialCommand;

    /* Make us critical */
    RtlSetProcessIsCritical(TRUE, NULL, FALSE);
    RtlSetThreadIsCritical(TRUE, NULL, FALSE);

    /* Raise our priority */
    SetBasePriority = 11;
    Status = NtSetInformationProcess(NtCurrentProcess(),
                                     ProcessBasePriority,
                                     (PVOID)&SetBasePriority,
                                     sizeof(SetBasePriority));
    ASSERT(NT_SUCCESS(Status));

    /* Save the debug flag if it was passed */
    if (DebugFlag) SmpDebug = DebugFlag != 0;

    /* Build the hard error parameters */
    Parameters[0] = (ULONG_PTR)&DbgString;
    Parameters[1] = Parameters[2] = Parameters[3] = 0;

    /* Enter SEH so we can terminate correctly if anything goes wrong */
    _SEH2_TRY
    {
        /* Initialize SMSS */
        Status = SmpInit(&InitialCommand, Handles);
        if (!NT_SUCCESS(Status))
        {
            DPRINT1("SMSS: SmpInit return failure - Status == %x\n", Status);
            RtlInitUnicodeString(&DbgString, L"Session Manager Initialization");
            Parameters[1] = Status;
            _SEH2_LEAVE;
        }

        /* Get the global flags */
        Status = NtQuerySystemInformation(SystemFlagsInformation,
                                          &Flags,
                                          sizeof(Flags),
                                          NULL);
        ASSERT(NT_SUCCESS(Status));

        /* Before executing the initial command check if the debug flag is on */
        if (Flags & (FLG_DEBUG_INITIAL_COMMAND | FLG_DEBUG_INITIAL_COMMAND_EX))
        {
            /* SMSS should launch ntsd with a few parameters at this point */
            DPRINT1("Global Flags Set to SMSS Debugging: Not yet supported\n");
        }

        /* Execute the initial command (Winlogon.exe) */
        Status = SmpExecuteInitialCommand(0, &InitialCommand, &Handles[1], NULL);
        if (!NT_SUCCESS(Status))
        {
            /* Fail and raise a hard error */
            DPRINT1("SMSS: Execute Initial Command failed\n");
            RtlInitUnicodeString(&DbgString,
                                 L"Session Manager ExecuteInitialCommand");
            Parameters[1] = Status;
            _SEH2_LEAVE;
        }

        /*  Check if we're already attached to a session */
        Status = SmpAcquirePrivilege(SE_LOAD_DRIVER_PRIVILEGE, &State);
        if (AttachedSessionId != -1)
        {
            /* Detach from it, we should be in no session right now */
            Status = NtSetSystemInformation(SystemSessionDetach,
                                            &AttachedSessionId,
                                            sizeof(AttachedSessionId));
            ASSERT(NT_SUCCESS(Status));
            AttachedSessionId = -1;
        }
        SmpReleasePrivilege(State);

        /* Wait on either CSRSS or Winlogon to die */
        Status = NtWaitForMultipleObjects(RTL_NUMBER_OF(Handles),
                                          Handles,
                                          WaitAny,
                                          FALSE,
                                          NULL);
        if (Status == STATUS_WAIT_0)
        {
            /* CSRSS is dead, get exit code and prepare for the hard error */
            RtlInitUnicodeString(&DbgString, L"Windows SubSystem");
            Status = NtQueryInformationProcess(Handles[0],
                                               ProcessBasicInformation,
                                               &ProcessInfo,
                                               sizeof(ProcessInfo),
                                               NULL);
            DPRINT1("SMSS: Windows subsystem terminated when it wasn't supposed to.\n");
        }
        else
        {
            /* The initial command is dead or we have another failure */
            RtlInitUnicodeString(&DbgString, L"Windows Logon Process");
            if (Status == STATUS_WAIT_1)
            {
                /* Winlogon.exe got terminated, get its exit code */
                Status = NtQueryInformationProcess(Handles[1],
                                                   ProcessBasicInformation,
                                                   &ProcessInfo,
                                                   sizeof(ProcessInfo),
                                                   NULL);
            }
            else
            {
                /* Something else satisfied our wait, so set the wait status */
                ProcessInfo.ExitStatus = Status;
                Status = STATUS_SUCCESS;
            }
            DPRINT1("SMSS: Initial command '%wZ' terminated when it wasn't supposed to.\n",
                    &InitialCommand);
        }

        /* Check if NtQueryInformationProcess was successful */
        if (NT_SUCCESS(Status))
        {
            /* Then we must have a valid exit status in the structure, use it */
            Parameters[1] = ProcessInfo.ExitStatus;
        }
        else
        {
            /* We really don't know what happened, so set a generic error */
            Parameters[1] = STATUS_UNSUCCESSFUL;
        }
    }
    _SEH2_EXCEPT(SmpUnhandledExceptionFilter(_SEH2_GetExceptionInformation()))
    {
        /* The filter should never return here */
        ASSERT(FALSE);
    }
    _SEH2_END;

    /* Something in the init loop failed, terminate SMSS */
    return SmpTerminate(Parameters, 1, RTL_NUMBER_OF(Parameters));
}
```

首先调用`SmpInit()`函数，用于初始化SMSS进程自身，代码如下代码块所示。首先为当前进程创建Tag堆，初始化其中的锁与链表。

创建`\\SmApiPort`端口，并且创建两个线程监控该端口。创建一个`\\Device\\VolumesSafeForWriteAccess`事件。加载注册表参数，其中使用比较多的`Pending file rename`操作就是在读取注册表过程中发现有待修改或待删除文件项目，就在` SmpProcessFileRenames()`函数中进行处理；根据注册表中的`KnownDlls`注册表项，初始化已知DLL的对象目录`\\KnownDlls`，并将注册表解析挂入全局列表中。

这其中会启动所有的子系统，每个子系统都是一个独立的进程，比如Win32的话就是启动`CSRSS.exe`进程。

最后调用`SmpLoadSubSystemsForMuSession()`为第一个会话加载所有的子系统。

```
NTSTATUS NTAPI SmpInit(IN PUNICODE_STRING InitialCommand, OUT PHANDLE ProcessHandle)
{
    NTSTATUS Status, Status2;
    OBJECT_ATTRIBUTES ObjectAttributes;
    UNICODE_STRING PortName, EventName;
    HANDLE EventHandle, PortHandle;
    ULONG HardErrorMode;

    /* Create the SMSS Heap */
    SmBaseTag = RtlCreateTagHeap(RtlGetProcessHeap(),
                                 0,
                                 L"SMSS!",
                                 L"INIT");
    SmpHeap = RtlGetProcessHeap();

    /* Enable hard errors */
    HardErrorMode = TRUE;
    NtSetInformationProcess(NtCurrentProcess(),
                            ProcessDefaultHardErrorMode,
                            &HardErrorMode,
                            sizeof(HardErrorMode));

    /* Initialize the subsystem list and the session list, plus their locks */
    RtlInitializeCriticalSection(&SmpKnownSubSysLock);
    InitializeListHead(&SmpKnownSubSysHead);
    RtlInitializeCriticalSection(&SmpSessionListLock);
    InitializeListHead(&SmpSessionListHead);

    /* Initialize the process list */
    InitializeListHead(&NativeProcessList);

    /* Initialize session parameters */
    SmpNextSessionId = 1;
    SmpNextSessionIdScanMode = 0;
    SmpDbgSsLoaded = FALSE;

    /* Create the initial security descriptors */
    Status = SmpCreateSecurityDescriptors(TRUE);
    if (!NT_SUCCESS(Status))
    {
        /* Fail */
        SMSS_CHECKPOINT(SmpCreateSecurityDescriptors, Status);
        return Status;
    }

    /* Initialize subsystem names */
    RtlInitUnicodeString(&SmpSubsystemName, L"NT-Session Manager");
    RtlInitUnicodeString(&PosixName, L"POSIX");
    RtlInitUnicodeString(&Os2Name, L"OS2");

    /* Create the SM API Port */
    RtlInitUnicodeString(&PortName, L"\\SmApiPort");
    InitializeObjectAttributes(&ObjectAttributes, &PortName, 0, NULL, NULL);
    Status = NtCreatePort(&PortHandle,
                          &ObjectAttributes,
                          sizeof(SB_CONNECTION_INFO),
                          sizeof(SM_API_MSG),
                          sizeof(SB_API_MSG) * 32);
    ASSERT(NT_SUCCESS(Status));
    SmpDebugPort = PortHandle;

    /* Create two SM API threads */
    Status = RtlCreateUserThread(NtCurrentProcess(),
                                 NULL,
                                 FALSE,
                                 0,
                                 0,
                                 0,
                                 SmpApiLoop,
                                 PortHandle,
                                 NULL,
                                 NULL);
    ASSERT(NT_SUCCESS(Status));
    Status = RtlCreateUserThread(NtCurrentProcess(),
                                 NULL,
                                 FALSE,
                                 0,
                                 0,
                                 0,
                                 SmpApiLoop,
                                 PortHandle,
                                 NULL,
                                 NULL);
    ASSERT(NT_SUCCESS(Status));

    /* Create the write event that autochk can set after running */
    RtlInitUnicodeString(&EventName, L"\\Device\\VolumesSafeForWriteAccess");
    InitializeObjectAttributes(&ObjectAttributes,
                               &EventName,
                               OBJ_PERMANENT,
                               NULL,
                               NULL);
    Status2 = NtCreateEvent(&EventHandle,
                            EVENT_ALL_ACCESS,
                            &ObjectAttributes,
                            0,
                            0);
    if (!NT_SUCCESS(Status2))
    {
        /* Should never really fail */
        DPRINT1("SMSS: Unable to create %wZ event - Status == %lx\n",
                &EventName, Status2);
        ASSERT(NT_SUCCESS(Status2));
    }

    /* Now initialize everything else based on the registry parameters */
    Status = SmpLoadDataFromRegistry(InitialCommand);
    if (NT_SUCCESS(Status))
    {
        /* Autochk should've run now. Set the event and save the CSRSS handle */
        *ProcessHandle = SmpWindowsSubSysProcess;
        NtSetEvent(EventHandle, 0);
        NtClose(EventHandle);
    }

    /* All done */
    return Status;
}
```

初始化完了会话进程，调用 `SmpExecuteInitialCommand` 启动进程`winlogon.exe`。

启动了两个进程之后，将当前进程挂到对应的会话中，然后等待两个进程的句柄（Winlogon.exe/CSRSS.exe），如果有一个进程退出则报告内核有系统关键进程被退出了。

### Win32子系统管理进程（CSRSS.exe） ###

`SMSS.exe`程序中的`SmpLoadSubSystemsForMuSession()`函数会负责启动子系统的管理进程，即`CSRSS.exe`进程。

CSRSS可执行文件比较简单，加载起来`CSRSRV.dll`并初始化后，主线程就终止了。

```
int _cdecl _main(int argc, char *argv[], char *envp[], int DebugFlag)
{
    KPRIORITY BasePriority = (8 + 1) + 4;
    NTSTATUS Status;
    //ULONG Response; // see the #if 0
    UNREFERENCED_PARAMETER(envp);
    UNREFERENCED_PARAMETER(DebugFlag);

    /* Set the Priority */
    NtSetInformationProcess(NtCurrentProcess(),
                            ProcessBasePriority,
                            &BasePriority,
                            sizeof(KPRIORITY));

    /* Give us IOPL so that we can access the VGA registers */
    Status = NtSetInformationProcess(NtCurrentProcess(),
                                     ProcessUserModeIOPL,
                                     NULL,
                                     0);
    if (!NT_SUCCESS(Status))
    {
        /* Raise a hard error */
        DPRINT1("CSRSS: Could not raise IOPL, Status: 0x%08lx\n", Status);
    }

    /* Initialize CSR through CSRSRV */
    Status = CsrServerInitialization(argc, argv);
    if (!NT_SUCCESS(Status))
    {
        /* Kill us */
        DPRINT1("CSRSS: Unable to initialize server, Status: 0x%08lx\n", Status);
        NtTerminateProcess(NtCurrentProcess(), Status);
    }

    /* Disable errors */
    CsrpSetDefaultProcessHardErrorMode();

    /* If this is Session 0, make sure killing us bugchecks the system */
    if (NtCurrentPeb()->SessionId == 0) RtlSetProcessIsCritical(TRUE, NULL, FALSE);

    /* Kill this thread. CSRSRV keeps us going */
    NtTerminateThread(NtCurrentThread(), Status);
    return 0;
}
```

`CsrServerInitialization`函数是CSRSRV模块入口函数，其中做了进程的基本初始化，然后将当前进程挂入进程队列中。创建`\\ApiPort`端口等待连接与通信，比如新建进程，新建线程等CSR类API调用的响应。创建`\\SbApiPort`端口，用于与SMSS进程的双向通信。

最后连接SM端口，即`\\SmApiPort`，与SMSS进程通信。

```
NTSTATUS NTAPI CsrServerInitialization(IN ULONG ArgumentCount, IN PCHAR Arguments[])
{
    NTSTATUS Status = STATUS_SUCCESS;

    /* Cache System Basic Information so we don't always request it */
    Status = NtQuerySystemInformation(SystemBasicInformation,
                                      &CsrNtSysInfo,
                                      sizeof(SYSTEM_BASIC_INFORMATION),
                                      NULL);
    if (!NT_SUCCESS(Status))
    {
        DPRINT1("CSRSRV:%s: NtQuerySystemInformation failed (Status=0x%08lx)\n",
                __FUNCTION__, Status);
        return Status;
    }

    /* Save our Heap */
    CsrHeap = RtlGetProcessHeap();

    /* Set our Security Descriptor to protect the process */
    Status = CsrSetProcessSecurity();
    if (!NT_SUCCESS(Status))
    {
        DPRINT1("CSRSRV:%s: CsrSetProcessSecurity failed (Status=0x%08lx)\n",
                __FUNCTION__, Status);
        return Status;
    }

    /* Set up Session Support */
    Status = CsrInitializeNtSessionList();
    if (!NT_SUCCESS(Status))
    {
        DPRINT1("CSRSRV:%s: CsrInitializeSessions failed (Status=0x%08lx)\n",
                __FUNCTION__, Status);
        return Status;
    }

    /* Set up Process Support and allocate the CSR Root Process */
    Status = CsrInitializeProcessStructure();
    if (!NT_SUCCESS(Status))
    {
        DPRINT1("CSRSRV:%s: CsrInitializeProcessStructure failed (Status=0x%08lx)\n",
                __FUNCTION__, Status);
        return Status;
    }

    /* Parse the command line */
    Status = CsrParseServerCommandLine(ArgumentCount, Arguments);
    if (!NT_SUCCESS(Status))
    {
        DPRINT1("CSRSRV:%s: CsrParseServerCommandLine failed (Status=0x%08lx)\n",
                __FUNCTION__, Status);
        return Status;
    }

    /* Finish to initialize the CSR Root Process */
    Status = CsrInitCsrRootProcess();
    if (!NT_SUCCESS(Status))
    {
        DPRINT1("CSRSRV:%s: CsrInitCsrRootProcess failed (Status=0x%08lx)\n",
                __FUNCTION__, Status);
        return Status;
    }

    /* Now initialize our API Port */
    Status = CsrApiPortInitialize();
    if (!NT_SUCCESS(Status))
    {
        DPRINT1("CSRSRV:%s: CsrApiPortInitialize failed (Status=0x%08lx)\n",
                __FUNCTION__, Status);
        return Status;
    }

    /* Initialize the API Port for SM communication */
    Status = CsrSbApiPortInitialize();
    if (!NT_SUCCESS(Status))
    {
        DPRINT1("CSRSRV:%s: CsrSbApiPortInitialize failed (Status=0x%08lx)\n",
                __FUNCTION__, Status);
        return Status;
    }

    /* We're all set! Connect to SM! */
    Status = SmConnectToSm(&CsrSbApiPortName,
                           CsrSbApiPort,
                           IMAGE_SUBSYSTEM_WINDOWS_GUI,
                           &CsrSmApiPort);
    if (!NT_SUCCESS(Status))
    {
        DPRINT1("CSRSRV:%s: SmConnectToSm failed (Status=0x%08lx)\n",
                __FUNCTION__, Status);
        return Status;
    }

    /* Have us handle Hard Errors */
    Status = NtSetDefaultHardErrorPort(CsrApiPort);
    if (!NT_SUCCESS(Status))
    {
        DPRINT1("CSRSRV:%s: NtSetDefaultHardErrorPort failed (Status=0x%08lx)\n",
                __FUNCTION__, Status);
        return Status;
    }

    /* Return status */
    return Status;
}
```

### 登录进程 WinLogon.exe ###

WinLogin.exe的代码入口位于文件`*\base\system\winlogon\winlogon.c`中。其代码如下所示，其完成了如下几份工作：

* 首先将当前进程注册给CSRSS进程，作为子系统的一个进程，并且初始化进程堆。
* 为登录桌面，用户桌面以及屏保桌面分别建立独立的桌面，并将当前会话切换到登录桌面。
* 启动操作注册表的远程过程调用（RPC）
* 启动服务管理进程（Services）
* 启动安全校验进程（Lsass）
* 加载Gina模块，并初始化，即显示登录界面
* 初始化SAS，创建隐藏窗口用于接收SAS通知，防止该组合键被劫持
* 最后进入消息循环

```
int WINAPI WinMain(
    IN HINSTANCE hInstance,
    IN HINSTANCE hPrevInstance,
    IN LPSTR lpCmdLine,
    IN int nShowCmd)
{
    ULONG HardErrorResponse;
    MSG Msg;

    UNREFERENCED_PARAMETER(hPrevInstance);
    UNREFERENCED_PARAMETER(lpCmdLine);
    UNREFERENCED_PARAMETER(nShowCmd);

    hAppInstance = hInstance;

    /* Make us critical */
    RtlSetProcessIsCritical(TRUE, NULL, FALSE);
    RtlSetThreadIsCritical(TRUE, NULL, FALSE);

    if (!RegisterLogonProcess(GetCurrentProcessId(), TRUE))
    {
        ERR("WL: Could not register logon process\n");
        NtRaiseHardError(STATUS_SYSTEM_PROCESS_TERMINATED, 0, 0, NULL, OptionOk, &HardErrorResponse);
        ExitProcess(1);
    }

    WLSession = (PWLSESSION)HeapAlloc(GetProcessHeap(), 0, sizeof(WLSESSION));
    if (!WLSession)
    {
        ERR("WL: Could not allocate memory for winlogon instance\n");
        NtRaiseHardError(STATUS_SYSTEM_PROCESS_TERMINATED, 0, 0, NULL, OptionOk, &HardErrorResponse);
        ExitProcess(1);
    }

    ZeroMemory(WLSession, sizeof(WLSESSION));
    WLSession->DialogTimeout = 120; /* 2 minutes */

    /* Initialize the dialog tracking list */
    InitDialogListHead();

    if (!CreateWindowStationAndDesktops(WLSession))
    {
        ERR("WL: Could not create window station and desktops\n");
        NtRaiseHardError(STATUS_SYSTEM_PROCESS_TERMINATED, 0, 0, NULL, OptionOk, &HardErrorResponse);
        ExitProcess(1);
    }

    LockWorkstation(WLSession);

    /* Load default keyboard layouts */
    if (!InitKeyboardLayouts())
    {
        ERR("WL: Could not preload keyboard layouts\n");
        NtRaiseHardError(STATUS_SYSTEM_PROCESS_TERMINATED, 0, 0, NULL, OptionOk, &HardErrorResponse);
        ExitProcess(1);
    }

    if (!StartRpcServer())
    {
        ERR("WL: Could not start the RPC server\n");
        NtRaiseHardError(STATUS_SYSTEM_PROCESS_TERMINATED, 0, 0, NULL, OptionOk, &HardErrorResponse);
        ExitProcess(1);
    }

    if (!StartServicesManager())
    {
        ERR("WL: Could not start services.exe\n");
        NtRaiseHardError(STATUS_SYSTEM_PROCESS_TERMINATED, 0, 0, NULL, OptionOk, &HardErrorResponse);
        ExitProcess(1);
    }

    if (!StartLsass())
    {
        ERR("WL: Failed to start lsass.exe service (error %lu)\n", GetLastError());
        NtRaiseHardError(STATUS_SYSTEM_PROCESS_TERMINATED, 0, 0, NULL, OptionOk, &HardErrorResponse);
        ExitProcess(1);
    }

    /* Wait for the LSA server */
    WaitForLsass();

    /* Init Notifications */
    InitNotifications();

    /* Load and initialize gina */
    if (!GinaInit(WLSession))
    {
        ERR("WL: Failed to initialize Gina\n");
        // FIXME: Retrieve the real name of the GINA DLL we were trying to load.
        // It is known only inside the GinaInit function...
        DialogBoxParam(hAppInstance, MAKEINTRESOURCE(IDD_GINALOADFAILED), GetDesktopWindow(), GinaLoadFailedWindowProc, (LPARAM)L"msgina.dll");
        HandleShutdown(WLSession, WLX_SAS_ACTION_SHUTDOWN_REBOOT);
        ExitProcess(1);
    }

    DisplayStatusMessage(WLSession, WLSession->WinlogonDesktop, IDS_REACTOSISSTARTINGUP);

    CallNotificationDlls(WLSession, StartupHandler);

    /* Create a hidden window to get SAS notifications */
    if (!InitializeSAS(WLSession))
    {
        ERR("WL: Failed to initialize SAS\n");
        ExitProcess(2);
    }

    // DisplayStatusMessage(Session, Session->WinlogonDesktop, IDS_PREPARENETWORKCONNECTIONS);
    // DisplayStatusMessage(Session, Session->WinlogonDesktop, IDS_APPLYINGCOMPUTERSETTINGS);

    /* Display logged out screen */
    WLSession->LogonState = STATE_INIT;
    RemoveStatusMessage(WLSession);

    /* Check for pending setup */
    if (GetSetupType() != 0)
    {
        /* Run setup and reboot when done */
        TRACE("WL: Setup mode detected\n");
        RunSetup();
    }
    else
    {
        PostMessageW(WLSession->SASWindow, WLX_WM_SAS, WLX_SAS_TYPE_CTRL_ALT_DEL, 0);
    }

    (void)LoadLibraryW(L"sfc_os.dll");

    /* Tell kernel that CurrentControlSet is good (needed
     * to support Last good known configuration boot) */
    NtInitializeRegistry(CM_BOOT_FLAG_ACCEPTED | 1);

    /* Message loop for the SAS window */
    while (GetMessageW(&Msg, WLSession->SASWindow, 0, 0))
    {
        TranslateMessage(&Msg);
        DispatchMessageW(&Msg);
    }

    CleanupNotifications();

    /* We never go there */

    return 0;
}
```

> 至于服务进程（services.exe）和本地安全进程（lsass.exe）在需要时再详细阅读其代码。

在输入登录信息，并且本地安全进程校验成功后，会返回成功，消息被返回到登录进程窗口，在其消息处理的函数`HandleLogon()`中会调用`StartUserShell()`启动本地shell，其实这里调用的是`msgina.dll`的`WlxActivateUserShell()`函数，其中查找`HKEY_LOCAL_MACHINE\\SOFTWARE\\Microsoft\\Windows NT\\CurrentVersion\\Winlogon`，获取其中的默认用户Shell，然后启动其中的进程。这里其实就是启动的`userinit.exe`进程。

**用户初始化进程（userinit.exe）**

userinit进程就比较简单了，它判定当前是要普通的系统启动（非引导），则调用`StartShell()`函数查找当前的默认Shell（即Explorer）进程。然后启动该进程，在完成了进程启动后，userinit进程就自动退出了。

至此，整个系统的引导过程的主线规划出来了，每个部分的功能也大致清晰，留待今后用到时即可慢慢调试与分析。


**未总结内容**

1. 无

**参考文章**

* WoW64 Wiki	[https://zh.wikipedia.org/wiki/WoW64](https://zh.wikipedia.org/wiki/WoW64)

**修订历史**

* 2017-09-04 11:21:23		完成文章

By Andy @2017-09-04 11:21:23