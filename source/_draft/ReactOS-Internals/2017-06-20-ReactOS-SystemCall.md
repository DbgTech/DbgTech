---
title: ReactOS 系统调用
date: 2017-06-20 17:15:33
tags:
- ReactOS
categories:
- ReactOS
---

上一篇文章中已经完成了ReactOS代码的编译，以及安装到虚拟机中，同时也研究了直接用Windbg进行调试的方法。也就是说调试它和调试Windows操作系统无太大差异，接下来就是坚持学习去了。单纯去看源码可能会迷失掉，ReactOS现有版本的代码量已经非常大了，涉及到内容也已经很繁杂，所以基于毛德操老师的《Windows内核情景分析》一书来学习。

后面的N篇文章就作为读书笔记，记录阅读中自认为的重点内容以及自己的一些思考！现在开始第一篇笔记，系统调用。系统调用之前，先简单说一下概述中自认为的重点的内容。

### 概述 ###

系统调用前面是概述内容，这个里面有几个想抄写到这里，作为记录！首先是毛老师在里面提到了现代意义上的操作系统的条件：

* 操作系统在内存中的映像必须受到保护，不会因为用户程序的不良行为而受到伤害
* 不同的用户程序之间也应该互相防护，不能因为一个程序不良行为或误操作而伤害另一个程序
* 除了保护用户数据与程序外，在物理内存中的位置应该可以浮动

再有就是Windows内核的简单描述，关于宏内核和Windows内核的关系，这里直接给出毛老师书中的Windows系统结构图，如图 1。

<div align="center">
![图1 Windows系统结构图](/img/2017-06-20-ReactOS-SystemCall-Windows-Architecture.jpg)
</div>

<!-- more -->

正如书中所述，其实从图上就可以看出，所谓Windows操作系统，至少应该是内核加上系统DLL还有子系统服务进程的总和。

还有几点也需要摘抄过来，对于一些概念的理解有帮助。系统调用界面下面就是Executive，即内核的管理层。管理层又具体分为对象管理，内存管理，进程管理，安全管理，I/O管理等独立模块。有些资料翻译为执行体，其实不太准确，翻译成管理层更容易理解。再者，微软所说的“内核”其实是内核中比较核心的，比较接近底层的一层，包含了设备驱动底层中断处理，异常处理等有关的功能。例如有一个说法，中断处理的内部不允许线程切换，线程切换只有在完成了中断处理以后才能进行，Windows内核规定只有在从所谓的“内核”层退出来时才可以进行线程切换（这里的内核就是不包含Executive的内核，即微软所称的“Kernel”，核心层），更明确一点就是IRQL从`DISPATCH_LEVEL`级别向`APC_LVEL`级别切换时，才会有线程的切换（在`DISPATCH_LEVEL`级别不允许有线程切换动作）。

### 系统调用 ###

系统调用简单说就是从用户程序调用内核层的系统函数，完成一些特定的操作，比如读取文件，写文件，打开驱动，发送网络数据包等。一般用户程序都是运行在Ring3层，而要调用的内核层的系统函数，本身是位于内核层的，这里就涉及到从用户态向内核态切换。

一般说来有三种方式可以将运行于用户空间的CPU转到系统空间：

* 中断（Interrupt）
	外部设备的中断请求到来，CPU会自动转入系统空间，从预定的地址开始执行指令。中断只会在两条指令之间发生，不会使得正在执行的指令半途而废。它属于被动行为，身不由己，完全是不可预测地异步发生。

* 异常(Exception)
	用户空间和系统空间都会发生，执行指令时的失败引起的“异常”。与中断不同，异常发生在执行某条指令过程中。但异常同样是被动行为，不可预测。当然也可以人为地在代码中安排异常而进入内核。

* 自陷（Trap）
	自陷是为了让CPU主动进入到系统空间，一般CPU都设置有“自陷”指令，系统调用通常就是通过自陷实现。与中断相比，自陷行为是主动的，由人为故意安排，相当于一次对系统空间子程序的调用。

自陷指令就包括了系统调用的int 2e，断点int3等。

Intel X86 系列CPU还提供了一种“调用门”机制，被认为是自陷机制的改进，几乎无人使用。但是在Windows系统中还是有两个“异常”是用调用门实现的，他们就是双重错误处理和机器检查。后来为了实现快速的系统调用，Intel的CPU又提供了一对指令sysenter/sysexit，称为快速系统调用机制。早先时候的Windows是通过自陷指令“int 0x2e”进入内核实现系统调用的，后面就改为快速调用方式实现，ReactOS也不例外。

下面简单记录一下Windbg调试ReadFile()函数调用的过程。这里用的调试方式可能不对（一直探索中），如果哪位有更好的方法欢迎邮件告知，感激不尽。具体怎么设置调试环境这里就不多罗嗦了，不明白的朋友可以参照ReactOS的编译，安装与调试的日志。

首先，我们找一个读取已知文件的调用（用于确定将Ring3的函数调用与Ring0层关联起来），这里以Notepad.exe进程读取文件为例。首先在桌面上建立一个 DebugTest.txt，内容随意；进入到内核调试，设置如下几个断点，其中0xb22bc020值为NotePad.exe进程的EPROCESS的值：

```
bp /p b22be7f8 kernel32!ReadFile
bp /p b22be7f8 ntdll!NtReadFile
bp /p b22be7f8 ntdll!KiFastSystemCall

bp /p b22be7f8 nt!KiFastCallEntry
bp /p b22be7f8 nt!KiFastCallEntryWithSingleStep

bp /p b22be7f8 nt!KiSystemCallTrampoline

bp /p b22be7f8 nt!NtReadFile

bp /p b22be7f8 nt!KiServiceExit

bp /p b22be7f8 nt!KiSystemCallSysExitReturn
bp /p b22be7f8 nt!KiSystemCallTrapReturn

bp /p b22be7f8 ntdll!KiFastSystemCallRet
```

> 这里给一个技巧，就是可以启动Notepad.exe进程（记事本）之后，从菜单中选择打开，直接弹出打开窗口，选中要打开的DebugTest.txt文件，方便后面点打开按钮。
> 一旦启动了内核调试，下了断点（尤其ReadFile这种大众函数），会被无数次断下来。防止这种现象出现，可以按照上述小技巧进行（虽然给出了完整调用顺序的断点们，但是看到后面就会发现，后面几个断点无法连续调试）。

断下来之后，查看一下当前打开的文件是否我们指定的文件呢，用于后面单步调试时用。从下面的调试内容看，打开的是`\Documents and Settings\Administrator\Desktop\DebugTest.txt`文件，即预期要打开的文件。

```
kd> kL
0012f7d4 0040532a kernel32!ReadFile+0x189
0012f834 00402af6 notepad!ReadText+0xba
0012f868 0040150d notepad!DoOpenFile+0x66
0012fad4 00403e72 notepad!DIALOG_FileOpen+0xbd
0012fae0 0040418c notepad!NOTEPAD_MenuCommand+0x42
0012fd1c 7c55141a notepad!NOTEPAD_WndProc+0x12c
0012fd4c 7c543680 user32!CALL_EXTERN_WNDPROC+0x1a
0012fdec 7c54675c user32!IntCallWindowProcW+0x3e0
0012fe1c 7c546184 user32!IntCallMessageProc+0x1ac
0012fe70 0040475f user32!DispatchMessageW+0x1d4
0012ff04 004066d0 notepad!wWinMain+0x26f
0012ff1c 004063cc notepad!wmain+0x20
0012ffb4 004065f1 notepad!__tmainCRTStartup+0x25c
0012ffc0 7c76e5b2 notepad!wWinMainCRTStartup+0x21
0012fff0 00000000 kernel32!BaseProcessStartup+0x42

kd> dd esp
0012f790  00000608 00000000 00000000 00000000
0012f7a0  0012f7b4 0014e798 00000019 00000000
0012f7b0  00000000 001301f4 00000005 00004000
0012f7c0  00000000 00000000 00000005 0014e790
0012f7d0  00000028 0012f834 0040532a 00000608
0012f7e0  0014e798 00000019 0012f820 00000000
0012f7f0  00000000 00000000 00000000 000a000d
0012f800  7c7ab5fe 00000000 00000000 00000000
kd> !handle 00000608
processor number 0, process b22be7f8
PROCESS b22be7f8  SessionId: 0  Cid: 02dc    Peb: 7ffd4000  ParentCid: 0248
    DirBase: 2cbec000  ObjectTable: e1774320  HandleCount:  35.
    Image: notepad.exe

Handle table at e1768000 with 35 Entries in use
0608: Object: b22a2de8  GrantedAccess: 00120089 Entry: e1768c10
Object: b22a2de8  Type: (b2683710) File
    ObjectHeader: b22a2dd0 (old version)
        HandleCount: 1  PointerCount: 1
        Directory Object: 00000000  Name: \Documents and Settings\Administrator\Desktop\DebugTest.txt {Partition1}
```

