---
title: Windows异常分发
date: 2017-05-21 11:33:32
tags:
- Windbg
- Exception
categories:
- 调试
---

#### 异常定义 ####

异常通常是CPU在执行指令时因为检测到预先定义的某个（或多个）条件而产生的同步事件。注意它与中断的区别。

>中断通常是有CPU外部的输入输出设备（硬件）所触发的，供外部设备通知CPU"有事情需要处理"，因此又叫做中断请求。
>中断请求的目的是希望CPU暂时停止执行当前正在执行的程序，转去执行中断请求所对应的中断处理例程。

异常的来源有三种：
1. 程序错误，即当CPU在执行程序指令时遇到操作数有错误或检测到指令规范中定义的非法情况。如除零操作，用户模式下的特权指令。
2. 某些特殊指令，即这些指令的预期就是产生相应的异常。如INT3指令。
3. 奔腾CPU引入的机器检查异常，即当CPU执行指令期间检测到CPU内部或外部的硬件错误。

异常进行分类，也可以分为三类:
1. 错误，导致错误类异常的情况通常可以被纠正。比如内存页错误。
2. 陷阱类异常，CPU报告陷阱类异常时，异常指令通常已经执行完毕，压入栈的CS/EIP值为异常返回后要执行指令。
3. 终止类异常，用来报告严重的错误，硬件错误，系统表中非法值等，这类异常不允许恢复执行。

<!-- more -->
|向量|助记|描述|类型|错误代码|来源|
|---|---|----|---|-------|---|
|0|#DE|除零错误|错误|无|div或idiv指令|
|1|#DB|调试异常，用于软件调试|错误/陷阱|无|任何代码或数据引用
|2| | NMI中断|中断|无| 不可屏蔽的外部中断
|3|#BP|断点|陷阱|无|int3指令
|4|#OF|溢出|陷阱|无|into指令
|5|#BR|数组越界|错误|无|bound指令
|6|#UD|无效指令（没有定义的指令）|错误|无|ud2指令, 或保留的指令
|7|#NM|数学协处理器不存在或不可用|错误|无|浮点或者wait/fwait指令
|8|#DF|双重错误|终止|有(0)|任何会产生异常的指令, NMI或者硬件中断
|9|#MF|协处理器执行浮点运算时, 至少有两个操作数不在一个段内(跨段)|错误|无|浮点指令
|10|#TS|无效TSS|错误|有|任务切换或访问TSS
|11|#NP|段不存在|错误|有|加载段寄存器或者访问系统段
|12|#SS|栈段故障|错误|有|栈操作或者加载段寄存器SS
|13|#GP|常规保护|错误|有|任何内存引用或其他保护异常
|14|#PF|页故障|错误|有|任何内存引用
|15|-||由Intel处理器保留，不能使用|无| -
|16|#MF|x87 FPU(浮点处理单元)浮点处理错误|错误|无|x87 FPU浮点指令或wait/fwait指令
|17|#AC|对齐检查|错误|有(0)|对内存数据引用
|18|#MC|机器检查|终止|无|错误代码(如果有的话)和来源是处理器型号相关的
|19|#XM|SIMD(单指令多数据)浮点异常|错误|无|sse/sse2/sse3浮点指令
|20~31||Intel公司保留, 建议不要使用|保留||
|32~255|用户自定义的中断|中断|外部中断, 或者int n指令||来自INTR的外部中断或INT n指令

#### 异常分发 #####

##### 实模式 #####

对于实模式下，会在地址空间的0x00000000处设置一个中断向量表（Interrupt Vector Table），大小为1K，每一个中断向量占据4个字节；高两字节用于段地址（CS），低两字节用于偏移地址（IP）；这样中断向量表就是256个表项，正好对应CPU的256个中断向量。当发生中断后，CPU根据中断号（IVT + Num \* 4）查找中断向量表找到中断处理函数，然后就可以跳转到对应的处理函数执行代码。

##### 保护模式 #####

在保护模式下，地址空间变为平坦模式。基于段寻址方式，通过GDT/LDT/IDT等来实现代码寻址，跳转，权限级别切换，以及中断和异常处理。

IDT即是保护模式下实现中断和异常的处理关键所在。IDT也是一张位于物理内存的线性表，共有256项（对应CPU支持最大256个中断向量）。IDT表总长度为4096个字节，每个表项是8字节（即一个中断描述符）。IDT表中的项分为三类，任务门，中断门，陷阱门。任务门由于任务切换，目前Windows上只有双重错误和不可屏蔽中断使用的是任务门。中断门用于描述中断处理例程入口。陷阱们描述符用于描述异常处理例程入口。

如下罚抄一遍《软件调试》中的IDT表设置一览（Vista 32位）

|向量号|门类型|处理例程/TSS选择子|中断/异常|说明|
|-----|-----|---------------|--------|---|
|00|中断|nt!KiTrap00|除零错误||
|01|中断|nt!KiTrap01|调试异常||
|02|任务|0x0058|不可屏蔽中断（NMI）|切换到系统线程处理该中断，使用的函数是KiTrap02|
|03|中断|nt!KiTrap03|断点||
|04|中断|nt!KiTrap04|溢出||
|05|中断|nt!KiTrap05|数组越界||
|06|中断|nt!KiTrap06|无效指令||
|07|中断|nt!KiTrap07|数学协处理器不存在或不可用||
|08|任务|0x0050|双重错误（Double Fault）|切换到系统线程处理改异常，执行KiTrap08|
|09|中断|nt!KiTrap09|协处理器段溢出||
|0a|中断|nt!KiTrap0a|无效的TSS||
|0b||nt!KiTrap0b|段不存在||
|0c||nt!KiTrap0c|栈段错误||
|0d||nt!KiTrap0d|一般保护错误||
|0e||nt!KiTrap0e|页错误||
|0f||nt!KiTrap0f|保留||
|10||nt!KiTrap10|浮点错误||
|11||nt!KiTrap11|内存错误||
|12||nt!KiTrap12|机器检查||
|13||nt!KiTrap13|SIMD浮点错误||
|14~1f|||保留|KiTrap0F会引发0x7F号蓝屏|
|20~29|00|NULL||（未使用）|
|2a||nt!KiGetTickCount|||
|2b||nt!KiCallbackReturn||从逆向调用返回|
|2c||nt!KiRaiseAssertion||断言|
|2d||nt!KiDebugService||调试服务|
|2e||nt!KiSystemService||系统服务|
|2f||nt!KiTrap0F|||
|30||hal!Halp8254ClockInterrupt|IRQ0|时钟中断|
|31~3F||驱动程序通过KINTERRUPT结构注册的处理例程|IRQ1~IRQ15|其他硬件设备的中断|
|40~FD||nt!KiUnexpectedInterruptX|N/A|没有使用|


