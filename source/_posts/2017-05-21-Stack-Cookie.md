---
title: Stack Cookie运行原理
date: 2017-05-21 11:33:32
tags:
- Windbg
- Stack Cookie
categories:
- 调试
---

在漏洞中，有一个最基本的漏洞形成原因就是栈溢出。为了防止程序缓冲区溢出而受到攻击，从VS2005开始的编译器中加入了基于Cookie的安全检查，来检测缓冲区溢出。这一篇来简单说说基于Cookie的安全检查。

我们以VS2008编译出来的Release版本为例看一下基于Cookie的安全检查如何运作。首先给出例子的代码：

```
#include "stdafx.h"
#include <Windows.h>

#pragma optimize("", off)

void StackCookie(LPCTSTR lpszInput)
{
	TCHAR szFstBuffer[3];
	_stprintf(szFstBuffer, _T("%s\n"), lpszInput);
	OutputDebugString(szFstBuffer);
}

#pragma optimize("", on)

int _tmain(int argc, _TCHAR* argv[])
{
	StackCookie(_T("DB"));
	system("pause");
	return 0;
}
```
<!-- more -->
代码比较简单，编写了一个StackCookie函数，参数为字符串。函数的作用就是将传参的字符串格式化到一个内部的缓存中，并用OutputDebugString()函数输出出来。很容易理解啦！

编译一下，Windbg启动程序，打开Disassembly窗口，main函数，StackCookie函数一眼望到头^_^：

```
StackCookie!wmain:
00f21040 68fc20f200      push    offset StackCookie!`string' (00f220fc)         // 00f220fc  44 00 42 00 00 00 00 00-70 61 75 73 65 00 00 00  D.B.....pause...
00f21045 e8b6ffffff      call    StackCookie!StackCookie (00f21000)
00f2104a 680421f200      push    offset StackCookie!`string' (00f22104)
00f2104f ff15a020f200    call    dword ptr [StackCookie!_imp__system (00f220a0)]
00f21055 83c408          add     esp,8
00f21058 33c0            xor     eax,eax
00f2105a c3              ret
StackCookie!__security_check_cookie:
00f2105b 3b0d0030f200    cmp     ecx,dword ptr [StackCookie!__security_cookie (00f23000)]
00f21061 7502            jne     StackCookie!__security_check_cookie+0xa (00f21065)
00f21063 f3c3            rep ret
00f21065 e9ac020000      jmp     StackCookie!__report_gsfailure (00f21316)

StackCookie!StackCookie:
00f21000 55              push    ebp
00f21001 8bec            mov     ebp,esp
00f21003 83ec0c          sub     esp,0Ch
00f21006 a10030f200      mov     eax,dword ptr [StackCookie!__security_cookie (00f23000)]
00f2100b 33c5            xor     eax,ebp
00f2100d 8945fc          mov     dword ptr [ebp-4],eax
00f21010 8b4508          mov     eax,dword ptr [ebp+8]
00f21013 50              push    eax
00f21014 68f420f200      push    offset StackCookie!`string' (00f220f4)
00f21019 8d4df4          lea     ecx,[ebp-0Ch]
00f2101c 51              push    ecx
00f2101d ff15a420f200    call    dword ptr [StackCookie!_imp___swprintf (00f220a4)]
00f21023 83c40c          add     esp,0Ch
00f21026 8d55f4          lea     edx,[ebp-0Ch]
00f21029 52              push    edx
00f2102a ff150020f200    call    dword ptr [StackCookie!_imp__OutputDebugStringW (00f22000)]
00f21030 8b4dfc          mov     ecx,dword ptr [ebp-4]
00f21033 33cd            xor     ecx,ebp
00f21035 e821000000      call    StackCookie!__security_check_cookie (00f2105b)
00f2103a 8be5            mov     esp,ebp
00f2103c 5d              pop     ebp
00f2103d c3              ret
00f2103e cc              int     3
00f2103f cc              int     3
```

从wmain()函数说起，程序默认是UNICODE的，所以程序中int _tmain(int argc, _TCHAR* argv[])被编译成了 wmain()方法（Oh My god，哪来那么多西红柿……）。那么多废话，赶紧说正题……

尽管无关紧要，看到了还是想说一下。好久之前用Windbg调试例子程序，一直在执行bp xxxx!main，总提示未落实的断点。鼓捣了好半天，发现_tmain被编译为了wmain，bp *!main能找到才怪。开始用Windbg，那也是一大把辛酸泪啊！

说正题！！！
程序首先压入了字符串参数L"DB"的指针，如汇编后的内存显示所示。然后调用了StackCookie函数，进入下面的函数中。非常规整的函数的前序，将ebp压栈，esp值放到ebp中，后面sub esp 0Ch。显然这个修改堆栈行为是为该函数的局部变量分配了栈空间。至于为啥是12个字节，见后面补充内容。注意了，这里分配了栈空间，但是栈空间的内容是随机的内容（并没有显示的初始化逻辑）。

接下来就是Cookie了，将程序的全局变量 __security_cookie值放入eax，然后将eax与ebp做了异或，接着就将eax值放入到了 ebp-4的位置。我们对这几个动作以及涉及内容一一作出说明。

先说\_\_security\_cookie这个全局变量，在前面有一篇研究CRT库初始化的笔记，里面说到了security cookie的初始化，其实最终计算完的数值就被保存到了这个全局变量中，用于基于Cookie的溢出检测功能中的。为了计算这个值且保证随机与唯一，可谓用值广泛；具体的内容可以参考之前的文章内容。尽管这样，为了使得Cookie在每一个函数调用中保持唯一性，就又将上述的值和ebp做了异或。

这么一来，一方面保证最终计算到的Cookie值在函数调用中唯一性。同时在之后检验时，可以保证ebp值也没有被篡改。再说Cookie被保存的位置 ebp-4，它位于前一个栈帧基址保存位置之后的四个字节。

```
    |---------|     // 低地址
    | Cookie  |
    |---------|
    | LastEBP |
    |---------|
    | RetAddr |
    |---------|
    | Params  |
    | ......  |
    |         |     // 高地址
