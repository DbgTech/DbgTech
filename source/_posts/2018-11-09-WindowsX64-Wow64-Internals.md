---
title: Windows WoW64浅析
date: 2018-11-09 20:15:33
tags:
- WoW64
- Windows
categories:
- 调试
---

WOW64（Windows-On-Windows 64bit）是X64 Windows操作系统的一个子系统，为32位应用程序提供运行环境。类似的还有WOW32子系统，负责在32位Windows系统上运行16位应用程序。

WoW64存在的原因还要从CPU的发展上开始说，X86指令集是一个指令集架构家族，最初在Intel 8086处理器中引入，开始它主要用于16位系统。从Intel 386处理器发布开始升级为32位，并且32位指令集一直保持了很久。了解32位系统的都知道32位CPU将内存空间限制到了4G（单一用户模式进程至少是这样）。随着RAM的越来越大，4G限制就成了瓶颈，系统无法使用更大的内存空间。于是2001年Intel发布了64位的IA64架构，它是一个全新的架构，架构设计非常清晰，比老的X86架构要更好。对于软件来说兼容性很重要，但是IA64处理器无法运行X86代码，这样问题就很严重了，已有的软件无法在新的CPU上运行。于是在2003年AMD发布了AMD64架构，它是对X86架构的增量更新，用于添加64位支持。这种架构的X64处理器可以执行X86代码，所以用户可以在X64处理器上运行现有的程序和操作系统。直到今天，X86和X64依旧是个人计算机和笔记本电脑使用CPU的主流。

如下为X64和X86指令，与X86指令相对应，X64指令需要增加额外的前缀字节（REX Prefix）表示使用64位寄存器。由于指针等数据大小翻倍，所以结构体中指针偏移大小也可能会增加。

```
// X86指令
0x8B, 0x52, 0x0C, // mov edx, dword ptr [edx+0Ch]

// X64指令
0x48, 0x8B, 0x52, 0x18, // mov rdx, qword ptr [rdx+18h]
```

一些指令在X86和X64上编码一致，比如短跳转。区分两种指令比较通用的方法是看指令是否携带了REX前缀字节（REX前缀字节用于表示使用64位寄存器或使用64位操作数）。REX前缀字节会覆盖一部分现存X86指令，因此在执行一块代码时需要告知X64处理器按照X86还是X64来解析指令。到底是怎么告知CPU要将代码按照X86解析还是按照X64解析呢？下面看Intel的白皮书给出的关于`IA-32e`如何区分兼容模式和64位模式。
<!--more-->
<div align="center">
![图1.64位模式中的代码段描述符](/img/2018-11-09-Wow64-code-segment-descriptor-in-64bit-mode.png)
</div>
从图1中的文字可知，Intel的CPU是根据代码段描述符的`CS.L`字段来确定，代码段描述符的该位在X86架构上没有使用的，这里被用于区分`IA-32e`下的兼容模式和X64模式。如果`CS.L=0`且处于`IA-32e`模式，那么CPU当前执行在兼容模式；如果`CS.L=0`且处于`IA-32e`模式，那么CPU当前运行在X64模式中。从这里可以看出，如果将两个代码段分别设置为不同值，其实在Ring3也是可以进行这种CPU模式切换的，这也就是WoW64所使用的方法。

Windows为了在X64系统上兼容32位程序设计了WoW64，用于在X64系统上执行32位应用程序。WoW64处理X64代码和X86代码之间的切换，并且为X86进程提供一个32位运行世界。由于X64是X86扩展，从32位代码向64位代码切换并不是那么困难，代码段描述符告诉处理器将代码当作X86代码还是X64代码，32位寄存器其实就是64位寄存器忽略高一半内容，32位模式RAM的4GB的编址和64位模式低4GB一样，因此从X86代码调用到X64代码，所有需要做的就是调用一个X64段选择子即完成了切换。

###WoW64简介###

WoW64层处理处理器的32位和64位模式切换以及模拟32位系统的事务。WOW64在用户模式下实现，作为32位ntdll.dll和内核之间的转换层，从技术上说WOW64是通过三个DLL实现，`Wow64.dll`是Windows NT内核的入口转换模块，实现了`Ntoskrnl.exe`入口的桩函数，在32位和64位调用之间进行转换，包括指针和调用栈的操控等；`Wow64win.dll`为32位GUI应用程序提供合适的入口指针，即`Win32k.sys`入口桩函数；`Wow64cpu.dll`负责将处理器的32位和64位的模式之间转换。

如下图1为典型的Wow64的实现原理图，上述的三个模块共同组成了WoW64模拟层：
<div align="center">
![图2 Wow64原理图](/img/2018-11-09-Wow64-32bit-process-on-64bit-os.jpg)
</div>
虽然从说法上看是在X64平台上提供了X86模拟环境，但实际上32位应用程序的执行并非模拟，而是由CPU直接解析执行，因此WoW64上的32位应用程序执行速度类似于32位Windows系统上的程序执行速度。当然这只是大致上来说，如果细究肯定不是完全一样。毕竟中间插入一层WoW64逻辑，相比于直接在32位系统上运行程序要多执行一部分代码，同时还包括X64和X86转换时的内存复制等操作，这些都会损耗性能，同时还会增加内存使用。

下面按照如下的几点对WoW64进行简单的分析。

1. 进程空间布局
2. 系统调用
3. 异常分发
4. 用户APC分发
5. 控制台支持
6. 用户回调
7. 注册表重定向
8. 文件系统重定向

###进程空间布局###

进程空间布局和32位系统上的进程类似，32位的系统模块会被映射到`0x80000000`以下的空间中，堆还是从低地址开始向上分布。因为WoW64进程是在X64进程基础上模拟X86环境，里面多出了前面说的三个模块`wow64cpu.dll`，`wow64.dll`，`wow64win.dll`，同时还包括X64的`ntdll.dll`。如上WoW64的简介中，三个模拟模块最终都要调用的64位的`ntdll.dll`中，因为它是进入内核入口。

**进程初始化**

WoW64进程本质上还是64位进程，在64位进程的基础上构造了32位运行环境，所以进程初始化时的第一个线程调用函数还是`ntdll!LdrInitializeThunk`，初始化安全`Cookie`后跳转到`ntdll!LdrpInitialize()`函数，紧接着会调用`ntdll!LdrpInitializeProcess`初始化进程中的内容，这个函数中会判断是否32位的EXE文件，即是否要加入WoW64层。如果是WoW64进程则加载`wow64.dll`，同时初始化`ntdll.dll`中的全局变量，如下所示：

```
0:000> x ntdll!Wow64*
00000000`77482da8 ntdll!Wow64ApcRoutine = <no type information>
00000000`77482f08 ntdll!Wow64PrepareForException = <no type information>
00000000`77482e18 ntdll!Wow64Handle = <no type information>
00000000`77482f10 ntdll!Wow64LdrpInitialize = <no type information>
```

初始化WoW64层时，首先调用`wow64!Wow64LdrpInitialize`，函数中进一步调用`wow64!ProcessInit`进行进程初始化，初始化内容简单列举如下：

1. 读取`Wow64ExecuteFlags`标记，设置进程的提交栈大小和最大栈值。
2. 加载`wow64log.dll`初始化WoW64的日志模块。
3. 初始化WoW64共享信息变量`wow64!Wow64SharedInformation`，并赋值全局变量`Ntdll32KiUserExceptionDispatcher`，`Ntdll32KiUserApcDispatcher`等用于WoW64的异常处理，APC分发，内核回调`user32.dll`模块等。
4. 调用函数`Wow64pInitializeFilePathRedirection`设置文件路径重定向。
5. 初始化`ServiceTables`全局变量，用于WoW64中系统调用时进行调用分发。
6. 用EXE全路径调用`CpuProcessInit()`函数，修改X64与X86切换代码内容。

在`wow64!ProcessInit`函数中开始时对`Wow64Info`的部分内容进行初始化，`Wow64Info`的位置有如下的一个指向关系，可以用作想要调试的人参考资料。

```
0:000> dt 0x7efdb000 ntdll!_TEB		// X64的 TEB
   +0x000 NtTib            : _NT_TIB
      +0x000 ExceptionList    : 0x00000000`7efdd000 // X86 TEB指针
   +0x008 StackBase        : 0x00000000`000bfd20 Void
   ...
   +0x1478 DeallocationStack : 0x00000000`001d0000 Void
   +0x1480 TlsSlots         : [64] (null)	// 0x14D0   保存了 WOW64_TLS_WOW64INFO
   ...
   +0x2030 lpPEB32          :               // X86 PEB结构体地址
   ...
   +0x20C0 lpX86SwitchTo64BitMode           // X86向X64切换的代码 地址