Windows对于异常的处理采用了统一的方式，无论是CPU产生的异常，还是通过系统API抛出的异常（比如RaiseException）。这里以XP系统为例，大致过一下Windows的异常处理过程。以一般保护错误为例，即0x0D号异常，相应的异常处理函数为nt!KiTrap0D

nt!KiTrap0D 中对一些特殊的情况做处理，比如V86模式；根据异常之前的CPU所处的模式，分别处理。当处理掉一些特殊情况后，函数最终会调用到nt!CommonDispatchException()函数中，对这个函数的调用不会返回。

```
nt!CommonDispatchException:
805430a8 83ec50          sub     esp,50h
805430ab 890424          mov     dword ptr [esp],eax            // 异常码
805430ae 33c0            xor     eax,eax
805430b0 89442404        mov     dword ptr [esp+4],eax          //
805430b4 89442408        mov     dword ptr [esp+8],eax          //
805430b8 895c240c        mov     dword ptr [esp+0Ch],ebx        // 异常地址
805430bc 894c2410        mov     dword ptr [esp+10h],ecx        // 参数个数
805430c0 83f900          cmp     ecx,0
805430c3 740c            je      nt!CommonDispatchException+0x29 (805430d1)
805430c5 8d5c2414        lea     ebx,[esp+14h]
805430c9 8913            mov     dword ptr [ebx],edx            // 三个参数
805430cb 897304          mov     dword ptr [ebx+4],esi          //
805430ce 897b08          mov     dword ptr [ebx+8],edi          //
805430d1 8bcc            mov     ecx,esp                        // 构造了 _EXCEPTION_RECORD 结构体（见下面结构体解析）
805430d3 f7457000000200  test    dword ptr [ebp+70h],20000h     // 判断异常之前的CPU所处模式
805430da 7407            je      nt!CommonDispatchException+0x3b (805430e3)
805430dc b8ffff0000      mov     eax,0FFFFh                     // Ring3
805430e1 eb03            jmp     nt!CommonDispatchException+0x3e (805430e6)
805430e3 8b456c          mov     eax,dword ptr [ebp+6Ch]        // _KTRAP_FRAME + 0x6C，即之前的CS寄存器
805430e6 83e001          and     eax,1
805430e9 6a01            push    1
805430eb 50              push    eax
805430ec 55              push    ebp
805430ed 6a00            push    0
805430ef 51              push    ecx
805430f0 e89dc3fbff      call    nt!KiDispatchException (804ff492)  // 函数原型见下面代码段
805430f5 8be5            mov     esp,ebp
805430f7 e920feffff      jmp     nt!KiExceptionExit (80542f1c)      // 异常分发完毕，直接跳转到异常退出函数，退出异常
805430fc 5b              pop     ebx
805430fd 648b3d04000000  mov     edi,dword ptr fs:[4]
80543104 81ef00010000    sub     edi,100h
8054310a 8bf4            mov     esi,esp
8054310c 8be7            mov     esp,edi
8054310e 8bef            mov     ebp,edi
80543110 53              push    ebx
80543111 b91d000000      mov     ecx,1Dh
80543116 f3a5            rep movs dword ptr es:[edi],dword ptr [esi]
80543118 8bce            mov     ecx,esi
8054311a 648b3d04000000  mov     edi,dword ptr fs:[4]
80543121 81ef8c000000    sub     edi,8Ch
80543127 8bdf            mov     ebx,edi
80543129 2bcf            sub     ecx,edi
8054312b 8bf5            mov     esi,ebp
8054312d c1e902          shr     ecx,2
80543130 f3a5            rep movs dword ptr es:[edi],dword ptr [esi]
80543132 8b4544          mov     eax,dword ptr [ebp+44h]
80543135 894344          mov     dword ptr [ebx+44h],eax
80543138 c6434801        mov     byte ptr [ebx+48h],1
8054313c 895d60          mov     dword ptr [ebp+60h],ebx
8054313f c7456844285480  mov     dword ptr [ebp+68h],offset nt!KiServiceExit2 (80542844)
80543146 c7453823000000  mov     dword ptr [ebp+38h],23h
8054314d c7453423000000  mov     dword ptr [ebp+34h],23h
80543154 c7455030000000  mov     dword ptr [ebp+50h],30h
8054315b 814d7000020000  or      dword ptr [ebp+70h],200h
80543162 c3              ret
```

如下是几个结构体的内部定义，结构体EXCEPTION_RECORD记录了异常信息；KTRAP_FRAME记录了陷阱帧内容，是发生异常时的CPU环境。后面我们会用到这个结构体，先在这里了解一下。