继续往下走，ReadFile()中的代码这里就不说了，如果想看一下它的内部逻辑，可以参考源代码（\reactos\dll\win32\kernel32\client\file\rw.c）。对于读取txt文件这种调用ReadFile()中，它最终会调用到ntdll!NtReadFile()函数，该函数的汇编如下（该函数没有对应的源代码，是编译中动态生成的，不要妄图找到它的源码），可以看到该函数与毛德操老师分析的0.3.3的汇编有一些不一样，eax寄存器中放的系统调用号是0xBF，即191，下面调用的SharedUserData!SystemCallStub，它的值在后面也给出了，其实就是ntdll!KiFastSystemCall()函数。

```
ntdll!NtReadFile:
001b:7c952d5d b8bf000000      mov     eax,0BFh
001b:7c952d62 ba0003fe7f      mov     edx,offset SharedUserData!SystemCallStub (7ffe0300)
001b:7c952d67 ff12            call    dword ptr [edx]
001b:7c952d69 c22400          ret     24h

// SharedUserData中数据
kd> dds 7ffe0300 L2
7ffe0300  7c92c8da ntdll!KiFastSystemCall
7ffe0304  7c92c8de ntdll!KiFastSystemCallRet
```

在ntdll!KiFastSystemCall()会调用`sysenter`指令，我们在这个指令的地方下个断点，断下来后，看一下栈和函数调用栈，如下面的代码块中所示。和上面的栈相比，多了返回地址 0x7c952d69，其他内容没有变化。再看一下寄存器，可以看到eax为0xBF，edx的内容为esp的值，即0x0012f788。

```
kd> dd esp
0012f788  7c952d69 7c790ec9 00000608 00000000
0012f798  00000000 00000000 0012f7b4 0014e798
0012f7a8  00000019 00000000 00000000 001301f4
0012f7b8  00000005 00004000 00000000 00000000
0012f7c8  00000005 0014e790 00000028 0012f834
0012f7d8  0040532a 00000608 0014e798 00000019
0012f7e8  0012f820 00000000 00000000 00000000
0012f7f8  00000000 000a000d 7c7ab5fe 00000000

kd> u 7c952d69
ntdll!ZwReadFile+0xc:
7c952d69 c22400          ret     24h
kd> u 7c790ec9
kernel32!ReadFile+0x189 [d:\reactos\sources\reactos\dll\win32\kernel32\client\file\rw.c @ 194]:
7c790ec9 8945fc          mov     dword ptr [ebp-4],eax
7c790ecc 817dfc03010000  cmp     dword ptr [ebp-4],103h
7c790ed3 751d            jne     kernel32!ReadFile+0x1b2 (7c790ef2)
7c790ed5 6a00            push    0
7c790ed7 6a00            push    0
7c790ed9 8b5508          mov     edx,dword ptr [ebp+8]
7c790edc 52              push    edx
7c790edd ff15a8c37a7c    call    dword ptr [kernel32!_imp__NtWaitForSingleObject (7c7ac3a8)]

kd> r
eax=000000bf ebx=7ffd5000 ecx=00000608 edx=0012f788 esi=00000000 edi=00000000
eip=7c92c8dc esp=0012f788 ebp=0012f7d4 iopl=0         nv up ei pl zr na pe nc
cs=001b  ss=0023  ds=0023  es=0023  fs=003b  gs=0000             efl=00000246
ntdll!KiFastSystemCall+0x2:
001b:7c92c8dc 0f34            sysenter
```

继续执行，`sysenter`指令执行之后，则进入 `nt!KiFastCallEntryWithSingleStep`函数中。该函数与`nt!KiFastCallEntry`函数内容基本相同，从ReactOS源码中找一下着两个函数，代码如下，内容为宏定义。以KiFastCallEntryWithSingleStep()为例解释一下代码。第一行FPO是设定给编译器，将调试信息直接记录到 `.debug$F`区段中。第二行，KiEnterTrap是一个宏，用于构建自陷框架；第三行KiCallHandler也为宏，调用处理函数。为了完整性，这里将宏的代码放到下面，在里面给出注释（或翻译原文注释）。KiFastCallEntry函数所在源文件为`reactos\ntoskrnl\ke\i386\trap.s`，而函数其中用到的宏的定义文件`reactos\ntoskrnl\include\internal\i386\asmmacro.S`中。

```
PUBLIC _KiFastCallEntry
.PROC _KiFastCallEntry
    FPO 0, 0, 0, 0, 1, FRAME_TRAP
    KiEnterTrap (KI_FAST_SYSTEM_CALL OR KI_NONVOLATILES_ONLY OR KI_DONT_SAVE_SEGS)
    KiCallHandler @KiSystemServiceHandler@8
.ENDP

PUBLIC _KiFastCallEntryWithSingleStep
.PROC _KiFastCallEntryWithSingleStep
    FPO 0, 0, 0, 0, 1, FRAME_TRAP
    KiEnterTrap (KI_FAST_SYSTEM_CALL OR KI_NONVOLATILES_ONLY OR KI_DONT_SAVE_SEGS)
    or dword ptr [ecx + KTRAP_FRAME_EFLAGS], EFLAGS_TF
    KiCallHandler @KiSystemServiceHandler@8
.ENDP
```

