---
title: Windows X64的Ring3层异常处理机制
date: 2017-08-04 09:52:23
tags:
- Windbg
- Exception
categories:
- 调试
---

前面自己总结了一下X86下的`__try/__except/__finally`等编译后的内容，以及异常分发到异常处理函数时的执行的过程。后面想看看X64上的内容，觉得X64的就不需要总结了，参考网上前行者的资料简单学习即可。当一点一点地看网上资料时，发现很多细节还是理解不透，无奈还是自己来总结一下。希望后来的朋友看到时不会有和我一样的感觉。后面会将参考到的资料列举一下，方便后来学习者参考，防止他们看这篇总结时也有我看他人学习笔记一样的感觉。

Visual C\+\+支持三种异常，即C\+\+异常处理，结构化异常处理，MFC异常。对于大多数C\+\+程序，应该使用类型安全的C\+\+异常处理，该处理可以确保在堆栈展开过程中调用对象析构函数；结构化异常处理为Windows提供供其自己的使用的，建议不要用于C\+\+或MFC编程；MFC的异常不在讨论之列。其中标准C\+\+异常处理是Visual C\+\+上对Windows结构化异常进一步扩展，加入了展开中的对象析构等内容，所以这里也不会去总结标准C\+\+的异常机制。

有前面的X86上的异常处理总结，这一篇总结起来就会相对简单一点。使用和上次相同的seh程序来解析（程序源码这里就不给出了）。与X86上类似的过程，在X64上也是一样，异常分发由RtlDispatchException函数负责，依次遍历“异常链表”，调用其过滤函数；如果过滤函数返回了`EXCEPTION_EXECUTE_HANDLER`则表示当前的异常处理函数负责处理该异常，然后进行全局展开、局部展开，最后调用异常处理函数，完成这一系列动作之后，程序恢复正常执行，执行`__try*`块（整个异常块）之后的代码；如果过滤函数返回继续执行即`EXCEPTION_CONTINUE_EXECUTION`，则返回异常出继续执行；如果返回`EXCEPTION_CONTINUE_SEARCH`，则继续遍历“异常链表”。当然没有处理异常最终会执行到线程起始处的异常处理函数，结果就是结束线程或进程。
<!--more-->
### 异常信息注册 ###

在X86上从异常处理块注册开始总结，那x64从哪里开始呢？其实也是要从类似的地方开始。X86上的异常处理块注册的方式是将异常信息放到栈上，而这种方式是可被缓存溢出攻击的，尽管Windows前前后后给出了很多缓解措施（SafeSEH等）。再一点，异常确实就是异常的，这就意味着异常并不是正常发生的情况；每一次调用使用了SEH的函数，这些构建异常链表的代码都要执行，这是非常不合理的设计。所以X64从设计之初就从异常信息块的注册上区别于x86的方式，对代码进行减肥。X64上，SEH变成了基于表的形式存在，也就是在代码编译时，这张表就被建立起来用来完整描述模块中的异常处理代码。表作为PE文件的一部分存在。当异常发生时，Windows解析这张表查找适当的异常处理函数。因为异常处理信息的查找线索放到了PE头中，这不会有缓存溢出攻击风险；其次，异常表是在编译过程中生成，不需要运行时额外指令（push/pop等指令）的执行消耗。X64这种基于表的异常处理也有缺点，首先它占用大量内存；其次虽然节省了正常代码执行过程中构建异常帧，但是它的异常处理的代价要比X86基于帧的异常处理高很多。

前面说过，X64模块上的异常处理信息是被保存到PE文件中，这里给出seh.exe的PE信息。从下面图1中可以看到，异常表保存的地方是PE文件中Exception目录中，它在seh.exe中的RVA为0x11000，大小为0x774字节。为了进一步验证，将seh.exe用IDA反汇编，查找偏移为0x11000位置内容，如图2所示。从图中可以看到，在0x140011000处为ExceptionDir表，每一项都是`RUNTIME_FUNCTION`类型的元素。

<div align="center">
![图1 PE文件中的异常信息表](/img/2017-08-04-Windows-X64-Try-Except-Finally-PEExceptionDirectory.jpg)
</div>

<div align="center">
![图2 IDA中Exception Directory](/img/2017-08-04-Windows-X64-Try-Except-Finally-IDA-ExceptionInfoTable.jpg)
</div>

保存在上述这个列表中的函数都是非叶函数，这些函数有什么特点呢？这些函数或者有分配栈空间，或者调用了其他的函数，或者有异常处理。在下面引用了一段MSDN对X64上关于函数类型的论述，可以体会一下官方解释。

> There are basically two types of functions. A function that requires a stack frame is called a frame function. A function that does not require a stack frame is called a leaf function.

> A frame function is a function that allocates stack space, calls other functions, saves nonvolatile registers, or uses exception handling. It also requires a function table entry. A frame function requires a prolog and an epilog. A frame function can dynamically allocate stack space and can employ a frame pointer. A frame function has the full capabilities of this calling standard at its disposal.

> If a frame function does not call another function then it is not required to align the stack (referenced in Section Stack Allocation).

> A leaf function is one that does not require a function table entry. It can't make changes to any nonvolatile registers, including RSP, which means that it can't call any functions or allocate stack space. It is allowed to leave the stack unaligned while it executes.

> 注：这块有个疑问，“如果帧函数不调用其他的函数，那么就不要求它对齐栈”，不太明白这句要表达什么意思。有知道的朋友欢迎留言给我，让我也长点知识^_^。

那么就是说一个`RUNTIME_FUNCTION`结构体代表了一个非页函数。从下述的代码块中可以看到该结构体的定义，定义比较简单，BeginAddress，EndAddress两个成员分别为函数开始和结束的RVA（这个如果不好理解，在后面看到异常分发代码时就理解了），UnWindData则是和这个函数相关的异常处理信息。从三个元素的数据类型ULONG类型可知它是无法表示X64模块中的指针的，即此处是一个相对于ImageBase的偏移值（即RVA），并不是指针或地址值。

> 从前面可知，异常函数表是编译器编译中形成。对于JIT编译器动态生成的函数可以使用RtlInstallFunctionTableCallback或RtlAddFunctionTable构建这么一个函数表，否则函数调用过程中会出现不可靠异常处理。

```
typedef struct _RUNTIME_FUNCTION {
    ULONG BeginAddress;
    ULONG EndAddress;
    ULONG UnwindData;
} RUNTIME_FUNCTION, *PRUNTIME_FUNCTION;
```

UnWindData成员的值其实是一个UNWIND_INFO结构体的RVA，UNWIND_INFO结构体保存了异常处理的信息，其定义如下代码块所示。其中各个成员的函数都做了注释，不再一一解释。结构体`UNWIND_CODE`描述了非叶函数执行时如何建立起栈空间。为了完美地展开栈帧，异常分发器需要了解栈帧中栈空间分配数量，非易失寄存器保存位置，以及其他操作栈的行为。为了恢复调用者的栈帧，这些信息是必须的。除了保存栈帧建立信息，`UNWIND_INFO`结构体也会描述异常处理信息，例如异常处理函数。只有在Flags域包含了`ENW FLAG EHANDLER`时，ExceptionHandler和ExceptionData成员才会存在，并且保存了异常处理函数及异常数据。

```
#define UNW_FLAG_NHANDLER 0x0		// 表示既没有 EXCEPT_FILTER 也没有 EXCEPT_HANDLER。
#define UNW_FLAG_EHANDLER 0x1		// 表示该函数有 EXCEPT_FILTER & EXCEPT_HANDLER。
#define UNW_FLAG_UHANDLER 0x2		// 表示该函数有 FINALLY_HANDLER。
#define UNW_FLAG_CHAININFO 0x4		// 表示该函数有多个 UNWIND_INFO，它们串接在一起（所谓的 chain）。

typedef struct _UNWIND_INFO {
    UBYTE Version         : 3;		// 版本号，目前为1
    UBYTE Flags           : 5;		// Flags，值为上述define定义的值
    UBYTE SizeOfProlog;				// Prolog 的大小（字节单位）
    UBYTE CountOfCodes;				// 展开代码的计数(表示当前UNWIND_INFO包含多少个UNWIND_CODE结构)
    UBYTE FrameRegister  : 4;		// 如果函数建立了栈帧，它表示栈帧的索引（相对于 CONTEXT::RAX 的偏移）。否则该成员的值为0。
    UBYTE FrameOffset    : 4;		// 表示FrameRegister距离函数最初栈顶（完成prolog，没有执行其他指令时的栈顶）偏移
    UNWIND_CODE UnwindCode[1];		// 展开代码数组 USHORT * n
									// UnwindCode 数组详细记录了函数修改栈、保存非易失性寄存器的指令
    //
    // The unwind codes are followed by an optional DWORD aligned field that
    // contains the exception handler address or a function table entry if
    // chained unwind information is specified. If an exception handler address
    // is specified, then it is followed by the language specified exception
    // handler data.
    //
	union {
        //
        // If (Flags & UNW_FLAG_EHANDLER)
        //
    	OPTIONAL ULONG ExceptionHandler;
        //
        // Else if (Flags & UNW_FLAG_CHAININFO)
        //
        OPTIONAL ULONG FunctionEntry;
	};
    //
    // If (Flags & UNW_FLAG_EHANDLER)
    //
    OPTIONAL ULONG ExceptionData[];
} UNWIND_INFO, *PUNWIND_INFO;
```

```
// 展开操作码
typedef enum _UNWIND_OP_CODES {
    UWOP_PUSH_NONVOL = 0, /* info == register number */
    UWOP_ALLOC_LARGE,     /* no info, alloc size in next 2 slots */
    UWOP_ALLOC_SMALL,     /* info == size of allocation / 8 - 1 */
    UWOP_SET_FPREG,       /* no info, FP = RSP + UNWIND_INFO.FPRegOffset*16 */
    UWOP_SAVE_NONVOL,     /* info == register number, offset in next slot */
    UWOP_SAVE_NONVOL_FAR, /* info == register number, offset in next 2 slots */
    UWOP_SAVE_XMM128,     /* info == XMM reg number, offset in next slot */
    UWOP_SAVE_XMM128_FAR, /* info == XMM reg number, offset in next 2 slots */
    UWOP_PUSH_MACHFRAME   /* info == 0: no error-code, 1: error-code */
} UNWIND_CODE_OPS;

typedef union _UNWIND_CODE {
	struct {
		UBYTE CodeOffset;		// Prolog 中的偏移量
		UBYTE UnwindOp : 4;		// 展开操作码
		UBYTE OpInfo : 4;		// 操作信息
	};
	USHORT FrameOffset;
} UNWIND_CODE, *PUNWIND_CODE;
```