```

下面看一下Cookie在栈中的位置，在如下的位置放一个断点：

```
00f21010 8b4508          mov     eax,dword ptr [ebp+8]
```

F5运行到此地之后，我们来看一下栈空间内容，如下：

```
0:000> dds esp
0036f950  00f23020 StackCookie!argv     // 随机值
0036f954  00f2301c StackCookie!envp     // 随机值
0036f958  e71b0746                      // Cookie
0036f95c  0036f9a8                      // 上一个栈帧的栈基址
0036f960  00f2104a StackCookie!wmain+0xa // StackCookie!StackCookie()函数调用的返回地址
0036f964  00f220fc StackCookie!`string'  // 字符串参数
0036f968  00f211c4 StackCookie!__tmainCRTStartup+0x10f [f:\dd\vctools\crt_bld\self_x86\crt\src\crtexe.c @ 583]
0036f96c  00000001
0036f970  00233d48
0036f974  002357d0
0036f978  e71b07b2
0036f97c  00000000
0036f980  00000000
0036f984  7efde000
0036f988  00000000
0036f98c  00000000
0036f990  0036f978
0036f994  bf6ea15b
0036f998  0036f9e4
0036f99c  00f21735 StackCookie!_except_handler4
0036f9a0  e7dfdfd2
0036f9a4  00000000
0036f9a8  0036f9b4
0036f9ac  754e336a kernel32!BaseThreadInitThunk+0x12
0036f9b0  7efde000
0036f9b4  0036f9f4
0036f9b8  77729902 ntdll!RtlInitializeExceptionChain+0x63
0036f9bc  7efde000
0036f9c0  7700156a
0036f9c4  00000000
0036f9c8  00000000
0036f9cc  7efde000
0:000> r
eax=e71b0746 ebx=00000000 ecx=6f09b6f8 edx=00000000 esi=00000001 edi=00f2337c
eip=00f21010 esp=0036f950 ebp=0036f95c iopl=0         nv up ei ng nz na po nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000282
StackCookie!StackCookie+0x10:
00f21010 8b4508          mov     eax,dword ptr [ebp+8] ss:002b:0036f964={StackCookie!`string' (00f220fc)}
```

从上面的调试内容中，就可以对Cookie校验中的内存布局有一个清晰的了解了。那么继续StackCookie!StackCookie()函数下面的代码。对函数 \_stprintf()与OutputDebugString()的调用，我们这里直接略过。下面就到了：

```
00f21030 8b4dfc          mov     ecx,dword ptr [ebp-4]
```

根据前面的对内存布局的分析，ebp-4 位置显然就是保存的Cookie了。紧接着将它与EBP做了异或，这么一来，ecx中保存的就应该是全局变量\_\_security_cookie中的值了。下面紧接着就调用了一个函数：__security_check_cookie()。

```
00f21035 e821000000      call    StackCookie!__security_check_cookie (00f2105b)
```

看这个函数，就发现它和\_\_security\_cookie 全局变量长得极其相似，基本可以断定是由CRT库提供的。我们先看一下这个函数干了点啥（见上面完整的汇编代码）。非常简单，将ecx值与\_\_security_cookie值做了对比，如果相等直接返回。如果不等，那么跳转到\_\_report\_gsfailure()函数。既然是CRT库中的代码，那就好办了，上源码！！！

先从CRT代码库源码中找到\_\_security\_check\_cookie()函数，它位于 *\Microsoft Visual Studio 9.0\VC\crt\src\intel\secchk.c，源代码如下所示：

```
/***
*__security_check_cookie(cookie) - check for buffer overrun
*
*Purpose:
*       Compiler helper.  Check if a local copy of the security cookie still
*       matches the global value.  If not, then report the fatal error.
*
*       The actual reporting is done by __report_gsfailure
*       since the cookie check routine must be minimal code that preserves
*       any registers used in returning the callee's result.
*
*Entry:
*       UINT_PTR cookie - local security cookie to check
*
*Exit:
*       Returns immediately if the local cookie matches the global version.
*       Otherwise, calls the failure reporting handler and exits.
*
*Exceptions:
*
*******************************************************************************/