```
nt!_EXCEPTION_RECORD
   +0x000 ExceptionCode    : Int4B                      // 异常代码
   +0x004 ExceptionFlags   : Uint4B                     // 异常标记
   +0x008 ExceptionRecord  : Ptr32 _EXCEPTION_RECORD    // 相关的另一个异常
   +0x00c ExceptionAddress : Ptr32 Void                 // 发生异常的地址
   +0x010 NumberParameters : Uint4B                     // 参数数组中元素的个数
   +0x014 ExceptionInformation : [15] Uint4B            // 参数数组
```

```
nt!_KTRAP_FRAME
   +0x000 DbgEbp           : Uint4B
   +0x004 DbgEip           : Uint4B
   +0x008 DbgArgMark       : Uint4B
   +0x00c DbgArgPointer    : Uint4B
   +0x010 TempSegCs        : Uint4B
   +0x014 TempEsp          : Uint4B
   +0x018 Dr0              : Uint4B
   +0x01c Dr1              : Uint4B
   +0x020 Dr2              : Uint4B
   +0x024 Dr3              : Uint4B
   +0x028 Dr6              : Uint4B
   +0x02c Dr7              : Uint4B
   +0x030 SegGs            : Uint4B
   +0x034 SegEs            : Uint4B
   +0x038 SegDs            : Uint4B
   +0x03c Edx              : Uint4B
   +0x040 Ecx              : Uint4B
   +0x044 Eax              : Uint4B
   +0x048 PreviousPreviousMode : Uint4B
   +0x04c ExceptionList    : Ptr32 _EXCEPTION_REGISTRATION_RECORD
   +0x050 SegFs            : Uint4B
   +0x054 Edi              : Uint4B
   +0x058 Esi              : Uint4B
   +0x05c Ebx              : Uint4B
   +0x060 Ebp              : Uint4B
   +0x064 ErrCode          : Uint4B
   +0x068 Eip              : Uint4B
   +0x06c SegCs            : Uint4B
   +0x070 EFlags           : Uint4B
   +0x074 HardwareEsp      : Uint4B
   +0x078 HardwareSegSs    : Uint4B
   +0x07c V86Es            : Uint4B
   +0x080 V86Ds            : Uint4B
   +0x084 V86Fs            : Uint4B
   +0x088 V86Gs            : Uint4B
```

nt!KiDispatchException()用于异常的分发，下面简单分析一下该函数的执行过程。代码不是完整的KiDispatchException的代码，删除了一部分无关的细支末节，在代码中添加注释说明函数过程，不再对函数进行分析。