ExceptionData其实是一个指向`SCOPE_TABLE`结构体的偏移，这个结构体是一个变长的结构体。ScopeTable用于表述每一个`__try/__except`或`__try/__finally`，每一个表项对应一个`__try`语句。与X86的ScopeTable相对比，这块就相对比较容易理解了。但是与X86不同的是X64上并没有记录try层级的tryLevel成员存在。x64上则是通过表项的先后来对应于try的先后与内外层。对于并列的多个try，先出现的try语句对应于ScopeTable中index较小的项；对于嵌套的try，最内层的try在ScopeTable中index越小，即遍历时要先遍历最内层try，再遍历外层try语句。

```
typedef struct _SCOPE_TABLE {
    ULONG Count;
    struct
    {
        ULONG BeginAddress;
        ULONG EndAddress;
        ULONG HandlerAddress;	// __except后()中代码（函数或代码块） 异常过滤器或异常处理器
        ULONG JumpTarget;		// __except后面{}代码块第一条指令的地址
    } ScopeRecord[1];
} SCOPE_TABLE, *PSCOPE_TABLE;
```

> 注：Windows X64的函数中没有压栈出栈操作，所有涉及栈操作的全都在调用者函数中或函数的prolog中完成，这样有利于遇到异常时进行展开。再者，x64的函数调用方式中，RCX/RDX/R8/R9等四个寄存器用于传递前四个参数（整数或指针），但是函数调用者依然要在栈上为这四个参数分配空间，即0x20 bytes的未初始化内存。即紧邻返回地址的是0x20 Bytes未初始化内存空间（register parameter area），接下来才是第五个参数（stack parameter area）。寄存器参数区总是要分配的，即使被调用的函数的参数少于四个；同时这个寄存器参数区是属于被调用函数的，它本来的用途是保存寄存器参数，以使得这些寄存器参数持久化，但是在有些情况下可以看到被调用函数中会拿着四个参数空间来保存非易失寄存器（如RBP，RBX，RDI，RSI，R12～R15等）。这么安排调用方式，一个不好的地方是函数调用中要最少消耗0x20的栈空间。

上面说了那么多，下面用Windbg解析一下Seh.exe的SehTest函数，如下代码块所示调试结果。

```
0:000> lm m seh			// 查一下seh 的基地址
Browse full module list
start             end                 module name
00000001`3f1d0000 00000001`3f1e4000   seh        (private pdb symbols)  D:\DebugWorks\Exception-Dispatch\X64-Exception\seh\x64\Release\seh.pdb

0:000> dt 001`3f1e1000 _RUNTIME_FUNCTION		// 根据前面PE文件解析看的结果知道异常列表位于0x11000偏移处
seh!_RUNTIME_FUNCTION
   +0x000 BeginAddress     : 0x1000				// SehTest起始地址的RVA值
   +0x004 EndAddress       : 0x1038				// SehTest结束地址的RVA值，从反汇编可知0x38是函数“开区间”，并不包含该值汇编
   +0x008 UnwindData       : 0xbd04				// 异常信息数据保存RVA