0:000> dt 7efde000 ntdll32!_PEB		// X86的 PEB，X64的TEB中的一项TLS指向PEB后的Wow64Info结构体
   +0x000 InheritedAddressSpace : 0 ''
   ...
   +0x23c pImageHeaderHash : (null)
   +0x240 TracingFlags     : 0
   +0x248 Wow64Info		   ; Wow64Info 结构体

0:000:x86> dt ntdll32!_TEB 0x7efdd000	// x86 TEB + 0xf70 偏移处 GdiBatchCount
   +0x000 NtTib            : _NT_TIB
   ...
   +0xf6c WinSockData      : (null)
   +0xf70 GdiBatchCount    : 0x7efdb000
   +0xf74 CurrentIdealProcessor : _PROCESSOR_NUMBER
   ...
```

```
typedef struct _WOW64INFO {
    ULONG NativeSystemPageSize;     // 模拟器所在本地系统的页面大小
    ULONG CpuFlags;
    WOW64_EXECUTE_OPTIONS Wow64ExecuteFlags;
} WOW64INFO, *PWOW64INFO;
```

读取注册表和加载`wow64log.dll`进行初始化这两部分很简单！接着就是调用`wow64!Wow64GetSharedInformation`函数初始化共享信息全局变量（`wow64!Wow64SharedInformation`）指向的是一个指针表，其成员内容如下所示，它们用于线程初始化，异常分发，APC分发等。

```
wow64!Wow64SharedInformation
0:000> dds 000000007747b120
00000000`7747b120  775497a9 ntdll32!LdrInitializeThunk			// wow64!Ntdll32LoaderInitRoutine
00000000`7747b124  77520154 ntdll32!KiUserExceptionDispatcher   // wow64!Ntdll32KiUserExceptionDispatcher
00000000`7747b128  77520058 ntdll32!KiUserApcDispatcher			// wow64!Ntdll32KiUserApcDispatcher
00000000`7747b12c  7752010c ntdll32!KiUserCallbackDispatcher    // wow64!Ntdll32KiUserCallbackDispatcher
00000000`7747b130  775af694 ntdll32!LdrHotPatchRoutine
00000000`7747b134  775427b1 ntdll32!ExpInterlockedPopEntrySListFault
00000000`7747b138  7754277b ntdll32!ExpInterlockedPopEntrySListResume
00000000`7747b13c  775427b3 ntdll32!ExpInterlockedPopEntrySListEnd
00000000`7747b140  775201e4 ntdll32!RtlUserThreadStart
00000000`7747b144  775b38a0 ntdll32!RtlpQueryProcessDebugInformationRemote
00000000`7747b148  7756a02d ntdll32!EtwpNotificationThread
00000000`7747b14c  77510000 ntdll32!`string' <PERF> (ntdll32+0x0)	// wow64!NtDll32Base
```

调用`wow64!Map64BitDlls`将X64的一些系统DLL，比如`kernel32.dll`所占用的基地址分配掉，防止32位相同名字DLL分配到这里？初始化全局变量`wow64!ServiceTables`，其中包含了`Nt`类函数分发表，控制台`con`类函数分发表，`Win32`GUI类函数分发表和`Csr`类函数分发表。获取Image名字，然后调用`wow64cpu!CpuProcessInit`函数判断X86和X64代码切换部分是否完整，不完整进行修改；`CpuProcessInit()`函数设置了`TEB32+0xC0`处为X86到X64转换代码指针。

接下来调用`wow64cpu!CpuGetContext`函数获取X86开始执行时的`CONTEXT`内容（打印一个结构内容如下），然后获取起始运行地址，并将地址转化为32位地址。

```
0:000> dt 0x00000014e5f0 ntdll32!_Context
   +0x000 ContextFlags     : 0x1002f
   +0x004 Dr0              : 0
   +0x008 Dr1              : 0x74d95cd0
   +0x00c Dr2              : 0
   +0x010 Dr3              : 0x74d95388
   +0x014 Dr6              : 0
   +0x018 Dr7              : 0x7737d96e
   +0x01c FloatSave        : _FLOATING_SAVE_AREA
   +0x08c SegGs            : 0x2b
   +0x090 SegFs            : 0x53
   +0x094 SegEs            : 0x2b
   +0x098 SegDs            : 0x2b
   +0x09c Edi              : 0
   +0x0a0 Esi              : 0
   +0x0a4 Ebx              : 0x7efde000
   +0x0a8 Edx              : 0
   +0x0ac Ecx              : 0
   +0x0b0 Eax              : 0xfbfcdd
   +0x0b4 Ebp              : 0
   +0x0b8 Eip              : 0x775201e4
   +0x0bc SegCs            : 0x23
   +0x0c0 EFlags           : 0x202
   +0x0c4 Esp              : 0x30fa10
   +0x0c8 SegSs            : 0x2b
   +0x0cc ExtendedRegisters : [512]  "???"
```

在`wow64!Wow64SetupInitialCall`函数中将X64栈上的X86的`CONTEXT`复制到32位栈上，其中先调用`CpuGetStackPointer`从TEB64的`WOW64_TLS_CPURESERVED`TLS项中保存的32位CPU信息，CPU信息结构中`0xC8`偏移处保存的32位栈帧信息，栈帧减去`0x2CC`(`CONTEXT`结构体大小)，然后再减去16字节，将之前获取的`CONTEXT`内容复制过去，然后再将`CONTEXT`在栈上指针和`ntdll32.dll`句柄放到多申请的16字节中，调用`CpuSetStackPointer`设置回TEB64的TLS的`CPU`保留信息项中。接着调用`wow64cpu!CpuSetInstructionPointer`将`Ntdll32LoaderInitRoutine`全局变量保存信息（`ntdll32!RtlUserThreadStart`）设置到CPU保留信息的`0xBC`偏移处，即`CONTEXT`的`Eip`字段。

调用`wow64!RunCpuSimulation()`进入CPU模拟，它调用`wow64cpu!CpuSimulate()`开始X86环境的模拟。在`wow64cpu!CpuSimulate`函数中，首先保存X64环境的寄存器到64位栈上，然后将`r14`设置为X64栈信息，`r12`寄存器设置为64位TEB地址，`r15`设置为快速（Turbo）系统调用表基地址，`r13`设置为X86的CPU保留信息结构体基地址。

查看X86环境块`CONTEXT`中是否用了浮点寄存器，如果使用了则将对应信息恢复到浮点寄存器中，并返回执行；否则是普通计算，则恢复各个寄存器内容，执行远跳转指令`jmp fword ptr [r14] ds:00000000`001deb40=0023:772d97a9`跳转到32位开始执行，可反汇编查看32位开始执行的函数为`ntdll32!LdrInitializeThunk`，即X86进程的进程初始化函数入口。

###系统调用###

对于应用程序来说比较重要的一部分就是系统调用，它们为应用程序正常运行提供了基础服务。无论X86还是X64的进程，如果能在Ring3层完成功能，则不需要进行系统调用，这个不存在和WoW64交互，直接加载32位模块调用函数即可，比如`StringCchCopy`等函数；再一种就是必须系统支持，比如读取文件内容`kernel32!CreateFileW()`，我们知道这些函数调用都会会进入`ntdll.dll`中，最终进入到内核。这种需要进入内核的调用是需要WoW64处理的内容。下面以`kernel32!CreateFileW`为例看WoW64如何处理系统调用。

```
int wmain(int argc, WCHAR* argv[])
{
	WCHAR szFilePath[MAX_PATH + 1] = L"C:\\Users\\Administrator\\Desktop\\Test\\Debug\\Test.pdb";
	HANDLE hFile = CreateFileW(szFilePath, GENERIC_READ, FILE_SHARE_READ, NULL,	OPEN_EXISTING, 0, NULL);
	if (hFile == INVALID_HANDLE_VALUE)
	{
		printf("Open File Error!\n");
	}

	return 0;
}
```

首先设置断点如下，要X64和X86的`ntdll.dll`都设置上断点，看一下最终调用到系统内核之前的所有调用栈。

```
0:000:x86> bl
 0 e x86 011aa5d0     0001 (0001)  0:**** Test!wmain
 1 e 76d79df0     0001 (0001)  0:**** ntdll!ZwCreateFile
 2 e x86 76ef00e4     0001 (0001)  0:**** ntdll32!ZwCreateFile