```
VOID
KiDispatchException (
    IN PEXCEPTION_RECORD ExceptionRecord,
    IN PKEXCEPTION_FRAME ExceptionFrame,
    IN PKTRAP_FRAME TrapFrame,
    IN KPROCESSOR_MODE PreviousMode,
    IN BOOLEAN FirstChance)
{
    CONTEXT ContextFrame;
    EXCEPTION_RECORD ExceptionRecord1, ExceptionRecord2;
    LONG Length;
    ULONG UserStack1;
    ULONG UserStack2;
    KeGetCurrentPrcb()->KeExceptionDispatchCount += 1;
    ContextFrame.ContextFlags = CONTEXT_FULL | CONTEXT_DEBUG_REGISTERS;
    if ((PreviousMode == UserMode) || KdDebuggerEnabled) {
        ContextFrame.ContextFlags |= CONTEXT_FLOATING_POINT;
        if (KeI386XMMIPresent) {
            ContextFrame.ContextFlags |= CONTEXT_EXTENDED_REGISTERS;
        }
    }

    KeContextFromKframes(TrapFrame, ExceptionFrame, &ContextFrame);     // 填充context结构的寄存器等内容
    switch (ExceptionRecord->ExceptionCode) {
        case STATUS_BREAKPOINT:
            ContextFrame.Eip--;                                         // 断点的话，eip退后一个字节，可以将断点还原接着执行
            break;
        case KI_EXCEPTION_ACCESS_VIOLATION:
            ExceptionRecord->ExceptionCode = STATUS_ACCESS_VIOLATION;
            if (PreviousMode == UserMode) {                             // 用户模式下检查是否是thunk问题
                if (KiCheckForAtlThunk(ExceptionRecord,&ContextFrame) != FALSE) {
                    goto Handled1;
                }
                if ((SharedUserData->ProcessorFeatures[PF_NX_ENABLED] == TRUE) &&
                    (ExceptionRecord->ExceptionInformation [0] == EXCEPTION_EXECUTE_FAULT)) {
                    if (((KeFeatureBits & KF_GLOBAL_32BIT_EXECUTE) != 0) ||
                        (PsGetCurrentProcess()->Pcb.Flags.ExecuteEnable != 0) ||
                        (((KeFeatureBits & KF_GLOBAL_32BIT_NOEXECUTE) == 0) &&
                         (PsGetCurrentProcess()->Pcb.Flags.ExecuteDisable == 0))) {
                        ExceptionRecord->ExceptionInformation [0] = 0;
                    }
                }
            }
            break;
    }

    if (PreviousMode == KernelMode)              // 内核模式
    {
        if (FirstChance == TRUE) {
            if ((KiDebugRoutine != NULL) &&      // 第一次错误 交由调试器处理
               (((KiDebugRoutine) (TrapFrame,
                                   ExceptionFrame,
                                   ExceptionRecord,
                                   &ContextFrame,
                                   PreviousMode,
                                   FALSE)) != FALSE)) {
                goto Handled1;
            }
            // 调试器无法处理，尝试找到一个异常函数
            if (RtlDispatchException(ExceptionRecord, &ContextFrame) == TRUE) {
                goto Handled1;
            }
        }
        // 第二次机会，直接发给调试器去处理
        if ((KiDebugRoutine != NULL) &&
            (((KiDebugRoutine) (TrapFrame,
                                ExceptionFrame,
                                ExceptionRecord,
                                &ContextFrame,
                                PreviousMode,
                                TRUE)) != FALSE)) {
            goto Handled1;
        }
        KeBugCheckEx(           // 处理不了直接 KeBugCheckEx，即蓝屏了
            KERNEL_MODE_EXCEPTION_NOT_HANDLED,
            ExceptionRecord->ExceptionCode,
            (ULONG)ExceptionRecord->ExceptionAddress,
            (ULONG)TrapFrame,
            0);
    } else {
        if (FirstChance == TRUE) {          // 用户模式下的异常
            if ((KiDebugRoutine != NULL) && // 用户模式下 有内核调试器，没有用户调试器
                ((PsGetCurrentProcess()->DebugPort == NULL &&
                  !KdIgnoreUmExceptions) ||
                 (KdIsThisAKdTrap(ExceptionRecord, &ContextFrame, UserMode)))) {
                if ((((KiDebugRoutine) (TrapFrame,
                                        ExceptionFrame,
                                        ExceptionRecord,
                                        &ContextFrame,
                                        PreviousMode,
                                        FALSE)) != FALSE)) {
                    goto Handled1;
                }
            }
            // 尝试发送数据用户层调试器
            if (DbgkForwardException(ExceptionRecord, TRUE, FALSE)) {
                goto Handled2;
            }
            ExceptionRecord1.ExceptionCode = 0; // satisfy no_opt compilation
        repeat:
            try {
                if (TrapFrame->HardwareSegSs != (KGDT_R3_DATA | RPL_MASK) ||
                    TrapFrame->EFlags & EFLAGS_V86_MASK ) {
                    ExceptionRecord2.ExceptionCode = STATUS_ACCESS_VIOLATION;
                    ExceptionRecord2.ExceptionFlags = 0;
                    ExceptionRecord2.NumberParameters = 0;
                    ExRaiseException(&ExceptionRecord2);
                }
                UserStack1 = (ContextFrame.Esp & ~CONTEXT_ROUND) - CONTEXT_ALIGNED_SIZE;  // CONTEXT结构入用户栈
                ProbeForWrite((PCHAR)UserStack1, CONTEXT_ALIGNED_SIZE, CONTEXT_ALIGN);
                RtlCopyMemory((PULONG)UserStack1, &ContextFrame, sizeof(CONTEXT));        // 将信息写入到用户态线程栈上
                Length = (sizeof(EXCEPTION_RECORD) - (EXCEPTION_MAXIMUM_PARAMETERS -
                         ExceptionRecord->NumberParameters) * sizeof(ULONG) +3) &
                         (~3);
                UserStack2 = UserStack1 - Length;       // 异常记录块入栈
                ProbeForWrite((PCHAR)(UserStack2 - 8), Length + 8, sizeof(ULONG));
                RtlCopyMemory((PULONG)UserStack2, ExceptionRecord, Length);               // 异常记录 结构体内容
                *(PULONG)(UserStack2 - sizeof(ULONG)) = UserStack1;
                *(PULONG)(UserStack2 - 2*sizeof(ULONG)) = UserStack2;
                //
                // 更新TrapFrame
                //
                KiSegSsToTrapFrame(TrapFrame, KGDT_R3_DATA);
                KiEspToTrapFrame(TrapFrame, (UserStack2 - sizeof(ULONG)*2));
                TrapFrame->SegCs = SANITIZE_SEG(KGDT_R3_CODE, PreviousMode);
                TrapFrame->SegDs = SANITIZE_SEG(KGDT_R3_DATA, PreviousMode);
                TrapFrame->SegEs = SANITIZE_SEG(KGDT_R3_DATA, PreviousMode);
                TrapFrame->SegFs = SANITIZE_SEG(KGDT_R3_TEB, PreviousMode);
                TrapFrame->SegGs = 0;

                //设置Eip是ntdll中的KiUserExceptionDispatcher，ntdll试图分发给异常处理器，如果异常被处理则可以继续执行
                //否则 NtRaiseException 继续进入内核 第二次处理
                TrapFrame->Eip = (ULONG)KeUserExceptionDispatcher;
                return;
            } except (KiCopyInformation(&ExceptionRecord1,
                        (GetExceptionInformation())->ExceptionRecord)) {
                if (ExceptionRecord1.ExceptionCode == STATUS_STACK_OVERFLOW) {
                    ExceptionRecord1.ExceptionAddress = ExceptionRecord->ExceptionAddress;
                    RtlCopyMemory((PVOID)ExceptionRecord,
                                  &ExceptionRecord1, sizeof(EXCEPTION_RECORD));
                    goto repeat;
                }
            }
        }
        //
        // 第二次处理异常 进程调试器处理异常/win32子系统处理异常 如果还是不能处理 进程终止 bugcheck
        //
        if (DbgkForwardException(ExceptionRecord, TRUE, TRUE)) {
            goto Handled2;
        } else if (DbgkForwardException(ExceptionRecord, FALSE, TRUE)) {
            goto Handled2;
        } else {
            ZwTerminateProcess(NtCurrentProcess(), ExceptionRecord->ExceptionCode);
            KeBugCheckEx(
                KERNEL_MODE_EXCEPTION_NOT_HANDLED,
                ExceptionRecord->ExceptionCode,
                (ULONG)ExceptionRecord->ExceptionAddress,
                (ULONG)TrapFrame,
                0);
        }
    }
Handled1:
    // 已经解决，把 ContextFrame移动到陷阱帧
    KeContextToKframes(TrapFrame, ExceptionFrame, &ContextFrame, ContextFrame.ContextFlags, PreviousMode);  

Handled2:
    return;     // 被调试器或者子系统解决的，不同更新trap帧直接返回
}
```