```
MACRO(KiEnterTrap, Flags)		// 传进来的参数表示 快速系统调用，不仅仅是Non Volatiles，不要保存寄存器
    LOCAL kernel_trap
    LOCAL not_v86_trap
    LOCAL set_sane_segs

    /* Check what kind of trap frame this trap requires  校验本地调用需要哪一类的自陷帧*/
    if (Flags AND KI_FAST_SYSTEM_CALL)	// 快速系统调用

        /* SYSENTER requires us to build a complete ring transition trap frame */
        // sysenter指令要求建立一个完整的级别转换 自陷帧，帧大小 0x68 = 104 （26*4）
        FrameSize = KTRAP_FRAME_EIP

        /* Fixup fs. cx is free to clobber cx可以自由使用，用于修复fs寄存器，指向内核PCR段（30h）解释见后文*/
        mov cx, KGDT_R0_PCR
        mov fs, cx

        /* Get pointer to the TSS 获取TSS的指针 偏移值为 40h */
        mov ecx, fs:[KPCR_TSS]

        /* Get a stack pointer 获取TSS中栈指针，KTSS结构体如后文所示 4h，即成员Esp0 */
        mov esp, [ecx + KTSS_ESP0]

        /* Set up a fake hardware trap frame 设置一个虚构的硬件自陷帧，上述并未进行栈操作，依旧保持Ring3进入时栈的情况。 */
        push KGDT_R3_DATA or RPL_MASK
        push edx
        pushfd
        push KGDT_R3_CODE or RPL_MASK
        push dword ptr ds:[KUSER_SHARED_SYSCALL_RET]	// 返回地址 ntdll!KiFastSystemCallRet 函数

    elseif (Flags AND KI_SOFTWARE_TRAP)

        /* Software traps need a complete non-ring transition trap frame 软件自陷 需要一个完整的非层级 转换自陷帧*/
        FrameSize = KTRAP_FRAME_ESP

        /* Software traps need to get their EIP from the caller's frame 从调用者栈帧中获取 EIP，即返回地址 */
        pop eax

    elseif (Flags AND KI_PUSH_FAKE_ERROR_CODE)

        /* If the trap doesn't have an error code, we'll make space for it */
        FrameSize = KTRAP_FRAME_EIP

    else

        /* The trap already has an error code, so just make space for the rest */
        FrameSize = KTRAP_FRAME_ERROR_CODE

    endif

    /* Make space for this frame 为栈帧预留空间 */
    sub esp, FrameSize

    /* Save nonvolatile registers 保存非可失寄存器 */
    mov [esp + KTRAP_FRAME_EBP], ebp		// 60h
    mov [esp + KTRAP_FRAME_EBX], ebx		// 5ch
    mov [esp + KTRAP_FRAME_ESI], esi		// 58h
    mov [esp + KTRAP_FRAME_EDI], edi		// 54h

    /* Save eax for system calls, for use by the C handler 为系统调用保存eax，eax内容为系统调用号*/
    mov [esp + KTRAP_FRAME_EAX], eax		// 44h

    /* Does the caller want nonvolatiles only? */
    if (NOT (Flags AND KI_NONVOLATILES_ONLY))	// 没有 KI_NONVOLATILES_ONLY标记，系统调用是有该标记的
        /* Otherwise, save the volatiles as well */
        mov [esp + KTRAP_FRAME_ECX], ecx	// 40h
        mov [esp + KTRAP_FRAME_EDX], edx	// 3Ch
    endif

    /* Save segment registers? 保存段寄存器 */
    if (Flags AND KI_DONT_SAVE_SEGS)		// 系统调用也有该标记

        /* Initialize TrapFrame segment registers with sane values */
        mov eax, KGDT_R3_DATA OR 3
        mov ecx, fs
        mov [esp + KTRAP_FRAME_DS], eax		// 38h
        mov [esp + KTRAP_FRAME_ES], eax		// 34h
        mov [esp + KTRAP_FRAME_FS], ecx		// 50h
        mov dword ptr [esp + KTRAP_FRAME_GS], 0	// 30h  GS

    else

        /* Check for V86 mode */
        test byte ptr [esp + KTRAP_FRAME_EFLAGS + 2], (EFLAGS_V86_MASK / HEX(10000))
        jz not_v86_trap

            /* Restore V8086 segments into Protected Mode segments */
            mov eax, [esp + KTRAP_FRAME_V86_DS]
            mov ecx, [esp + KTRAP_FRAME_V86_ES]
            mov [esp + KTRAP_FRAME_DS], eax
            mov [esp + KTRAP_FRAME_ES], ecx
            mov eax, [esp + KTRAP_FRAME_V86_FS]
            mov ecx, [esp + KTRAP_FRAME_V86_GS]
            mov [esp + KTRAP_FRAME_FS], eax
            mov [esp + KTRAP_FRAME_GS], ecx
            jmp set_sane_segs

        not_v86_trap:

            /* Save segment selectors */
            mov eax, ds
            mov ecx, es
            mov [esp + KTRAP_FRAME_DS], eax
            mov [esp + KTRAP_FRAME_ES], ecx
            mov eax, fs
            mov ecx, gs
            mov [esp + KTRAP_FRAME_FS], eax
            mov [esp + KTRAP_FRAME_GS], ecx

    endif

set_sane_segs:
    /* Load correct data segments 加载正确的数据 段寄存器 */
    mov ax, KGDT_R3_DATA OR RPL_MASK
    mov ds, ax
    mov es, ax

    /* Fast system calls have fs already fixed 快速系统调用已经将FS寄存器修正，指向了KPCR 结构体了 */
    if (Flags AND KI_FAST_SYSTEM_CALL)

        /* Enable interrupts and set a sane FS value 开启中断，设置FS值*/
        or dword ptr [esp + KTRAP_FRAME_EFLAGS], EFLAGS_INTERRUPT_MASK
        mov dword ptr [esp + KTRAP_FRAME_FS], KGDT_R3_TEB or RPL_MASK

        /* Set sane active EFLAGS */
        push 2
        popfd

        /* Point edx to the usermode parameters 处理参数，将edx内容加8字节，指向Ring3层的参数列表 */
        add edx, 8
    else
        /* Otherwise fix fs now */
        mov ax, KGDT_R0_PCR
        mov fs, ax
    endif

#if DBG
    /* Keep the frame chain intact */
    mov eax, [esp + KTRAP_FRAME_EIP]		// 68h
    mov [esp + KTRAP_FRAME_DEBUGEIP], eax	// 04h
    mov [esp + KTRAP_FRAME_DEBUGEBP], ebp	// 00h
    mov ebp, esp
#endif

    /* Set parameter 1 (ECX) to point to the frame 将ECX设置为指向 自陷帧 */
    mov ecx, esp

    /* Clear direction flag 清除数据传送方向 */
    cld

ENDM

MACRO(KiCallHandler, Handler)
#if DBG
    /* Use a call to get the return address for back traces */
    call Handler
#else
    /* Use the faster jmp */
    jmp Handler
#endif
    nop
ENDM
```

内核中fs内容被初始化为 30h，转换为GDT的索引为6，如下为GDT索引为6（第七项）的项目的内容，如下，很明显是地址位于0xFFDFF000，范围为0～0x1FFF的内存区块，它内容其实就是`nt!_KPCR`结构体（该内存块中还有其他内容）。解析一下该结构体，如下。TSS的指针位于`nt!_KPCR`结构体0x40的位置。

```
kd> dg 30
                                  P Si Gr Pr Lo
Sel    Base     Limit     Type    l ze an es ng Flags
---- -------- -------- ---------- - -- -- -- -- --------
0030 ffdff000 00001fff Data RW Ac 0 Bg Pg P  Nl 00000c93
```