```

首先进入到X86的`ntdll.dll`中的`ntdll32!ZwCreateFile`函数，如果是在X86系统上，那么这个函数内部就会执行中断或快速系统调用进入内核，但是在WoW64上显然这里是无法进入内核的。

```
0:000:x86> k
ChildEBP RetAddr
0028f86c 769fc76b ntdll32!ZwCreateFile
0028f910 764d40ae KERNELBASE!CreateFileW+0x35e
0028f93c 011aa640 kernel32!CreateFileWImplementation+0x69
0028fc50 011ff506 Test!wmain+0x70 [c:\users\administrator\desktop\test\test\test.cpp @ 146]
0028fc9c 011ff3df Test!__tmainCRTStartup+0x116 [f:\dd\vctools\crt_bld\self_x86\crt\src\crt0.c @ 266]
0028fca4 764d343d Test!wmainCRTStartup+0xf [f:\dd\vctools\crt_bld\self_x86\crt\src\crt0.c @ 182]
0028fcb0 76f09832 kernel32!BaseThreadInitThunk+0xe
0028fcf0 76f09805 ntdll32!__RtlUserThreadStart+0x70
0028fd08 00000000 ntdll32!_RtlUserThreadStart+0x1b
```

反汇编看一下`ntdll32!ZwCreateFile`函数的内容，可以发现在`ecx`寄存器中存入系统调用号，`edx`保存了栈指针，然后调用的是`fs:[0C0h]`。

```
ntdll32!ZwCreateFile:
76ef00e4 b852000000      mov     eax,52h
76ef00e9 33c9            xor     ecx,ecx
76ef00eb 8d542404        lea     edx,[esp+4]
76ef00ef 64ff15c0000000  call    dword ptr fs:[0C0h]
76ef00f6 83c404          add     esp,4
76ef00f9 c22c00          ret     2Ch
```

我们知道X86系统上`fs`寄存器中其实保存的是`TEB`的基地址，`fs:[0C0h]`即TEB中`0xC0`偏移处的内容，如下：

```
0:000:x86> dt 0x7efdd000 ntdll32!_TEB
   +0x000 NtTib            : _NT_TIB
   ......
   +0x044 User32Reserved   : [26] 0
   +0x0ac UserReserved     : [5] 0
   +0x0c0 WOW32Reserved    : 0x746b2320 Void
   +0x0c4 CurrentLocale    : 0x804
   ......
   +0xf6c WinSockData      : (null)
   +0xf70 GdiBatchCount    : 0x7efdb000
   +0xf74 CurrentIdealProcessor : _PROCESSOR_NUMBER
   ......
```

注意这里是X86的`TEB`，在`0xC0`偏移处为`WOW32Reserved`字段，名字就可知它与WoW64相关的内容，反汇编其内容，如下代码。

```
wow64cpu!X86SwitchTo64BitMode:
746b2320 ea1e276b743300  jmp     0033:746B271E
```

在上一节中看到过这个全局变量，在`wow64cpu!CpuProcessInit`中判断，如果没有进行初始化，则会将这块代码进行初始化。它的代码很简单，远跳转到`0033:746B271E`，根据CPU那块知识可知这是在将CPU从`IA32e`的兼容模式向64位切换。

```
0:000> dg 0x23
                                                    P Si Gr Pr Lo
Sel        Base              Limit          Type    l ze an es ng Flags
---- ----------------- ----------------- ---------- - -- -- -- -- --------
0023 00000000`00000000 00000000`ffffffff Code RE Ac 3 Bg Pg P  Nl 00000cfb

1: kd> dg 0x33
                                                    P Si Gr Pr Lo
Sel        Base              Limit          Type    l ze an es ng Flags
---- ----------------- ----------------- ---------- - -- -- -- -- --------
0033 00000000`00000000 00000000`00000000 Code RE Ac 3 Nb By P  Lo 000002fb
```

> 其中Sel表示选择子，Base和Limit分别是基地址和边界，Type是段的类型，RE代表只读和可执行,Ac表示被访问过，Pl为3是ring3特权级，Bg(Big)表示为32位代码，Gran表示粒度，Pg意味着粒度的单位是内存而(4KB) , Pres代表Present即这个段是否在内存中，Long下的N1表示Not Long，意味着这不是64位代码，Lo表示这是64位代码。

切换到X64模式后，执行的内容如下（从jmp后面的地址可以看到是这里的代码）。

```
wow64cpu!CpupReturnFromSimulatedCode:
00000000`746b271e 67448b0424      mov     r8d,dword ptr [esp] ds:00000000`0028f86c=76ef00f6
00000000`746b2723 458985bc000000  mov     dword ptr [r13+0BCh],r8d
00000000`746b272a 4189a5c8000000  mov     dword ptr [r13+0C8h],esp
00000000`746b2731 498ba42480140000 mov     rsp,qword ptr [r12+1480h]
00000000`746b2739 4983a4248014000000 and   qword ptr [r12+1480h],0
00000000`746b2742 448bda          mov     r11d,edx
wow64cpu!TurboDispatchJumpAddressStart:
00000000`746b2745 41ff24cf        jmp     qword ptr [r15+rcx*8]
```

在上一节中的WoW64环境初始化中知道，`r13`寄存器指向CPU保留信息，其实是一个指针加上`CONTEXT`，将返回地址放到`CONTEXT`的`EIP`字段，`esp`寄存器放到`ESP`字段；`r12`寄存器指向X64环境中的TEB指针，其中`0x1480`偏移为TLS指针数组，这个偏移处为`WOW64_TLS_STACKPTR64`，即保存了上次离开X64环境时X64栈的栈顶。这里将栈恢复到`rsp`寄存器中，继续`CpuSimulate`函数中跳到X86时的状态继续执行。清空`WOW64_TLS_STACKPTR64`TLS内容，保存X86的栈指针（`edx`保存）到`r11`。

接下来是一个跳转，这个跳转用于选择系统调用方式，WoW64提供了一种快速(Turbo)方式，即X86栈上已经形成了可以进行X64系统调用的形式，根据`ecx`值确定系统调用参数个数和方式，可以直接用X86栈上内容进行系统调用。这种分发所用的表如下所示，不同的值表示不同的调用方式。

```
0:000> dqs @r15
00000000`746b2450  00000000`746b2749 wow64cpu!TurboDispatchJumpAddressEnd
00000000`746b2458  00000000`746b2dba wow64cpu!Thunk0Arg
00000000`746b2460  00000000`746b2bce wow64cpu!Thunk0ArgReloadState
00000000`746b2468  00000000`746b2d6a wow64cpu!Thunk1ArgSp
00000000`746b2470  00000000`746b2d7b wow64cpu!Thunk1ArgNSp
00000000`746b2478  00000000`746b2d77 wow64cpu!Thunk2ArgNSpNSp
00000000`746b2480  00000000`746b2c8a wow64cpu!Thunk2ArgNSpNSpReloadState
00000000`746b2488  00000000`746b2d84 wow64cpu!Thunk2ArgSpNSp
00000000`746b2490  00000000`746b2d66 wow64cpu!Thunk2ArgSpSp
00000000`746b2498  00000000`746b2d55 wow64cpu!Thunk2ArgNSpSp
00000000`746b24a0  00000000`746b2d73 wow64cpu!Thunk3ArgNSpNSpNSp
00000000`746b24a8  00000000`746b2d62 wow64cpu!Thunk3ArgSpSpSp
00000000`746b24b0  00000000`746b2d8d wow64cpu!Thunk3ArgSpNSpNSp
00000000`746b24b8  00000000`746b2bc3 wow64cpu!Thunk3ArgSpNSpNSpReloadState
00000000`746b24c0  00000000`746b2d9e wow64cpu!Thunk3ArgSpSpNSp
00000000`746b24c8  00000000`746b2d51 wow64cpu!Thunk3ArgNSpSpNSp
00000000`746b24d0  00000000`746b2d80 wow64cpu!Thunk3ArgSpNSpSp
00000000`746b24d8  00000000`746b2d6f wow64cpu!Thunk4ArgNSpNSpNSpNSp
00000000`746b24e0  00000000`746b2d9a wow64cpu!Thunk4ArgSpSpNSpNSp
00000000`746b24e8  00000000`746b2af4 wow64cpu!Thunk4ArgSpSpNSpNSpReloadState
00000000`746b24f0  00000000`746b2dab wow64cpu!Thunk4ArgSpNSpNSpNSp
00000000`746b24f8  00000000`746b2bbf wow64cpu!Thunk4ArgSpNSpNSpNSpReloadState
00000000`746b2500  00000000`746b2d4d wow64cpu!Thunk4ArgNSpSpNSpNSp
00000000`746b2508  00000000`746b2d5e wow64cpu!Thunk4ArgSpSpSpNSp
00000000`746b2510  00000000`746b27be wow64cpu!QuerySystemTime
00000000`746b2518  00000000`746b2783 wow64cpu!GetCurrentProcessorNumber
00000000`746b2520  00000000`746b2991 wow64cpu!ReadWriteFile
00000000`746b2528  00000000`746b28d7 wow64cpu!DeviceIoctlFile
00000000`746b2530  00000000`746b2a42 wow64cpu!RemoveIoCompletion
00000000`746b2538  00000000`746b27fc wow64cpu!WaitForMultipleObjects
00000000`746b2540  00000000`746b2803 wow64cpu!WaitForMultipleObjects32
00000000`746b2548  00000000`746b2782 wow64cpu!ThunkNone
```

如下`ntdll32!ZwDelayExecution`函数就是用了这种调用方式，它将`ecx`寄存器设置为6，在进入`WoW64`后实际调用的函数为`wow64cpu!Thunk2ArgNSpNSpReloadState`，在该函数中就调用了`wow64cpu!CpupSyscallStub`直接进入内核。

```
0:000:x86> u ntdll32!ZwDelayExecution
ntdll32!ZwDelayExecution:
7743fdac b831000000      mov     eax,31h
7743fdb1 b906000000      mov     ecx,6
7743fdb6 8d542404        lea     edx,[esp+4]
7743fdba 64ff15c0000000  call    dword ptr fs:[0C0h]
7743fdc1 83c404          add     esp,4
7743fdc4 c20800          ret     8
```

这里看一下另一个用到系统调用的模块，即`User32.dll`，以`user32!CreateWindowExW`为例，调用到系统调用层时汇编如下所示，`ecx`寄存器内容为0，而调用号为`0x1076`

```
0:000:x86> u 75b1a950
USER32!NtUserCreateWindowEx:
75b1a950 b876100000      mov     eax,1076h
75b1a955 b900000000      mov     ecx,0
75b1a95a 8d542404        lea     edx,[esp+4]
75b1a95e 64ff15c0000000  call    dword ptr fs:[0C0h]
75b1a965 83c404          add     esp,4
75b1a968 c23c00          ret     3Ch
```

但是对于`ntdll32.dll`转过来的系统调用都不走这种快速方式，而是需要在WoW64中进行参数的调整，然后再进行调用。`ecx`值为0，这里直接跳转到如下的内容上继续执行。

```
wow64cpu!TurboDispatchJumpAddressEnd:
00000000`746b2749 4189b5a4000000  mov     dword ptr [r13+0A4h],esi
00000000`746b2750 4189bda0000000  mov     dword ptr [r13+0A0h],edi
00000000`746b2757 41899da8000000  mov     dword ptr [r13+0A8h],ebx
00000000`746b275e 4189adb8000000  mov     dword ptr [r13+0B8h],ebp
00000000`746b2765 9c              pushfq
00000000`746b2766 5b              pop     rbx
00000000`746b2767 41899dc4000000  mov     dword ptr [r13+0C4h],ebx
00000000`746b276e 8bc8            mov     ecx,eax
00000000`746b2770 ff150ae9ffff    call    qword ptr [wow64cpu!_imp_Wow64SystemServiceEx
00000000`746b2776 418985b4000000  mov     dword ptr [r13+0B4h],eax
00000000`746b277d e98ffeffff      jmp     wow64cpu!CpuSimulate+0x61 (00000000`746b2611)