从上面的函数可以看出，对于内核模式，首先会看是否系统正在被调试（KiDebugRoutine变量不为空），如果被调试，则调用 KiDebugRoutine，它实际指向的是nt!KdpTrap，该函数完成内核调试的后续内容（这里不再继续介绍，以后有机会单独学习一下）；否则调用RtlDispatchException()函数进行异常分发。分发异常后依然处理不了当前的异常，则会再次抛出异常，那么系统则进入第二次机会的异常处理，再次调用KiDebugRoutine所指向的函数。如果没有调试，则直接调用 KeBugCheckEx()发起蓝屏请求。

对于用户模式和内核的流程类似。如果没有用户模式的调试器，但是存在系统内核正在被调试，则直接调用KiDebugRoutine指向函数，通知内核调试器。如果没有内核调试器，则调用 DbgkForwardException()尝试将异常分发给用户层调试器。将异常分发给用户层调试器失败后，则准备将异常在用户模式下进行分发。这时则要将异常相关的信息告知Ring3层，所以将CONTEXT / ExceptionRecord复制到用户栈中，同时对TrapFrame信息进行调整，使得CPU回到Ring3层时从 KeUserExceptionDispatcher()函数开始执行（该函数后面分析）。如果用户层异常分发时判断有调试器，且调试器不处理异常，则进入第二次异常处理，同样首先尝试分发给调试器，如果调试器依然不处理，则最终将调用ZwTerminateProcess()终结进程。

从上面可以看到，Ring3和Ring0的异常分发大同小异。首先会尝试分发给调试器，一旦异常被分发给了调试器，则进程会被挂起（调试器设置为不处理的异常除外）。分发给调试之后就会进行将异常分发给异常处理器。如果异常处理器不处理，则进入了第二轮的异常分发。如下借用一下《软件调试》中的异常分发的图^_^。

<div align="center">
![KiDispatchException流程图](/img/2017-06-08-KiDispatchException-DispatchFlow.jpg)
</div>

对于进入调试器的异常，后续的内容不在此处总结，后续可以总结一片异常如何分发给调试器。这里主要关注一下分发给异常处理器的流程。接下来就是调用RtlDispatchException()函数了，在Win2000的源码中，代码逻辑有一些出入，不能参考Win2000的代码了，此处直接反汇编一下XP中该函数的代码，代码如下:

```
char __stdcall RtlDispatchException(int a1, int a2)
{
  unsigned int v2; // eax@1
  unsigned int v3; // ebx@1
  int v4; // esi@2
  unsigned __int32 v5; // edi@3
  void *v6; // ecx@6
  int v7; // eax@9
  int v8; // edi@9
  unsigned int v9; // eax@17
  unsigned __int32 v10; // eax@25
  unsigned __int32 v11; // ecx@25
  unsigned int v13; // [sp+4h] [bp-64h]@16
  int v14; // [sp+8h] [bp-60h]@16
  int v15; // [sp+Ch] [bp-5Ch]@16
  int v16; // [sp+14h] [bp-54h]@16
  unsigned int v17; // [sp+54h] [bp-14h]@9
  unsigned __int32 LowLimit; // [sp+58h] [bp-10h]@1
  unsigned __int32 HighLimit; // [sp+5Ch] [bp-Ch]@1
  unsigned int v20; // [sp+60h] [bp-8h]@1
  char v21; // [sp+67h] [bp-1h]@1

  v21 = 0;
  RtlpGetStackLimits(&LowLimit, &HighLimit);        // 获取栈的上下界限
  v2 = RtlpGetRegistrationHead();                   // 获取 ETHREAD中的异常处理器的头字段
  v20 = 0;
  v3 = v2;
  if ( v2 != 0xFFFFFFFF )                           // 有异常处理器
  {
    v4 = a1;
    while ( 1 )
    {
      v5 = v3 + 8;
      if ( v3 >= LowLimit && v5 <= HighLimit )
        break;
      if ( v3 & 3
        || KeGetCurrentIrql() < 2u
        || (v10 = __readfsdword(0x20), v11 = *(_DWORD *)(v10 + 2152), !*(_DWORD *)(v10 + 2164))
        || v5 > v11
        || v3 < v11 - 0x3000 )
      {
LABEL_32:
        *(_DWORD *)(v4 + 4) |= 8u;
        return v21;
      }
      HighLimit = *(_DWORD *)(v10 + 2152);
      LowLimit = v11 - 0x3000;
LABEL_29:
      if ( v3 == 0xFFFFFFFF )                   // 遍历到结尾，则退出
        return v21;
    }

    if ( v3 & 3 || !(unsigned __int8)RtlIsValidHandler(*(_DWORD *)(v3 + 4)) )   // 处理器是否是有效处理器
      goto LABEL_32;
    if ( byte_4837EE & 0x80 )
      a1 = RtlpLogExceptionHandler(v4, a2, 0, v3, 16); 
    v7 = RtlpExecuteHandlerForException(v6, v4, v3, a2, (int)&v17, *(_DWORD *)(v3 + 4));    // 执行异常处理器
    v8 = v7;
    if ( byte_4837EE & 0x80 )
      RtlpLogLastExceptionDisposition(a1, v7);
    if ( v20 == v3 )
    {
      *(_DWORD *)(v4 + 4) &= 0xFFFFFFEF;
      v20 = 0;
    }
    if ( v8 )
    {
      if ( v8 != 1 )
      {
        if ( v8 == 2 )      // 异常嵌套，特殊处理
        {
          v9 = v17;
          *(_DWORD *)(v4 + 4) |= 0x10u;
          if ( v9 > v20 )
            v20 = v9;
        }
        else                // 异常未处理，继续抛出异常
        {
          v13 = 0xC0000026;
          v14 = 1;
          v15 = v4;
          v16 = 0;
          ((void (__stdcall *)(unsigned int *))RtlRaiseException)(&v13);
        }
LABEL_22:
        v3 = *(_DWORD *)v3;     // 指向下一个异常处理器
        goto LABEL_29;
      }
    }
    else
    {
      if ( !(*(_BYTE *)(v4 + 4) & 1) )
        return 1;
      v13 = 0xC0000025;
      v14 = 1;
      v15 = v4;
      v16 = 0;
      ((void (__stdcall *)(unsigned int *))RtlRaiseException)(&v13);
    }
    if ( *(_BYTE *)(v4 + 4) & 8 )
      return v21;
    goto LABEL_22;      // 没有处理，则继续下一个异常处理器
  }
  return v21;
}
```