```
kd> dt nt!_KPCR ffdff000
   +0x000 NtTib            : _NT_TIB
   +0x000 Used_ExceptionList : 0xf70c1c1c _EXCEPTION_REGISTRATION_RECORD
   +0x004 Used_StackBase   : (null)
   +0x008 Spare2           : (null)
   +0x00c TssCopy          : 0x8069d000
   +0x010 ContextSwitches  : 0x5459e3
   +0x014 SetMemberCopy    : 1
   +0x018 Used_Self        : 0x7ffdf000
   +0x01c SelfPcr          : 0xffdff000 _KPCR	// 即_KPCR在内存中的地址，本次调试中即 0xffdff000。
   +0x020 Prcb             : 0xffdff120 _KPRCB	// 它的位置在_KPCR结构体基址 + 0x120 的位置上
   +0x024 Irql             : 0 ''
   +0x028 IRR              : 0
   +0x02c IrrActive        : 0
   +0x030 IDR              : 0xffff2078
   +0x034 KdVersionBlock   : 0x8058f248
   +0x038 IDT              : 0x806a0400 _KIDTENTRY
   +0x03c GDT              : 0x806a0000 _KGDTENTRY
   +0x040 TSS              : 0x8069d000 _KTSS
   +0x044 MajorVersion     : 1
   +0x046 MinorVersion     : 1
   +0x048 SetMember        : 1
   +0x04c StallScaleFactor : 0xead
   +0x050 SpareUnused      : 0 ''
   +0x051 Number           : 0 ''
   +0x052 Spare0           : 0 ''
   +0x053 SecondLevelCacheAssociativity : 0 ''
   +0x054 VdmAlert         : 0
   +0x058 KernelReserved   : [14] 0
   +0x090 SecondLevelCacheSize : 0
   +0x094 HalReserved      : [16] 0
kd> dt nt!_KPRCB 0xffdff120
   +0x000 MinorVersion     : 1
   +0x002 MajorVersion     : 1
   +0x004 CurrentThread    : 0xb22bdc30 _KTHREAD
   +0x008 NextThread       : (null) 
   +0x00c IdleThread       : 0x80599370 _KTHREAD
   +0x010 Number           : 0 ''
   +0x011 Reserved         : 0 ''
   +0x012 BuildType        : 3
   +0x014 SetMember        : 1
   +0x018 CpuType          : 0x6 ''
   +0x019 CpuID            : 0x1 ''
   +0x01a CpuStep          : 0xa09
   +0x01c ProcessorState   : _KPROCESSOR_STATE
   +0x33c KernelReserved   : [16] 0
   +0x37c HalReserved      : [16] 0
   +0x3bc PrcbPad0         : [92]  ""
   +0x418 LockQueue        : [33] _KSPIN_LOCK_QUEUE
   +0x520 NpxThread        : (null) 
   +0x524 InterruptCount   : 0x26c2
   +0x528 KernelTime       : 0x1284
   +0x52c UserTime         : 0x3f
   +0x530 DpcTime          : 3
   +0x534 DebugDpcTime     : 0
   +0x538 InterruptTime    : 0x65
   +0x53c AdjustDpcThreshold : 7
   +0x540 PageColor        : 0
   +0x544 SkipTick         : 0 ''
   +0x545 DebuggerSavedIRQL : 0 ''
   +0x546 NodeColor        : 0 ''
   +0x547 Spare1           : 0 ''
   +0x548 NodeShiftedColor : 0
   +0x54c ParentNode       : 0x805b0e40 _KNODE
   +0x550 MultiThreadProcessorSet : 1
   +0x554 MultiThreadSetMaster : 0xffdff120 _KPRCB
   +0x558 SecondaryColorMask : 0x3f
   +0x55c Sleeping         : 0
   +0x560 CcFastReadNoWait : 0
   +0x564 CcFastReadWait   : 0
   +0x568 CcFastReadNotPossible : 0
   +0x56c CcCopyReadNoWait : 0
   +0x570 CcCopyReadWait   : 0
   +0x574 CcCopyReadNoWaitMiss : 0
   +0x578 KeAlignmentFixupCount : 0
   +0x57c SpareCounter0    : 0
   +0x580 KeDcacheFlushCount : 0
   +0x584 KeExceptionDispatchCount : 0x291
   +0x588 KeFirstLevelTbFills : 0
   +0x58c KeFloatingEmulationCount : 0
   +0x590 KeIcacheFlushCount : 0
   +0x594 KeSecondLevelTbFills : 0
   +0x598 KeSystemCalls    : 0x5f4c69
   +0x59c IoReadOperationCount : 0
   +0x5a0 IoWriteOperationCount : 0
   +0x5a4 IoOtherOperationCount : 0
   +0x5a8 IoReadTransferCount : _LARGE_INTEGER 0x0
   +0x5b0 IoWriteTransferCount : _LARGE_INTEGER 0x0
   +0x5b8 IoOtherTransferCount : _LARGE_INTEGER 0x0
   +0x5c0 SpareCounter1    : [8] 0
   +0x5e0 PPLookasideList  : [16] _PP_LOOKASIDE_LIST
   +0x660 PPNPagedLookasideList : [32] _PP_LOOKASIDE_LIST
   +0x760 PPPagedLookasideList : [32] _PP_LOOKASIDE_LIST
   +0x860 PacketBarrier    : 0
   +0x864 ReverseStall     : 0
   +0x868 IpiFrame         : (null) 
   +0x86c PrcbPad2         : [52]  ""
   +0x8a0 CurrentPacket    : [3] (null) 
   +0x8ac TargetSet        : 0
   +0x8b0 WorkerRoutine    : (null) 
   +0x8b4 IpiFrozen        : 0
   +0x8b8 PrcbPad3         : [40]  ""
   +0x8e0 RequestSummary   : 0
   +0x8e4 SignalDone       : (null) 
   +0x8e8 PrcbPad4         : [56]  ""
   +0x920 DpcData          : [2] _KDPC_DATA
   +0x948 DpcStack         : 0xf7981000 
   +0x94c MaximumDpcQueueDepth : 4
   +0x950 DpcRequestRate   : 0
   +0x954 MinimumDpcRate   : 3
   +0x958 DpcInterruptRequested : 0 ''
   +0x959 DpcThreadRequested : 0 ''
   +0x95a DpcRoutineActive : 0 ''
   +0x95b DpcThreadActive  : 0 ''
   +0x95c PrcbLock         : 0
   +0x960 DpcLastCount     : 0x1ee0
   +0x964 TimerHand        : 0xc2
   +0x968 TimerRequest     : 0
   +0x96c DpcThread        : (null) 
   +0x970 DpcEvent         : _KEVENT
   +0x980 ThreadDpcEnable  : 0 ''
   +0x981 QuantumEnd       : 0 ''
   +0x982 PrcbPad50        : 0 ''
   +0x983 IdleSchedule     : 0 ''
   +0x984 DpcSetEventRequest : 0
   +0x988 PrcbPad5         : [18]  ""
   +0x99c TickOffset       : 0
   +0x9a0 CallDpc          : _KDPC
   +0x9c0 PrcbPad7         : [8] 0
   +0x9e0 WaitListHead     : _LIST_ENTRY [ 0xffdffb00 - 0xffdffb00 ]
   +0x9e8 ReadySummary     : 0x101
   +0x9ec QueueIndex       : 1
   +0x9f0 DispatcherReadyListHead : [32] _LIST_ENTRY [ 0xb2695b78 - 0xb2695b78 ]
   +0xaf0 DeferredReadyListHead : _SINGLE_LIST_ENTRY
   +0xaf4 PrcbPad72        : [11] 0
   +0xb20 ChainedInterruptList : (null) 
   +0xb24 LookasideIrpFloat : 512
   +0xb28 MmPageFaultCount : 0
   +0xb2c MmCopyOnWriteCount : 0
   +0xb30 MmTransitionCount : 242
   +0xb34 MmCacheTransitionCount : 0
   +0xb38 MmDemandZeroCount : 7575
   +0xb3c MmPageReadCount  : 0
   +0xb40 MmPageReadIoCount : 0
   +0xb44 MmCacheReadCount : 0
   +0xb48 MmCacheIoCount   : 0
   +0xb4c MmDirtyPagesWriteCount : 0
   +0xb50 MmDirtyWriteIoCount : 0
   +0xb54 MmMappedPagesWriteCount : 0
   +0xb58 MmMappedWriteIoCount : 0
   +0xb5c SpareFields0     : [1] 0
   +0xb60 VendorString     : [13]  "GenuineIntel"
   +0xb6d InitialApicId    : 0 ''
   +0xb6e LogicalProcessorsPerPhysicalProcessor : 0x1 ''
   +0xb70 MHz              : 0xd40
   +0xb74 FeatureBits      : 0x20033fff
   +0xb78 UpdateSignature  : _LARGE_INTEGER 0x19`00000000
   +0xb80 IsrTime          : _LARGE_INTEGER 0x0
   +0xb88 SpareField1      : _LARGE_INTEGER 0x0
   +0xb90 NpxSaveArea      : _FX_SAVE_AREA
   +0xda0 PowerState       : _PROCESSOR_POWER_STATE
```

```
kd> dt nt!_KTSS
   +0x000 Backlink         : Uint2B
   +0x002 Reserved0        : Uint2B
   +0x004 Esp0             : Uint4B
   +0x008 Ss0              : Uint2B
   +0x00a Reserved1        : Uint2B
   +0x00c NotUsed1         : [4] Uint4B
   +0x01c CR3              : Uint4B
   +0x020 Eip              : Uint4B
   +0x024 EFlags           : Uint4B
   +0x028 Eax              : Uint4B
   +0x02c Ecx              : Uint4B
   +0x030 Edx              : Uint4B
   +0x034 Ebx              : Uint4B
   +0x038 Esp              : Uint4B
   +0x03c Ebp              : Uint4B
   +0x040 Esi              : Uint4B
   +0x044 Edi              : Uint4B
   +0x048 Es               : Uint2B
   +0x04a Reserved2        : Uint2B
   +0x04c Cs               : Uint2B
   +0x04e Reserved3        : Uint2B
   +0x050 Ss               : Uint2B
   +0x052 Reserved4        : Uint2B
   +0x054 Ds               : Uint2B
   +0x056 Reserved5        : Uint2B
   +0x058 Fs               : Uint2B
   +0x05a Reserved6        : Uint2B
   +0x05c Gs               : Uint2B
   +0x05e Reserved7        : Uint2B
   +0x060 LDT              : Uint2B
   +0x062 Reserved8        : Uint2B
   +0x064 Flags            : Uint2B
   +0x066 IoMapBase        : Uint2B
   +0x068 IoMaps           : [1] _KiIoAccessMap
   +0x208c IntDirectionMap  : [32] UChar