//////////////////////////////////////////////////////////////
0:000> dt ntdll32!_CONTEXT 00000000001dfd24		// CPUReserved
   -0x004 XXX
   +0x000 ContextFlags     : 0x1002f
   +0x004 Dr0              : 0
   +0x008 Dr1              : 0
   +0x00c Dr2              : 0
   +0x010 Dr3              : 0
   +0x014 Dr6              : 0
   +0x018 Dr7              : 0
   +0x01c FloatSave        : _FLOATING_SAVE_AREA
   +0x08c SegGs            : 0x2b
   +0x090 SegFs            : 0x53
   +0x094 SegEs            : 0x2b
   +0x098 SegDs            : 0x2b
   +0x09c Edi              : 0
   +0x0a0 Esi              : 0
   +0x0a4 Ebx              : 0x7efde000
   +0x0a8 Edx              : 0
   +0x0ac Ecx              : 0
   +0x0b0 Eax              : 0
   +0x0b4 Ebp              : 0x38fb3c
   +0x0b8 Eip              : 0x772c00f6
   +0x0bc SegCs            : 0x23
   +0x0c0 EFlags           : 0x246
   +0x0c4 Esp              : 0x38f7f8
   +0x0c8 SegSs            : 0x2b
   +0x0cc ExtendedRegisters : [512]  "???"
```

继续执行过程中，将X86转过来时寄存器的内容依次放到`r13`指向的CPU保留信息（包含的`CONTEXT`）中。保存完CPU信息后，调用`wow64!Wow64SystemServiceEx`函数进行函数调用分发。

在该函数中，根据系统调用号，获取系统调用类别（四类，如下`wow64.dll`全局变量所保存内容，初始化时对该变量进行过初始化。），将系统调用号右移12位，然后与上3，该值用于选取`wow64!ServiceTables`中的四类之一。从上面`ntdll32.dll`过来的为第一类，而`user32.dll`过来的调用为第二类。

```
wow64!ServiceTables
0:000> dqs 7475aa00
00000000`7475aa00  00000000`747592a0 wow64!sdwhnt32JumpTable
00000000`7475aa08  00000000`00000000
00000000`7475aa10  00000000`000003e8
00000000`7475aa18  00000000`74759fc0 wow64!sdwhnt32Number
00000000`7475aa20  00000000`00000000
00000000`7475aa28  00000000`00000000

00000000`7475aa30  00000000`7470fae0 wow64win!sdwhwin32JumpTable
00000000`7475aa38  00000000`00000000
00000000`7475aa40  00000000`000003e8
00000000`7475aa48  00000000`747114b0 wow64win!sdwhwin32Number
00000000`7475aa50  00000000`00000000
00000000`7475aa58  00000000`7470e110 wow64win!sdwhwin32ErrorCase

00000000`7475aa60  00000000`74711b40 wow64win!sdwhconJumpTable
00000000`7475aa68  00000000`00000000
00000000`7475aa70  00000000`000003e8
00000000`7475aa78  00000000`74711e60 wow64win!sdwhconNumber
00000000`7475aa80  00000000`00000000
00000000`7475aa88  00000000`74711820 wow64win!sdwhconErrorCase

00000000`7475aa90  00000000`747591e0 wow64!sdwhbaseJumpTable
00000000`7475aa98  00000000`00000000
00000000`7475aaa0  00000000`000003e8
00000000`7475aaa8  00000000`74759258 wow64!sdwhbaseNumber
00000000`7475aab0  00000000`00000000
00000000`7475aab8  00000000`74759160 wow64!sdwhbaseErrorCase
```

这里对于`ntdll32.dll`调用过来的，会查找`wow64!sdwhnt32JumpTable`表，我们这里查找到的函数就是`wow64!whNtCreateFile`了，进入该函数中，将X86上的参数整合到X64栈上，然后调用`ntdll!ZwCreateFile`，这时的调用栈如下。

```
0:000> !wow64exts.k
Walking 64bit Stack...
Child-SP          RetAddr           Call Site
00000000`0016e028 00000000`7473c25b ntdll!ZwCreateFile
00000000`0016e030 00000000`7472d18f wow64!whNtCreateFile+0x10f
00000000`0016e100 00000000`746b2776 wow64!Wow64SystemServiceEx+0xd7
00000000`0016e9c0 00000000`7472d286 wow64cpu!TurboDispatchJumpAddressEnd+0x2d
00000000`0016ea80 00000000`7472c69e wow64!RunCpuSimulation+0xa
00000000`0016ead0 00000000`76d44223 wow64!Wow64LdrpInitialize+0x42a
00000000`0016f020 00000000`76da9a60 ntdll!LdrpInitializeProcess+0x17e3
00000000`0016f510 00000000`76d5374e ntdll! ?? ::FNODOBFM::`string'+0x22a50
00000000`0016f580 00000000`00000000 ntdll!LdrInitializeThunk+0xe

Walking 32bit Stack...
ChildEBP RetAddr
0028f86c 769fc76b ntdll32!NtCreateFile+0x12
0028f910 764d40ae KERNELBASE!CreateFileW+0x35e
0028f93c 011aa640 kernel32!CreateFileWImplementation+0x69
0028fc50 011ff506 Test!wmain+0x70 [c:\users\administrator\desktop\test\test\test.cpp @ 146]
0028fc9c 011ff3df Test!__tmainCRTStartup+0x116 [f:\dd\vctools\crt_bld\self_x86\crt\src\crt0.c @ 266]
0028fca4 764d343d Test!wmainCRTStartup+0xf [f:\dd\vctools\crt_bld\self_x86\crt\src\crt0.c @ 182]
0028fcb0 76f09832 kernel32!BaseThreadInitThunk+0xe
0028fcf0 76f09805 ntdll32!__RtlUserThreadStart+0x70
0028fd08 00000000 ntdll32!_RtlUserThreadStart+0x1b
```