调用RtlpGetStackLimits()获取栈的上下限，用于判断遍历是否超出范围。RtlpGetRegistrationHead()获取注册的异常处理器链表头，从反汇编的代码（如下）看，直接读取的是fs:[0h]的内容，其实就是_KPCR结构的NtTib成员的第一个成员ExceptionList。该成员记录了当前栈上注册的异常处理器链表的表头。遍历异常处理器表，调用函数RtlIsValidHandler()判断注册函数是否有效处理器，同时调用RtlpExecuteHandlerForException()函数来调用注册的异常处理器函数，是否是否要处理当前异常。

```
nt!RtlpGetRegistrationHead:
80547fd8 64a100000000    mov     eax,dword ptr fs:[00000000h]
80547fde c3              ret
```

```
0: kd> dt _KPCR ffdff000
nt!_KPCR
   +0x000 NtTib            : _NT_TIB
   +0x01c SelfPcr          : 0xffdff000 _KPCR
   +0x020 Prcb             : 0xffdff120 _KPRCB
   +0x024 Irql             : 0 ''
   +0x028 IRR              : 0
   +0x02c IrrActive        : 0
   +0x030 IDR              : 0xffffffff
   +0x034 KdVersionBlock   : 0x8054e2b8 Void
   +0x038 IDT              : 0x8003f400 _KIDTENTRY
   +0x03c GDT              : 0x8003f000 _KGDTENTRY
   +0x040 TSS              : 0x80042000 _KTSS
   +0x044 MajorVersion     : 1
   +0x046 MinorVersion     : 1
   +0x048 SetMember        : 1
   +0x04c StallScaleFactor : 0xd40
   +0x050 DebugActive      : 0 ''
   +0x051 Number           : 0 ''
   +0x052 Spare0           : 0 ''
   +0x053 SecondLevelCacheAssociativity : 0 ''
   +0x054 VdmAlert         : 0
   +0x058 KernelReserved   : [14] 0
   +0x090 SecondLevelCacheSize : 0
   +0x094 HalReserved      : [16] 0
   +0x0d4 InterruptMode    : 0
   +0x0d8 Spare1           : 0 ''
   +0x0dc KernelReserved2  : [17] 0
   +0x120 PrcbData         : _KPRCB
0: kd> dx -r1 (*((ntkrpamp!_NT_TIB *)0xffffffffffdff000))
(*((ntkrpamp!_NT_TIB *)0xffffffffffdff000))                 [Type: _NT_TIB]
    [+0x000] ExceptionList    : 0xffffffffb002f324 [Type: _EXCEPTION_REGISTRATION_RECORD *]
    [+0x004] StackBase        : 0xffffffffb002fc70 [Type: void *]
    [+0x008] StackLimit       : 0xffffffffb002c000 [Type: void *]
    [+0x00c] SubSystemTib     : 0x0 [Type: void *]
    [+0x010] FiberData        : 0x0 [Type: void *]
    [+0x010] Version          : 0x0 [Type: unsigned long]
    [+0x014] ArbitraryUserPointer : 0x0 [Type: void *]
    [+0x018] Self             : 0x7ffdd000 [Type: _NT_TIB *]

```

对于用户层的异常分发，如前面所述从内核中回到Ring3层时，将TrapFrame更新，进入KeUserExceptionDispatcher指向的函数继续执行。它指向的函数是 ntdll!KiUserExceptionDispatcher()，如下摘取Win2000源码中代码。从例程的描述看，该函数在从内核模式返回，分发用户模式异常时会进入。如果一个基于栈帧的处理器处理了异常，则回到发生异常处继续执行。否则最后一次机会处理会执行。