0:000> uf seh!SehTest
seh!SehTest [c:\users\administrator\desktop\x64-exception\seh\seh\seh.cpp @ 31]:
   31 00000001`3f1d1000 4883ec58        sub     rsp,58h
   32 00000001`3f1d1004 c744246000000000 mov     dword ptr [rsp+60h],0
   36 00000001`3f1d100c c744246011111111 mov     dword ptr [rsp+60h],11111111h
   37 00000001`3f1d1014 eb08            jmp     seh!SehTest+0x1e (00000001`3f1d101e)  Branch

seh!SehTest+0x1e [c:\users\administrator\desktop\x64-exception\seh\seh\seh.cpp @ 51]:
   51 00000001`3f1d101e c704250000000033333333 mov dword ptr [0],33333333h
   55 00000001`3f1d1029 c744246030333333 mov     dword ptr [rsp+60h],33333330h
   57 00000001`3f1d1031 eb00            jmp     seh!SehTest+0x33 (00000001`3f1d1033)  Branch

seh!SehTest+0x33 [c:\users\administrator\desktop\x64-exception\seh\seh\seh.cpp @ 64]:
   64 00000001`3f1d1033 4883c458        add     rsp,58h
   64 00000001`3f1d1037 c3              ret
	  00000001`3f1d1038 cc              int     3

0:000> dt 001`3f1d0000 + 0xbd04 _UNWIND_INFO		// 异常信息
seh!_UNWIND_INFO
   +0x000 Version          : 0y001
   +0x000 Flags            : 0y00011 (0x3)			// UNW_FLAG_EHANDLER | UNW_FLAG_UHANDLER
   +0x001 SizeOfProlog     : 0x4 ''					// 处理栈的汇编代码所占字节数
   +0x002 CountOfCodes     : 0x1 ''					// UnwindCode数组元素个数
   +0x003 FrameRegister    : 0y0000
   +0x003 FrameOffset      : 0y0000
   +0x004 UnwindCode       : [1] _UNWIND_CODE
0:000> dt 001`3f1d0000 + 0xbd04 + 4 _UNWIND_CODE	//
seh!_UNWIND_CODE
   +0x000 CodeOffset       : 0x4 ''			// prolog 后的第一个指令偏移
   +0x001 UnwindOp         : 0y0010			// UWOP_ALLOC_SMALL
   +0x001 OpInfo           : 0y1010			// 10
   +0x000 FrameOffset      : 0xa204
0:000> dd 001`3f1d0000 + 0xbd04
00000001`3f1dbd04  00010419 0000a204 0000144c 00000003
00000001`3f1dbd14  0000100c 00001016 00009370 00001016
00000001`3f1dbd24  0000101e 00001029 000093c0 00000000
00000001`3f1dbd34  0000101e 00001033 000093d0 00001033
00000001`3f1dbd44  00010401 00004204 0000bd78 00000000
00000001`3f1dbd54  00000000 0000c47a 0000a000 00000000
00000001`3f1dbd64  00000000 00000000 00000000 00000000
00000001`3f1dbd74  00000000 0000bfa0 00000000 0000bfb4
0:000> u 001`3f1d0000 + 0000144c 					// UNWIND_INFO后有一个异常处理函数
seh!__C_specific_handler:
00000001`3f1d144c 488bc4          mov     rax,rsp
00000001`3f1d144f 48895808        mov     qword ptr [rax+8],rbx
00000001`3f1d1453 48896818        mov     qword ptr [rax+18h],rbp
00000001`3f1d1457 48897020        mov     qword ptr [rax+20h],rsi
00000001`3f1d145b 48895010        mov     qword ptr [rax+10h],rdx
00000001`3f1d145f 57              push    rdi
00000001`3f1d1460 4154            push    r12
00000001`3f1d1462 4155            push    r13
0:000> dt 0000001`3f1dbd04 + 0c _SCOPE_TABLE		// 异常处理函数RVA 后面就是ScopeTable了，本例中有三个try，表大小为3
seh!_SCOPE_TABLE
   +0x000 Count            : 3
   +0x004 ScopeRecord      : [1] <unnamed-tag>
```

前面仅仅对`_UNWIND_CODE`结构体内容作了注释，并没有解释它的含义。这里以调试中出现的这个`_UNWIND_CODE`中的内容为例，解释一下。根据前面的叙述，`_UNWIND_CODE`是用作描述当前函数的栈操作，非易失寄存器存储等使用。CodeOffset意思是从prolog开始到结束prolog操作最后一个指令的下一条指令的偏移，即`sub rsp,58h`指令后面一条指令的偏移（相对于prolog起始地址）；UnwindOp为展开造作吗，0y10即2，对应于`UWOP_ALLOC_SMALL`，它的含义是在栈上分配一个小尺寸的空间，大小为OpInfo成员乘以8再加上8，本例中即为88字节，而函数prolog的`sub rsp,58h`指令保留的空间及88字节。跟多相关的内容可以参考MSDN的官方解释：[https://msdn.microsoft.com/en-us/library/ck9asaa9.aspx](https://msdn.microsoft.com/en-us/library/ck9asaa9.aspx)

### 异常分发 ###

上一节总结了一下X64上异常表的“注册”，以及异常表的内容。这块覆盖一些之前总结过的异常分发的内容，总结一下X64中如何用基于异常表的新形式分发异常。这里直接将WRK中的X64的RtlDispatchException函数借用过来用作说明，具体内容见注释。源代码路径为`*\WRK-v1.2\WRK-v1.2\base\ntos\rtl\amd64\exdsptch.c
`

```
// 异常分发中用的几个返回值
#define EXCEPTION_NONCONTINUABLE   0x0001
#define EXCEPTION_UNWINDING        0x0002
#define EXCEPTION_EXIT_UNWIND      0x0004
#define EXCEPTION_STACK_INVALID    0x0008
#define EXCEPTION_NESTED_CALL      0x0010
#define EXCEPTION_TARGET_UNWIND    0x0020
#define EXCEPTION_COLLIDED_UNWIND  0x0040
#define EXCEPTION_UNWIND           0x0066

/*++
Routine Description:
    This function attempts to dispatch an exception to a frame based
    handler by searching backwards through the stack based call frames.
    The search begins with the frame specified in the context record and
    continues backward until either a handler is found that handles the
    exception, the stack is found to be invalid (i.e., out of limits or
    unaligned), or the end of the call hierarchy is reached.

    As each frame is encounter, the PC where control left the corresponding
    function is determined and used to lookup exception handler information
    in the runtime function table built by the linker. If the respective
    routine has an exception handler, then the handler is called. If the
    handler does not handle the exception, then the prologue of the routine
    is executed backwards to "unwind" the effect of the prologue and then
    the next frame is examined.
	函数向后回溯基于栈的调用帧，尝试将异常分发给基于帧的处理器。搜索开始于Context记录中指定的帧，
    直到找到处理该异常的处理器，或者栈帧变成无效，再或者到了调用层次的末尾。

    每次遇到一个栈帧，控制离开对应函数的PC值确定了，并且用于在链接器构建的运行时函数表中寻找异常处理器信息。
    如果每一个函数都有一个异常处理器，就会调用异常处理器。如果异常处理器不处理该异常，然后函数的prologue就
    会被逆向执行以“展开”prologue对栈的影响，然后再检查下一个栈帧。
Arguments:
    ExceptionRecord - Supplies a pointer to an exception record. 异常记录的指针
    ContextRecord - Supplies a pointer to a context record.	异常环境记录的指针

Return Value:
    If the exception is handled by one of the frame based handlers, then
    a value of TRUE is returned. Otherwise a value of FALSE is returned.
	如果异常被处理，返回TRUE，否则返回FALSE
--*/

BOOLEAN RtlDispatchException ( IN PEXCEPTION_RECORD ExceptionRecord, IN PCONTEXT ContextRecord )
{
    BOOLEAN Completion = FALSE;
    CONTEXT ContextRecord1;
    ULONG64 ControlPc;
    DISPATCHER_CONTEXT DispatcherContext;
    EXCEPTION_DISPOSITION Disposition;
    ULONG64 EstablisherFrame;
    ULONG ExceptionFlags;
    PEXCEPTION_ROUTINE ExceptionRoutine;
    PRUNTIME_FUNCTION FunctionEntry;
    PVOID HandlerData;
    ULONG64 HighLimit;
    PUNWIND_HISTORY_TABLE HistoryTable;
    ULONG64 ImageBase;
    ULONG Index;
    ULONG64 LowLimit;
    ULONG64 NestedFrame;
    BOOLEAN Repeat;
    ULONG ScopeIndex;
    UNWIND_HISTORY_TABLE UnwindTable;

	// 调用注册的向量化异常处理器
	if ( (unsigned __int8)RtlpCallVectoredHandlers(ExceptionRecord, ContextRecord, 0i64) )
	{
		Completion = 1;
		return Completion;
	}

    //
    // Get current stack limits, copy the context record, get the initial
    // PC value, capture the exception flags, and set the nested exception
    // frame pointer.
    //

    RtlpGetStackLimits(&LowLimit, &HighLimit);	// 获取栈帧界限，防止遍历中超出栈帧界限
    RtlpCopyContext(&ContextRecord1, ContextRecord);	// 拷贝异常环境记录
    ControlPc = (ULONG64)ExceptionRecord->ExceptionAddress;	// 发生异常时的执行代码的地址（RIP）
    ExceptionFlags = ExceptionRecord->ExceptionFlags & EXCEPTION_NONCONTINUABLE;
    NestedFrame = 0;

    //
    // Initialize the unwind history table.	初始化展开历史表，用于二次调用时加速
    //

    HistoryTable = &UnwindTable;
    HistoryTable->Count = 0;
    HistoryTable->Search = UNWIND_HISTORY_TABLE_NONE;
    HistoryTable->LowAddress = - 1;
    HistoryTable->HighAddress = 0;

    //
    // Start with the frame specified by the context record and search
    // backwards through the call frame hierarchy attempting to find an
    // exception handler that will handle the exception.
    // 搜索开始于Context找那个指定的栈帧，向后搜索调用栈帧层次尝试找到可以处理该异常的异常处理函数。

    do {
        //
        // Lookup the function table entry using the point at which control
        // left the procedure. 控制离开异常函数时的代码执行指针值 查找函数表（即发生异常时的RIP）
        //

		// 这个函数代码不再列出，函数作用就是搜索给出ControlPC所在RUNTIME_FUNCTION表项。
        // 如果HistoryTable有值，且指定搜索方式，则从历史表中搜索，否则从映像的异常函数表中搜索（二分法）
        FunctionEntry = RtlLookupFunctionEntry(ControlPc, &ImageBase, HistoryTable);

        //
        // If there is a function table entry for the routine, then virtually
        // unwind to the caller of the current routine to obtain the virtual
        // frame pointer of the establisher and check if there is an exception
        // handler for the frame.
        // 如果在函数表中有一个发生异常函数对应的表项，然后虚拟展开当前函数到调用者中
        // 获取调用者的虚拟帧指针，确定该函数帧是否有异常处理器

        if (FunctionEntry != NULL) { 	// X64中比较关键的一个函数，后面分析
            ExceptionRoutine = RtlVirtualUnwind(UNW_FLAG_EHANDLER,
                                                ImageBase,
                                                ControlPc,
                                                FunctionEntry,
                                                &ContextRecord1,
                                                &HandlerData,
                                                &EstablisherFrame,
                                                NULL);

            //
            // If the establisher frame pointer is not within the specified
            // stack limits or the established frame pointer is unaligned,
            // then set the stack invalid flag in the exception record and
            // return exception not handled. Otherwise, check if the current
            // routine has an exception handler.
            // 栈帧建立者的栈帧不再特定栈界限或建立起来的栈帧指针没有对其，设置栈帧无效，返回异常未处理。

            if (RtlpIsFrameInBounds(&LowLimit, EstablisherFrame, &HighLimit) == FALSE) {
                ExceptionFlags |= EXCEPTION_STACK_INVALID;
                break;

            } else if (ExceptionRoutine != NULL) {

                //
                // The frame has an exception handler.
                //
                // A linkage routine written in assembler is used to actually
                // call the actual exception handler. This is required by the
                // exception handler that is associated with the linkage
                // routine so it can have access to two sets of dispatcher
                // context when it is called.
                //
                // Call the language specific handler.
                // 如果栈帧有异常处理器。调用语言指定异常处理器

                ScopeIndex = 0;
                do {

                    //
                    // Log the exception if exception logging is enabled.
                    // 记录异常发生日志
                    ExceptionRecord->ExceptionFlags = ExceptionFlags;
                    if ((NtGlobalFlag & FLG_ENABLE_EXCEPTION_LOGGING) != 0) {
                        Index = RtlpLogExceptionHandler(ExceptionRecord,
                                                        &ContextRecord1,
                                                        ControlPc,
                                                        FunctionEntry,
                                                        sizeof(RUNTIME_FUNCTION));
                    }

                    //
                    // Clear repeat, set the dispatcher context, and call the
                    // exception handler.
                    // 设置分发器环境，调用异常处理器
                    Repeat = FALSE;
                    DispatcherContext.ControlPc = ControlPc;
                    DispatcherContext.ImageBase = ImageBase;
                    DispatcherContext.FunctionEntry = FunctionEntry;
                    DispatcherContext.EstablisherFrame = EstablisherFrame;
                    DispatcherContext.ContextRecord = &ContextRecord1;
                    DispatcherContext.LanguageHandler = ExceptionRoutine;
                    DispatcherContext.HandlerData = HandlerData;
                    DispatcherContext.HistoryTable = HistoryTable;
                    DispatcherContext.ScopeIndex = ScopeIndex;
                    // 借用链接器的函数，调用真正的异常处理器函数（X64上就是 __C_specific_handler）
                    Disposition = RtlpExecuteHandlerForException(ExceptionRecord,
                    											 EstablisherFrame,
                                                                 ContextRecord,
                                                                 &DispatcherContext);

                    if ((NtGlobalFlag & FLG_ENABLE_EXCEPTION_LOGGING) != 0) {
                        RtlpLogLastExceptionDisposition(Index, Disposition);
                    }

                    //
                    // Propagate noncontinuable exception flag.
                    //
                    ExceptionFlags |=
                        (ExceptionRecord->ExceptionFlags & EXCEPTION_NONCONTINUABLE);

                    //
                    // If the current scan is within a nested context and the
                    // frame just examined is the end of the nested region,
                    // then clear the nested context frame and the nested
                    // exception flag in the exception flags.
                    // 如果当前的遍历位于一个嵌套的上下文环境，并且当前栈帧是嵌套栈帧的结束
                    // 清楚嵌套上下文环境 和嵌套异常标记。
                    if (NestedFrame == EstablisherFrame) {
                        ExceptionFlags &= (~EXCEPTION_NESTED_CALL);
                        NestedFrame = 0;
                    }

                    //
                    // Case on the handler disposition.
                    // 返回值处理
                    switch (Disposition) {
                        //
                        // The disposition is to continue execution.
                        //
                        // If the exception is not continuable, then raise
                        // the exception STATUS_NONCONTINUABLE_EXCEPTION.
                        // Otherwise return exception handled.
                        // 继续执行。如果异常时不可继续执行，则抛出STATUS_NONCONTINUABLE_EXCEPTION异常，否则返回异常被处理
                    case ExceptionContinueExecution :
                        if ((ExceptionFlags & EXCEPTION_NONCONTINUABLE) != 0) {
                            RtlRaiseStatus(STATUS_NONCONTINUABLE_EXCEPTION);
                        } else {
                            Completion = TRUE;
                            goto DispatchExit;
                        }

                        //
                        // The disposition is to continue the search.
                        //
                        // Get next frame address and continue the search.
                        // 如果是继续搜索，获取下一个栈帧地址，并且继续搜索
                    case ExceptionContinueSearch :
                        break;

                        //
                        // The disposition is nested exception.
                        //
                        // Set the nested context frame to the establisher frame
                        // address and set the nested exception flag in the
                        // exception flags.
                        // 如果返回值是一个嵌套异常，将嵌套上下文栈帧设置到建立者栈帧地址，并且在异常标记中设置嵌套标记
                    case ExceptionNestedException :
                        ExceptionFlags |= EXCEPTION_NESTED_CALL;
                        if (DispatcherContext.EstablisherFrame > NestedFrame) {
                            NestedFrame = DispatcherContext.EstablisherFrame;
                        }
                        break;

                        //
                        // The dispostion is collided unwind.
                        //
                        // A collided unwind occurs when an exception dispatch
                        // encounters a previous call to an unwind handler. In
                        // this case the previous unwound frames must be skipped.
                        // 出现了异常冲突。如果异常分发遇到了对前一个展开处理的调用，则发生了展开冲突。
                        // 这种情况下先前的展开帧必须跳过
                    case ExceptionCollidedUnwind:
                        ControlPc = DispatcherContext.ControlPc;
                        ImageBase = DispatcherContext.ImageBase;
                        FunctionEntry = DispatcherContext.FunctionEntry;
                        EstablisherFrame = DispatcherContext.EstablisherFrame;
                        RtlpCopyContext(&ContextRecord1,
                                        DispatcherContext.ContextRecord);

                        ContextRecord1.Rip = ControlPc;
                        ExceptionRoutine = DispatcherContext.LanguageHandler;
                        HandlerData = DispatcherContext.HandlerData;
                        HistoryTable = DispatcherContext.HistoryTable;
                        ScopeIndex = DispatcherContext.ScopeIndex;
                        Repeat = TRUE;
                        break;

                        //
                        // All other disposition values are invalid.
                        //
                        // Raise invalid disposition exception.
                        // 其他的返回值都被认为是无效的，发起无效异常处理
                    default :
                        RtlRaiseStatus(STATUS_INVALID_DISPOSITION);
                    }
                } while (Repeat != FALSE);
            }

        } else {

            //
            // If the old control PC is the same as the return address,
            // then no progress is being made and the function tables are
            // most likely malformed.
            // 如果老的控制PC 和当前的返回值相同，那么可能函数表最可能被破坏掉了
            if (ControlPc == *(PULONG64)(ContextRecord1.Rsp)) {
                break;
            }

            //
            // Set the point where control left the current function by
            // obtaining the return address from the top of the stack.
            // 这种情况可能是页函数，需要通过从栈顶获取返回值，将控制回到上一层函数。即非叶函数中。
            ContextRecord1.Rip = *(PULONG64)(ContextRecord1.Rsp);
            ContextRecord1.Rsp += 8;
        }

        //
        // Set point at which control left the previous routine.
        // 进入上一层函数调用栈帧，继续搜索异常处理器
        ControlPc = ContextRecord1.Rip;
    } while (RtlpIsFrameInBounds(&LowLimit, (ULONG64)ContextRecord1.Rsp, &HighLimit) == TRUE);

    //
    // Set final exception flags and return exception not handled.
    // 设置最后的异常标记，返回异常没有处理。
    ExceptionRecord->ExceptionFlags = ExceptionFlags;

    //
    // Call vectored continue handlers.
    //
DispatchExit:

    return Completion;
}
```

总结一下上述的函数的执行过程，首先获取栈界限以及异常地址等信息，寻找该异常地址对应的异常函数表中表项。如果当前函数为叶函数，则回退到它的调用函数中（对应的ControlPC也需要修改），再继续搜索异常函数表中表项。如果找到了异常函数表表项，则对当前的函数栈帧进行虚拟展开，获取异常处理函数（语言相关的处理函数 \_\_C_specific_handler）。获取到异常处理函数之后，则调用该异常处理函数，根据返回值确定是否继续搜索异常处理器，还是继续执行程序。

关于异常处理的内容在下一节做简单总结，下面对X64上独有的这个RtlVirtualUnwind()函数做一个简单分析。同样适用WRK中的源代码为依据，RtlVirtualUnwind()函数代码如下代码块所示。

```
/*++
Routine Description:
    This function virtually unwinds the specified function by executing its
    prologue code backward or its epilogue code forward.

    If a context pointers record is specified, then the address where each
    nonvolatile registers is restored from is recorded in the appropriate
    element of the context pointers record.
	这个函数通过反向执行prologue或正向执行epilogue来虚拟展开指定函数。如果指定了Context指针记录，
    每一个非易失寄存器的地址记录在Context适当的元素记录中。
Arguments:
    HandlerType - Supplies the handler type expected for the virtual unwind.
        This may be either an exception or an unwind handler.提供虚拟展开需要的异常处理类型，或者是异常或者是展开

    ImageBase - Supplies the base address of the image that contains the
        function being unwound. 被展开函数所在映像的基地址

    ControlPc - Supplies the address where control left the specified
        function. 控制离开特定函数的地址

    FunctionEntry - Supplies the address of the function table entry for the
        specified function. 特定函数的函数表项地址

    ContextRecord - Supplies the address of a context record. 异常上下文记录的地址

    HandlerData - Supplies a pointer to a variable that receives a pointer
        the the language handler data. 指向语言异常处理数据的指针

    EstablisherFrame - Supplies a pointer to a variable that receives the
        the establisher frame pointer value. 指向建立栈帧的函数的栈帧指针

    ContextPointers - Supplies an optional pointer to a context pointers
        record.	可选参数，指向非易失上下文记录的指针。

Return Value:
    If control did not leave the specified function in either the prologue
    or an epilogue and a handler of the proper type is associated with the
    function, then the address of the language specific exception handler
    is returned. Otherwise, NULL is returned.
--*/
PEXCEPTION_ROUTINE RtlVirtualUnwind (
    IN ULONG HandlerType,
    IN ULONG64 ImageBase,
    IN ULONG64 ControlPc,
    IN PRUNTIME_FUNCTION FunctionEntry,
    IN OUT PCONTEXT ContextRecord,
    OUT PVOID *HandlerData,
    OUT PULONG64 EstablisherFrame,
    IN OUT PKNONVOLATILE_CONTEXT_POINTERS ContextPointers OPTIONAL )
{

    ULONG64 BranchBase;
    ULONG64 BranchTarget;
    LONG Displacement;
    ULONG FrameRegister;
    ULONG Index;
    LOGICAL InEpilogue;
    PULONG64 IntegerAddress;
    PULONG64 IntegerRegister;
    PUCHAR NextByte;
    PRUNTIME_FUNCTION PrimaryFunctionEntry;
    ULONG PrologOffset;
    ULONG RegisterNumber;
    PUNWIND_INFO UnwindInfo;

    //
    // If the specified function does not use a frame pointer, then the
    // establisher frame is the contents of the stack pointer. This may
    // not actually be the real establisher frame if control left the
    // function from within the prologue. In this case the establisher
    // frame may be not required since control has not actually entered
    // the function and prologue entries cannot refer to the establisher
    // frame before it has been established, i.e., if it has not been
    // established, then no save unwind codes should be encountered during
    // the unwind operation.
    //
    // If the specified function uses a frame pointer and control left the
    // function outside of the prologue or the unwind information contains
    // a chained information structure, then the establisher frame is the
    // contents of the frame pointer.
    //
    // If the specified function uses a frame pointer and control left the
    // function from within the prologue, then the set frame pointer unwind
    // code must be looked up in the unwind codes to determine if the
    // contents of the stack pointer or the contents of the frame pointer
    // should be used for the establisher frame. This may not actually be
    // the real establisher frame. In this case the establisher frame may
    // not be required since control has not actually entered the function
    // and prologue entries cannot refer to the establisher frame before it
    // has been established, i.e., if it has not been established, then no
    // save unwind codes should be encountered during the unwind operation.
    //
    // N.B. The correctness of these assumptions is based on the ordering of
    //      unwind codes.
    // 如果指定的函数没有使用帧指针，然后建立者帧就是栈指针中的内容。如果从之离开函数时
    // 执行地址位于prologue中，那么这个栈指针可能不是真正的建立者栈帧。这种情况下，可能
    // 不要求建立帧栈帧，因为控制并没有真正进入当前函数，并且prologue代码在栈帧建立起来
    // 之前并没有引用建立者栈帧。也就是说，如果栈帧还没有被建立起来，在展开操作中不应该
    // 碰到保存的展开代码。
    //
    // 如果指定的函数使用栈帧指针，并且执行控制（RIP）离开函数是在prologue之外，或展开
    // 信息包含了链式信息结构，那么建立者栈帧就使用栈帧指针的内容。
	//
    // 如果指定函数使用了栈帧指针，并且控制离开函数时位于prologue中，那么设置帧指针的
    // 展开代码必须从保存的展开代码中查找，确定是否栈指针或栈帧指针的内容应该用于建立帧栈帧。
    // 这个内容也不一定是真实的建立者栈帧。这种情况下，不应该要求建立者栈帧，因为控制并没有
    // 真正地进入到函数，并且在栈帧建立之前，prologue条目都不能引用建立者栈帧。也就是说，
    // 如果栈帧没有建立起来，在展开操作中不应该用到保存的展开代码。
    //
    // 这些对应的假设是基于展开代码顺序的。
    //
    UnwindInfo = (PUNWIND_INFO)(FunctionEntry->UnwindData + ImageBase);		// 展开数据
    PrologOffset = (ULONG)(ControlPc - (FunctionEntry->BeginAddress + ImageBase));
    if (UnwindInfo->FrameRegister == 0) {
        *EstablisherFrame = ContextRecord->Rsp;		// 没有指定栈帧指针，直接使用Rsp

    } else if ((PrologOffset >= UnwindInfo->SizeOfProlog) ||
               ((UnwindInfo->Flags & UNW_FLAG_CHAININFO) != 0)) {
		// 发生异常的指令不在 Prolog范围中，则当前函数栈帧已经建立，就是上述第二种情况
        *EstablisherFrame = (&ContextRecord->Rax)[UnwindInfo->FrameRegister];
        *EstablisherFrame -= UnwindInfo->FrameOffset * 16;
    } else {
        Index = 0;
        while (Index < UnwindInfo->CountOfCodes) {
            if (UnwindInfo->UnwindCode[Index].UnwindOp == UWOP_SET_FPREG) {
                break;
            }

            Index += 1;
        }
		// 如果指令已经将构建栈帧的指令执行完了，则要按照帧指针即偏移计算帧指针，否则直接用RSP做帧指针
        if (PrologOffset >= UnwindInfo->UnwindCode[Index].CodeOffset) {
            *EstablisherFrame = (&ContextRecord->Rax)[UnwindInfo->FrameRegister];
            *EstablisherFrame -= UnwindInfo->FrameOffset * 16;

        } else {
            *EstablisherFrame = ContextRecord->Rsp;
        }
    }

    //
    // If the point at which control left the specified function is in an
    // epilogue, then emulate the execution of the epilogue forward and
    // return no exception handler.
    // 如果控制离开指定函数时位于epilogue中，要将epilogue执行完，返回到调用函数中

    IntegerRegister = &ContextRecord->Rax;
    NextByte = (PUCHAR)ControlPc;

    //
    // Check for one of:	// X64的epilogue做了优化，只有如下几种情况，修改RSP。再就是非易失寄存器的恢复
    //
    //   add rsp, imm8
    //       or
    //   add rsp, imm32
    //       or
    //   lea rsp, -disp8[fp]
    //       or
    //   lea rsp, -disp32[fp]
    //

    if ((NextByte[0] == SIZE64_PREFIX) &&
        (NextByte[1] == ADD_IMM8_OP) &&
        (NextByte[2] == 0xc4)) {

        //
        // add rsp, imm8.
        //

        NextByte += 4;

    } else if ((NextByte[0] == SIZE64_PREFIX) &&
               (NextByte[1] == ADD_IMM32_OP) &&
               (NextByte[2] == 0xc4)) {
        //
        // add rsp, imm32.
        //
        NextByte += 7;
    } else if (((NextByte[0] & 0xfe) == SIZE64_PREFIX) &&
               (NextByte[1] == LEA_OP)) {

        FrameRegister = ((NextByte[0] & 0x1) << 3) | (NextByte[2] & 0x7);
        if ((FrameRegister != 0) &&
            (FrameRegister == UnwindInfo->FrameRegister)) {

            if ((NextByte[2] & 0xf8) == 0x60) {
                //
                // lea rsp, disp8[fp].
                //
                NextByte += 4;
            } else if ((NextByte[2] &0xf8) == 0xa0) {
                //
                // lea rsp, disp32[fp].
                //
                NextByte += 7;
            }
        }
    }

    //
    // Check for any number of:
    //
    //   pop nonvolatile-integer-register[0..15].
    //	弹出栈上的非易失寄存器(这块并没有弹出而是仅仅查找函数结尾的return/jmp指令)

    while (TRUE) {
        if ((NextByte[0] & 0xf8) == POP_OP) {
            NextByte += 1;

        } else if (IS_REX_PREFIX(NextByte[0]) &&
                   ((NextByte[1] & 0xf8) == POP_OP)) {

            NextByte += 2;

        } else {
            break;
        }
    }

    //
    // If the next instruction is a return or an appropriate jump, then
    // control is currently in an epilogue and execution of the epilogue
    // should be emulated. Otherwise, execution is not in an epilogue and
    // the prologue should be unwound.
    // 如果处理完非易失性寄存器的操作代码后，后面是return或适当的jump，则异常指令位于epilogue中
    // 直接将epilogue模拟执行完毕，否则要将prologue模拟展开

    InEpilogue = FALSE;
    if ((NextByte[0] == RET_OP) ||
        (NextByte[0] == RET_OP_2) ||
        ((NextByte[0] == REP_PREFIX) && (NextByte[1] == RET_OP))) {
        //
        // A return is an unambiguous indication of an epilogue. 返回指令确定无疑是位于epilogue中
        //
        InEpilogue = TRUE;
    } else if ((NextByte[0] == JMP_IMM8_OP) || (NextByte[0] == JMP_IMM32_OP)) {

        //
        // An unconditional branch to a target that is equal to the start of
        // or outside of this routine is logically a call to another function.
        // 非条件分支，跳转到另外一个函数起始或当前函数外部 逻辑上讲是对其他函数的调用

        BranchTarget = (ULONG64)NextByte - ImageBase;
        if (NextByte[0] == JMP_IMM8_OP) {
            BranchTarget += 2 + (CHAR)NextByte[1];

        } else {
            BranchTarget += 5 + *((LONG UNALIGNED *)&NextByte[1]);
        }

        //
        // Determine whether the branch target refers to code within this
        // function. If not, then it is an epilogue indicator.
        //
        // A branch to the start of self implies a recursive call, so
        // is treated as an epilogue.
        // 确定是否分支目标指向当前这个函数，如果不指向当前函数则表明是epilogue代码
        // 跳转函数自己开始处意味着递归调用，所以也当作epilogue处理
        if (BranchTarget < FunctionEntry->BeginAddress ||
            BranchTarget >= FunctionEntry->EndAddress) {

            //
            // The branch target is outside of the region described by
            // this function entry. See whether it is contained within
            // an indirect function entry associated with this same
            // function.
            //
            // If not, then the branch target really is outside of
            // this function.
            //
			// 判断跳转的目标和当前函数是否同一个
            PrimaryFunctionEntry = RtlpSameFunction(FunctionEntry,
                                                    ImageBase,
                                                    BranchTarget + ImageBase);

			// 对于并非同一个，或是同一个函数但是是函数起始位置（递归了），则也是位于Epilogue
            if ((PrimaryFunctionEntry == NULL) ||
                (BranchTarget == PrimaryFunctionEntry->BeginAddress)) {

                InEpilogue = TRUE;
            }

        } else if ((BranchTarget == FunctionEntry->BeginAddress) &&
                   ((UnwindInfo->Flags & UNW_FLAG_CHAININFO) == 0)) {
			// 递归了，并且没有CHAININFO，则认为是位于Epilogue
            InEpilogue = TRUE;
        }

    } else if ((NextByte[0] == JMP_IND_OP) && (NextByte[1] == 0x25)) {

        //
        // An unconditional jump indirect.
        //
        // This is a jmp outside of the function, probably a tail call
        // to an import function.
        // 非条件的间接跳转
        InEpilogue = TRUE;
    } else if (((NextByte[0] & 0xf8) == SIZE64_PREFIX) &&
               (NextByte[1] == 0xff) &&
               (NextByte[2] & 0x38) == 0x20) {

        //
        // This is an indirect jump opcode: 0x48 0xff /4.  The 64-bit
        // flag (REX.W) is always redundant here, so its presence is
        // overloaded to indicate a branch out of the function - a tail
        // call.
        //
        // Such an opcode is an unambiguous epilogue indication.
        // 间接跳转，64位标识被省略，尾部调用。要跳转出当前函数，必然当前位置位于epilogue

        InEpilogue = TRUE;
    }
	// 对于在Epilogue的情况来说，需要先将epilogue模拟执行完毕。
    // 依次是将分配栈帧回退掉，恢复非易失寄存器，根据jump/return找到返回地址
    if (InEpilogue != FALSE) {
        NextByte = (PUCHAR)ControlPc;
        //
        // Emulate one of (if any):
        //
        //   add rsp, imm8
        //       or
        //   add rsp, imm32
        //       or
        //   lea rsp, disp8[frame-register]
        //       or
        //   lea rsp, disp32[frame-register]
        //
        if ((NextByte[0] & 0xf8) == SIZE64_PREFIX) {
            if (NextByte[1] == ADD_IMM8_OP) {
                //
                // add rsp, imm8.
                //
                ContextRecord->Rsp += (CHAR)NextByte[3];
                NextByte += 4;
            } else if (NextByte[1] == ADD_IMM32_OP) {
                //
                // add rsp, imm32.
                //
                Displacement = NextByte[3] | (NextByte[4] << 8);
                Displacement |= (NextByte[5] << 16) | (NextByte[6] << 24);
                ContextRecord->Rsp += Displacement;
                NextByte += 7;
            } else if (NextByte[1] == LEA_OP) {
                if ((NextByte[2] & 0xf8) == 0x60) {
                    //
                    // lea rsp, disp8[frame-register].
                    //
                    ContextRecord->Rsp = IntegerRegister[FrameRegister];
                    ContextRecord->Rsp += (CHAR)NextByte[3];
                    NextByte += 4;
                } else if ((NextByte[2] & 0xf8) == 0xa0) {
                    //
                    // lea rsp, disp32[frame-register].
                    //
                    Displacement = NextByte[3] | (NextByte[4] << 8);
                    Displacement |= (NextByte[5] << 16) | (NextByte[6] << 24);
                    ContextRecord->Rsp = IntegerRegister[FrameRegister];
                    ContextRecord->Rsp += Displacement;
                    NextByte += 7;
                }
            }
        }

        //
        // Emulate any number of (if any):
        //
        //   pop nonvolatile-integer-register.
        //   弹出非易失整数寄存器（RBP/RBX等）
        while (TRUE) {
            if ((NextByte[0] & 0xf8) == POP_OP) {
                //
                // pop nonvolatile-integer-register[0..7]
                //
                RegisterNumber = NextByte[0] & 0x7;
                IntegerAddress = (PULONG64)ContextRecord->Rsp;
                IntegerRegister[RegisterNumber] = *IntegerAddress;
                if (ARGUMENT_PRESENT(ContextPointers)) {
                    ContextPointers->IntegerContext[RegisterNumber] = IntegerAddress;
                }
                ContextRecord->Rsp += 8;
                NextByte += 1;
            } else if (IS_REX_PREFIX(NextByte[0]) && ((NextByte[1] & 0xf8) == POP_OP)) {
                //
                // pop nonvolatile-integer-register[8..15]
                //
                RegisterNumber = ((NextByte[0] & 1) << 3) | (NextByte[1] & 0x7);
                IntegerAddress = (PULONG64)ContextRecord->Rsp;
                IntegerRegister[RegisterNumber] = *IntegerAddress;
                if (ARGUMENT_PRESENT(ContextPointers)) {
                    ContextPointers->IntegerContext[RegisterNumber] = IntegerAddress;
                }
                ContextRecord->Rsp += 8;
                NextByte += 2;
            } else {
                break;
            }
        }

        //
        // Emulate return and return null exception handler.
        //
        // Note: this instruction might in fact be a jmp, however
        //       we want to emulate a return regardless.
        // 注意这块的指令或许是jmp，但是我们是想要模拟一个return
        ContextRecord->Rip = *(PULONG64)(ContextRecord->Rsp);
        ContextRecord->Rsp += 8;
        return NULL;
    }

    //
    // Control left the specified function outside an epilogue. Unwind the
    // subject function and any chained unwind information.
    // 控制是从指定函数的epilogue之外返回，那么就要展开目标函数和任意链接展开信息
    FunctionEntry = RtlpUnwindPrologue(ImageBase,
                                       ControlPc,
                                       *EstablisherFrame,
                                       FunctionEntry,
                                       ContextRecord,
                                       ContextPointers);

    //
    // If control left the specified function outside of the prologue and
    // the function has a handler that matches the specified type, then
    // return the address of the language specific exception handler.
    // Otherwise, return NULL.
    // 如果控制是从指定函数prologue之外离开，并且函数有异常处理器符合本次异常类型，
    // 那么就返回语言特定异常处理函数（__C_except_handler），否则返回NULL
    UnwindInfo = (PUNWIND_INFO)(FunctionEntry->UnwindData + ImageBase);
    PrologOffset = (ULONG)(ControlPc - (FunctionEntry->BeginAddress + ImageBase));
    if ((PrologOffset >= UnwindInfo->SizeOfProlog) &&
        ((UnwindInfo->Flags & HandlerType) != 0)) {
        Index = UnwindInfo->CountOfCodes;
        if ((Index & 1) != 0) {
            Index += 1;
        }

        *HandlerData = &UnwindInfo->UnwindCode[Index + 2];	// +2 绕过DWORD，即__C_specific_handler，找到ScopeTable
        return (PEXCEPTION_ROUTINE)(*((PULONG)&UnwindInfo->UnwindCode[Index]) + ImageBase);

    } else {
        return NULL;
    }
}
```

总结一下上面这个函数，做虚拟展开是为了寻找匹配的异常处理函数，并且获取到可以调用异常处理函数的Context信息，以便于调用异常处理函数。根据发生异常的位置（调用发生异常函数的位置），确定是否位于prologue/epilogue，或者位于函数真正逻辑代码中，对他们分别进行不同处理。对于位于prologue的情况要将已经执行的代码模拟回退掉；对于位于函数真正逻辑中的情况要将整个prologue回退掉；对于位于epilogue的情况，则只需要将剩余代码模拟执行完毕，退到调用它的函数位置即可。对于prologue和epilogue的模拟比较简单，他们的指令都是固定集中，做对应后对RSP和非可变寄存器操作即可。

在这个函数中调用了两个函数，RtlpSameFunction和RtlpUnwindPrologue。两个函数也不是特别难理解，理解了上述两个函数，他们应该都不是什么问题。对于想要了解prologue和epilogue的模拟执行的，可以研究一下RtlpunwindPrologue（）函数。

从上面的代码中可以看到，RtlDispatchException负责遍历X64模块“注册”的异常列表；对于每一个函数的异常表项调用RtlVirtualUnwind()进行模拟展开，获取该函数对应的异常处理函数地址以及异常信息表（ScopeTable）；如果有异常处理函数，则有RtlDispatchException调用异常处理函数过滤是否可以处理当前异常；如果没有异常函数，根据虚拟展开后的Context中Rip确定当前函数的返回地址，进一步循环异常表。通过上述一系列动作，就完成了异常列表的遍历与异常函数调用。

那么下面内容就明显了，就是调用异常处理函数了。在前面总结异常信息注册时，为SehTest函数的注册的异常处理函数即`__C_specific_handler`函数（其实所有的异常处理函数都是这个函数），下面一节简要分析一下这个函数。

### 异常处理和堆栈展开 ###

在RtlDispatchException()函数中对异常处理函数的调用是通过RtlpExecuteHandlerForException()间接完成的，它只是做了一个"中继"，没有额外内容，不再看它了，直接来看`_C_specific_handler`函数。

从WRK中也没发现`_C_specific_handler`函数的定义，应该是被编译为lib进行链接了。这里将SehTest中该函数的逆向代码放在这里，下面就对这段代码做简单分析，用于了解异常处理的过程。在分析该函数之前，先看一个结构体`DISPATCHER_CONTEXT`，它的内容如下所示。

```
typedef struct _DISPATCHER_CONTEXT {
    ULONG64               ControlPc;			// 虚拟展开后的当前IP
    ULONG64               ImageBase;			// 映像基地址
    PRUNTIME_FUNCTION     FunctionEntry;		// 运行时函数表，注册的异常处理信息
    ULONG64               EstablisherFrame;		// 当前ControlPC对应栈帧
    ULONG64               TargetIp;				// 展开中ScopeTable遍历的终止条件
    PCONTEXT              ContextRecord;		// 虚拟展开中获取的ControlPC的运行时上下文
    PEXCEPTION_ROUTINE    LanguageHandler;		// _C_specific_handler
    PVOID                 HandlerData;			// ScopeTable
    PUNWIND_HISTORY_TABLE HistoryTable;			// 历史表，加速函数表搜索
    ULONG                 ScopeIndex;			// Sopetable索引下表
    ULONG                 Fill0;				// 暂无用
} DISPATCHER_CONTEXT, *PDISPATCHER_CONTEXT;
```

```
#define EXCEPTION_NONCONTINUABLE 0x1    // Noncontinuable exception
#define EXCEPTION_UNWINDING 0x2         // Unwind is in progress
#define EXCEPTION_EXIT_UNWIND 0x4       // Exit unwind is in progress
#define EXCEPTION_STACK_INVALID 0x8     // Stack out of limits or unaligned
#define EXCEPTION_NESTED_CALL 0x10      // Nested exception handler call
#define EXCEPTION_TARGET_UNWIND 0x20    // Target unwind in progress
#define EXCEPTION_COLLIDED_UNWIND 0x40  // Collided exception handler call
```


```
signed __int64 __fastcall _C_specific_handler(_EXCEPTION_RECORD *ExceptionRecord,
											  void *EstablisherFrame,
                                              _CONTEXT *ContextRecord,
                                              _DISPATCHER_CONTEXT *DispatcherContext)
{
  unsigned __int64 ImageBase; // r15@1
  unsigned int *pHandlerData; // rsi@1
  unsigned __int64 ContorlOffset; // r12@1
  _DISPATCHER_CONTEXT *pDispatcherContext; // r14@1
  void *pEstablisherFrame; // rbp@1
  _EXCEPTION_RECORD *pExceptionRecord; // r13@1
  unsigned int ScopeTableIndex; // edi@2
  signed __int64 pFilterFunc; // rbx@3
  int nResult; // eax@8
  unsigned int v14; // ebp@18
  unsigned __int64 TargetIpOffset; // rdi@18
  _DWORD *pScopeTable_ItemJumpTarget; // rbx@19
  unsigned __int64 rvaBeginAddress; // rcx@20
  unsigned __int64 rvaEndAddress; // rax@21
  _EXCEPTION_RECORD *pExceptionRecord1; // [sp+30h] [bp-38h]@2
  _CONTEXT *pContextRecord1; // [sp+38h] [bp-30h]@2
  void *v21; // [sp+78h] [bp+10h]@1

  v21 = EstablisherFrame;
  ImageBase = DispatcherContext->ImageBase;
  pHandlerData = (unsigned int *)DispatcherContext->HandlerData;// _SCOPE_TABLE
  ContorlOffset = DispatcherContext->ControlPc - ImageBase;
  pDispatcherContext = DispatcherContext;
  pEstablisherFrame = EstablisherFrame;
  pExceptionRecord = ExceptionRecord;
  if ( ExceptionRecord->ExceptionFlags & 0x66 ) // pExceptionRecord->ExceptionFlags & EXCEPTION_UNWIND，判断是否展开过程
  {
    v14 = 0;
    TargetIpOffset = DispatcherContext->TargetIp - ImageBase;
    if ( *pHandlerData )
    {
      pScopeTable_ItemJumpTarget = pHandlerData + 4;
      do
      {
        rvaBeginAddress = *(pScopeTable_ItemJumpTarget - 3);// BeginAddress
        if ( ContorlOffset >= rvaBeginAddress )
        {
          rvaEndAddress = *(pScopeTable_ItemJumpTarget - 2);// EndAddress
          if ( ContorlOffset < rvaEndAddress )
          {
            if ( TargetIpOffset >= rvaBeginAddress
              && TargetIpOffset < rvaEndAddress
              && pExceptionRecord->ExceptionFlags & 0x20 )	// 到了目的地址，即展开的终点
            {
              return 1i64;
            }

            if ( *pScopeTable_ItemJumpTarget )		// 表示当前为 __try/__except 块
            {
			  // 当前的ExceptHandler块的地址就是将要执行的ExceptHandler的地址，直接返回找到了目标块即可
			  // 这是本次展开操作的终点
              if ( TargetIpOffset == *pScopeTable_ItemJumpTarget )
                return 1i64;
            }
            else
            {
			  // 当前为__try/__finally 块
			  // 将Dispatcher_Context->ScopeIndex ++ ，越过这个__try/__finally
			  // 在执行此finally 的时候产生异常并被捕获后，展开将继续执行，并越过这个finally块。
              LOBYTE(rvaBeginAddress) = 1;
              pDispatcherContext->ControlPc = ImageBase + rvaEndAddress;
              ((void (__fastcall *)(unsigned __int64, void *))(ImageBase + *(pScopeTable_ItemJumpTarget - 1)))(
                rvaBeginAddress,
                v21);                           // 调用HandlerAddress指向的函数
            }
          }
        }
        ++v14;                                  // 遍历计数
        pScopeTable_ItemJumpTarget += 4;        // ScopeTable的下一个Item
      }
      while ( v14 < *pHandlerData );
    }
  }
  else                                          // 异常处理
  {
	/**
	* 异常操作相对比较简单，遍历SCOPE_TABLE 中的SCOPE_ENTRY 数组，依次寻找包含目标代码的__try/__except 块，并执行其filter 函数
	* 根据filter 函数的返回值
	* EXCEPTION_CONTINUE_EXECUTION 	直接返回 ExceptionContinueExecution
	* EXCEPTION_EXECUTE_HANDLER 	调用RtlUnwindEx 执行handler 函数
	* EXCEPTION_CONTINUE_SEARCH 	继续遍历
	*/
    ScopeTableIndex = 0;
    pExceptionRecord1 = ExceptionRecord;
    pContextRecord1 = ContextRecord;
    if ( *pHandlerData )
    {
      pFilterFunc = (signed __int64)(pHandlerData + 3);
      do
      {
        if ( ContorlOffset >= *(_DWORD *)(pFilterFunc - 8)// BeginAddress
          && ContorlOffset < *(_DWORD *)(pFilterFunc - 4)// EndAddress
          && *(_DWORD *)(pFilterFunc + 4) )     // JumpTarget
        {
          if ( *(_DWORD *)pFilterFunc == 1 )    // EXCEPTION_EXECUTE_HANDLER = 1
            goto LABEL_33;
          nResult = ((int (__fastcall *)(_EXCEPTION_RECORD **, void *))(ImageBase + *(_DWORD *)pFilterFunc))(
                      &pExceptionRecord1,
                      pEstablisherFrame);
          if ( nResult < 0 )                    // EXCEPTION_CONTINUE_EXECUTION = -1
            return 0i64;
          if ( nResult > 0 )
          {
LABEL_33:
            if ( pExceptionRecord->ExceptionCode == 0xE06D7363 && *(_QWORD *)pDestructExceptionObject )
            {
              if ( IsNonwritableInCurrentImage(pDestructExceptionObject) )
                ((void (__fastcall *)(_EXCEPTION_RECORD *, signed __int64))pDestructExceptionObject)(
                  pExceptionRecord,
                  1i64);
            }
            NLG_Notify(ImageBase + *(_DWORD *)(pFilterFunc + 4), pEstablisherFrame, 1i64);
            RtlUnwindEx(
              pEstablisherFrame,
              (PVOID)(ImageBase + *(_DWORD *)(pFilterFunc + 4)),
              pExceptionRecord,
              (PVOID)pExceptionRecord->ExceptionCode,
              pDispatcherContext->ContextRecord,
              pDispatcherContext->HistoryTable);
            _NLG_Return2();
          }
        }
        ++ScopeTableIndex;                      // EXCEPTION_CONTINUE_SEARCH = 0 继续循环
        pFilterFunc += 16i64;                   // 循环ScopeTable下一个表项
      }
      while ( ScopeTableIndex < *pHandlerData );// Index不超过表大小
    }
  }
  return 1i64;
}
```

参照X86中的异常处理函数逻辑，该函数也是类似逻辑。分两种情况（说明在异常处理时会两次进入），一种是异常处理时进入该函数中遍历ScopeTable，并按照是否有Filter函数，如果有则调用该函数，返回异常可以被处理（即1），则进入异常展开阶段；否则继续遍历ScopeTable直到找到可以处理项或到结尾。第二次进入该函数是展开过程中，依然是遍历ScopeTable，确定是否到达了目标try语句，其中对于finally语句则直接调用。

在找到了可以处理当前异常的ScopeTable表项后，则调用 RtlUnwindEx()函数进行展开。

```
/*++
Routine Description:
    This function initiates an unwind of procedure call frames. The machine
    state at the time of the call to unwind is captured in a context record
    and the unwinding flag is set in the exception flags of the exception
    record. If the TargetFrame parameter is not specified, then the exit unwind
    flag is also set in the exception flags of the exception record. A backward
    scan through the procedure call frames is then performed to find the target
    of the unwind operation.

    As each frame is encounter, the PC where control left the corresponding
    function is determined and used to lookup exception handler information
    in the runtime function table built by the linker. If the respective
    routine has an exception handler, then the handler is called.
	这个函数发起函数调用栈帧的展开。CPU当前的状态保存在了Context记录中，并且展开标记在异常记录结构体中有设置。
    如果TargetFrame参数没有指定，那么退出展开标记会设置到异常记录的异常标记中。向后扫描函数调用栈帧，逐一查找
    展开操作代码。
    每遇到一个栈帧，离开当前函数的控制PC就是确定的，并且用于在链接器构建的运行时函数表中查找异常处理信息。
    如果函数有异常处理器，那么就调用它的处理函数。
Arguments:
    TargetFrame - Supplies an optional pointer to the call frame that is the
        target of the unwind. If this parameter is not specified, then an exit
        unwind is performed.	该参数不指定，则退出展开过程
    TargetIp - Supplies an optional instruction address that specifies the
        continuation address of the unwind. This address is ignored if the
        target frame parameter is not specified. 指定了继续执行执行的地址
    ExceptionRecord - Supplies an optional pointer to an exception record.
		异常信息记录
    ReturnValue - Supplies a value that is to be placed in the integer
        function return register just before continuing execution.
    OriginalContext - Supplies a pointer to a context record that can be used
        to store context during the unwind operation.
    HistoryTable - Supplies an optional pointer to an unwind history table.
Return Value:
    None.
--*/
VOID RtlUnwindEx (
    IN PVOID TargetFrame OPTIONAL,
    IN PVOID TargetIp OPTIONAL,
    IN PEXCEPTION_RECORD ExceptionRecord OPTIONAL,
    IN PVOID ReturnValue,
    IN PCONTEXT OriginalContext,
    IN PUNWIND_HISTORY_TABLE HistoryTable OPTIONAL)
{

    ULONG64 ControlPc;
    PCONTEXT CurrentContext;
    DISPATCHER_CONTEXT DispatcherContext;
    EXCEPTION_DISPOSITION Disposition;
    ULONG64 EstablisherFrame;
    ULONG ExceptionFlags;
    EXCEPTION_RECORD ExceptionRecord1;
    PEXCEPTION_ROUTINE ExceptionRoutine;
    PRUNTIME_FUNCTION FunctionEntry;
    PVOID HandlerData;
    ULONG64 HighLimit;
    ULONG64 ImageBase;
    CONTEXT LocalContext;
    ULONG64 LowLimit;
    PCONTEXT PreviousContext;
    ULONG ScopeIndex;
    PCONTEXT TempContext;

    //
    // Get current stack limits, capture the current context, virtually
    // unwind to the caller of this routine, get the initial PC value, and
    // set the unwind target address.
    // 获取当前的栈限制，当前的上下文，展开的目标地址等
    CurrentContext = OriginalContext;
    PreviousContext = &LocalContext;
    RtlpGetStackLimits(&LowLimit, &HighLimit);
    RtlCaptureContext(CurrentContext);

    //
    // If a history table is specified, then set to search history table.
    // 如果传入了历史表，那么就设置从历史表中搜索
    if (ARGUMENT_PRESENT(HistoryTable)) {
        HistoryTable->Search = UNWIND_HISTORY_TABLE_GLOBAL;
    }

    //
    // If an exception record is not specified, then build a local exception
    // record for use in calling exception handlers during the unwind operation.
    // 没有指定异常记录，则构建一个本地的用于在展开操作中调用异常处理函数
    if (ARGUMENT_PRESENT(ExceptionRecord) == FALSE) {
        ExceptionRecord = &ExceptionRecord1;
        ExceptionRecord1.ExceptionCode = STATUS_UNWIND;
        ExceptionRecord1.ExceptionRecord = NULL;
        ExceptionRecord1.ExceptionAddress = (PVOID)CurrentContext->Rip;
        ExceptionRecord1.NumberParameters = 0;
    }

    //
    // If the target frame of the unwind is specified, then a normal unwind
    // is being performed. Otherwise, an exit unwind is being performed.
    // 如果指定了展开的目标栈帧，那么执行正常的展开操作。否则就是执行退出展开。
    ExceptionFlags = EXCEPTION_UNWINDING;
    if (ARGUMENT_PRESENT(TargetFrame) == FALSE) {
        ExceptionFlags |= EXCEPTION_EXIT_UNWIND;
    }

    //
    // Scan backward through the call frame hierarchy and call exception
    // handlers until the target frame of the unwind is reached.
    // 向后扫描调用帧层，调用异常处理函数，直到到达展开的目标栈帧。
    do {

        //
        // Lookup the function table entry using the point at which control
        // left the procedure.
        // 查询函数表，找到对应的表象
        ControlPc = CurrentContext->Rip;
        FunctionEntry = RtlLookupFunctionEntry(ControlPc,
                                               &ImageBase,
                                               HistoryTable);
        //
        // If there is a function table entry for the routine, then virtually
        // unwind to the caller of the routine to obtain the virtual frame
        // pointer of the establisher, but don't update the context record.
        // 如果找到了函数表条目，然后虚拟展开，获取虚拟栈帧指针，但是不更新上下文记录
        if (FunctionEntry != NULL) {
            RtlpCopyContext(PreviousContext, CurrentContext);
            ExceptionRoutine = RtlVirtualUnwind(UNW_FLAG_UHANDLER,
                                                ImageBase,
                                                ControlPc,
                                                FunctionEntry,
                                                PreviousContext,
                                                &HandlerData,
                                                &EstablisherFrame,
                                                NULL);

            //
            // If the establisher frame pointer is not within the specified
            // stack limits, the establisher frame pointer is unaligned, or
            // the target frame is below the establisher frame and an exit
            // unwind is not being performed, then raise a bad stack status.
            // Otherwise, check to determine if the current routine has an
            // exception handler.
            // 如果建立者栈帧不在指定的栈界限内容，或者栈帧没有对齐，或者目标栈帧比建立者栈帧小
            // 那么就是出错了，抛出坏栈错误。
            if ((RtlpIsFrameInBounds(&LowLimit, EstablisherFrame, &HighLimit) == FALSE) ||
                 ((ARGUMENT_PRESENT(TargetFrame) != FALSE) &&
                  ((ULONG64)TargetFrame < EstablisherFrame))) {

                RtlRaiseStatus(STATUS_BAD_STACK);

            } else if (ExceptionRoutine != NULL) {

                //
                // The frame has a exception handler.
                //
                // A linkage routine written in assembler is used to actually
                // call the actual exception handler. This is required by the
                // exception handler that is associated with the linkage
                // routine so it can have access to two sets of dispatcher
                // context when it is called.
                //
                // Call the language specific handler.
                // 如果栈帧对应异常处理函数，则调用语言特定处理函数 __C_Specific_handler
                DispatcherContext.TargetIp = (ULONG64)TargetIp;	// 赋值TargetIp，注意在异常分发中没有赋值
                ScopeIndex = 0;
                do {

                    //
                    // If the establisher frame is the target of the unwind
                    // operation, then set the target unwind flag.
                    // 如果当前栈帧就是展开的目标栈帧，则设置  EXCEPTION_TARGET_UNWIND 标记
                    if ((ULONG64)TargetFrame == EstablisherFrame) {
                        ExceptionFlags |= EXCEPTION_TARGET_UNWIND;
                    }

                    ExceptionRecord->ExceptionFlags = ExceptionFlags;

                    //
                    // Set the specified return value and target IP in case
                    // the exception handler directly continues execution.
                    // 设置指定的返回值和目标IP，防止异常处理函数直接继续执行，不返回
                    CurrentContext->Rax = (ULONG64)ReturnValue;

                    //
                    // Set the dispatcher context and call the termination
                    // handler.
                    // 设置分发上下文，并且调用处理函数
                    DispatcherContext.ControlPc = ControlPc;
                    DispatcherContext.ImageBase = ImageBase;
                    DispatcherContext.FunctionEntry = FunctionEntry;
                    DispatcherContext.EstablisherFrame = EstablisherFrame;
                    DispatcherContext.ContextRecord = CurrentContext;
                    DispatcherContext.LanguageHandler = ExceptionRoutine;
                    DispatcherContext.HandlerData = HandlerData;
                    DispatcherContext.HistoryTable = HistoryTable;
                    DispatcherContext.ScopeIndex = ScopeIndex;
                    Disposition =
                        RtlpExecuteHandlerForUnwind(ExceptionRecord,
                                                    EstablisherFrame,
                                                    CurrentContext,
                                                    &DispatcherContext);

                    //
                    // Clear target unwind and collided unwind flags.
                    // 清除标记
                    ExceptionFlags &=
                        ~(EXCEPTION_COLLIDED_UNWIND | EXCEPTION_TARGET_UNWIND);

                    //
                    // Case on the handler disposition.
                    // 调用结果
                    switch (Disposition) {
                        //
                        // The disposition is to continue the search.
                        //
                        // If the target frame has not been reached, then
                        // swap context pointers.
                        // 继续执行，交换上下文指针
                    case ExceptionContinueSearch :
                        if (EstablisherFrame != (ULONG64)TargetFrame) {
                            TempContext = CurrentContext;
                            CurrentContext = PreviousContext;
                            PreviousContext = TempContext;
                        }
                        break;

                        //
                        // The disposition is collided unwind.
                        //
                        // Copy the context of the previous unwind and
                        // virtually unwind to the caller of the establisher,
                        // then set the target of the current unwind to the
                        // dispatcher context of the previous unwind, and
                        // reexecute the exception handler from the collided
                        // frame with the collided unwind flag set in the
                        // exception record.
                        // 展开冲突
                    case ExceptionCollidedUnwind :
                        ControlPc = DispatcherContext.ControlPc;
                        ImageBase = DispatcherContext.ImageBase;
                        FunctionEntry = DispatcherContext.FunctionEntry;
                        RtlpCopyContext(OriginalContext,
                                        DispatcherContext.ContextRecord);

                        CurrentContext = OriginalContext;
                        PreviousContext = &LocalContext;
                        RtlpCopyContext(PreviousContext, CurrentContext);
                        RtlVirtualUnwind(UNW_FLAG_NHANDLER,
                                         ImageBase,
                                         ControlPc,
                                         FunctionEntry,
                                         PreviousContext,
                                         &HandlerData,
                                         &EstablisherFrame,
                                         NULL);

                        EstablisherFrame = DispatcherContext.EstablisherFrame;
                        ExceptionRoutine = DispatcherContext.LanguageHandler;
                        HandlerData = DispatcherContext.HandlerData;
                        HistoryTable = DispatcherContext.HistoryTable;
                        ScopeIndex = DispatcherContext.ScopeIndex;
                        ExceptionFlags |= EXCEPTION_COLLIDED_UNWIND;
                        break;

                        //
                        // All other disposition values are invalid.
                        //
                        // Raise invalid disposition exception.
                        //
                    default :
                        RtlRaiseStatus(STATUS_INVALID_DISPOSITION);
                    }
                } while ((ExceptionFlags & EXCEPTION_COLLIDED_UNWIND) != 0);

            } else {
                //
                // If the target frame has not been reached, then swap
                // context pointers.
                // 没有到达目标栈帧，则交换上下文记录指针，继续搜索
                if (EstablisherFrame != (ULONG64)TargetFrame) {
                    TempContext = CurrentContext;
                    CurrentContext = PreviousContext;
                    PreviousContext = TempContext;
                }
            }

        } else {

            //
            // Set the point where control left the current function by
            // obtaining the return address from the top of the stack.
            // 控制离开当前函数，获取返回值，继续遍历之前的函数。
            CurrentContext->Rip = *(PULONG64)(CurrentContext->Rsp);
            CurrentContext->Rsp += 8;
        }
    } while ((RtlpIsFrameInBounds(&LowLimit, EstablisherFrame, &HighLimit) == TRUE) &&
             (EstablisherFrame != (ULONG64)TargetFrame));

    //
    // If the establisher stack pointer is equal to the target frame pointer,
    // then continue execution. Otherwise, an exit unwind was performed or the
    // target of the unwind did not exist and the debugger and subsystem are
    // given a second chance to handle the unwind.
    // 建立者栈帧 和目标栈帧相同，那么继续执行。否则执行展开退出，如果展开目标不存在，调试器和子系统
    // 会第二次处理展开
    if (EstablisherFrame == (ULONG64)TargetFrame) {
        CurrentContext->Rax = (ULONG64)ReturnValue;
        if (ExceptionRecord->ExceptionCode != STATUS_UNWIND_CONSOLIDATE) {
            CurrentContext->Rip = (ULONG64)TargetIp;
        }

        RtlRestoreContext(CurrentContext, ExceptionRecord);

    } else {
        // 对错误情况进行处理
        // If the old control PC is the same as the new control PC, then
        // no progress is being made and the function tables are most likely
        // malformed. Otherwise, give the debugger and subsystem a second
        // chance to handle the exception.
        if (ControlPc == CurrentContext->Rip) {
            RtlRaiseStatus(STATUS_BAD_FUNCTION_TABLE);
        } else {
            ZwRaiseException(ExceptionRecord, CurrentContext, FALSE);
        }
    }
}
```

通过注释对RtlUnwindEx()函数也就有了比较充分的了解，这里不再重复这个过程了。将整个异常处理的过程简单叙述一下，以帮助整体理解。

异常发生后，首先CPU会在内核内进行一些处理（这块见之前一些文章总结与分析），对于Ring3层的异常最终会调回到Ring3层的RtlDispatchException()函数（该函数内核和Ring3的逻辑是相同的），RtlDispatchException会根据ControlPC（即异常发生点的代码地址）查找运行时函数表（`RUNTIME_FUNCTIONS`）来查找目标表项。在找到目标表项后，调用RtlVirtualUnwind()函数进行虚拟展开，一方面找到当前函数的异常处理函数（`_C_specific_handler`）以调用它来过滤异常；另一方面，虚拟展开中会将当前栈帧展开，相关的内容填写到复制的一份上下文中，用于调用异常处理函数使用；再就是在虚拟展开中会获取到当前异常信息中的ScopeTable等数据。调用`_C_specific_handler`函数对ScopeTable遍历，进行异常过滤函数调用（可能是一个代码块，或简单地就是一个`EXCEPTION_EXECUTE_HANDLER`值）。如果异常过滤函数返回0，则继续遍历；如果异常过滤函数返回-1，则返回到异常发生位置继续执行；如果异常过滤函数返回1（处理异常），则进行异常展开，调用RtlUnwindEx()。异常展开函数类似异常分发过程，进行虚拟展开并在此调用异常处理函数`_C_specific_handler`（但是不同于异常分发，带有展开标记）。对于有finally语句的try块，则直接调用该代码块，否则继续进行展开；展开栈帧到了目标栈帧，则返回到`_C_specific_handler`函数，由于上下文已经被恢复到TargetIP（即异常处理的except代码块），`_C_specific_handler`函数直接return去继续执行接下来的代码。

这里将正常的异常处理过程过了一遍，还有许多内容没有详细分析总结，奈何能力不足，心有余力不足，待到以后资料阅读够多，再详细补充。

**未总结内容**

1. “ExceptionNestedException 和 ExceptionCollidedUnwind”

**参考文章**

* SEH分析笔记（X86篇）   [http://boxcounter.com/technique/2011-11-04-seh-x64/](http://boxcounter.com/technique/2011-11-04-seh-x64/)
* Exceptional Behavior -- x64 Structured Exception Handling		[http://www.osronline.com/article.cfm?article=469](http://www.osronline.com/article.cfm?article=469)
* 配置64位的程序（Visual C\+\+） [https://msdn.microsoft.com/zh-cn/library/h2k70f3s.aspx](https://msdn.microsoft.com/zh-cn/library/h2k70f3s.aspx)
* 异常处理 (x64) [https://msdn.microsoft.com/zh-cn/library/1eyas8tf.aspx](https://msdn.microsoft.com/zh-cn/library/1eyas8tf.aspx)
* Programming against the x64 exception handling support[http://www.nynaeve.net/?p=113](http://www.nynaeve.net/?p=113)
* SEH X64 [http://m.blog.csdn.net/qq_18218335/article/details/72722320](http://m.blog.csdn.net/qq_18218335/article/details/72722320)

**修订历史**

* 2017-08-16 11:21:23		完成文章


By Andy @2017-08-16 20:13:23