等到`ntdll!ZwCreateFile`和`ntdll!ZwCreateFile`依次返回后，保存系统调用返回结果（一般是错误值，放到`CONTEXT`的EAX字段），然后继续跳转，`wow64cpu!CpuSimulate+0x61`这块内容很熟悉了，即初始化时会经过此处，判断是否在协处理器上运行，如果是则恢复浮点寄存器，否则恢复X86环境中的寄存器内容（之前进入X64时已经保存），进而跳转到32位代码环境中继续执行。

```
00000000`746b2611 4183a5d002000001 and     dword ptr [r13+2D0h],1 ds:00000000`0016fff0=00000000
00000000`746b2619 0f84af000000    je      wow64cpu!CpuSimulate+0x11e (00000000`746b26ce)
00000000`746b261f 410f288570010000 movaps  xmm0,xmmword ptr [r13+170h]
00000000`746b2627 410f288d80010000 movaps  xmm1,xmmword ptr [r13+180h]
00000000`746b262f 410f289590010000 movaps  xmm2,xmmword ptr [r13+190h]
00000000`746b2637 410f289da0010000 movaps  xmm3,xmmword ptr [r13+1A0h]
00000000`746b263f 410f28a5b0010000 movaps  xmm4,xmmword ptr [r13+1B0h]
00000000`746b2647 410f28adc0010000 movaps  xmm5,xmmword ptr [r13+1C0h]
00000000`746b264f 418b8db0000000  mov     ecx,dword ptr [r13+0B0h]
00000000`746b2656 418b95ac000000  mov     edx,dword ptr [r13+0ACh]
00000000`746b265d 4183a5d0020000fe and     dword ptr [r13+2D0h],0FFFFFFFEh
00000000`746b2665 418bbda0000000  mov     edi,dword ptr [r13+0A0h]
00000000`746b266c 418bb5a4000000  mov     esi,dword ptr [r13+0A4h]
00000000`746b2673 418b9da8000000  mov     ebx,dword ptr [r13+0A8h]
00000000`746b267a 418badb8000000  mov     ebp,dword ptr [r13+0B8h]
00000000`746b2681 418b85b4000000  mov     eax,dword ptr [r13+0B4h]
00000000`746b2688 4989a42480140000 mov     qword ptr [r12+1480h],rsp
00000000`746b2690 66c74424082300  mov     word ptr [rsp+8],23h
00000000`746b2697 66c74424202b00  mov     word ptr [rsp+20h],2Bh
00000000`746b269e 458b85c4000000  mov     r8d,dword ptr [r13+0C4h]
00000000`746b26a5 4181a5c4000000fffeffff and dword ptr [r13+0C4h],0FFFFFEFFh
00000000`746b26b0 4489442410      mov     dword ptr [rsp+10h],r8d
00000000`746b26b5 458b85c8000000  mov     r8d,dword ptr [r13+0C8h]
00000000`746b26bc 4c89442418      mov     qword ptr [rsp+18h],r8
00000000`746b26c1 458b85bc000000  mov     r8d,dword ptr [r13+0BCh]
00000000`746b26c8 4c890424        mov     qword ptr [rsp],r8
00000000`746b26cc 48cf            iretq

// 如下恢复X86环境中的寄存器内容
00000000`746b26ce 418bbda0000000  mov     edi,dword ptr [r13+0A0h]
00000000`746b26d5 418bb5a4000000  mov     esi,dword ptr [r13+0A4h]
00000000`746b26dc 418b9da8000000  mov     ebx,dword ptr [r13+0A8h]
00000000`746b26e3 418badb8000000  mov     ebp,dword ptr [r13+0B8h]
00000000`746b26ea 418b85b4000000  mov     eax,dword ptr [r13+0B4h]
00000000`746b26f1 4989a42480140000 mov     qword ptr [r12+1480h],rsp
00000000`746b26f9 41c7460423000000 mov     dword ptr [r14+4],23h
00000000`746b2701 41b82b000000    mov     r8d,2Bh
00000000`746b2707 418ed0          mov     ss,r8w
00000000`746b270a 418ba5c8000000  mov     esp,dword ptr [r13+0C8h]
00000000`746b2711 458b8dbc000000  mov     r9d,dword ptr [r13+0BCh]
00000000`746b2718 45890e          mov     dword ptr [r14],r9d
00000000`746b271b 41ff2e          jmp     fword ptr [r14] ds:00000000`0016ea30=002376ef00f6
```

从这里就回到了X86中`ntdll32!ZwCreateFile`中继续执行。

**补充**

当然了X64调试器是提供了扩展模块专门用来调试WoW64进程，即`wow64exts.dll`，在Windbg中可以使用`.load wow64exts`命令来加载该扩展。

```
0:000:x86> !wow64exts.help

Wow64 debugger extension commands:

sw:            Switch between 32-bit and 64-bit mode
k <count>:     Combined 32/64 stack trace
info:          Dumps information about some important wow64 structures
r [addr]:      Dumps x86 CONTEXT
lf:            Dump/Set log flags
l2f:           Enable logging to file
```

比如`!wow64exts.info`命令执行后列举出了X86和X64的基础信息，包括PEB，TEB，堆栈信息以及TEB64中的TLS内容。

```
0:000:x86> !wow64exts.info
PEB32: 0x7efde000
PEB64: 0x7efdf000

Wow64 information for current thread:

TEB32: 0x7efdd000
TEB64: 0x7efdb000

32 bit, StackBase   : 0x290000
        StackLimit  : 0x28d000
        Deallocation: 0x190000

64 bit, StackBase   : 0x16fd20
        StackLimit  : 0x16c000
        Deallocation: 0x130000

Wow64 TLS slots:
WOW64_TLS_STACKPTR64:       0x000000000016e9c0	// X64环境转向X86时的栈
WOW64_TLS_CPURESERVED:      0x000000000016fd20  // CPU保留信息，在0x4偏移处为X86的CONTEXT
WOW64_TLS_INCPUSIMULATION:  0x0000000000000000
WOW64_TLS_LOCALTHREADHEAP:  0x0000000000000000
WOW64_TLS_EXCEPTIONADDR:    0x0000000000000000
WOW64_TLS_USERCALLBACKDATA: 0x0000000000000000
WOW64_TLS_EXTENDED_FLOAT:   0x0000000000000000
WOW64_TLS_APCLIST:          0x0000000000000000
WOW64_TLS_FILESYSREDIR:     0x0000000000000000
WOW64_TLS_LASTWOWCALL:      0x0000000000000000
WOW64_TLS_WOW64INFO:        0x000000007efde248	// WoW64Info 结构体指针，在PEB32末尾
```

###异常分发###

从前面知道X86环境是WoW64子系统"模拟"出来的，所以WoW64就相当于X86环境的内核，所以对于异常分发来说从内核通过调用`ntdll!KiUserExceptionDispatcher`函数将异常分发到Ring3尝试找异常处理函数，我们关系的是WoW64对于异常的处理方式，这里只需要看一下从X64内核将异常转到`ntdll32!KiUserExceptionDispatcher`这一段即可。这里有从网上找来的一段关于WoW64处理异常的论述：

> WoW64通过`ntdll.dll`的`KiUserExceptionDispatcher`勾住了异常分发过程。无论何时当64位内核将要给一个`WoW64`进程分发一个异常时，Wow64会捕获住原生的异常以及用户模式下的环境记录（context record），然后准备一个32位异常和环境记录，并且按照原生32位内核所做的那样将他分发出去。

从`ntdll!KiUserExceptionDispatcher`函数开始看一X64的Ring3异常分发到WoW64的异常分发这个过程，它的汇编代码如下，首先判断全局变量`ntdll!Wow64PrepareForException`是否赋值，它其实是在加载`wow64.dll`模块时将该模块同名导出函数赋值给该变量了。

如果调试的话可以注意到，其实这时代码执行处于X86的栈上，所以要将X86栈上的数据放到X64栈上，并将栈切换到X64栈上去，通过调用`ntdll!Wow64PrepareForException`来完成这个工作。在`ntdll!Wow64PrepareForException`函数中会调用`wow64!CpuResetToConsistentState`，它主要是将异常信息机构复制到`wow64cpu!RecoverException64`变量中，将异常对应的上下文结果复制到`wow64cpu!RecoverContext64`，然后切换栈，转向使用64位代码的专用栈。

```
ntdll!KiUserExceptionDispatcher:
00000000`772cb610 fc              cld
00000000`772cb611 488b05f0780c00  mov     rax,qword ptr [ntdll!Wow64PrepareForException]
00000000`772cb618 4885c0          test    rax,rax
00000000`772cb61b 740f            je      ntdll!KiUserExceptionDispatcher+0x1c (00000000`772cb62c)
00000000`772cb61d 488bcc          mov     rcx,rsp
00000000`772cb620 4881c1f0040000  add     rcx,4F0h
00000000`772cb627 488bd4          mov     rdx,rsp
00000000`772cb62a ffd0            call    rax
00000000`772cb62c 488bcc          mov     rcx,rsp
00000000`772cb62f 4881c1f0040000  add     rcx,4F0h
00000000`772cb636 488bd4          mov     rdx,rsp
00000000`772cb639 e8324afdff      call    ntdll!RtlDispatchException (00000000`772a0070)
00000000`772cb63e 84c0            test    al,al
00000000`772cb640 740c            je      ntdll!KiUserExceptionDispatcher+0x3e (00000000`772cb64e)
00000000`772cb642 488bcc          mov     rcx,rsp
00000000`772cb645 33d2            xor     edx,edx
00000000`772cb647 e8c4010000      call    ntdll!RtlRestoreContext (00000000`772cb810)
00000000`772cb64c eb15            jmp     ntdll!KiUserExceptionDispatcher+0x53 (00000000`772cb663)
00000000`772cb64e 488bcc          mov     rcx,rsp
00000000`772cb651 4881c1f0040000  add     rcx,4F0h
00000000`772cb658 488bd4          mov     rdx,rsp
00000000`772cb65b 4532c0          xor     r8b,r8b
00000000`772cb65e e85df5ffff      call    ntdll!NtRaiseException (00000000`772cabc0)
00000000`772cb663 8bc8            mov     ecx,eax
00000000`772cb665 e886d30500      call    ntdll!RtlRaiseStatus (00000000`773289f0)
```

紧接着就会调用`ntdll!RtlDispatchException`函数，这个函数完成异常分发，它的两个参数就是从X86栈上拷贝过来的数据，异常记录和环境结构体，如下所示：

```
0:000> dt ntdll!_EXCEPTION_RECORD   00000000003af7f0
   +0x000 ExceptionCode    : 0n-1073741819  // 0x00000000c0000005
   +0x004 ExceptionFlags   : 0
   +0x008 ExceptionRecord  : (null)
   +0x010 ExceptionAddress : 0x00000000`00f6a629 Void
   +0x018 NumberParameters : 2
   +0x020 ExceptionInformation : [15] 1

0:000> dt ntdll!_CONTEXT   00000000003af300
   +0x000 P1Home           : 0x7745304d`50000063
   +0x008 P2Home           : 0x77597390
   +0x010 P3Home           : 0x151ab0`00000000
   +0x018 P4Home           : 0x150000`40000062
   +0x020 P5Home           : 0x3af404
   +0x028 P6Home           : 0x1000000
   +0x030 ContextFlags     : 0x10005f
   +0x034 MxCsr            : 0x1f80
   +0x038 SegCs            : 0x23
   +0x03a SegDs            : 0x2b
   +0x03c SegEs            : 0x2b
   +0x03e SegFs            : 0x53
   +0x040 SegGs            : 0x2b
   +0x042 SegSs            : 0x2b
   +0x044 EFlags           : 0x10286
   +0x048 Dr0              : 0
   +0x050 Dr1              : 0
   +0x058 Dr2              : 0
   +0x060 Dr3              : 0
   +0x068 Dr6              : 0
   +0x070 Dr7              : 0
   +0x078 Rax              : 0
   +0x080 Rcx              : 0
   +0x088 Rdx              : 0x153058
   +0x090 Rbx              : 0x7efde000
   +0x098 Rsp              : 0x3afa38
   +0x0a0 Rbp              : 0x3afb2c
   +0x0a8 Rsi              : 0
   +0x0b0 Rdi              : 0x3afb14
   +0x0b8 R8               : 0x2b
   +0x0c0 R9               : 0x7743fb1a
   +0x0c8 R10              : 0
   +0x0d0 R11              : 0x246
   +0x0d8 R12              : 0x7efdb000
   +0x0e0 R13              : 0x23fd20
   +0x0e8 R14              : 0x23ea60
   +0x0f0 R15              : 0x748c2450
   +0x0f8 Rip              : 0xf6a629
   +0x100 FltSave          : _XSAVE_FORMAT
   +0x100 Header           : [2] _M128A
   +0x120 Legacy           : [8] _M128A
   +0x1a0 Xmm0             : _M128A
   +0x1b0 Xmm1             : _M128A
   +0x1c0 Xmm2             : _M128A
   +0x1d0 Xmm3             : _M128A
   +0x1e0 Xmm4             : _M128A
   +0x1f0 Xmm5             : _M128A
   +0x200 Xmm6             : _M128A
   +0x210 Xmm7             : _M128A
   +0x220 Xmm8             : _M128A
   +0x230 Xmm9             : _M128A
   +0x240 Xmm10            : _M128A
   +0x250 Xmm11            : _M128A
   +0x260 Xmm12            : _M128A
   +0x270 Xmm13            : _M128A
   +0x280 Xmm14            : _M128A
   +0x290 Xmm15            : _M128A
   +0x300 VectorRegister   : [26] _M128A
   +0x4a0 VectorControl    : 0x630150`00000000
   +0x4a8 DebugControl     : 0x77494dcd`00630150
   +0x4b0 LastBranchToRip  : 0
   +0x4b8 LastBranchFromRip : 0
   +0x4c0 LastExceptionToRip : 0
   +0x4c8 LastExceptionFromRip : 0
```

这里的异常分发其实是X64下的异常分发，那么它就是要按照X64的异常数据结构进行分发了，看一下当前线程中安装的异常链。

```
0:000> !exchain
8 stack frames, scanning for handlers...
Frame 0x02: wow64cpu!CpupReturnFromSimulatedCode (00000000`748c271e)
  ehandler wow64cpu!CpupSimulateHandler (00000000`748c2560)
Frame 0x03: wow64!RunCpuSimulation+0xa (00000000`74c6d286)
  ehandler wow64!_C_specific_handler (00000000`74c8e48e)
Frame 0x04: wow64!Wow64LdrpInitialize+0x42a (00000000`74c6c69e)
  ehandler wow64!_GSHandlerCheck (00000000`74c8e3d0)
Frame 0x05: ntdll!LdrpInitializeProcess+0x17e3 (00000000`77294223)
  ehandler ntdll!_GSHandlerCheck (00000000`772c0a54)
Frame 0x06: ntdll! ?? ::FNODOBFM::`string'+0x22a50 (00000000`772f9a60)
  ehandler ntdll!_C_specific_handler (00000000`772b730c)
```

其实对Ring3层异常分发起作用的是为`wow64!RunCpuSimulation`函数设置的异常处理。当依次调用异常链上的过滤函数都没有响应时，就会执行到这里的异常过滤函数，即`wow64!_C_specific_handler`，它直接调用X64位的`ntdll!_C_specific_handler`进行处理。

`ntdll!_C_specific_handler`函数的过程就不详细说明了，它会遍历`wow64!RunCpuSimulation`函数的ScopeTable(分层try)，依次调用它们的过滤函数，其实这个里面只有一层，它的过滤函数如下：

```
wow64!Wow64pLongJmp+0x652:
00000000`74c8ec52 4055            push    rbp
00000000`74c8ec54 4883ec20        sub     rsp,20h
00000000`74c8ec58 488bea          mov     rbp,rdx
00000000`74c8ec5b 48894d30        mov     qword ptr [rbp+30h],rcx
00000000`74c8ec5f 48894d28        mov     qword ptr [rbp+28h],rcx
00000000`74c8ec63 488b4d28        mov     rcx,qword ptr [rbp+28h]
00000000`74c8ec67 e88cddfdff      call    wow64!Pass64bitExceptionTo32Bit (00000000`74c6c9f8)
00000000`74c8ec6c c7452001000000  mov     dword ptr [rbp+20h],1
00000000`74c8ec73 8b4520          mov     eax,dword ptr [rbp+20h]
00000000`74c8ec76 4883c420        add     rsp,20h
00000000`74c8ec7a 5d              pop     rbp
00000000`74c8ec7b c3              ret
```

函数`wow64!Pass64bitExceptionTo32Bit`将X64的异常信息转换为X86的异常信息，其实就是`CONTEXT`和异常记录的构造。构造完成之后调用`wow64!Wow64SetupExceptionDispatch`函数，该函数中整理X86栈上的信息，并且设置跳回X86时要执行的地址，为全局变量`wow64!Ntdll32KiUserExceptionDispatcher`的值，它的值在初始化时设置为`ntdll32!KiUserExceptionDispatcher`，其实就是Ring3的异常分发起始函数。

```
00000000`74c6c85f 8b0d3be70200    mov     ecx,dword ptr [wow64!Ntdll32KiUserExceptionDispatcher (00000000`74c9afa0)]
00000000`74c6c865 ff15c556ffff    call    qword ptr [wow64!_imp_CpuSetInstructionPointer (00000000`74c61f30)]
```

接下来回到`ntdll!_C_specific_handler`函数时，返回值为`EXCEPTION_EXECUTE_HANDLER`，则需要进行栈展开。在展开过程中就会调用到`SCOPE_TABLE`中的`JumpTarget`字段所指偏移处函数，将该`SCOPE_TABLE`打印出来，看到偏移`0xd288`处为跳转指令，继续执行`call qword ptr [wow64!_imp_CpuSimulate]`。

```
0:000> lm m wow64
start             end                 module name
00000000`74c60000 00000000`74c9f000   wow64

0:000> dd wow64 + 00034164
00000000`74c94164  0000d280 0000d288 0002ec52 0000d288

0:000> u wow64!RunCpuSimulation
wow64!RunCpuSimulation:
00000000`74c6d27c 4883ec48        sub     rsp,48h
00000000`74c6d280 ff15524cffff    call    qword ptr [wow64!_imp_CpuSimulate (00000000`74c61ed8)]
00000000`74c6d286 eb00            jmp     wow64!RunCpuSimulation+0xc (00000000`74c6d288)
00000000`74c6d288 ebf6            jmp     wow64!RunCpuSimulation+0x4 (00000000`74c6d280) // Handler
00000000`74c6d28a 4883c448        add     rsp,48h
00000000`74c6d28e c3              ret
```

根据前面，已经将X86执行环境设置为异常处理的环境了，起点为`ntdll32!KiUserExceptionDispatcher`，一旦进入模拟状态就会进行X86下的异常分发。

###用户APC分发###

`WoW64`通过`ntdll!KiUserApcDispatcher`也勾住了用户模式APC的递交过程。无论何时当64位内核将要给一个`WoW64`进程分发一个用户模式APC时，Wow64把32位APC地址映射到一个更高的64位地址空间范围中。然后64位`ntdll.dll`捕获住原生的APC以及用户模式下的环境记录，将它映射到一个32位地址。然后为它准备一个32位用户模式APC和环境记录，并且按照原生32位内核所做的那样将它分发出去。

如下为插入用户层APC时的调用栈。

```
0:000> !wow64exts.k
Walking 64bit Stack...
Child-SP          RetAddr           Call Site
00000000`001bdb90 00000000`74c6d18f wow64!whNtQueueApcThread+0x2a
00000000`001bdbd0 00000000`748c2776 wow64!Wow64SystemServiceEx+0xd7
00000000`001be490 00000000`74c6d286 wow64cpu!TurboDispatchJumpAddressEnd+0x2d
00000000`001be550 00000000`74c6c69e wow64!RunCpuSimulation+0xa
00000000`001be5a0 00000000`77294223 wow64!Wow64LdrpInitialize+0x42a
00000000`001beaf0 00000000`772f9a60 ntdll!LdrpInitializeProcess+0x17e3
00000000`001befe0 00000000`772a374e ntdll! ?? ::FNODOBFM::`string'+0x22a50
00000000`001bf050 00000000`00000000 ntdll!LdrInitializeThunk+0xe

Walking 32bit Stack...
ChildEBP RetAddr
0040fe04 759f3ec6 ntdll32!NtQueueApcThread+0x12
0040fe2c 013f90ab KERNELBASE!QueueUserAPC+0x6b
0040ff0c 0146f506 Test!wmain+0x4b [c:\users\administrator\desktop\test\test\test.cpp @ 168]
0040ff58 0146f3df Test!__tmainCRTStartup+0x116 [f:\dd\vctools\crt_bld\self_x86\crt\src\crt0.c @ 266]
0040ff60 754e343d Test!wmainCRTStartup+0xf [f:\dd\vctools\crt_bld\self_x86\crt\src\crt0.c @ 182]
0040ff6c 77459832 kernel32!BaseThreadInitThunk+0xe
0040ffac 77459805 ntdll32!__RtlUserThreadStart+0x70
0040ffc4 00000000 ntdll32!_RtlUserThreadStart+0x1b
```