```

从上述的内容可知，KiEnterTrap 在系统调用中就是构建了自陷帧，这个自陷帧的整体结构是什么样的呢？下面给出它的内容。其实代码中能看到的构建动作就是从0x64和0x78两个地方为界的两个不同块。对于INT 2Eh自陷指令，0x68～0x78这些系统栈内容是由指令自动入栈的；对于快速系统调用SYSENTER，它则不会负责这些内容，那就是上面看到的汇编代码中故意构建的自陷帧。至于V86模式的那个内容，目前不在考虑范围内。下面给出了本次调试中，各个字段对应的值，如下代码块所示。将其中的Edx字段做个验证，证明是从上面ReadFile函数中调用下来的，如下代码块中内容所示。

```
kd> dt nt!_KTRAP_FRAME 0xf70c1d64
   +0x000 DbgEbp           : 0x12f7d4
   +0x004 DbgEip           : 0x7c92c8de
   +0x008 DbgArgMark       : 0x12f788
   +0x00c DbgArgPointer    : 0
   +0x010 TempSegCs        : 0
   +0x014 TempEsp          : 0
   +0x018 Dr0              : 0
   +0x01c Dr1              : 0
   +0x020 Dr2              : 0
   +0x024 Dr3              : 0
   +0x028 Dr6              : 0x23
   +0x02c Dr7              : 0x23
   +0x030 SegGs            : 0
   +0x034 SegEs            : 0x23
   +0x038 SegDs            : 0x23
   +0x03c Edx              : 0x12f788
   +0x040 Ecx              : 0x8069d000
   +0x044 Eax              : 0xbf
   +0x048 PreviousPreviousMode : 0
   +0x04c ExceptionList    : (null) 
   +0x050 SegFs            : 0x3b
   +0x054 Edi              : 0
   +0x058 Esi              : 0
   +0x05c Ebx              : 0x7ffd5000
   +0x060 Ebp              : 0x12f7d4
   +0x064 ErrCode          : 0x146

   +0x068 Eip              : 0x7c92c8de		// R3 EIP ntdll!KiFastSystemCallRet
   +0x06c SegCs            : 0x1b			// R3 CS
   +0x070 EFlags           : 0x346			// EFLAGS
   +0x074 HardwareEsp      : 0x12f788		// R3层的栈顶，见下面对该值内容输出
   +0x078 HardwareSegSs    : 0x23			// R3 SS 段寄存器

   +0x07c V86Es            : 0
   +0x080 V86Ds            : 0
   +0x084 V86Fs            : 0
   +0x088 V86Gs            : 0
```

```
kd> dds 0012f788			// 调用到内核时栈顶内容，与前述内容相同。
0012f788  7c952d69 ntdll!ZwReadFile+0xc
0012f78c  7c790ec9 kernel32!ReadFile+0x189 [d:\reactos\sources\reactos\dll\win32\kernel32\client\file\rw.c @ 194]
0012f790  000007a0
0012f794  00000000
0012f798  00000000
0012f79c  00000000
0012f7a0  0012f7b4
0012f7a4  0014ec40
0012f7a8  00000019
0012f7ac  00000000
0012f7b0  00000000
```

至此，调试已经到了构建完自陷栈帧的内容了，ReactOS接下来是调用了`nt!KiSystemServiceHandler`函数，该函数为快速调用，一共两个参数，两个参数分别位于ECX和EDX中，如下，即刚刚构建的TRAP_FRAME和用户层栈空间中的参数列表（注意EDX指向的是返回地址，而非直接就是参数列表）。其中ECX寄存器的值0xf70c1d64即为自陷框架的帧指针，见上述自陷帧内容分析。

```
kd> r
eax=7c92c8de ebx=7ffd5000 ecx=f70c1d64 edx=0012f790 esi=00000000 edi=00000000
eip=80505850 esp=f70c1d60 ebp=f70c1d64 iopl=0         nv up di pl nz na po nc
cs=0008  ss=0010  ds=0023  es=0023  fs=0030  gs=0000             efl=00000002
nt!KiSystemServiceHandler:
80505850 8bff            mov     edi,edi
```

下面贴出ReactOS中该函数的源码，简单分析一下该函数。

```
DECLSPEC_NORETURN
VOID
FASTCALL
KiSystemServiceHandler(IN PKTRAP_FRAME TrapFrame,
                       IN PVOID Arguments)
{
    PKTHREAD Thread;
    PKSERVICE_TABLE_DESCRIPTOR DescriptorTable;
    ULONG Id, Offset, StackBytes;
    NTSTATUS Status;
    PVOID Handler;
    ULONG SystemCallNumber = TrapFrame->Eax;	// TrapFrame参数即上面汇编构建的自陷帧

    /* Get the current thread 获取当前线程的 KTHREAD结构体指针 */
    Thread = KeGetCurrentThread();

    /* Set debug header 设置调试值，将自陷帧中的调试相关的寄存器设置值，例如DbgEip DbgEbp PreviousPreviousMode等 */
    KiFillTrapFrameDebug(TrapFrame);

    /* Chain trap frames 将自陷帧连起来，Edx成员赋值为 线程当前的自陷帧字段，现在有了新的自陷帧，要保存老的自陷帧（如果有的话） */
    TrapFrame->Edx = (ULONG_PTR)Thread->TrapFrame;

    /* No error code 清除错误码 */
    TrapFrame->ErrCode = 0;

    /* Save previous mode 保存 先前模式*/
    TrapFrame->PreviousPreviousMode = Thread->PreviousMode;

    /* Save the SEH chain and terminate it for now 保存SEH链，将线程SEH链置空 */
    TrapFrame->ExceptionList = KeGetPcr()->NtTib.ExceptionList;
    KeGetPcr()->NtTib.ExceptionList = EXCEPTION_CHAIN_END;

    /* Default to debugging disabled */
    TrapFrame->Dr7 = 0;

    /* Check if the frame was from user mode */
    if (KiUserTrap(TrapFrame))
    {
        /* Check for active debugging 检验是否被调试，如果被调试将DR* 寄存器保存，且将内核调试的DR*还原到寄存器中 */
        if (KeGetCurrentThread()->Header.DebugActive & 0xFF)
        {
            /* Handle debug registers */
            KiHandleDebugRegistersOnTrapEntry(TrapFrame);
        }
    }

    /* Set thread fields 将KTREAD中的 自陷帧指针字段 指向当前的自陷帧 且设置 先前模式 */
    Thread->TrapFrame = TrapFrame;
    Thread->PreviousMode = KiUserTrap(TrapFrame);

    /* Enable interrupts 开启中断 */
    _enable();

    /* Decode the system call number 解析系统调用号 */
    /*
    	系统调用号，正常是小于 0x1000，而Win32Sys的系统调用号大于0x1000，位于不同的系统调用表中
        第一个表达式，将确定是调用0号表，还是1号表，Id即系统调用号
    */
    Offset = (SystemCallNumber >> SERVICE_TABLE_SHIFT) & SERVICE_TABLE_MASK;
    Id = SystemCallNumber & SERVICE_NUMBER_MASK;

    /* Get descriptor table 获取本次要使用的系统调用表 基址 */
    DescriptorTable = (PVOID)((ULONG_PTR)Thread->ServiceTable + Offset);

    /* Validate the system call number 校验系统调用号的有效性 */
    if (__builtin_expect(Id >= DescriptorTable->Limit, 0))
    {
        /* Check if this is a GUI call 校验是否是GUI调用 */
        if (!(Offset & SERVICE_TABLE_TEST))
        {
            /* Fail the call */
            Status = STATUS_INVALID_SYSTEM_SERVICE;
            goto ExitCall;
        }

        /* Convert us to a GUI thread -- must wrap in ASM to get new EBP */
        /* 将我们的线程转成GUI线程，必须包含在ASM中，获取新的EBP */
        Status = KiConvertToGuiThread();

        /* Reload trap frame and descriptor table pointer from new stack 重新加载栈帧和描述符表指针 */
        TrapFrame = *(volatile PVOID*)&Thread->TrapFrame;
        DescriptorTable = (PVOID)(*(volatile ULONG_PTR*)&Thread->ServiceTable + Offset);

        if (!NT_SUCCESS(Status))
        {
            /* Set the last error and fail */
            goto ExitCall;
        }

        /* Validate the system call number again */
        if (Id >= DescriptorTable->Limit)
        {
            /* Fail the call */
            Status = STATUS_INVALID_SYSTEM_SERVICE;
            goto ExitCall;
        }
    }

    /* Check if this is a GUI call 校验是否是GUI调用 */
    if (__builtin_expect(Offset & SERVICE_TABLE_TEST, 0))
    {
        /* Get the batch count and flush if necessary */
        if (NtCurrentTeb()->GdiBatchCount) KeGdiFlushUserBatch();
    }

    /* Increase system call count 增加系统调用计数 */
    KeGetCurrentPrcb()->KeSystemCalls++;

    /* FIXME: Increase individual counts on debug systems */
    //KiIncreaseSystemCallCount(DescriptorTable, Id);

    /* Get stack bytes 获取所有参数占据的空间大小，单位是字节 */
    StackBytes = DescriptorTable->Number[Id];

    /* Probe caller stack */
    if (__builtin_expect((Arguments < (PVOID)MmUserProbeAddress) && !(KiUserTrap(TrapFrame)), 0))
    {
        /* Access violation */
        UNIMPLEMENTED_FATAL();
    }

    /* Call pre-service debug hook  */
    KiDbgPreServiceHook(SystemCallNumber, Arguments);

    /* Get the handler and make the system call 获取处理函数地址，并且调用 系统调用，其中KiSystemCallTrampoline()在后面给出了 */
    Handler = (PVOID)DescriptorTable->Base[Id];
    Status = KiSystemCallTrampoline(Handler, Arguments, StackBytes);

    /* Call post-service debug hook */
    Status = KiDbgPostServiceHook(SystemCallNumber, Status);

    /* Make sure we're exiting correctly */
    KiExitSystemCallDebugChecks(Id, TrapFrame);

    /* Restore the old trap frame */
ExitCall:
    Thread->TrapFrame = (PKTRAP_FRAME)TrapFrame->Edx;	// 将老的 自陷帧恢复到 KTHREAD 结构中

    /* Exit from system call 退出系统调用  */
    KiServiceExit(TrapFrame, Status);
}