```
        page
        subttl  "User Exception Dispatcher"
;++
;
; VOID
; KiUserExceptionDispatcher (
;    IN PEXCEPTION_RECORD ExceptionRecord,
;    IN PCONTEXT ContextRecord
;    )
;
; Routine Description:
;
;    This routine is entered on return from kernel mode to dispatch a user
;    mode exception. If a frame based handler handles the exception, then
;    the execution is continued. Else last chance processing is performed.
;
;    NOTE:  This procedure is not called, but rather dispatched to.
;           It depends on there not being a return address on the stack
;           (assumption w.r.t. argument offsets.)
;
; Arguments:
;
;    ExceptionRecord (esp+0) - Supplies a pointer to an exception record.
;
;    ContextRecord (esp+4) - Supplies a pointer to a context frame.
;
; Return Value:
;
;    None.
;
;--

cPublicProc _KiUserExceptionDispatcher      ,2
.FPO (0, 2, 0, 0, 0, 0)

        mov     ecx, [esp+4]            ; (ecx)->context record
        mov     ebx, [esp]              ; (ebx)->exception record

; attempt to dispatch the exception     // 尝试 分发异常
        stdCall   _RtlDispatchException, <ebx, ecx>

;
; If the return status is TRUE, then the exception was handled and execution
; should be continued with the NtContinue service in case the context was
; changed. If the return statusn is FALSE, then the exception was not handled
; and ZwRaiseException is called to perform last chance exception processing.
;
; 如果返回状态为TRUE，那么异常就是被处理了，因为上下文已经变化，代码继续执行应该使用NtContinue服务。
; 如果返回状态是FLASE，异常没有被处理，调用ZwRaiseException()函数执行最后一次异常处理。
;
        or      al,al
        je      short kued10

;
; Continue execution.
;

        pop     ebx                     ; (ebx)->exception record
        pop     ecx                     ; (ecx)->context record

; continue execution
        stdCall   _ZwContinue, <ecx, 0>
        jmp     short kued20            ; join common code

;
; Last chance processing.
;
;   (esp+0) = ExceptionRecord
;   (esp+4) = ContextRecord
;

kued10: pop     ebx                     ; (ebx)->exception record
        pop     ecx                     ; (ecx)->context record

; ecx - context record
; ebx - exception record
; perform last chance processiong 
; 执行最后一次处理，第三个参数为 0，及FirstChance为FALSE，发起第二轮异常处理
; 其实走到此处说明有调试器调试，但是调试器也选择不处理异常
        stdCall   _ZwRaiseException, <ebx, ecx, 0>

;
; Common code for nonsuccessful completion of the continue or raiseexception
; services. Use the return status as the exception code, set noncontinuable
; exception and attempt to raise another exception. Note the stack grows
; and eventually this loop will end.
;
; 如果 ZwContinue() 返回非0，则继续执行失败。将返回值作为异常码，并设置
; 不可继续执行的标记位，再次抛出异常。

.FPO(0, 0, 0, 0, 0, 0)

kued20: add     esp, -ExceptionRecordSize ; allocate stack space
        mov     [esp]+ErExceptionCode, eax ; set exception code
        mov     dword ptr [esp]+ErExceptionFlags, EXCEPTION_NONCONTINUABLE
        mov     [esp]+ErExceptionRecord,ebx ; set associated exception record
        mov     dword ptr [esp]+ErNumberParameters, 0
                                        ; set number of parameters
; esp - addr of exception record
        stdCall   _RtlRaiseException, <esp>
; never return
        stdRET    _KiUserExceptionDispatcher

stdENDP _KiUserExceptionDispatcher

        page
        subttl  "Raise User Exception Dispatcher"
```

用户层异常分发，调用的是_RtlDispatchException()函数，及ntdll!RtlDispatchException()，见后文分析。如果异常分发返回了TRUE，则会调用
ZwContinue()函数继续代码执行，该函数为系统调用，容以后再做介绍。如果ZwContinue返回了，则说明当前异常无法再继续执行了，将返回码作为
异常码，并设置不可继续执行标记，调用RtlRaiseException()再次抛出异常。如果ntdll!RtlDispatchException()返回的是FALSE，则进行最后一次处理，
即发起第二次异常分发。

```
0: kd> u ntdll!ZwContinue
ntdll!ZwContinue:
7c92d05e b820000000      mov     eax,20h
7c92d063 ba0003fe7f      mov     edx,offset SharedUserData!SystemCallStub (7ffe0300)
7c92d068 ff12            call    dword ptr [edx]
7c92d06a c20800          ret     8
7c92d06d 90              nop
```

与内核层的异常分发基本类似，不同的是Ring3层的异常分发首先调用了 ntdll!RtlCallVectoredExceptionHandlers()，用于调用注册的向量化异常处理器。Windows提供了AddVectoredExceptionHandler()/RemoveVectoredExceptionHandler()函数，用于添加和删除向量化异常处理器，添加的处理器被放到了ntdll!RtlpCalloutEntryList指向的链表中，函数调用中可以看到，首先获取ntdll!RtlpCalloutEntryList表头，判断是否有表项存在。遍历每一个表项，解密函数指针，并调用注册的处理器。如果某个异常处理器返回了EXCEPTION_CONTINUE_EXECUTION（-1），则结束遍历，返回 1。

```
char __stdcall RtlCallVectoredExceptionHandlers(int a1, int a2)
{
  char result; // al@2
  int i; // esi@4
  int (__stdcall *v4)(int *); // eax@5
  int v5; // [sp+4h] [bp-8h]@4
  int v6; // [sp+8h] [bp-4h]@4
  char v7; // [sp+17h] [bp+Bh]@8

  if ( (int *)RtlpCalloutEntryList == &RtlpCalloutEntryList )	// 列表为空，直接返回
  {
    result = 0;
  }
  else
  {
    v5 = a1;
    v6 = a2;
    RtlEnterCriticalSection(&RtlpCalloutEntryLock);
    for ( i = RtlpCalloutEntryList; ; i = *(_DWORD *)i )		// 遍历注册的向量化的异常处理器
    {
      if ( (int *)i == &RtlpCalloutEntryList )
      {
        v7 = 0;
        goto LABEL_9;
      }
      v4 = (int (__stdcall *)(int *))RtlDecodePointer(*(_DWORD *)(i + 8));
      if ( v4(&v5) == -1 )										// 调用异常函数，-1（EXCEPTION_CONTINUE_EXECUTION）
        break;
    }
    v7 = 1;
LABEL_9:
    RtlLeaveCriticalSection(&RtlpCalloutEntryLock);
    result = v7;
  }
  return result;
}
```

如果向量化异常处理器没有处理异常，则会走到栈上注册的异常处理器的遍历流程。剩下部分的代码逻辑和上述的内核中的RtlDispatchException()函数逻辑就基本相同了，此处不再详述。