这里还是需要最终调用到X64的内核插入APC，分发回来同样也是从X64的`ntdll.dll`开始，即从`ntdll!KiUserApcDispatcher`开始分发，首先将分发函数（`KERNELBASE!RtlDispatchAPC`）指针进行解码，判断该指针地址是否超过`0x80000000`，WoW64的APC来说它的该函数在`0x80000000`地址之下，那么调用`wow64!Wow64ApcRoutine`函数。

函数`wow64!Wow64ApcRoutine`构建X86的运行时环境，并且设置X86下APC的分发函数，即用`wow64!Ntdll32KiUserApcDispatcher`全局变量包含值（`ntdll32!KiUserApcDispatcher`函数地址）。完成环境设置，进入X86模拟环境继续执行即可。

###控制台支持###

因为控制台是由`csrss.exe`在用户模式下实现的，它只是单个原生二进制可执行文件，所以32应用程序在64位Windows上执行不能执行控制台`I/O`。类似于专门有一个特殊的`rpcrt4.dll`用来将32位`RPC`适配成64位`RPC`，WoW64的32位`kernel32.dll`中有专门的代码来调用到Wow中，以便在与`Csrss.exe`和`Conhost.exe`交互过程中对参数进行适配。

以32位的`kernel32!WriteConsoleInternal`为例来看一下这个逻辑，如下为该函数的反汇编。