FORCEINLINE
NTSTATUS
KiSystemCallTrampoline(IN PVOID Handler,
                       IN PVOID Arguments,
                       IN ULONG StackBytes)
{
    __asm
    {
        mov ecx, StackBytes
        mov esi, Arguments
        mov eax, Handler
        sub esp, ecx
        mov edi, esp
        shr ecx, 2
        rep movsd
        call eax
    }
    /* Return with result in EAX */
}

```

其中的ServiceTable指向的是一个`nt!_KSERVICE_TABLE_DESCRIPTOR`结构的表，大小为2，第一项有内容，第二项的Limit为0，其实就是没有内容，如下所示。这个表如何查呢？其实就是根据函数调用号做索引值，取Base数组的元素，Number数组的元素，两个分别为对应的系统调用函数以及该函数的参数个数。如下面给出的函数`nt!NtAcceptConnectPort`，它是表的第一个元素，对应参数内存用量为 0x18字节，即6个参数，在下面给出了这个函数的原型。注意到上面会提到Win32Sys的系统调用，它存在于一个独立的`nt!_KSERVICE_TABLE_DESCRIPTOR`结构中，用`KeServiceDescriptorTableShadow`全局变量单独指向，而普通的系统调用位于`KeServiceDescriptorTable`全局变量指向的上述结构中。一个线程初始化完成后，它的系统调用表的字段`ServiceTable`（KTHREAD结构中）的值等于全局变量`KeServiceDescriptorTable`的值。

```
kd> dt nt!KSERVICE_TABLE_DESCRIPTOR 0x805b0aa0
   +0x000 Base             : 0x80573c68  -> 0x8048bc40	// 保存的是一个函数数组的指针
   +0x004 Count            : 0xb2696008  -> 0
   +0x008 Limit            : 0x128						// 一共有多少表项
   +0x00c Number           : 0x80574108  "???"			// 每一个函数对应的参数个数，是一个UCHAR的数组

kd> dt 805b0aac  _KSERVICE_TABLE_DESCRIPTOR
nt!_KSERVICE_TABLE_DESCRIPTOR
   +0x000 Base             : 0x80574108  -> 0x2c2c2018
   +0x004 Count            : 0xf76da868  -> 0xf7674c90
   +0x008 Limit            : 0
   +0x00c Number           : 0x000002aa  "--- memory read error at address 0x000002aa ---"
```

```
kd> db 0x80574108 L1
80574108  18                                               .
kd> dds 0x80573c68 L1
80573c68  8048bc40 nt!NtAcceptConnectPort [d:\reactos\sources\reactos\ntoskrnl\lpc\complete.c @ 46]
```

```
NTSYSCALLAPI
NTSTATUS
NTAPI
NtAcceptConnectPort(
    _Out_ PHANDLE PortHandle,
    _In_opt_ PVOID PortContext,
    _In_ PPORT_MESSAGE ConnectionRequest,
    _In_ BOOLEAN AcceptConnection,
    _Inout_opt_ PPORT_VIEW ServerView,
    _Out_opt_ PREMOTE_PORT_VIEW ClientView
);
```

在校验完系统调用号和参数的有效性后，就该到调用对应的函数了。这里是通过调用 `KiSystemCallTrampoline` 完成的，它里面的汇编也比较简单，根据参数个数，将参数从用户栈上复制到系统栈上，然后调用对应的函数即可，不再多说。具体的`nt!ReadFile`的逻辑不在此处学习了，后面可以单独阅读。

调用完函数，接下来就是返回了。先恢复线程的陷阱帧，因为当前这个陷阱帧马上就会被丢弃掉，要将先前的陷阱帧赋值到线程的KTHREAD结构体的TrapFrame字段中。然后调用 `KiServiceExit`函数。该函数有两个参数，一个是陷阱帧指针，另外一个是返回值Status。函数的具体操作见函数中的注释。

> 这里做个说明，在前面的调试内容还是比较容易搞出来。但是当退出系统调用时，断点断到KiServiceExit函数还是比较容易的，要从它往下调试时，发现不是出现双重错误，就是其他的一些异常，总是无法调试。所以，下面的内容没有真实调试出来的数据了，只做一下简单的代码分析。
> 这块不太确定是否由于调试器的事件影响了系统的退出的一些调用，造成了出现双重错误。如果有人知道，或有更好的调试方法，欢迎您写邮件告知，感激不尽。后面如果了解了原因，或是有了更好的方法可以继续追踪调试，再补一下这块的调试数据。

其中有一个地方要注意一下，就是函数`KiCheckForApcDelivery`的调用，它用作APC分发，这个函数后面想专门总结一下APC的机制，再详细解析吧！

```
DECLSPEC_NORETURN
VOID
FASTCALL
KiServiceExit(IN PKTRAP_FRAME TrapFrame,
              IN NTSTATUS Status)
{
    ASSERT((TrapFrame->EFlags & EFLAGS_V86_MASK) == 0);
    ASSERT(!KiIsFrameEdited(TrapFrame));

    /* Copy the status into EAX 将返回值复制给陷阱帧的 Eax寄存器 */
    TrapFrame->Eax = Status;

    /* Common trap exit code 普通自陷 退出代码 */
    KiCommonExit(TrapFrame, FALSE);

    /* Restore previous mode 恢复当前线程 的先前模式，被保存在了ProviousPreviousMode */
    KeGetCurrentThread()->PreviousMode = (CCHAR)TrapFrame->PreviousPreviousMode;

    /* Check for user mode exit 校验用户模式的 退出 */
    if (KiUserTrap(TrapFrame))
    {
        /* Check if we were single stepping 校验是否单步执行 */
        if (TrapFrame->EFlags & EFLAGS_TF)
        {
            /* Must use the IRET handler */
            KiSystemCallTrapReturn(TrapFrame);
        }
        else
        {
            /* We can use the sysexit handler */
            KiFastCallExitHandler(TrapFrame);
            UNREACHABLE;
        }
    }

    /* Exit to kernel mode */
    KiSystemCallReturn(TrapFrame);
}