```
char __stdcall RtlDispatchException(int a1, int a2)
{
  int v2; // esi@1
  unsigned int v3; // ebx@2
  unsigned int v4; // eax@6
  int v5; // eax@11
  int v6; // edi@11
  unsigned int v8; // eax@27
  unsigned int v9; // [sp+4h] [bp-64h]@26
  int v10; // [sp+8h] [bp-60h]@26
  int v11; // [sp+Ch] [bp-5Ch]@26
  int v12; // [sp+14h] [bp-54h]@26
  unsigned int v13; // [sp+54h] [bp-14h]@11
  int v14; // [sp+58h] [bp-10h]@10
  unsigned int v15; // [sp+5Ch] [bp-Ch]@2
  unsigned int v16; // [sp+60h] [bp-8h]@2
  char v17; // [sp+67h] [bp-1h]@1
  unsigned int v18; // [sp+70h] [bp+8h]@2

  v2 = a1;
  v17 = 0;
  if ( RtlCallVectoredExceptionHandlers(a1, a2) )// 调用注册的 向量化的异常处理函数
    return 1;
  RtlpGetStackLimits(&v16, &v15);
  v18 = 0;
  v3 = RtlpGetRegistrationHead();               // 获取栈上注册的异常处理函数列表的头
  if ( v3 != 0xFFFFFFFF )
  {
    while ( 1 )
    {
      if ( v3 < v16
        || v3 + 8 > v15
        || v3 & 3
        || (v4 = *(_DWORD *)(v3 + 4), v4 >= v16) && v4 < v15
        || !(unsigned __int8)RtlIsValidHandler(*(_DWORD *)(v3 + 4)) )
      {
        *(_DWORD *)(v2 + 4) |= 8u;              // _EXCEPTION_RECORD->ExceptionFlags    0x8(EXCEPTION_STACK_INVALID)
        return v17;
      }
      if ( byte_7C99E3FA & 0x80 )
        v14 = RtlpLogExceptionHandler(v2, a2, 0, v3, 16);
      v5 = RtlpExecuteHandlerForException(v2, v3, a2, &v13, *(_DWORD *)(v3 + 4));// 执行异常处理函数
      v6 = v5;
      if ( byte_7C99E3FA & 0x80 )
        RtlpLogLastExceptionDisposition(v14, v5);
      if ( v18 == v3 )
      {
        *(_DWORD *)(v2 + 4) &= 0xFFFFFFEF;      // _EXCEPTION_RECORD->ExceptionFlags
        v18 = 0;
      }
      if ( !v6 )
        break;
      if ( v6 == 1 )
        goto LABEL_20;
      if ( v6 == 2 )
      {
        v8 = v13;
        *(_DWORD *)(v2 + 4) |= 0x10u;           // EXCEPTION_NESTED_CALL  嵌套调用
        if ( v8 > v18 )
          v18 = v8;
      }
      else
      {
        v9 = 0xC0000026;                        // EXCEPTION_INVALID_DISPOSITION
        v10 = 1;
        v11 = v2;
        v12 = 0;
        ((void (__stdcall *)(unsigned int *))RtlRaiseException)(&v9);
      }
LABEL_21:
      v3 = *(_DWORD *)v3;
      if ( v3 == 0xFFFFFFFF )                   // 异常列表到结尾了
        return v17;
    }
    if ( !(*(_BYTE *)(v2 + 4) & 1) )
      return 1;
    v9 = 0xC0000025;                            // EXCEPTION_NONCONTINUABLE_EXCEPTION
    v10 = 1;
    v11 = v2;
    v12 = 0;
    ((void (__stdcall *)(unsigned int *))RtlRaiseException)(&v9);
LABEL_20:
    if ( *(_BYTE *)(v2 + 4) & 8 )               // _EXCEPTION_RECORD->ExceptionFlags    0x8(EXCEPTION_STACK_INVALID)
      return v17;
    goto LABEL_21;
  }
  return v17;
}
```

执行异常处理函数，会调用RtlpExecuteHandlerForException()，下面列举出了XP上这个函数的调用顺序，可知最终调用了异常处理函数。RtlDispatchException()函数中的调用上述函数时，最后一个传参为\*(_DWORD \*)(v3 + 4)，即异常处理函数。

```
int __thiscall RtlpExecuteHandlerForException(void *this, int a1, int a2, int a3, int a4, int a5)
{
  return ExecuteHandler((int)this, (int)sub_7C9232BC, a1, a2, a3, a4, a5);
}

int __fastcall ExecuteHandler(int ecx0, int edx0, int a1, int a2, int a3, int a4, int a5)
{
  return ExecuteHandler2(ecx0, edx0, a1, a2, a3, a4, (int (__stdcall *)(int, int, int, int, int, int))a5);
}

int __fastcall ExecuteHandler2(int a1, int a2, int a3, int a4, int a5, int a6, int (__stdcall *a7)(int, int, int, int, int, int))
{
  int v8; // [sp-Ch] [bp-Ch]@0

  return a7(a3, a4, a5, a6, v8, a2);
}

```

这里有一点需要注意，看上面函数，可能会觉得如果这些函数都不处理这个异常呢？或者说我写程序压根没有写try-except代码来处理异常。其实系统会为每一个线程设置一个默认的异常处理函数。这个默认的异常处理函数就是UnhandledExceptionFilter()，就是说即使你在写代码时一个try-except也不用，发生了异常也是可以被处理的，处理的函数就是UnhandledExceptionFilter()。

后面的内容不再写了，于是乎就留下了如下的几个问题：
1 内核调试 nt!KdpTrap
2 Ring3层的异常向调试器分发（DbgkForwardException()）
3 UnhandledExceptionFilter()的处理流程
4 try-except / try-catch（C++）编译成的代码如何处理异常
这样就要以后再慢慢总结了，一下总结出来有点略多啊！！！:)

还是有好多边角没顾及到，后面看仔细了再补充!

**参考文档**

1. ReactOS源码
2. Win2000部分源码
3. 《软件调试》

**修订历史**

* 2017-05-24 10:32:22	完成博文

By Andy  @ 2017-05-24 10:32:22