```
0:000:x86> u kernel32!WriteConsoleInternal
kernel32!WriteConsoleInternal:
754e12d5 b802200000      mov     eax,2002h
754e12da b900000000      mov     ecx,0
754e12df 8d542404        lea     edx,[esp+4]
754e12e3 64ff15c0000000  call    dword ptr fs:[0C0h]
754e12ea 83c404          add     esp,4
754e12ed c21400          ret     14h
```

注意这里`ecx`寄存器值为0，函数调用ID号为`0x2002h`，根据前面的四类函数映射可知它会使用`wow64!ServiceTables`表中的第三种映射。它对应的WoW64中函数为`wow64win!whWriteConsoleInternal`，这个函数接下来调用一系列函数，就如前面所述，整理参数，然后调用64位的RPC到`csrss.exe`或`conhost.exe`中。

另外一类与此类似即`Csr`类的函数，如下的例子中，它即使用`0x30003`的系统调用号，映射到`wow64!ServiceTables`表中的第四类函数表（索引为3）中。

```
0:000:x86> u kernel32!NtWow64CsrBasepCreateProcess
kernel32!NtWow64CsrBasepCreateProcess:
754e8de8 b803300000      mov     eax,3003h
754e8ded 33c9            xor     ecx,ecx
754e8def 8d542404        lea     edx,[esp+4]
754e8df3 64ff15c0000000  call    dword ptr fs:[0C0h]
754e8dfa 83c404          add     esp,4
754e8dfd c20400          ret     4
```

映射到WoW64中的函数为`wow64!whNtWow64CsrBasepCreateProcess`，它会将数据转换后调用X64对应的函数进行处理。

###用户回调###

从内核回调用户层的函数，这里主要是调用`user32.dll`中的回调表中函数。内核返回后进入的函数为`ntdll!KiUserCallbackDispatcher`，函数汇编如下所示：