FORCEINLINE
VOID
KiCommonExit(IN PKTRAP_FRAME TrapFrame, BOOLEAN SkipPreviousMode)
{
    /* Disable interrupts until we return 关闭中断 */
    _disable();

    /* Check for APC delivery 校验APC分发 */
    KiCheckForApcDelivery(TrapFrame);

    /* Restore the SEH handler chain 恢复 SEH处理函数链，将用户层注册的异常链恢复回去 */
    KeGetPcr()->NtTib.ExceptionList = TrapFrame->ExceptionList;

    /* Check if there are active debug registers 校验调试寄存器是否被设置，被设置了要进行保存 和 恢复 */
    if (__builtin_expect(TrapFrame->Dr7 & ~DR7_RESERVED_MASK, 0))
    {
        /* Check if the frame was from user mode or v86 mode */
        if (KiUserTrap(TrapFrame) ||
            (TrapFrame->EFlags & EFLAGS_V86_MASK))
        {
            /* Handle debug registers 处理调试寄存器 */
            KiHandleDebugRegistersOnTrapExit(TrapFrame);
        }
    }

    /* Debugging checks 如果是调试模式，跳过先前模式字段的设置 */
    KiExitTrapDebugChecks(TrapFrame, SkipPreviousMode);
}
```

根据是否是单步执行，最终退出时，有两种退出方式。一种是单步的话调用`KiSystemCallTrapReturn`，而非单步的话调用`KiFastCallExitHandler`，它其实是一个全局变量，它指向的内容为`KiSystemCallSysExitReturn`。如下给出了两个函数的定义，他们又是来自同一个宏定义，只是编译标记不同。先看一下这个公共的宏定义。两个函数的参数都是指向自陷帧的指针。

```
KiTrapExitStub KiSystemCallSysExitReturn, (KI_RESTORE_EAX OR KI_RESTORE_FS OR KI_RESTORE_EFLAGS OR KI_EXIT_SYSCALL)
KiTrapExitStub KiSystemCallTrapReturn,    (KI_RESTORE_EAX OR KI_RESTORE_FS OR KI_EXIT_IRET)


MACRO(KiTrapExitStub, Name, Flags)
    LOCAL ret8_instruction
    LOCAL not_nested_int

PUBLIC @&Name&@4
@&Name&@4:

    if (Flags AND KI_EXIT_RET8) OR (Flags AND KI_EXIT_IRET)

        /* This is the IRET frame */
        OffsetEsp = KTRAP_FRAME_EIP

    elseif (Flags AND KI_RESTORE_EFLAGS)

        /* We will pop EFlags off the stack 赋值EFlags在Trap_Frame中的偏移值 */
        OffsetEsp = KTRAP_FRAME_EFLAGS

    else

        OffsetEsp = 0

    endif

    if (Flags AND KI_EDITED_FRAME)

        /* Load the requested ESP */
        mov esp, [ecx + KTRAP_FRAME_TEMPESP]

        /* Put return address on the new stack */
        push [ecx + KTRAP_FRAME_EIP]

        /* Put EFLAGS on the new stack */
        push [ecx + KTRAP_FRAME_EFLAGS]

    else

        /* Point esp to an appropriate member of the frame esp指向自陷帧中适当的成员 */
        lea esp, [ecx + OffsetEsp]

    endif

    /* Restore non volatiles 恢复非易失的寄存器 */
    mov ebx, [ecx + KTRAP_FRAME_EBX]
    mov esi, [ecx + KTRAP_FRAME_ESI]
    mov edi, [ecx + KTRAP_FRAME_EDI]
    mov ebp, [ecx + KTRAP_FRAME_EBP]

    if (Flags AND KI_RESTORE_EAX)

        /* Restore eax 注意此处恢复了EAX寄存器，只是针对特定的自陷/异常或中断 */
        mov eax, [ecx + KTRAP_FRAME_EAX]

    endif

    if (Flags AND KI_RESTORE_ECX_EDX)

        /* Restore volatiles */
        mov edx, [ecx + KTRAP_FRAME_EDX]
        mov ecx, [ecx + KTRAP_FRAME_ECX]

    elseif (Flags AND KI_EXIT_JMP)

        /* Load return address into edx */
        mov edx, [esp - OffsetEsp + KTRAP_FRAME_EIP]

    elseif (Flags AND KI_EXIT_SYSCALL)	// 快速系统调用退出

        /* Set sysexit parameters 设置sysexit指令的参数，即将返回地址设置到EDX中，用户层栈指针设置到ECX中 */
        mov edx, [esp - OffsetEsp + KTRAP_FRAME_EIP]
        mov ecx, [esp - OffsetEsp + KTRAP_FRAME_ESP]

        /* Keep interrupts disabled until the sti / sysexit */
        and byte ptr [esp - OffsetEsp + KTRAP_FRAME_EFLAGS + 1], NOT (EFLAGS_INTERRUPT_MASK / HEX(100))

    endif

    if (Flags AND KI_RESTORE_SEGMENTS)

        /* Restore segments for user mode 恢复用户模式下的段寄存器 */
        mov ds, [esp - OffsetEsp + KTRAP_FRAME_DS]
        mov es, [esp - OffsetEsp + KTRAP_FRAME_ES]
        mov gs, [esp - OffsetEsp + KTRAP_FRAME_GS]

    endif

    if ((Flags AND KI_RESTORE_FS) OR (Flags AND KI_RESTORE_SEGMENTS))

        /* Restore user mode FS 恢复用户模式的FS寄存器，它在用户模式下指向的是 TIB，即TEB，这里只是将进入内核时保存的FS值恢复回去*/
        mov fs, [esp - OffsetEsp + KTRAP_FRAME_FS]

    endif

    if (Flags AND KI_RESTORE_EFLAGS)

        if (Flags AND KI_EXIT_RET8)

            /* Check if we return from a nested interrupt, i.e. an interrupt
               that occurred in the ret8 return path between restoring
               EFLAGS and returning with the ret instruction. */
            // 这里会校验是否从嵌套的中断中返回，即在ret8返回路径上恢复EFLAGS寄存器和ret指令之间到来中断
            cmp dword ptr [esp], offset ret8_instruction
            jne not_nested_int

            /* This is a nested interrupt, so we have 2 IRET frames.
               Skip the first, and go directly to the previous return address.
               Do not pass Go. Do not collect $200 */
            // 对于嵌套中断，这里就有了两个iret帧，跳过第一个iret帧，直接进入到先前的返回地址
            add esp, 12

not_nested_int:
            /* We are at the IRET frame, so push EFLAGS first 在iret帧中， 首先压栈EFLAGS */
            push dword ptr [esp + 8]

        endif

        /* Restore EFLAGS 出栈EFLAGS */
        popfd

    endif

    if (Flags AND KI_EXIT_SYSCALL)

        /* Enable interrupts and return to user mode.
           Both must follow directly after another to be "atomic". */
        // 开启中断，返回到用户模式，接下来的必须是一个原子指令
        sti
        sysexit

    elseif (Flags AND KI_EXIT_IRET)

        /* Return with iret */
        iretd

    elseif (Flags AND KI_EXIT_JMP)

        /* Return to kernel mode with a jmp */
        jmp edx

    elseif (Flags AND KI_EXIT_RET8)

        /* Return to kernel mode with a ret 8 返回到内核模式 */
ret8_instruction:
        ret 8

    elseif (Flags AND KI_EXIT_RET)

        /* Return to kernel mode with a ret */
        ret

    endif

ENDM
```

再直接一点，直接看这两个函数的汇编代码，如下所示。ECX中保存了TrapFrame指针，即指向自陷帧；ecx+70h即调整系统栈到调用之前；恢复非易失寄存器；恢复EAX，之前的函数`KiServiceExit`将系统调用结果Status值放入了自陷帧的Eax字段中，在返回用户模式之前，要将该值恢复给EAX寄存器，作为返回值；

```
nt!KiSystemCallSysExitReturn:
80403edd 8d6170          lea     esp,[ecx+70h]				// esp指向 EFlags字段
80403ee0 8b595c          mov     ebx,dword ptr [ecx+5Ch]
80403ee3 8b7158          mov     esi,dword ptr [ecx+58h]
80403ee6 8b7954          mov     edi,dword ptr [ecx+54h]
80403ee9 8b6960          mov     ebp,dword ptr [ecx+60h]
80403eec 8b4144          mov     eax,dword ptr [ecx+44h]
80403eef 8b5424f8        mov     edx,dword ptr [esp-8]		// 返回地址 Eip
80403ef3 8b4c2404        mov     ecx,dword ptr [esp+4]		// Esp字段
80403ef7 80642401fd      and     byte ptr [esp+1],0FDh		// 调整标记位，将中断位屏蔽掉
80403efc 8e6424e0        mov     fs,word ptr [esp-20h]		// -20h 为之前SegFS字段
80403f00 9d              popfd								// 将EFlags字段出栈，esp+4
80403f01 fb              sti								// 开中断
80403f02 0f35            sysexit							// 退出内核模式