void __declspec(naked) __fastcall __security_check_cookie(UINT_PTR cookie)
{
    /* x86 version written in asm to preserve all regs */
    __asm {
        cmp ecx, __security_cookie
        jne failure
        rep ret /* REP to avoid AMD branch prediction penalty */
failure:
        jmp __report_gsfailure
    }
}
```

代码就是用汇编编写的，和我们上面从exe的反汇编代码中看到的一模一样。直接进入__report_gsfailure()函数中。该函数位于文件 *\Microsoft Visual Studio 9.0\VC\crt\src\gs_report.c中。

```
#pragma optimize("", off)   /* force an EBP frame on x86, no stack packing */

#if defined (_X86_)
__declspec(noreturn) void __cdecl __report_gsfailure(void)
#else  /* defined (_X86_) */
__declspec(noreturn) void __cdecl __report_gsfailure(ULONGLONG StackCookie)
#endif  /* defined (_X86_) */
{
    volatile UINT_PTR cookie[2];

    /*
     * Set up a fake exception, and report it via UnhandledExceptionFilter.
     * We can't raise a true exception because the stack (and therefore
     * exception handling) can't be trusted after a buffer overrun.  The
     * exception should appear as if it originated after the call to
     * __security_check_cookie, so it is attributed to the function where the
     * buffer overrun was detected.
     */

#if defined (_X86_)
    /*
     * On x86, we reserve some extra stack which won't be used.  That is to
     * preserve as much of the call frame as possible when the function with
     * the buffer overrun entered __security_check_cookie with a JMP instead of
     * a CALL, after the calling frame has been released in the epilogue of
     * that function.
     */

    volatile ULONG dw[(sizeof(CONTEXT) + sizeof(EXCEPTION_RECORD))/sizeof(ULONG)];

    /*
     * Save the state in the context record immediately.  Hopefully, since
     * opts are disabled, this will happen without modifying ECX, which has
     * the local cookie which failed the check.
     */

    __asm {
        mov dword ptr [GS_ContextRecord.Eax], eax
        mov dword ptr [GS_ContextRecord.Ecx], ecx
        mov dword ptr [GS_ContextRecord.Edx], edx
        mov dword ptr [GS_ContextRecord.Ebx], ebx
        mov dword ptr [GS_ContextRecord.Esi], esi
        mov dword ptr [GS_ContextRecord.Edi], edi
        mov word ptr [GS_ContextRecord.SegSs], ss
        mov word ptr [GS_ContextRecord.SegCs], cs
        mov word ptr [GS_ContextRecord.SegDs], ds
        mov word ptr [GS_ContextRecord.SegEs], es
        mov word ptr [GS_ContextRecord.SegFs], fs
        mov word ptr [GS_ContextRecord.SegGs], gs
        pushfd
        pop [GS_ContextRecord.EFlags]

        /*
         * Set the context EBP/EIP/ESP to the values which would be found
         * in the caller to __security_check_cookie.
         */

        mov eax, [ebp]
        mov dword ptr [GS_ContextRecord.Ebp], eax
        mov eax, [ebp+4]
        mov dword ptr [GS_ContextRecord.Eip], eax
        lea eax, [ebp+8]
        mov dword ptr [GS_ContextRecord.Esp], eax

        /*
         * Make sure the dummy stack space looks referenced.
         */

        mov eax, dword ptr dw
    }

    GS_ContextRecord.ContextFlags = CONTEXT_CONTROL;

    GS_ExceptionRecord.ExceptionAddress  = (PVOID)(ULONG_PTR)GS_ContextRecord.Eip;

#elif defined (_AMD64_)
    /*
     * On AMD64/EM64T, __security_check_cookie branched to __report_gsfailure
     * with a JMP, not CALL, so call RtlVirtualUnwind once to get the context
     * of the caller to __security_check_cookie, which is the function with
     * the buffer overrun.
     */

    ULONG64 ControlPc;
    ULONG64 EstablisherFrame;
    PRUNTIME_FUNCTION FunctionEntry;
    PVOID HandlerData;
    ULONG64 ImageBase;

    RtlCaptureContext(&GS_ContextRecord);
    ControlPc = GS_ContextRecord.Rip;
    FunctionEntry = RtlLookupFunctionEntry(ControlPc, &ImageBase, NULL);
    if (FunctionEntry != NULL) {
        RtlVirtualUnwind(UNW_FLAG_NHANDLER,
                         ImageBase,
                         ControlPc,
                         FunctionEntry,
                         &GS_ContextRecord,
                         &HandlerData,
                         &EstablisherFrame,
                         NULL);
    } else {
        GS_ContextRecord.Rip = (ULONGLONG) _ReturnAddress();
        GS_ContextRecord.Rsp = (ULONGLONG) _AddressOfReturnAddress()+8;
    }

    GS_ExceptionRecord.ExceptionAddress = (PVOID)GS_ContextRecord.Rip;
    GS_ContextRecord.Rcx = StackCookie;

#elif defined (_IA64_)
    /*
     * On IA64, __security_check_cookie branched to __report_gsfailure with a
     * true call, so call RtlVirtualUnwind twice to get the context of the
     * caller to __security_check_cookie, which is the function with the
     * buffer overrun.
     */

    ULONGLONG ControlPc;
    FRAME_POINTERS EstablisherFrame;
    PRUNTIME_FUNCTION FunctionEntry;
    BOOLEAN InFunction;
    ULONGLONG ImageBase;
    ULONGLONG TargetGp;
    int frames;

    RtlCaptureContext(&GS_ContextRecord);
    ControlPc = GS_ContextRecord.BrRp;

    for (frames = 0; frames < 2; ++frames)
    {
        FunctionEntry = RtlLookupFunctionEntry(ControlPc, &ImageBase, &TargetGp);
        if (FunctionEntry == NULL)
        {
            break;
        }
        ControlPc = RtlVirtualUnwind(ImageBase,
                                     ControlPc,
                                     FunctionEntry,
                                     &GS_ContextRecord,
                                     &InFunction,
                                     &EstablisherFrame,
                                     NULL);
        PROGRAM_COUNTER_TO_CONTEXT(&GS_ContextRecord, ControlPc);
    }

    GS_ExceptionRecord.ExceptionAddress  = (PVOID)ControlPc;
    GS_ContextRecord.IntV0 = StackCookie;

#else  /* defined (_IA64_) */
#error Unknown platform!
#endif  /* defined (_IA64_) */

    GS_ExceptionRecord.ExceptionCode  = STATUS_STACK_BUFFER_OVERRUN;
    GS_ExceptionRecord.ExceptionFlags = EXCEPTION_NONCONTINUABLE;

    /*
     * Save the global cookie and cookie complement locally - using an array
     * to defeat any potential stack-packing.
     */

    cookie[0] = __security_cookie;
    cookie[1] = __security_cookie_complement;

#if defined (_CRTBLD) && !defined (_SYSCRT)
    DebuggerWasPresent = IsDebuggerPresent();
    _CRT_DEBUGGER_HOOK(_CRT_DEBUGGER_GSFAILURE);
#endif  /* defined (_CRTBLD) && !defined (_SYSCRT) */

    /* Make sure any filter already in place is deleted. */
    SetUnhandledExceptionFilter(NULL);

    UnhandledExceptionFilter((EXCEPTION_POINTERS *)&GS_ExceptionPointers);

#if defined (_CRTBLD) && !defined (_SYSCRT)
    /*
     * If we make it back from Watson, then the user may have asked to debug
     * the app.  If we weren’t under a debugger before invoking Watson,
     * re-signal the VS CRT debugger hook, so a newly attached debugger gets
     * a chance to break into the process.
     */
    if (!DebuggerWasPresent)
    {
        _CRT_DEBUGGER_HOOK(_CRT_DEBUGGER_GSFAILURE);
    }
#endif  /* defined (_CRTBLD) && !defined (_SYSCRT) */

    TerminateProcess(GetCurrentProcess(), STATUS_STACK_BUFFER_OVERRUN);
}
```

源码逻辑很明确了，首先分配一个数组变量，大小为sizeof(CONTEXT) + sizeof(EXCEPTION_RECORD)，用这块内存保存了环境上下文，以及异常结构体。其实这里就是构造了异常发生时的环境。然后调用SetUnhandledExceptionFilter(NULL)，清除注册的异常处理函数。直接调用UnhandledExceptionFilter()函数。即调用了异常处理流程中最终未处理异常的处置函数（异常的分发过程，会在一篇笔记中有专门记录，这里就不过多分析了）。对于具有调试器调试的程序，UnhandledExceptionFilter()函数调用后会挂起到调试器中，函数不会返回。对于没有调试的程序，并且也没有即时调试器，那么该函数返回后，直接调用了TerminateProcess()，终止了进程。

到这里呢，关于基于Cookie的安全检测的运作原理就说完了。但是关于这个程序，还想说一点。

细心的人会注意到，程序中传参为字符串_T("DB")，及三个字符长（D，B，结束符）。而在StackCookie()函数中，缓存的大小为3，并且在做字符串格式化时，格式化的字符串为 _T("%s\n")，如果将_T("DB")格式化到该字符串中，结果应该是四个字符（即D，B，\n，字符串结束符），而保存该字符串的缓存szFstBuffer，它只声明了3个字符的大小。将四个字符复制到3个字符大的空间，那岂不是妥妥地溢出了么？Oh My God，我发现了一处缓存溢出Bug！但是令人匪夷所思的是为何程序还可以完好无损地执行，且顺利结束。挂上了Windbg的即时调试，这货居然也默不作声？难道是程序逻辑分析的有问题？

其实并不是啦，不是Windbg不够专业，不够敬业，也不是程序不存在缓冲区溢出。其实是编译器满足CPU的一些原则（内存对齐），编译出了一个略微缓存溢出也没有问题的程序。同样地，细心的人儿会注意到上面分析中设置断点的位置，打印出了栈的内容中（参考上面打印的栈内容），没错就是注释中有两个"//随机值"的地方，其实这8个字节就是为szFstBuffer变量分配的内容了（8字节，四个WCHAR字符），本应该是3个WCHAR字符，但是为了满足CPU的内存对齐原则，分配了8个字节。\_stprintf()函数调用时，8个字节的内存恰好保存了四个WCHAR字符，不会出现问题。但其实这个地方还是有缓存溢出，只是没有出现问题而已。

VS2008就问你了，敢不敢把程序编译成Debug版本？我分分钟让你变量校验失败，梆梆打脸！！！见下图

<div align="center">
![栈Cookie校验错误提示](/img/2017-05-21-Stack-Cookie-DebugStackVarCheck.jpg)
</div>

Windbg就问你了，你敢不敢把传参修改为_T("DBG")？我分分钟让你的程序挂起啊！敢不敢让我启动程序啊，我非得给你准确指出你已经碾压了Cookie（见下面栈回溯）。

```
0:000> kv
ChildEBP RetAddr  Args to Child
0012fc14 7c92de7a 7c801e3a ffffffff c0000409 ntdll!KiFastSystemCallRet (FPO: [0,0,0])
0012fc18 7c801e3a ffffffff c0000409 0012ff60 ntdll!NtTerminateProcess+0xc (FPO: [2,0,0])
0012fc28 0040141a ffffffff c0000409 e12e4d3b kernel32!TerminateProcess+0x20 (FPO: [2,0,0])
0012ff60 0040103a 00420044 000a0031 e13c0000 StackCookie!__report_gsfailure+0x104 (FPO: [Non-Fpo]) (CONV: cdecl) [f:\dd\vctools\crt_bld\self_x86\crt\src\gs_report.c @ 320]
0012ff74 0040104a 004020fc 004011c4 00000001 StackCookie!StackCookie+0x3a (CONV: cdecl) [d:\debugworks\stack-cookie\stackcookie\stackcookie\stackcookie.cpp @ 14]
0012ff7c 004011c4 00000001 00392980 00392a18 StackCookie!wmain+0xa (FPO: [2,0,0]) (CONV: cdecl) [d:\debugworks\stack-cookie\stackcookie\stackcookie\stackcookie.cpp @ 21]
0012ffc0 7c81776f 0184f6f2 0184f79a 7ffde000 StackCookie!__tmainCRTStartup+0x10f (FPO: [Non-Fpo]) (CONV: cdecl) [f:\dd\vctools\crt_bld\self_x86\crt\src\crtexe.c @ 583]
0012fff0 00000000 0040130c 00000000 78746341 kernel32!BaseProcessStart+0x23 (FPO: [Non-Fpo])
```

** 参考文献 **
1. 《软件调试》 张银奎著
2. VS2008 CRT源代码

** 修订历史 **

* 2017-05-21 12:27:32	完成博文


By Andy @2017-05-21 12:27:32