```
0:000> uf ntdll!KiUserCallbackDispatcher
ntdll!KiUserCallbackDispatch:
00000000`772cb5d0 488b4c2420      mov     rcx,qword ptr [rsp+20h]
00000000`772cb5d5 8b542428        mov     edx,dword ptr [rsp+28h]
00000000`772cb5d9 448b44242c      mov     r8d,dword ptr [rsp+2Ch]
00000000`772cb5de 65488b042560000000 mov   rax,qword ptr gs:[60h]
00000000`772cb5e7 4c8b4858        mov     r9,qword ptr [rax+58h]
00000000`772cb5eb 43ff14c1        call    qword ptr [r9+r8*8]
00000000`772cb5ef 33c9            xor     ecx,ecx
00000000`772cb5f1 33d2            xor     edx,edx
00000000`772cb5f3 448bc0          mov     r8d,eax
00000000`772cb5f6 e8f5e2ffff      call    ntdll!NtCallbackReturn (00000000`772c98f0)
00000000`772cb5fb 8bf0            mov     esi,eax
00000000`772cb5fd 8bce            mov     ecx,esi
00000000`772cb5ff e8ecd30500      call    ntdll!RtlRaiseStatus (00000000`773289f0)
00000000`772cb604 ebf7            jmp     ntdll!KiUserCallbackDispatcherContinue+0xe (00000000`772cb5fd)
```

这里可以看到它取出`gs:[60h]`处的值，即PEB的地址，然后获取PEB的`0x58`偏移处的指针，如下，偏移处为`KernelCallbackTable`字段，即内核回调表。看一下内核回调表的内容，如下所示。

```
0:000> dt 7efdf000  ntdll!_PEB
   +0x000 InheritedAddressSpace : 0 ''
   ......
   +0x050 ReservedBits0    : 0y000000000000000000000000000 (0)
   +0x058 KernelCallbackTable : 0x00000000`748d1510 Void
   +0x058 UserSharedInfoPtr : 0x00000000`748d1510 Void
   +0x060 SystemReserved   : [1] 0
   ......

0:000> dqs 748d1510
00000000`748d1510  00000000`74902868 wow64win!whcbfnCOPYDATA
00000000`748d1518  00000000`749029fc wow64win!whcbfnCOPYGLOBALDATA
00000000`748d1520  00000000`74902b40 wow64win!whcbfnDWORD
00000000`748d1528  00000000`74902dc4 wow64win!whcbfnNCDESTROY
00000000`748d1530  00000000`74902f10 wow64win!whcbfnDWORDOPTINLPMSG
00000000`748d1538  00000000`74903080 wow64win!whcbfnINOUTDRAG
00000000`748d1540  00000000`7490325c wow64win!whcbfnGETTEXTLENGTHS
00000000`748d1548  00000000`749033cc wow64win!whcbfnINCNTOUTSTRING
00000000`748d1550  00000000`74903560 wow64win!whcbfnINCNTOUTSTRINGNULL
00000000`748d1558  00000000`749036ec wow64win!whcbfnINLPCOMPAREITEMSTRUCT
00000000`748d1560  00000000`74903894 wow64win!whcbfnINLPCREATESTRUCT
00000000`748d1568  00000000`74903a94 wow64win!whcbfnINLPDELETEITEMSTRUCT
00000000`748d1570  00000000`74903c04 wow64win!whcbfnINLPDRAWITEMSTRUCT
00000000`748d1578  00000000`74903d90 wow64win!whcbfnINLPHELPINFOSTRUCT
00000000`748d1580  00000000`74903f1c wow64win!whcbfnINLPHLPSTRUCT
00000000`748d1588  00000000`749040a8 wow64win!whcbfnINLPMDICREATESTRUCT
```

从表里可知WoW64将X64原始进程的回调表内容替换成了WoW64的内容，以`wow64win!whcbfnCOPYDATA`为例，在该函数中将WoW64的回调函数ID号映射为X86的，并且将回调数据进行整合，调用`wow64!Wow64KiUserCallbackDispatcher`函数。

函数`wow64!Wow64KiUserCallbackDispatcher`和前面的APC分发类似，设置回调X86时的执行地址为全局变量`Ntdll32KiUserCallbackDispatcher`(即`ntdll32!KiUserCallbackDispatcher`)，然后将执行切回X86模拟环境。

进入`ntdll32!KiUserCallbackDispatcher`函数后就是纯X86的内核回调分发了。

###注册表重定向###

应用程序和组件程序将它们的配置数据保留在注册表中，组件程序在安装的过程中，当它们被注册的时候，通常将配置数据写到注册表中。如果同样的组件即安装注册了一个32位二进制文件，又安装了一个64位二进制文件，那么，最后被注册的组件将会覆盖掉以前组件的注册，因为他们填写在相同的位置上。为了以透明的方式解决这个问题，并且无须对32位组件进行任何代码修饰，注册表被分成了两个部分：原生的和Wow64的。在默认情况下，32位组件访问32位视图，64位组件访问64位视图，这为32位和64位组件提供了一个安全的环境，并且将32位应用程序的状态与64位应用程序的状态隔开来。

为了实现这一点，Wow64截取了所有要打开注册表的系统调用，并且重新解释这些注册表键的路径，将它们指向注册表的64位视图。注册表的打开逻辑和其他的系统调用流程没有差别，只是在WoW64中在调用X64内核中之前对打开路径进行了修改，以`ntdll32!ZwOpenKeyEx`为例，它会调用到WoW64中的`wow64!whNtOpenKeyEx`函数，该函数进一步调用`wow64!Wow64NtOpenKey`，它会调用`wow64!ConstructKernelKeyPath`对打开的注册表路径进行修正。在X64上受到重定向影响的注册表路径有如下几个：

```
// 64位程序的注册信息存储键
HKLM/Software
HKEY_CLASSES_ROOT
HKEY_CURRENT_USER/Software/Classes
HKEY_LOCAL_MACHINE/Software
HKEY_USERS/*/Software/Classes
HKEY_USERS/*_Classes
//32位程序的注册信息重定向存储键
HKLM/Software/WOW6432node
HKEY_CLASSES_ROOT/WOW6432node
HKEY_CURRENT_USER/Software/Classes/WOW6432node
HKEY_LOCAL_MACHINE/Software/WOW6432node
HKEY_USERS/*/Software/Classes/WOW6432node
HKEY_USERS/*_Classes/WOW6432node
```

在打开注册表路径，获取注册表句柄时，参数中加入`KEY_WOW64_64KEY/KEY_WOW64_32KEY`分别用于32位程序访问64位程序注册表和64位程序访问32位程序注册表。

###文件系统重定向###

为了维护应用程序的兼容性，已经降低从Win32到64位Windows的应用程序移植代价，系统目录名称仍然保持不变。因此，`\Windows\System32`文件夹包含了原生的64位映像文件。因为Wow64勾住了所有的系统调用，所以它会解释所有与路径相关的API，将`\Windows\System32`文件夹的名称替换为`\Windows\Syswow64`。Wow64也将`\Windows\LastGood`重定向到`\Windows\LastGodd\Syswow64`，将`\Windows\Regedit.exe`重定向到`\Windows\syswow64\Regedit.exe`。通过使用系统环境变量，`%PROGRAMFILE%`环境变量对于32位程序被设置为`\Program File (x86)`，而对于64位应用程序被设置为`\Program File`文件夹，`CommonProgramFiles`和`CommonProgramFiles(x86)`也存在，它们总是指向32位的位置，而`ProgramW6432`和`CommonProgramWP6432`则无条件指向64位位置。

X86所有的系统调用和操作都已经被WoW64给截取了，所以只需要在涉及上述这些路径中进行路径修改即可。在WoW64中使用`wow64!RedirectObjectAttributes`函数将路径进行修改。在`wow64.dll`中包括`wow64!RedirectDosPathUnicode`，`wow64!Wow64ShallowThunkAllocObjectAttributes32TO64_FNC`和`wow64!RedirectObjectName`等函数都会调用到它。

如下两个函数可以用于打开和关闭文件重定向，它们内部会调用`ntdll32!RtlWow64EnableFsRedirectionEx`，最终用于操作X64的TEB内容。

```
kernel32!Wow64DisableWow64FsRedirection	// 关闭系统重定向
kernel32!Wow64RevertWow64FsRedirection  // 打开系统重定向
```

**参考文章**

1. [https://baike.baidu.com/item/WOW64/2155695](https://baike.baidu.com/item/WOW64/2155695)
2. [https://bbs.pediy.com/thread-221236.htm](https://bbs.pediy.com/thread-221236.htm)
3. [https://www.corsix.org/content/dll-injection-and-wow64](https://www.corsix.org/content/dll-injection-and-wow64)
4. [http://blog.rewolf.pl/blog/?p=102](http://blog.rewolf.pl/blog/?p=102)
5. [https://www.jianshu.com/p/acf43755a042](https://www.jianshu.com/p/acf43755a042)

By Andy @2018-11-09 20:20:38