nt!KiSystemCallTrapReturn:
80403f04 8d6168          lea     esp,[ecx+68h]				// 指向返回地址Eip字段
80403f07 8b595c          mov     ebx,dword ptr [ecx+5Ch]	// 恢复非易失寄存器
80403f0a 8b7158          mov     esi,dword ptr [ecx+58h]
80403f0d 8b7954          mov     edi,dword ptr [ecx+54h]
80403f10 8b6960          mov     ebp,dword ptr [ecx+60h]
80403f13 8b4144          mov     eax,dword ptr [ecx+44h]	// 恢复返回值到EAX中
80403f16 8e6424e8        mov     fs,word ptr [esp-18h]		// 将原始FS用户模式的FS值恢复到FS寄存器，指向TIB结构基址
80403f1a cf              iretd								// 返回
```

其中`sysexit`指令完成的工作包括将寄存器EDX内容置入EIP中，将ECX内容置入ESP寄存器，自动根据SYSENTER_CS_MSR修改CS和SS段寄存器的内容。由于前面设置的返回地址为`KiFastSystemCallRet`，所以调用sysexit后，则进入到函数KiFastSystemCallRet中继续执行。下面随意给出一个函数返回时的sysexit执行后的寄存器情况，如下。可知ECX与ESP值相同，EDX与EIP值相同。

```
kd> r
eax=00000001 ebx=7ffd4000 ecx=0012f8fc edx=7c92c8de esi=00000000 edi=00000000
eip=7c92c8de esp=0012f8fc ebp=0012f9c0 iopl=0         nv up ei pl nz na po nc
cs=001b  ss=0023  ds=0023  es=0023  fs=003b  gs=0000             efl=00000202
ntdll!KiFastSystemCallRet:
001b:7c92c8de c3              ret
```

对于`nt!KiSystemCallTrapReturn`函数来说，它也是做了恢复寄存器的操作，但是最后通过iretd返回，该指令其实是int 2eh指令的逆指令，即从栈中弹出EIP，CS，EFLAGS，ESP，SS等寄存器，然后跳转到EIP指定地址继续执行。其实在调试的情况下，sysenter形式进入的内核，但是其实返回时在`KiServiceExit`函数中调用的是本函数，至于原因还不太清楚。调用`nt!KiSystemCallTrapReturn`的条件是TrapFrame中的EFlags字段有单步标识，则通过iretd的形式返回，即调用到本函数中了。如下是NtReadFile调用返回时的情况，返回后的地址为NtReadFile+0xC的指令，寄存器内容可以看出，EAX作为返回值为0（表示成功）。查看内核中系统栈的内容，可以发现EIP，ESP，SS，CS，EFLAGS等值可以对上。

```
kd> kL
ChildEBP RetAddr
0012f788 7c790ec9 ntdll!ZwReadFile+0xc
0012f7d4 0040532a kernel32!ReadFile+0x189
0012f834 00402af6 notepad!ReadText+0xba
0012f868 0040150d notepad!DoOpenFile+0x66
0012fad4 00403e72 notepad!DIALOG_FileOpen+0xbd
0012fae0 0040418c notepad!NOTEPAD_MenuCommand+0x42
0012fd1c 7c55141a notepad!NOTEPAD_WndProc+0x12c
0012fd4c 7c543680 user32!CALL_EXTERN_WNDPROC+0x1a
0012fdec 7c54675c user32!IntCallWindowProcW+0x3e0
0012fe1c 7c546184 user32!IntCallMessageProc+0x1ac
0012fe70 0040475f user32!DispatchMessageW+0x1d4
0012ff04 004066d0 notepad!wWinMain+0x26f
0012ff1c 004063cc notepad!wmain+0x20
0012ffb4 004065f1 notepad!__tmainCRTStartup+0x25c
0012ffc0 7c76e5b2 notepad!wWinMainCRTStartup+0x21
0012fff0 00000000 kernel32!BaseProcessStartup+0x42

ntdll!NtReadFile:
001b:7c952d5d b8bf000000      mov     eax,0BFh
001b:7c952d62 ba0003fe7f      mov     edx,offset SharedUserData!SystemCallStub (7ffe0300)
001b:7c952d67 ff12            call    dword ptr [edx]
001b:7c952d69 c22400          ret     24h		// ntdll!ZwReadFile+0xc 所指向的汇编

kd> r
eax=00000000 ebx=7ffd4000 ecx=0012f788 edx=7c92c8de esi=00000000 edi=00000000
eip=7c952d69 esp=0012f78c ebp=0012f7d4 iopl=0         nv up ei pl zr na pe nc
cs=001b  ss=0023  ds=0023  es=0023  fs=003b  gs=0000             efl=00000246
ntdll!ZwReadFile+0xc:
001b:7c952d69 c22400          ret     24h
kd> dd esp
0012f78c  7c790ec9 00000334 00000000 00000000
0012f79c  00000000 0012f7b4 0015b448 00000019
0012f7ac  00000000 00000000 00000000 00000019
0012f7bc  00004000 00000000 00000000 00000005
0012f7cc  0015b440 00000028 0012f834 0040532a
0012f7dc  00000334 0015b448 00000019 0012f820
0012f7ec  00000000 00000000 00000000 00000000
0012f7fc  000a000d 7c7ab5fe 00000000 00000000
kd> !pcr
KPCR for Processor 0 at ffdff000:
    Major 1 Minor 1
	NtTib.ExceptionList: f70c9c90
	    NtTib.StackBase: 00000000
	   NtTib.StackLimit: 00000000
	 NtTib.SubSystemTib: 8069d000
	      NtTib.Version: 00e4ac62
	  NtTib.UserPointer: 00000001
	      NtTib.SelfTib: 7ffdf000

	            SelfPcr: ffdff000
	               Prcb: ffdff120
	               Irql: 00000000
	                IRR: 00000000
	                IDR: ffff2078
	      InterruptMode: 00000000
	                IDT: 806a0400
	                GDT: 806a0000
	                TSS: 8069d000

	      CurrentThread: 00000000
	         NextThread: 00000000
	         IdleThread: 00000000

	          DpcQueue:

// 内核栈中的内容情况
kd> dt nt!_KTSS 8069d000 -y Esp0
   +0x004 Esp0 : 0xf70c9de0
kd> dds 0xf70c9de0 - 0x14
f70c9dcc  7c952d6a ntdll!ZwReadFile+0xd
f70c9dd0  0000001b
f70c9dd4  00000246
f70c9dd8  0012f78c
f70c9ddc  00000023
f70c9de0  00000000
f70c9de4  00000000
f70c9de8  00000000
f70c9dec  00000000
```

至于为啥在调试时以SYSENTER进入内核，而返回时却是使用IRETD，暂时不得而知，可能后面通过阅读代码的深入可以窥探到原因。那就将此问题暂时作为疑问留下吧！！！

至此，通过调试方式将系统调用过了一遍，理了理其中的各种关系。当然了日志是一遍分析一遍整理，其中不免有一些错误。

-------------------
** 问题 **

1. 进入内核时，sysenter指令所在函数KiFastSystemCall()是什么时候放入内核和用户共享内存部分的？
2. 快速调用用的MSR寄存器是什么时候设置的？
3. IDT表是什么时机的构建的，它的内容是如何构建的？如何从Windbg查看这些内容？
	定义在trap.S汇编文件中。
4. 系统调用表如何初始化，以及Win32Sys系统调用表如何初始化？当然包括在系统调用中如何将普通线程切换为Win32线程?
5. 为何调试情况下以SYSENTER进入内核，而返回时却是使用IRETD的方式？

** 参考文档 **

1. 《Windows内核情景分析-基于ReactOS》
2. ReactOS 0.4.5版源码

** 修订历史 **

* 2017-06-25 16:19:23 完成文章
* 2017-09-06 09:28:12 修改语言表达


By Andy @ 2017-06-25 19:44:32

<div align="center">
**声明**
*“博客原创文章遵守《[知识共享许可协议](https://creativecommons.org/licenses/)》，转载文章请标明[出处](https://dbglife.github.io/)”*
</div>