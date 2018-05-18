---
title: Windows X86的ring3结构化异常处理
date: 2017-07-26 20:52:23
tags:
- Windbg
- Exception
categories:
- 调试
---

前面总结过Windows的异常分发机制，但是对于分发到指定异常处理器后的内容没有进一步总结。这个里面涉及到异常过滤和栈“展开”等内容，尤其栈“展开”一直不明白怎么回事。既然不想糊里糊涂，那就总结一下搞明白，"纸上得来终觉浅，绝知此事要躬行"。

这篇博文以seh程序为例来解析实际的应用程序中异常过滤和异常处理以及栈展开的内容。程序的代码如下代码块所示。代码非常简单，仅仅是main函数中调用了一个SehTest，在该函数中有三个\_\_try块，一个为独立的代码块，另外两个嵌套。

```
#include "stdafx.h"
#include <Windows.h>

int Filter_0(LPEXCEPTION_POINTERS p_exinfo)
{
	if(p_exinfo->ExceptionRecord->ExceptionCode == EXCEPTION_ACCESS_VIOLATION)
	{
		printf("存储保护异常\n");
		return 1;
	}
	else
		return 0;
}

int Filter_2(LPEXCEPTION_POINTERS p_exinfo)
{
	if(p_exinfo->ExceptionRecord->ExceptionCode == EXCEPTION_INT_DIVIDE_BY_ZERO)
	{
		printf("被0除异常\n");
		return 1;
	}
	else
		return 0;
}

VOID SehTest()
{
	ULONG ulVal = 0;

	__try // 第一个 __try 域
	{
		ulVal = 0x11111111; // 最后一位为1表示“在 __try 代码块中”
	}
	__except(Filter_0(GetExceptionInformation()))
	{
		ulVal = 0x11111110; // 最后一位为0表示“在 __except/__finally 代码块中”
	}

	__try // 第二个 __try 域
	{
		ulVal = 0x22222222;

		__try // 第三个 __try 域
		{
			ulVal = 0x33333333;

			*((ULONG*)NULL) = ulVal; // 触发异常
		}
		__finally
		{
			ulVal = 0x33333330;
		}
	}
	__except(Filter_2(GetExceptionInformation()))
	{
		ulVal = 0x22222220;
	}

	return;
}

int _tmain(int argc, _TCHAR* argv[])
{
	SehTest();
	return 0;
}
```

<!-- more -->

在前面的两篇博文中（一篇Windows异常分发，一篇Windows异常处理），已经看到了RtlDispatchException负责异常分发，遇到适当的异常处理函数，则调用该处理函数。此处调用的处理器是需要程序实际执行过程中注册到TEB起始的NtTib.ExceptionList字段中的，而这个注册过程就是`__try`等关键字编译后的内容。下面就从这个SEH的`__try`编译后的代码开始来总结。

由于SehTest函数比较简单，如果用IDA F5之后，就剩下一个赋值操作了，关于异常处理的操作都不可见了。所以，这里将SehTest()函数的汇编代码放到这里。

```
.text:0042C640 ; Attributes: bp-based frame
.text:0042C640
.text:0042C640 ; void __cdecl SehTest()
.text:0042C640 ?SehTest@@YAXXZ proc near
.text:0042C640
.text:0042C640 var_E4          = byte ptr -0E4h
.text:0042C640 ulVal           = dword ptr -20h
.text:0042C640 ms_exc          = CPPEH_RECORD ptr -18h
.text:0042C640
.text:0042C640                 push    ebp			   ; EBP, 前一个push ebp
.text:0042C641                 mov     ebp, esp
.text:0042C643                 push    0FFFFFFFEh      ; trylevel = TRYLEVEL_INVALID
.text:0042C645                 push    offset stru_48FD98 ;  Scopetable
.text:0042C64A                 push    offset __VCrtDbgReportW_SEH ; handler 其实就是_except_handler4
.text:0042C64F                 mov     eax, large fs:0 ; prev 指针
.text:0042C655                 push    eax
.text:0042C656                 add     esp, 0FFFFFF2Ch ; //// 栈扩充，非try代码（////之间为非try代码）
.text:0042C65C                 push    ebx
.text:0042C65D                 push    esi
.text:0042C65E                 push    edi
.text:0042C65F                 lea     edi, [ebp+var_E4]
.text:0042C665                 mov     ecx, 33h
.text:0042C66A                 mov     eax, 0CCCCCCCCh
.text:0042C66F                 rep stosd               ; ////
.text:0042C671                 mov     eax, ___security_cookie
.text:0042C676                 xor     [ebp+ms_exc.registration.ScopeTable], eax ; ebp-8 对ScopeTable进行异或加密
.text:0042C679                 xor     eax, ebp        ; GS Cookie 它也不是try代码，是程序的Cookie校验
.text:0042C67B                 push    eax
.text:0042C67C                 lea     eax, [ebp+ms_exc.registration] ; 取栈上registration 结构体指针
.text:0042C67F                 mov     large fs:0, eax ; 将TEB中的异常列表指针替换掉
.text:0042C685                 mov     [ebp+ms_exc.old_esp], esp ; 保存 esp，这个在展开时会用到
.text:0042C688                 mov     [ebp+ulVal], 0
.text:0042C68F                 mov     [ebp+ms_exc.registration.TryLevel], 0 ; 设置TryLevel为0，即scopetable第一项负责当异常
.text:0042C696                 mov     [ebp+ulVal], 11111111h
.text:0042C69D                 mov     [ebp+ms_exc.registration.TryLevel], 0FFFFFFFEh ; 将trylevel恢复，表明没有异常处理负责异常了
.text:0042C6A4                 jmp     short loc_42C6C4
.text:0042C6A6 ; ---------------------------------------------------------------------------
.text:0042C6A6
.text:0042C6A6 $LN7:                                   ; DATA XREF: .rdata:stru_48FD98o
.text:0042C6A6                 mov     eax, [ebp+ms_exc.exc_ptr] ; Exception filter 0 for function 42C640
.text:0042C6A9                 push    eax             ; p_exinfo
.text:0042C6AA                 call    j_?Filter_0@@YAHPAU_EXCEPTION_POINTERS@@@Z ; Filter_0(_EXCEPTION_POINTERS *)
.text:0042C6AF                 add     esp, 4
.text:0042C6B2
.text:0042C6B2 $LN9:
.text:0042C6B2                 retn
.text:0042C6B3 ; ---------------------------------------------------------------------------
.text:0042C6B3
.text:0042C6B3 $LN8:                                   ; DATA XREF: .rdata:stru_48FD98o
.text:0042C6B3                 mov     esp, [ebp+ms_exc.old_esp] ; Exception handler 0 for function 42C640
.text:0042C6B6                 mov     [ebp+ulVal], 11111110h
.text:0042C6BD                 mov     [ebp+ms_exc.registration.TryLevel], 0FFFFFFFEh
.text:0042C6C4
.text:0042C6C4 loc_42C6C4:                             ; CODE XREF: SehTest(void)+64j
.text:0042C6C4                 mov     [ebp+ms_exc.registration.TryLevel], 1 ; 修改Traylevel为1，scopetable表第二项负责当前块中异常
.text:0042C6CB                 mov     [ebp+ulVal], 22222222h
.text:0042C6D2                 mov     [ebp+ms_exc.registration.TryLevel], 2 ; 含义同 .text:0042C6C4
.text:0042C6D9                 mov     [ebp+ulVal], 33333333h
.text:0042C6E0                 mov     eax, [ebp+ulVal]
.text:0042C6E3                 mov     large ds:0, eax
.text:0042C6E8                 mov     [ebp+ms_exc.registration.TryLevel], 1 ; 退出trylevel为2范围，进入到trylevel 1范围
.text:0042C6EF                 call    $LN15           ; Finally handler 2 for function 42C640
.text:0042C6F4 ; ---------------------------------------------------------------------------
.text:0042C6F4
.text:0042C6F4 loc_42C6F4:                             ; CODE XREF: SehTest(void):$LN16j
.text:0042C6F4                 jmp     short $LN18
.text:0042C6F6 ; ---------------------------------------------------------------------------
.text:0042C6F6
.text:0042C6F6 $LN15:                                  ; CODE XREF: SehTest(void)+AFj
.text:0042C6F6                                         ; DATA XREF: .rdata:stru_48FD98o
.text:0042C6F6                 mov     [ebp+ulVal], 33333330h ; Finally handler 2 for function 42C640
.text:0042C6FD
.text:0042C6FD $LN16:
.text:0042C6FD                 retn
.text:0042C6FE ; ---------------------------------------------------------------------------
.text:0042C6FE
.text:0042C6FE $LN18:                                  ; CODE XREF: SehTest(void):loc_42C6F4j
.text:0042C6FE                 mov     [ebp+ms_exc.registration.TryLevel], 0FFFFFFFEh ; 将trylevel设置为TRYLEVEL_INVALID
.text:0042C705                 jmp     short loc_42C725
.text:0042C707 ; ---------------------------------------------------------------------------
.text:0042C707
.text:0042C707 $LN11:                                  ; DATA XREF: .rdata:stru_48FD98o
.text:0042C707                 mov     eax, [ebp+ms_exc.exc_ptr] ; Exception filter 1 for function 42C640
.text:0042C70A                 push    eax             ; p_exinfo
.text:0042C70B                 call    j_?Filter_2@@YAHPAU_EXCEPTION_POINTERS@@@Z ; Filter_2(_EXCEPTION_POINTERS *)
.text:0042C710                 add     esp, 4
.text:0042C713
.text:0042C713 $LN13:
.text:0042C713                 retn
.text:0042C714 ; ---------------------------------------------------------------------------
.text:0042C714
.text:0042C714 $LN12:                                  ; DATA XREF: .rdata:stru_48FD98o
.text:0042C714                 mov     esp, [ebp+ms_exc.old_esp] ; Exception handler 1 for function 42C640
.text:0042C717                 mov     [ebp+ulVal], 22222220h
.text:0042C71E                 mov     [ebp+ms_exc.registration.TryLevel], 0FFFFFFFEh
.text:0042C725
.text:0042C725 loc_42C725:                             ; CODE XREF: SehTest(void)+C5j	; 退出函数准备
.text:0042C725                 push    edx
.text:0042C726                 mov     ecx, ebp        ; frame
.text:0042C728                 push    eax
.text:0042C729                 lea     edx, v          ; v
.text:0042C72F                 call    j_@_RTC_CheckStackVars@8 ; _RTC_CheckStackVars(x,x)
.text:0042C734                 pop     eax
.text:0042C735                 pop     edx
.text:0042C736                 mov     ecx, [ebp+ms_exc.registration.Next]				; 撤销掉注册的异常块
.text:0042C739                 mov     large fs:0, ecx									; 重新赋值 TEB中异常链表头
.text:0042C740                 pop     ecx
.text:0042C741                 pop     edi
.text:0042C742                 pop     esi
.text:0042C743                 pop     ebx
.text:0042C744                 add     esp, 0E4h
.text:0042C74A                 cmp     ebp, esp
.text:0042C74C                 call    j___RTC_CheckEsp
.text:0042C751                 mov     esp, ebp
.text:0042C753                 pop     ebp
.text:0042C754                 retn
.text:0042C754 ; ---------------------------------------------------------------------------
.text:0042C755                 align 4
.text:0042C758 ; _RTC_framedesc v
.text:0042C758 v               _RTC_framedesc <1, offset dword_42C760>
.text:0042C758                                         ; DATA XREF: SehTest(void)+E9o
.text:0042C760 dword_42C760    dd 0FFFFFFE0h, 4        ; DATA XREF: SehTest(void):vo
.text:0042C768                 dd offset aUlval        ; "ulVal"
.text:0042C76C aUlval          db 'ulVal',0            ; DATA XREF: SehTest(void)+128o
.text:0042C76C ?SehTest@@YAXXZ endp
.text:0042C76C
.text:0042C772                 db 4Eh dup(0CCh)
.text:0042C7C0
```

对于使用了`__try`/`__except`的函数来说，它会注册异常处理信息。看如下的汇编代码中，从代码开始到`.text:0042C685`这一行构造了异常处理信息，即`EXCEPTION_REGISTRATION_RECORD`结构体（结构体的定义如下代码块所示）。从代码量上看绝不止这两个字段，其实在VS中编译时，结构体被扩展了（如下`_EXCEPTION_REGISTRATION`结构体所示）。其中prev和handler字段即`_EXCEPTION_REGISTRATION_RECORD`定义中的Next和Handler字段。

```
typedef struct _EXCEPTION_REGISTRATION_RECORD {
   struct _EXCEPTION_REGISTRATION_RECORD *Next;
   PEXCEPTION_ROUTINE Handler;
} EXCEPTION_REGISTRATION_RECORD;
```

typedef struct _EXCEPTION_REGISTRATION PEXCEPTION_REGISTRATION;
struct _EXCEPTION_REGISTRATION{
   DWORD old_esp;
   PEXCEPTION_POINTERS xpointers;
   struct _EXCEPTION_REGISTRATION *prev;
   void (*handler)(PEXCEPTION_RECORD, PEXCEPTION_REGISTRATION, PCONTEXT, PEXCEPTION_RECORD);
   struct scopetable_entry *scopetable;
   int trylevel;
   int _ebp;
};

从SehTest()函数中构造`_EXCEPTION_REGISTRATION`结构体可知，Scopetable指向的是一个全局变量，如下的代码块，其中注释中标识出来了Scopetable表的三项内容。从中可以看到FilterFunc和HandlerFunc就是SehTest函数中代码的部分，分别对应着exept中括号中的代码，以及except后面大括号内的内容，即对应的过滤表达式（函数）和处理代码。可以注意到第三个表项的FilterFunc字段为0，其实这是一个`__try/__finally`语句块，它显然是没有过滤表达式的，所以这个字段被设置为空了。下面将SehTest函数中的三个try对应的scopetable及其内部个字段对应函数实时调试内容列举了，并做了注释。这里不再过多解释。第二个内容是handler字段，这个字段指向的内容`__VCrtDbgReportW_SEH`，而这个代码地方直接jmp到了`_except_handler4`函数。`_except_handler4`这个函数后面会解释，这里先不做解析。

```
.rdata:0048FD98 stru_48FD98     dd 0FFFFFFFEh           ; GSCookieOffset
.rdata:0048FD98                 dd 0                    ; GSCookieXOROffset ; SEH scope table for function 42C640
.rdata:0048FD98                 dd 0FFFFFF0Ch           ; EHCookieOffset
.rdata:0048FD98                 dd 0                    ; EHCookieXOROffset

.rdata:0048FD98                 dd 0FFFFFFFEh           ; ScopeRecord.EnclosingLevel
.rdata:0048FD98                 dd offset $LN7          ; ScopeRecord.FilterFunc
.rdata:0048FD98                 dd offset $LN8          ; ScopeRecord.HandlerFunc

.rdata:0048FD98                 dd 0FFFFFFFEh           ; ScopeRecord.EnclosingLevel
.rdata:0048FD98                 dd offset $LN11         ; ScopeRecord.FilterFunc
.rdata:0048FD98                 dd offset $LN12         ; ScopeRecord.HandlerFunc

.rdata:0048FD98                 dd 1                    ; ScopeRecord.EnclosingLevel
.rdata:0048FD98                 dd 0                    ; ScopeRecord.FilterFunc
.rdata:0048FD98                 dd offset $LN15         ; ScopeRecord.HandlerFunc

.rdata:0048FDCC                 dd 0
.rdata:0048FDD0                 dd 0
.rdata:0048FDD4                 dd 0

//// 实际调试中Scopetable表的内容
0:000> dds 0138fd98
0138fd98  fffffffe
0138fd9c  00000000
0138fda0  ffffff0c
0138fda4  00000000	// 做校验用的

0138fda8  fffffffe
0138fdac  0132c6a6 seh!SehTest+0x66 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 38]
0138fdb0  0132c6b3 seh!SehTest+0x73 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 38]

0138fdb4  fffffffe
0138fdb8  0132c707 seh!SehTest+0xc7 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 58]
0138fdbc  0132c714 seh!SehTest+0xd4 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 58]

0138fdc0  00000001
0138fdc4  00000000
0138fdc8  0132c6f6 seh!SehTest+0xb6 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 55]

0138fdcc  00000000
0138fdd0  00000000
0138fdd4  00000000
```

```
0:000> u 0132c6a6
seh!SehTest+0x66 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 38]:
0132c6a6 8b45ec          mov     eax,dword ptr [ebp-14h]			; 注册数据结构的成员 xpointer
0132c6a9 50              push    eax
0132c6aa e8fbe8ffff      call    seh!ILT+4005(?Filter_0YAHPAU_EXCEPTION_POINTERSZ) (0132afaa)	; Filter_0
0132c6af 83c404          add     esp,4
0132c6b2 c3              ret

0:000> u 0132c6b3
seh!SehTest+0x73 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 38]:
0132c6b3 8b65e8          mov     esp,dword ptr [ebp-18h]			; 栈展开中删除之前的调用栈，开始处理（old_esp字段）
0132c6b6 c745e010111111  mov     dword ptr [ebp-20h],11111110h
0132c6bd c745fcfeffffff  mov     dword ptr [ebp-4],0FFFFFFFEh

0132c6c4 c745fc01000000  mov     dword ptr [ebp-4],1				; 第二个try块的代码
0132c6cb c745e022222222  mov     dword ptr [ebp-20h],22222222h
0132c6d2 c745fc02000000  mov     dword ptr [ebp-4],2
0132c6d9 c745e033333333  mov     dword ptr [ebp-20h],33333333h
0132c6e0 8b45e0          mov     eax,dword ptr [ebp-20h]

0:000> u 0132c707
seh!SehTest+0xc7 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 58]:
0132c707 8b45ec          mov     eax,dword ptr [ebp-14h]			; 注册数据结构的成员 xpointer
0132c70a 50              push    eax
0132c70b e8fed9ffff      call    seh!ILT+265(?Filter_2YAHPAU_EXCEPTION_POINTERSZ) (0132a10e)	; Filter_2
0132c710 83c404          add     esp,4
0132c713 c3              ret

0:000> u 0132c714
seh!SehTest+0xd4 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 58]:
0132c714 8b65e8          mov     esp,dword ptr [ebp-18h]			; 栈展开中删除之前的调用栈，开始处理
0132c717 c745e020222222  mov     dword ptr [ebp-20h],22222220h
0132c71e c745fcfeffffff  mov     dword ptr [ebp-4],0FFFFFFFEh

0132c725 52              push    edx								; 校验栈变量，不属于try代码
0132c726 8bcd            mov     ecx,ebp
0132c728 50              push    eax
0132c729 8d1558c73201    lea     edx,[seh!SehTest+0x118 (0132c758)]
0132c72f e84edeffff      call    seh!ILT+1405(_RTC_CheckStackVars (0132a582)

0:000> u 0132c6f6
seh!SehTest+0xb6 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 55]:
0132c6f6 c745e030333333  mov     dword ptr [ebp-20h],33333330h		; finaly处理块，按照函数代码，该赋值结束，即退出函数
0132c6fd c3              ret

0132c6fe c745fcfeffffff  mov     dword ptr [ebp-4],0FFFFFFFEh
0132c705 eb1e            jmp     seh!SehTest+0xe5 (0132c725)
0132c707 8b45ec          mov     eax,dword ptr [ebp-14h]
0132c70a 50              push    eax
0132c70b e8fed9ffff      call    seh!ILT+265(?Filter_2YAHPAU_EXCEPTION_POINTERSZ) (0132a10e)
```

再向下执行，就到了第一个try的范围内，如下代码立决了执行到将ulVal设置为0x11111111值的地方，将当时的代码做反汇编，即可发现前一句汇编为将ebp-4位置设置为了0，这个位置其实就是栈上构造的`_EXCEPTION_REGISTRATION`结构体中的tryLevel成员，如下查看`_EXCEPTION_REGISTRATION`结构体在内存中的内容即可发现值已经被设置为0了。将该成员设置为0的意思是如果在执行过程中遇到了异常，遍历处理器时，用0作为索引值找ScopeTable中的表项。这里索引为0的表项即`.rdata:0048FD98`处（IDA的反汇编内容）的表项，即代码的第一个try语句。

```
0:000> r
eax=0038f654 ebx=7efde000 ecx=00000000 edx=00192de8 esi=00000000 edi=0038f64c
eip=0132c696 esp=0038f570 ebp=0038f664 iopl=0         nv up ei ng nz na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000286
seh!SehTest+0x56:
0132c696 c745e011111111  mov     dword ptr [ebp-20h],11111111h ss:002b:0038f644=00000000

0:000> u eip - 7
seh!SehTest+0x4f [c:\users\administrator\desktop\seh\seh\seh.cpp @ 34]:
0132c68f c745fc00000000  mov     dword ptr [ebp-4],0
0132c696 c745e011111111  mov     dword ptr [ebp-20h],11111111h
0132c69d c745fcfeffffff  mov     dword ptr [ebp-4],0FFFFFFFEh
0132c6a4 eb1e            jmp     seh!SehTest+0x84 (0132c6c4)
0132c6a6 8b45ec          mov     eax,dword ptr [ebp-14h]
0132c6a9 50              push    eax
0132c6aa e8fbe8ffff      call    seh!ILT+4005(?Filter_0YAHPAU_EXCEPTION_POINTERSZ) (0132afaa)
0132c6af 83c404          add     esp,4

0:000> dds ebp - 30
0038f634  cccccccc
0038f638  cccccccc
0038f63c  cccccccc
0038f640  cccccccc
0038f644  00000000
0038f648  cccccccc
0038f64c  0038f570
0038f650  023d9cf2
0038f654  0038f774
0038f658  0132a528 seh!ILT+1315(__except_handler4)
0038f65c  acd90a7d
0038f660  00000000
0038f664  0038f738
0038f668  0132c7e3 seh!wmain+0x23 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 70]
0038f66c  00000000
0038f670  00000000
0038f674  7efde000
0038f678  cccccccc
0038f67c  cccccccc
```

将上述代码接着执行，执行到`0132c6c4 c745fc01000000  mov     dword ptr [ebp-4],1`处，即第二个try语句的地方，可知tryLevel被设置为了0xFFFFFFFE，即TRYLEVEL_INVALID值，表明当前没有异常处理器负责当前块中的异常。再向下的try语句的代码类似，会依次修改tryLevel的值，以确定异常处理的表项索引。不再一一解释。

```
0:000> p
eax=0038f654 ebx=7efde000 ecx=00000000 edx=00192de8 esi=00000000 edi=0038f64c
eip=0132c6c4 esp=0038f570 ebp=0038f664 iopl=0         nv up ei ng nz na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000286
seh!SehTest+0x84:
0132c6c4 c745fc01000000  mov     dword ptr [ebp-4],1  ss:002b:0038f660=fffffffe
0:000> r
eax=0038f654 ebx=7efde000 ecx=00000000 edx=00192de8 esi=00000000 edi=0038f64c
eip=0132c6c4 esp=0038f570 ebp=0038f664 iopl=0         nv up ei ng nz na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000286
seh!SehTest+0x84:
0132c6c4 c745fc01000000  mov     dword ptr [ebp-4],1  ss:002b:0038f660=fffffffe
0:000> dds ebp - 30
0038f634  cccccccc
0038f638  cccccccc
0038f63c  cccccccc
0038f640  cccccccc
0038f644  11111111
0038f648  cccccccc
0038f64c  0038f570
0038f650  023d9cf2
0038f654  0038f774
0038f658  0132a528 seh!ILT+1315(__except_handler4)
0038f65c  acd90a7d	//
0038f660  fffffffe	// trylevel
0038f664  0038f738
0038f668  0132c7e3 seh!wmain+0x23 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 70]
0038f66c  00000000
0038f670  00000000
0038f674  7efde000
0038f678  cccccccc
0038f67c  cccccccc
0038f680  cccccccc
0038f684  cccccccc
```

当代码执行到如下的位置时（即`ulVal = 0`语句），可以看一下调用栈，如下代码所示。从汇编代码可以看到这一条汇编语句是向0x00000000的内存位置赋值，即向空指针赋值，这肯定会引起异常的。从前面的注册的异常块可知，当进行异常链表遍历时，肯定会遍历到我们注册的这个异常块，调用对应的过滤代码，以判断是否可以处理当前的异常。前面分析知道，异常结构中的handler字段指向的是`seh!_except_handler4`（或者说最终会跳到该函数）。这里可以在这个函数上设置一个断点，看是否真的会在异常分发中走到该函数。

```
0:000> k
ChildEBP RetAddr
0038f664 0132c7e3 seh!SehTest+0xa3 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 51]
0038f738 0132d166 seh!wmain+0x23 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 70]
0038f784 0132d03f seh!__tmainCRTStartup+0x116 [f:\dd\vctools\crt_bld\self_x86\crt\src\crt0.c @ 266]
0038f78c 74e0336a seh!wmainCRTStartup+0xf [f:\dd\vctools\crt_bld\self_x86\crt\src\crt0.c @ 182]
0038f798 774e9902 kernel32!BaseThreadInitThunk+0xe
0038f7d8 774e98d5 ntdll!__RtlUserThreadStart+0x70
0038f7f0 00000000 ntdll!_RtlUserThreadStart+0x1b

0:000> r
eax=33333333 ebx=7efde000 ecx=00000000 edx=00192de8 esi=00000000 edi=0038f64c
eip=0132c6e3 esp=0038f570 ebp=0038f664 iopl=0         nv up ei ng nz na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000286
seh!SehTest+0xa3:
0132c6e3 a300000000      mov     dword ptr ds:[00000000h],eax ds:002b:00000000=????????

0:000> u 0132a528
seh!ILT+1315(__except_handler4):
0132a528 e953280000      jmp     seh!_except_handler4 (0132cd80)
seh!ILT+1320(?_ValidateReadYAHPBXIZ):
0132a52d e9ae300400      jmp     seh!_ValidateRead (0136d5e0)

0:000> bl
 0 e 0132c7c0     0001 (0001)  0:**** seh!wmain
 1 e 0132a528     0001 (0001)  0:**** seh!ILT+1315(__except_handler4)
```

单步执行以下，即可发现调试器断下来了，给出了Access violation的诊断信息，即我们赋值空指针造成的。这在异常处理一篇中可知异常首先会被分发给调试器（如果调试器存在）。如果让调试器不处理，单纯给出继续执行的指令，异常就会继续分发。“继续执行”命令执行后，可以发现再次断到调试器，这次是断点造成的程序断到调试器，断点1即我们上面给`__except_handler4`函数设置的断点。查看一下但钱的堆栈，`ntdll!KiUserExceptionDispatcher`函数即ring3层将异常分发给异常处理函数的起点（遍历异常链表的起点）。

```
0:000> p
(1e08.1848): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
eax=33333333 ebx=7efde000 ecx=00000000 edx=00192de8 esi=00000000 edi=0038f64c
eip=0132c6e3 esp=0038f570 ebp=0038f664 iopl=0         nv up ei ng nz na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00010286
seh!SehTest+0xa3:
0132c6e3 a300000000      mov     dword ptr ds:[00000000h],eax ds:002b:00000000=????????
0:000> g
Breakpoint 1 hit
eax=00000000 ebx=00000000 ecx=0132a528 edx=7751353d esi=00000000 edi=00000000
eip=0132a528 esp=0038efd4 ebp=0038eff4 iopl=0         nv up ei pl zr na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
seh!ILT+1315(__except_handler4):
0132a528 e953280000      jmp     seh!_except_handler4 (0132cd80)
0:000> kv
ChildEBP RetAddr  Args to Child
0038eff4 775134fb 0038f0bc 0038f654 0038f10c seh!ILT+1315(__except_handler4)
0038f018 7751349c 0038f0bc 0038f654 0038f10c ntdll!ExecuteHandler+0x24 (FPO: [5,0,3])
0038f0a4 774c0143 0038f0bc 0038f10c 0038f0bc ntdll!RtlDispatchException+0x127 (FPO: [2,25,4])
0038f0a4 00000000 0038f0bc 0038f10c 0038f0bc ntdll!KiUserExceptionDispatcher+0xf (FPO: [2,0,0]) (CONTEXT @ 00000008)
```

根据`int RtlDispatchException(PEXCEPTION_RECORD pExcptRec, CONTEXT * pContext)`函数原型可知，第一个参数和第二个参数分别是异常记录结构体和上下文环境结构体。在windbg中看一下这两个结构体内容，如下代码块。从两个结构体可以非常清楚看到异常发生时的情况，与我们上面执行时的代码一致。

```
0:000> .exr 0038f0bc
ExceptionAddress: 0132c6e3 (seh!SehTest+0x000000a3)
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000000
NumberParameters: 2
   Parameter[0]: 00000001
   Parameter[1]: 00000000
Attempt to write to address 00000000

0:000> .cxr 0038f10c
eax=33333333 ebx=7efde000 ecx=00000000 edx=00192de8 esi=00000000 edi=0038f64c
eip=0132c6e3 esp=0038f570 ebp=0038f664 iopl=0         nv up ei ng nz na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00010386
seh!SehTest+0xa3:
0132c6e3 a300000000      mov     dword ptr ds:[00000000h],eax ds:002b:00000000=????????
0:000> k
  *** Stack trace for last set context - .thread/.cxr resets it
ChildEBP RetAddr
0038f664 0132c7e3 seh!SehTest+0xa3 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 51]
0038f738 0132d166 seh!wmain+0x23 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 70]
0038f784 0132d03f seh!__tmainCRTStartup+0x116 [f:\dd\vctools\crt_bld\self_x86\crt\src\crt0.c @ 266]
0038f78c 74e0336a seh!wmainCRTStartup+0xf [f:\dd\vctools\crt_bld\self_x86\crt\src\crt0.c @ 182]
0038f798 774e9902 kernel32!BaseThreadInitThunk+0xe
0038f7d8 774e98d5 ntdll!__RtlUserThreadStart+0x70
0038f7f0 00000000 ntdll!_RtlUserThreadStart+0x1b

0:000> u seh!SehTest+0xa3
seh!SehTest+0xa3 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 51]:
0132c6e3 a300000000      mov     dword ptr ds:[00000000h],eax
0132c6e8 c745fc01000000  mov     dword ptr [ebp-4],1
0132c6ef e802000000      call    seh!SehTest+0xb6 (0132c6f6)
0132c6f4 eb08            jmp     seh!SehTest+0xbe (0132c6fe)
0132c6f6 c745e030333333  mov     dword ptr [ebp-20h],33333330h
0132c6fd c3              ret
```

从断点处，即`_except_handler4`函数的调用起始，看一下该函数。它有四个参数，从栈中看一下三个参数，前三个参数不用说分别是异常信息记录结构体，注册到异常链表的异常处理结构体指针，上下文内容结构体。最后一个是DispatcherContext，没有找到比较正式的资料，仅仅从网上搜索到一些相关资料，结合实际调试的内容分析一下，也不知道这个结构体的实际大小，暂时写这么多，今后知道了更多内容，加以补充。

```
0:000> dds esp
0038efd4  77513529 ntdll!ExecuteHandler2+0x26
0038efd8  0038f0bc
0038efdc  0038f654
0038efe0  0038f10c
0038efe4  0038f090	// _DISPATCHER_CONTEXT *DispatcherContext
0038efe8  0038f654
```

```
_EXCEPTION_DISPOSITION __cdecl _except_handler4(_EXCEPTION_RECORD *ExceptionRecord, _EXCEPTION_REGISTRATION_RECORD *EstablisherFrame, _CONTEXT *ContextRecord, void *DispatcherContext)

_DISPATCHER_CONTEXT *DispatcherContext

typedef union _FRAME_POINTERS {
    struct {
        ULONG MemoryStackFp;
        ULONG BackingStoreFp;
    };
    ULONGLONG FramePointers;           // used to force 8-byte alignment
} FRAME_POINTERS, *PFRAME_POINTERS;

typedef struct _xDISPATCHER_CONTEXT {
    FRAME_POINTERS EstablisherFrame;
    ULONG ControlPc;
    struct _RUNTIME_FUNCTION *FunctionEntry;
    PCONTEXT ContextRecord;
} DispatcherContext, *pDispatcherContext;

0:000> dds 0038f090
0038f090  00000000
0038f094  0000004d
0038f098  00390000		// 栈底
0038f09c  0038d000		// 当前栈 交付的内存块的“最高”位置，即压栈极限，再压就缺页中断了
0038f0a0  00000000
0038f0a4  0038f664		// 上一个 EXCEPTION_REGISTRATION_RECORD 结构体指针
0038f0a8  774c0143 ntdll!KiUserExceptionDispatcher+0xf
0038f0ac  0038f0bc		// _EXCEPTION_RECORD *ExceptionRecord
0038f0b0  0038f10c		// _CONTEXT *ContextRecord
0038f0b4  0038f0bc		// _EXCEPTION_RECORD *ExceptionRecord
0038f0b8  0038f10c		// _CONTEXT *ContextRecord
0038f0bc  c0000005		// 错误码
0038f0c0  00000000
0038f0c4  00000000
0038f0c8  0132c6e3 seh!SehTest+0xa3 [c:\users\administrator\desktop\seh\seh\seh.cpp @ 51] // 返回地址
0038f0cc  00000002		// TryLevel
0038f0d0  00000001
0038f0d4  00000000
0038f0d8  00000001
0038f0dc  00000000
0038f0e0  00000000
0038f0e4  00000000
0038f0e8  00000000
0038f0ec  00000000
0038f0f0  00000000
0038f0f4  00000000
0038f0f8  00000000
0038f0fc  00000000
0038f100  00000000
0038f104  00000000
0038f108  00000000
0038f10c  0001007f
```

```
typedef struct _EXCEPTION_RECORD {
   DWORD ExceptionCode;
   DWORD ExceptionFlags;
   struct _EXCEPTION_RECORD *ExceptionRecord;
   PVOID ExceptionAddress;
   DWORD NumberParameters;
   ULONG_PTR ExceptionInformation[EXCEPTION_MAXIMUM_PARAMETERS];
} EXCEPTION_RECORD;
```

对于`_except_handler4`函数也没有官方正式的源代码，这里以IDA反汇编处来的内容为准做一个简单分析。

```
.text:0042CD80 ; Attributes: bp-based frame
.text:0042CD80
.text:0042CD80 ; _EXCEPTION_DISPOSITION __cdecl _except_handler4(_EXCEPTION_RECORD *ExceptionRecord, _EXCEPTION_REGISTRATION_RECORD *EstablisherFrame, _CONTEXT *ContextRecord, void *DispatcherContext)
.text:0042CD80 __except_handler4 proc near             ; CODE XREF: ?SehTest@@YAXXZ_SEHj
.text:0042CD80
.text:0042CD80 FilterFunc      = dword ptr -30h
.text:0042CD80 ScopeTable      = dword ptr -2Ch
.text:0042CD80 TryLevel        = dword ptr -28h
.text:0042CD80 Revalidate      = byte ptr -21h
.text:0042CD80 RegistrationNode= dword ptr -20h
.text:0042CD80 ScopeTableRecord= dword ptr -1Ch
.text:0042CD80 FramePointer    = dword ptr -18h
.text:0042CD80 EnclosingLevel  = dword ptr -14h
.text:0042CD80 FilterResult    = dword ptr -10h
.text:0042CD80 Disposition     = dword ptr -0Ch
.text:0042CD80 ExceptionPointers= _EXCEPTION_POINTERS ptr -8
.text:0042CD80 ExceptionRecord = dword ptr  8
.text:0042CD80 EstablisherFrame= dword ptr  0Ch
.text:0042CD80 ContextRecord   = dword ptr  10h
.text:0042CD80 DispatcherContext= dword ptr  14h
.text:0042CD80
.text:0042CD80                 mov     edi, edi
.text:0042CD82                 push    ebp
.text:0042CD83                 mov     ebp, esp
.text:0042CD85                 sub     esp, 30h        ; prolog 函数前序
.text:0042CD88                 mov     [ebp+Revalidate], 0
.text:0042CD8C                 mov     [ebp+Disposition], 1
.text:0042CD93                 mov     eax, [ebp+EstablisherFrame] ; 本函数第二个参数 EstablisherFrame
.text:0042CD96                 sub     eax, 8
.text:0042CD99                 mov     [ebp+RegistrationNode], eax ; 扩展的_EXCEPTION_REGISTRATION结构体指针 赋值局部变量
.text:0042CD9C                 mov     ecx, [ebp+RegistrationNode] ; RegistrationNode,即注册的异常节点，从old_esp算起
.text:0042CD9F                 add     ecx, 18h					   ; 指向_EXCEPTION_REGISTRATION结构体指针 _ebp陈冠
.text:0042CDA2                 mov     [ebp+FramePointer], ecx 	   ; 赋值局部变量FramePointer _EXCEPTION_REGISTRATION
.text:0042CDA5                 mov     edx, [ebp+RegistrationNode]
.text:0042CDA8                 mov     eax, [edx+10h]  			   ; ScopeTable 指针
.text:0042CDAB                 xor     eax, ___security_cookie 	   ; 解密，与Cookie做异或
.text:0042CDB1                 mov     [ebp+ScopeTable], eax	   ; ScopeTable局部变量赋值
.text:0042CDB4                 mov     ecx, [ebp+FramePointer]	   ;
.text:0042CDB7                 push    ecx             ; FramePointer
.text:0042CDB8                 mov     edx, [ebp+ScopeTable]
.text:0042CDBB                 push    edx             ; ScopeTable
.text:0042CDBC                 call    ValidateLocalCookies        ; Cookie 校验
.text:0042CDC1                 add     esp, 8
.text:0042CDC4                 mov     eax, [ebp+ExceptionRecord]
.text:0042CDC7                 mov     ecx, [eax+4]    ; ExceptionFlags
.text:0042CDCA                 and     ecx, 66h        ; pExceptionRecord->ExceptionFlags & EXCEPTION_UNWIND，判断是异常处理过程还是展开过程
.text:0042CDCD                 jnz     loc_42CEF0	   ; 跳去做展开操作
.text:0042CDD3                 mov     edx, [ebp+ExceptionRecord]		// 本地构建_EXCEPTION_POINTERS，异常记录与上下文指针
.text:0042CDD6                 mov     [ebp+ExceptionPointers.ExceptionRecord], edx
.text:0042CDD9                 mov     eax, [ebp+ContextRecord]
.text:0042CDDC                 mov     [ebp+ExceptionPointers.ContextRecord], eax
.text:0042CDDF                 mov     ecx, [ebp+RegistrationNode]
.text:0042CDE2                 lea     edx, [ebp+ExceptionPointers]
.text:0042CDE5                 mov     [ecx+4], edx    ; 注册到异常列表异常块的PEXCEPTION_POINTERS类型指针赋值
.text:0042CDE8                 mov     eax, [ebp+RegistrationNode]
.text:0042CDEB                 mov     ecx, [eax+14h]  ; tryLevel 指针
.text:0042CDEE                 mov     [ebp+TryLevel], ecx	; 赋值 TryLevel局部变量
.text:0042CDF1                 jmp     short loc_42CDF9
.text:0042CDF3 ; ---------------------------------------------------------------------------
.text:0042CDF3
.text:0042CDF3 loc_42CDF3:                             ; CODE XREF: __except_handler4:loc_42CEE9j
.text:0042CDF3                 mov     edx, [ebp+EnclosingLevel] ; EXCEPTION_CONTINUE_SEARCH（0）
.text:0042CDF6                 mov     [ebp+TryLevel], edx
.text:0042CDF9
.text:0042CDF9 loc_42CDF9:                             ; CODE XREF: __except_handler4+71j
.text:0042CDF9                 cmp     [ebp+TryLevel], 0FFFFFFFEh ; TRYLEVEL_INVALID
.text:0042CDFD                 jz      loc_42CEEE				  ; TryLevel无效，没有在异常处理范围内
.text:0042CE03                 mov     eax, [ebp+TryLevel]
.text:0042CE06                 imul    eax, 0Ch        ; scopetable一个Item是12个字节，level，filter，handler
.text:0042CE09                 mov     ecx, [ebp+ScopeTable]
.text:0042CE0C                 lea     edx, [ecx+eax+10h] ; 这里加10h 是略过其实16个校验字节
.text:0042CE10                 mov     [ebp+ScopeTableRecord], edx ; ScopeTableRecord 直接指向了对应的表项
.text:0042CE13                 mov     eax, [ebp+ScopeTableRecord]
.text:0042CE16                 mov     ecx, [eax+4]    ; Filter函数
.text:0042CE19                 mov     [ebp+FilterFunc], ecx
.text:0042CE1C                 mov     edx, [ebp+ScopeTableRecord]
.text:0042CE1F                 mov     eax, [edx]      ; LastTryLevel
.text:0042CE21                 mov     [ebp+EnclosingLevel], eax	; EnclosingLevel是外层try在Scope表中的索引，用于循环
.text:0042CE24                 cmp     [ebp+FilterFunc], 0			; 过滤函数为空，则继续循环外层
.text:0042CE28                 jz      loc_42CEE9
.text:0042CE2E                 mov     edx, [ebp+FramePointer]		; 如果过滤函数非空，则调用
.text:0042CE31                 mov     ecx, [ebp+FilterFunc]
.text:0042CE34                 call    j_@_EH4_CallFilterFunc@8 ; 调用过滤函数
.text:0042CE39                 mov     [ebp+FilterResult], eax ; 返回结果
.text:0042CE3C                 mov     [ebp+Revalidate], 1
.text:0042CE40                 cmp     [ebp+FilterResult], 0 ; 结果是否为 EXCEPTION_CONTINUE_SEARCH（0）
.text:0042CE44                 jge     short loc_42CE57 ; 大于等于0，即0或1，EXCEPTION_EXECUTE_HANDLER (1)
.text:0042CE46                 mov     [ebp+Disposition], 0
.text:0042CE4D                 jmp     loc_42CEEE      ; EXCEPTION_CONTINUE_EXECUTION (-1)
.text:0042CE52 ; ---------------------------------------------------------------------------
.text:0042CE52                 jmp     loc_42CEE9
.text:0042CE57 ; ---------------------------------------------------------------------------
.text:0042CE57
.text:0042CE57 loc_42CE57:                             ; CODE XREF: __except_handler4+C4j
.text:0042CE57                 cmp     [ebp+FilterResult], 0
.text:0042CE5B                 jle     loc_42CEE9      ; EXCEPTION_CONTINUE_SEARCH（0）返回值等于0，继续搜索
.text:0042CE61                 mov     ecx, [ebp+ExceptionRecord] ; EXCEPTION_EXECUTE_HANDLER (1) 异常可以被处理
.text:0042CE64                 cmp     dword ptr [ecx], 0E06D7363h
.text:0042CE6A                 jnz     short loc_42CE95
.text:0042CE6C                 cmp     ds:__pDestructExceptionObject, 0	; C++中要将抛出异常的对象析构掉。
.text:0042CE73                 jz      short loc_42CE95
.text:0042CE75                 push    offset __pDestructExceptionObject ; pTarget
.text:0042CE7A                 call    j___IsNonwritableInCurrentImage
.text:0042CE7F                 add     esp, 4
.text:0042CE82                 test    eax, eax
.text:0042CE84                 jz      short loc_42CE95
.text:0042CE86                 push    1               ; fThrowNotAllowed
.text:0042CE88                 mov     edx, [ebp+ExceptionRecord]
.text:0042CE8B                 push    edx             ; pExcept
.text:0042CE8C                 call    ds:__pDestructExceptionObject ; 调用析构的函数
.text:0042CE92                 add     esp, 8
.text:0042CE95
.text:0042CE95 loc_42CE95:                             ; CODE XREF: __except_handler4+EAj
.text:0042CE95                                         ; __except_handler4+F3j ...
.text:0042CE95                 mov     ecx, [ebp+RegistrationNode]
.text:0042CE98                 add     ecx, 8          ; TargetFrame
.text:0042CE9B                 call    j_@_EH4_GlobalUnwind@4 ; _EH4_GlobalUnwind(x) ; 全局展开
.text:0042CEA0                 mov     eax, [ebp+RegistrationNode]
.text:0042CEA3                 mov     ecx, [eax+14h]
.text:0042CEA6                 cmp     ecx, [ebp+TryLevel]
.text:0042CEA9                 jz      short loc_42CEC2
.text:0042CEAB                 push    offset ___security_cookie
.text:0042CEB0                 mov     edx, [ebp+FramePointer]
.text:0042CEB3                 push    edx
.text:0042CEB4                 mov     ecx, [ebp+RegistrationNode]
.text:0042CEB7                 add     ecx, 8
.text:0042CEBA                 mov     edx, [ebp+TryLevel]
.text:0042CEBD                 call    j_@_EH4_LocalUnwind@16 ; _EH4_LocalUnwind(x,x,x,x) ; 局部展开
.text:0042CEC2
.text:0042CEC2 loc_42CEC2:                             ; CODE XREF: __except_handler4+129j
.text:0042CEC2                 mov     eax, [ebp+RegistrationNode]
.text:0042CEC5                 mov     ecx, [ebp+EnclosingLevel]
.text:0042CEC8                 mov     [eax+14h], ecx
.text:0042CECB                 mov     edx, [ebp+FramePointer]
.text:0042CECE                 push    edx             ; FramePointer
.text:0042CECF                 mov     eax, [ebp+ScopeTable]
.text:0042CED2                 push    eax             ; ScopeTable
.text:0042CED3                 call    ValidateLocalCookies					; 再一次校验Cookie，防止被利用
.text:0042CED8                 add     esp, 8
.text:0042CEDB                 mov     edx, [ebp+FramePointer]
.text:0042CEDE                 mov     ecx, [ebp+ScopeTableRecord]
.text:0042CEE1                 mov     ecx, [ecx+8]
.text:0042CEE4                 call    j_@_EH4_TransferToHandler@8 ; _EH4_TransferToHandler(x,x) ; 调用异常处理函数
.text:0042CEE9
.text:0042CEE9 loc_42CEE9:                             ; CODE XREF: __except_handler4+A8j
.text:0042CEE9                                         ; __except_handler4+D2j ...
.text:0042CEE9                 jmp     loc_42CDF3      ; EXCEPTION_CONTINUE_SEARCH（0）
.text:0042CEEE ; ---------------------------------------------------------------------------
.text:0042CEEE
.text:0042CEEE loc_42CEEE:                             ; CODE XREF: __except_handler4+7Dj
.text:0042CEEE                                         ; __except_handler4+CDj
.text:0042CEEE                 jmp     short loc_42CF16
.text:0042CEF0 ; ---------------------------------------------------------------------------
.text:0042CEF0
.text:0042CEF0 loc_42CEF0:                             ; CODE XREF: __except_handler4+4Dj ; 对于展开调用，直接调用局部展开
.text:0042CEF0                 mov     edx, [ebp+RegistrationNode]
.text:0042CEF3                 cmp     dword ptr [edx+14h], 0FFFFFFFEh
.text:0042CEF7                 jz      short loc_42CF16
.text:0042CEF9                 push    offset ___security_cookie
.text:0042CEFE                 mov     eax, [ebp+FramePointer]
.text:0042CF01                 push    eax
.text:0042CF02                 mov     ecx, [ebp+RegistrationNode]
.text:0042CF05                 add     ecx, 8
.text:0042CF08                 mov     edx, 0FFFFFFFEh
.text:0042CF0D                 call    j_@_EH4_LocalUnwind@16 ; _EH4_LocalUnwind(x,x,x,x)
.text:0042CF12                 mov     [ebp+Revalidate], 1
.text:0042CF16
.text:0042CF16 loc_42CF16:                             ; CODE XREF: __except_handler4:loc_42CEEEj
.text:0042CF16                                         ; __except_handler4+177j
.text:0042CF16                 movzx   ecx, [ebp+Revalidate]
.text:0042CF1A                 test    ecx, ecx
.text:0042CF1C                 jz      short loc_42CF2E
.text:0042CF1E                 mov     edx, [ebp+FramePointer]
.text:0042CF21                 push    edx             ; FramePointer
.text:0042CF22                 mov     eax, [ebp+ScopeTable]
.text:0042CF25                 push    eax             ; ScopeTable
.text:0042CF26                 call    ValidateLocalCookies
.text:0042CF2B                 add     esp, 8
.text:0042CF2E
.text:0042CF2E loc_42CF2E:                             ; CODE XREF: __except_handler4+19Cj
.text:0042CF2E                 mov     eax, [ebp+Disposition]
.text:0042CF31                 mov     esp, ebp
.text:0042CF33                 pop     ebp
.text:0042CF34                 retn
.text:0042CF34 __except_handler4 endp
.text:0042CF34
```

偶然从网上找到了一份`_except_handler4`的代码，放到这里，和上面的汇编做一个对比吧。对于不想看汇编的朋友，可以直接参考这份源码，与汇编基本相同。

```
/***
*ValidateLocalCookies - perform local cookie validation during SEH processing
*
*Purpose:
*   Perform the security checks for _except_handler4. 执行_except_handler4调用中的安全校验
*
*Entry:
*   CookieCheckFunction - (CRT DLL only) pointer to __security_check_cookie in
*       the target image
*   ScopeTable - pointer to the unencoded scope table from the exception
*       registration record
*   FramePointer - EBP frame pointer for the function that established the
*       exception registration record
*
*Return:
*   If the security checks fail, the process is terminated via a Watson dump.
*   如果安全校验失败，进程会被终止
*******************************************************************************/

static __declspec(guard(ignore)) void
ValidateLocalCookies(
#if defined(CRTDLL)
    IN PCOOKIE_CHECK    CookieCheckFunction,
#endif
    IN PEH4_SCOPETABLE  ScopeTable,
    IN PCHAR            FramePointer
    )
{
    UINT_PTR        GSCookie;
    UINT_PTR        EHCookie;

    if (ScopeTable->GSCookieOffset != NO_GS_COOKIE)
    {
        GSCookie = *(PUINT_PTR)(FramePointer + ScopeTable->GSCookieOffset);
        GSCookie ^= (UINT_PTR)(FramePointer + ScopeTable->GSCookieXOROffset);
#if defined(CRTDLL)
        _GUARD_CHECK_ICALL(CookieCheckFunction);
        (*CookieCheckFunction)(GSCookie);
#else
        __security_check_cookie(GSCookie);
#endif
    }

    EHCookie = *(PUINT_PTR)(FramePointer + ScopeTable->EHCookieOffset);
    EHCookie ^= (UINT_PTR)(FramePointer + ScopeTable->EHCookieXOROffset);
#if defined(CRTDLL)
    _GUARD_CHECK_ICALL(CookieCheckFunction);
    (*CookieCheckFunction)(EHCookie);
#else
    __security_check_cookie(EHCookie);
#endif
}

/***
*_except_handler4 - (non-CRT DLL only) SEH handler with security cookie checks
*
*_except_handler4_common - (CRT DLL only) actual SEH implementation called by
*                          _except_handler4 stub
*
*Purpose:
*   Implement structured exception handling for functions which have __try/
*   __except/__finally.  This version of SEH also performs security cookie
*   checks to detect buffer overruns which potentially corrupt the on-stack
*   exception handling data, terminating the process before such corruption
*   can be exploited.
*
*   Call exception and termination handlers as necessary, based on the current
*   execution point within the function.
*
*Entry:
*   CookiePointer - (CRT DLL only) pointer to the global security cookie to be
*       used to decode the scope table pointer in the exception registration
*       record
*   CookieCheckFunction - (CRT DLL only) pointer to __security_check_cookie in
*       the calling image
*   ExceptionRecord - pointer to the exception being dispatched
*   EstablisherFrame - pointer to the on-stack exception registration record
*       for this function
*   ContextRecord - pointer to a context record for the point of exception
*   DispatcherContext - pointer to the exception dispatcher or unwind
*       dispatcher context
*
*Return:
*   If the security checks fail, the process is terminated via a Watson dump.
*
*   If an exception is being dispatched and the exception is handled by an
*   __except filter for this exception frame, then this function does not
*   return.  Instead, it calls RtlUnwind and transfers control to the __except
*   block corresponding to the accepting __except filter.  Otherwise, an
*   exception disposition of continue execution or continue search is returned.
*
*   If an unwind is being dispatched, then each termination handler (__finally)
*   is called and a value of continue search is returned.
*
*******************************************************************************/

EXCEPTION_DISPOSITION
#if !defined(CRTDLL)
_except_handler4(
#else
_except_handler4_common(
    IN PUINT_PTR                        CookiePointer,
    IN PCOOKIE_CHECK                    CookieCheckFunction,
#endif
    IN PEXCEPTION_RECORD                ExceptionRecord,
    IN PEXCEPTION_REGISTRATION_RECORD   EstablisherFrame,
    IN OUT PCONTEXT                     ContextRecord,
    IN OUT PVOID                        DispatcherContext
    )
{
    PEH4_EXCEPTION_REGISTRATION_RECORD  RegistrationNode;
    PCHAR                               FramePointer;
    PEH4_SCOPETABLE                     ScopeTable;
    ULONG                               TryLevel;
    ULONG                               EnclosingLevel;
    EXCEPTION_POINTERS                  ExceptionPointers;
    PEH4_SCOPETABLE_RECORD              ScopeTableRecord;
    PEXCEPTION_FILTER                   FilterFunc;
    LONG                                FilterResult;
    BOOLEAN                             Revalidate = FALSE;
    EXCEPTION_DISPOSITION               Disposition = ExceptionContinueSearch;
    //
    // We are passed a registration record which is a field offset from the
    // start of our true registration record.
    //

    RegistrationNode =
        (PEH4_EXCEPTION_REGISTRATION_RECORD)
        ( (PCHAR)EstablisherFrame -
          FIELD_OFFSET(EH4_EXCEPTION_REGISTRATION_RECORD, SubRecord) );

    //
    // The EBP frame pointer in the function corresponding to the registration
    // record will be immediately following the record.  If the function uses
    // FPO, this is a "virtual" frame pointer outside of exception handling,
    // but it's still the EBP value set when calling into the handlers or
    // filters.
    // 在注册的异常块信息之后就是EBP帧指针，对于省略栈帧情况，在异常处理之外，这里是一个虚拟的栈帧值

    FramePointer = (PCHAR)(RegistrationNode + 1);

    //
    // Retrieve the scope table pointer, which encodes where we find the local
    // security cookies within the function's frame, as well as how the guarded
    // blocks in the target function are laid out.  This pointer was XORed with
    // the image-local global security cookie when originally stored, to avoid
    // attacks which spoof the table to address valid local cookies elsewhere
    // on the stack.
    //

#if defined(CRTDLL)
    ScopeTable = (PEH4_SCOPETABLE)
                    (RegistrationNode->EncodedScopeTable ^ *CookiePointer);
#else
    ScopeTable = (PEH4_SCOPETABLE)
                    (RegistrationNode->EncodedScopeTable ^ __security_cookie);
#endif

    //
    // Perform the initial security cookie validation.
    //

    ValidateLocalCookies(
#if defined(CRTDLL)
        CookieCheckFunction,
#endif
        ScopeTable,
        FramePointer
        );

    __except_validate_context_record(ContextRecord);

    //
    // Security checks have passed, begin actual exception handling.
    //

    if (IS_DISPATCHING(ExceptionRecord->ExceptionFlags))
    {
        //
        // An exception dispatch is in progress.  First build the
        // EXCEPTION_POINTERS record queried by the _exception_info intrinsic
        // and save it in the exception registration record so the __except
        // filter can find it.
        //

        ExceptionPointers.ExceptionRecord = ExceptionRecord;
        ExceptionPointers.ContextRecord = ContextRecord;
        RegistrationNode->ExceptionPointers = &ExceptionPointers;

        //
        // Scan the scope table and call the appropriate __except filters until
        // we find one that accepts the exception.
        //

        for (TryLevel = RegistrationNode->TryLevel;
             TryLevel != TOPMOST_TRY_LEVEL;
             TryLevel = EnclosingLevel)
        {
            ScopeTableRecord = &ScopeTable->ScopeRecord[TryLevel];
            FilterFunc = ScopeTableRecord->FilterFunc;
            EnclosingLevel = ScopeTableRecord->EnclosingLevel;

            if (FilterFunc != NULL)
            {
                //
                // The current scope table record is for an __except.
                // Call the __except filter to see if we've found an
                // accepting handler.
                //

                FilterResult = _EH4_CallFilterFunc(FilterFunc, FramePointer);
                Revalidate = TRUE;

                //
                // If the __except filter returned a negative result, then
                // dismiss the exception.  If it returned a positive result,
                // unwind to the accepting exception handler.  Otherwise keep
                // searching for an exception filter.
                //

                if (FilterResult < 0) 	// -1 继续执行，再次对异常代码执行一次
                {
                    Disposition = ExceptionContinueExecution;
                    break;
                }
                else if (FilterResult > 0)	// 执行异常处理器，这时要进行展开动作
                {
#if !defined(_NTSUBSET_)
                    //
                    // If we're handling a thrown C++ exception, let the C++
                    // exception handler destruct the thrown object.  This call
                    // is through a function pointer to avoid linking to the
                    // C++ EH support unless it's already present.  Don't call
                    // the function pointer unless it's in read-only memory.
                    //

                    if (ExceptionRecord->ExceptionCode == EH_EXCEPTION_NUMBER &&
                        _pDestructExceptionObject != NULL &&
                        _IsNonwritableInCurrentImage((PBYTE)&_pDestructExceptionObject))
                    {
                        (*_pDestructExceptionObject)(ExceptionRecord, TRUE);
                    }
#endif  // !defined(_NTSUBSET_)



                    //
                    // Unwind all registration nodes below this one, then unwind
                    // the nested __try levels.
                    //
                    _EH4_GlobalUnwind2(
                        &RegistrationNode->SubRecord,
                        ExceptionRecord
                        );

                    if (RegistrationNode->TryLevel != TryLevel)
                    {
                        _EH4_LocalUnwind(
                            &RegistrationNode->SubRecord,
                            TryLevel,
                            FramePointer,
#if defined(CRTDLL)
                            CookiePointer
#else
                            &__security_cookie
#endif
                            );
                    }

                    //
                    // Set the __try level to the enclosing level, since it is
                    // the enclosing level, if any, that guards the __except
                    // handler.
                    //

                    RegistrationNode->TryLevel = EnclosingLevel;

                    //
                    // Redo the security checks, in case any __except filters
                    // or __finally handlers have caused overruns in the target
                    // frame.
                    //

                    ValidateLocalCookies(
#if defined(CRTDLL)
                        CookieCheckFunction,
#endif
                        ScopeTable,
                        FramePointer
                        );

                    //
                    // Call the __except handler.  This call will not return.
                    // The __except handler will reload ESP from the
                    // registration record upon entry.  The EBP frame pointer
                    // for the handler is directly after the registration node.
                    //

                    _EH4_TransferToHandler(
                        ScopeTableRecord->u.HandlerAddress,
                        FramePointer
                        );
                }
            }
        }
    }
    else
    {
        //
        // An exception unwind is in progress, and this isn't the target of the
        // unwind.  Unwind any active __try levels in this function, calling
        // the applicable __finally handlers.
        //

        if (RegistrationNode->TryLevel != TOPMOST_TRY_LEVEL)
        {
            _EH4_LocalUnwind(
                &RegistrationNode->SubRecord,
                TOPMOST_TRY_LEVEL,
                FramePointer,
#if defined(CRTDLL)
                CookiePointer
#else
                &__security_cookie
#endif
                );
            Revalidate = TRUE;
        }
    }

    //
    // If we called any __except filters or __finally handlers, then redo the
    // security checks, in case those funclets have caused overruns in the
    // target frame.
    //

    if (Revalidate)
    {
        ValidateLocalCookies(
#if defined(CRTDLL)
            CookieCheckFunction,
#endif
            ScopeTable,
            FramePointer
            );
    }

    //
    // Continue searching for exception or termination handlers in previous
    // registration records higher up the stack, or resume execution if we're
    // here because an __except filter returned a negative result.
    //

    return Disposition;
}
```

在`_except_handler4`函数中有两个函数`_EH4_CallFilterFunc`和`_EH4_TransferToHandler`，他们都是跳转到对应的函数去，代码比较简单，列举出IDA的反汇编代码如下。

```
.text:004312D2 ; __fastcall _EH4_CallFilterFunc(x, x)
.text:004312D2 @_EH4_CallFilterFunc@8 proc near        ; CODE XREF: _EH4_CallFilterFunc(x,x)j
.text:004312D2                 push    ebp
.text:004312D3                 push    esi
.text:004312D4                 push    edi
.text:004312D5                 push    ebx
.text:004312D6                 mov     ebp, edx
.text:004312D8                 xor     eax, eax
.text:004312DA                 xor     ebx, ebx
.text:004312DC                 xor     edx, edx
.text:004312DE                 xor     esi, esi
.text:004312E0                 xor     edi, edi
.text:004312E2                 call    ecx
.text:004312E4                 pop     ebx
.text:004312E5                 pop     edi
.text:004312E6                 pop     esi
.text:004312E7                 pop     ebp
.text:004312E8                 retn
.text:004312E8 @_EH4_CallFilterFunc@8 endp

.text:004312E9 ; __fastcall _EH4_TransferToHandler(x, x)
.text:004312E9 @_EH4_TransferToHandler@8 proc near     ; CODE XREF: _EH4_TransferToHandler(x,x)j
.text:004312E9                 mov     ebp, edx
.text:004312EB                 mov     esi, ecx
.text:004312ED                 mov     eax, ecx
.text:004312EF                 push    1
.text:004312F1                 call    j___NLG_Notify
.text:004312F6                 xor     eax, eax
.text:004312F8                 xor     ebx, ebx
.text:004312FA                 xor     ecx, ecx
.text:004312FC                 xor     edx, edx
.text:004312FE                 xor     edi, edi
.text:00431300                 jmp     esi
.text:00431300 @_EH4_TransferToHandler@8 endp
```

由上述的`_except_handler4`代码可知，异常处理时有三种情况，`EXCEPTION_CONTINUE_SEARCH`，`EXCEPTION_CONTINUE_EXECUTION`，`EXCEPTION_EXECUTE_HANDLER`。`EXCEPTION_CONTINUE_SEARCH`比较好理解，继续搜索异常处理函数；`EXCEPTION_CONTINUE_EXECUTION`是退出异常处理，继续执行异常代码，从代码中可以看到，赋值局部变量后，直接break掉了搜索循环；最后一个`EXCEPTION_EXECUTE_HANDLER`是要执行异常处理函数，从上面的分析中可以知道异常处理函数并不一定处于异常发生的地方，可能在调用栈很高的位置，这个时候要执行它就需要将栈帧回退到它所在函数。对于正常的执行过程中，执行了处理函数，是接着要执行后面的代码的，那异常处理函数（代码段）之后的函数栈帧怎么办呢？就是要进行展开了，一方面退掉他们所占用的栈，另一方面释放掉他们申请的资源。

```
void __stdcall RtlUnwind(PVOID TargetFrame, PVOID TargetIp, PEXCEPTION_RECORD ExceptionRecord, PVOID ReturnValue)
{
  int v4; // ebp@0
  PEXCEPTION_RECORD v5; // esi@1
  unsigned int v6; // eax@5
  unsigned int v7; // ebx@5
  void *v8; // eax@12
  int v9; // eax@15
  unsigned int v10; // eax@16
  unsigned int v11; // [sp+8h] [bp-37Ch]@2
  int v12; // [sp+Ch] [bp-378h]@2
  int v13; // [sp+10h] [bp-374h]@2
  int v14; // [sp+14h] [bp-370h]@2
  int v15; // [sp+18h] [bp-36Ch]@2
  EXCEPTION_RECORD v16; // [sp+58h] [bp-32Ch]@20
  unsigned int v17; // [sp+A8h] [bp-2DCh]@15
  unsigned int LowLimit; // [sp+ACh] [bp-2D8h]@1
  unsigned int HighLimit; // [sp+B0h] [bp-2D4h]@1
  CONTEXT Context; // [sp+B4h] [bp-2D0h]@5
  int retaddr; // [sp+388h] [bp+4h]@2

  v5 = ExceptionRecord;
  RtlpGetStackLimits(&LowLimit, &HighLimit);    // 获取当前堆栈的最大最小值，防止后面获取值超出范围
  if ( !ExceptionRecord )						// 构建EXCEPTION_RECORD结构体
  {
    v5 = (PEXCEPTION_RECORD)&v11;
    v11 = 0xC0000027;                           // STATUS_UNWIND
    v12 = 0;
    v13 = 0;
    v14 = retaddr;
    v15 = 0;
  }
  if ( TargetFrame )
    v5->ExceptionFlags |= 2u;                   // EXCEPTION_UNWINDING
  else
    v5->ExceptionFlags |= 6u;					// EXCEPTION_UNWINDING | EXCEPTION_EXIT_UNWIND
  Context.ContextFlags = 0x10007;
  RtlpCaptureContext(v4, (int)&Context);        // 填充Context结构体
  Context.Esp += 16;
  Context.Eax = (unsigned int)ReturnValue;
  RtlpGetRegistrationHead();                    // 获取线程异常链头指针
  v7 = v6;
  while ( v7 != 0xFFFFFFFF )                    // 开始遍历异常链，直到目标Frame
  {
    if ( (PVOID)v7 == TargetFrame )             // 到了目标帧，直接返回
    {
      ZwContinue(&Context, 0);
    }
    else if ( TargetFrame && (unsigned int)TargetFrame < v7 )// 当前帧大于目标帧地址，这是出现了错误的。
    {
      v16.NumberParameters = 0;
      v16.ExceptionCode = 0xC0000029;           // STATUS_INVALID_UNWIND_TARGET 展开目标无效，抛出异常
      v16.ExceptionFlags = 1;
      v16.ExceptionRecord = v5;
      RtlRaiseException(&v16);
    }
    if ( v7 < LowLimit
      || v7 + 8 > HighLimit
      || v7 & 3
      || (v8 = *(void **)(v7 + 4), (unsigned int)v8 >= LowLimit) && (unsigned int)v8 < HighLimit
      || !RtlIsValidHandler(v8, 0) )            // 当前帧异常，超出栈限制，且其中的处理函数无效，则抛出异常
    {
      v16.NumberParameters = 0;
      v16.ExceptionCode = 0xC0000028;           // STATUS_BAD_STACK
      v16.ExceptionFlags = 1;
      v16.ExceptionRecord = v5;
      RtlRaiseException(&v16);
    }
    v9 = RtlpExecuteHandlerForUnwind(v5, v7, &Context, &v17, *(_DWORD *)(v7 + 4)) - 1;// 调用异常处理函数（except块）
    if ( v9 )
    {
      if ( v9 != 2 )
      {
        v16.NumberParameters = 0;
        v16.ExceptionCode = 0xC0000026;         // STATUS_INVALID_DISPOSITION
        v16.ExceptionFlags = 1;
        v16.ExceptionRecord = v5;
        RtlRaiseException(&v16);
      }
      v7 = v17;
    }
    v10 = v7;
    v7 = *(_DWORD *)v7;                         // 继续循环
    RtlpUnlinkHandler((int *)v10);
  }
  if ( TargetFrame == (PVOID)-1 )
    ZwContinue(&Context, 0);
  else
    ZwRaiseException(v5, &Context, 0);
}
```

从上述反汇编的代码中可以看到，展开过程最终会调用`RtlpExecuteHandlerForUnwind`函数，它其实就是一个用它自己最后一个参数作为函数指针调用该函数指针指向函数的包装。从v7指向异常信息注册块可知，函数指针指向的是`_except_handler4`函数，只是传参不同。代码中的`ExceptionRecord->ExceptionFlags`标记中带有`EXCEPTION_UNWINDING`，表明是展开操作，在`_except_handler4`中就会走到不同的路径，即局部展开中。这里就解释了一个问题，为什么`_except_handler4`在异常处理时为什么会被两次调用，一次用于异常处理函数搜索，一次用于栈展开。

这里要说一下全局展开（`_EH4_GlobalUnwind`）和局部展开（`_EH4_LocalUnwind`）的区别，比较容易混淆两个概念。全局展开即遍历TEB中异常链表中当前遍历中命中的异常信息记录节点之前的所有节点（遍历中要对每一个节点执行局部展开）。而局部展开是针对某一个具体的异常信息记录节点，即对应一个函数的异常处理信息。它内部可能包含多个try语句，并且try可能进行嵌套。局部展开的工作就是执行该节点中每一个try语句的处理函数（代码块）。

```
int __usercall _local_unwind4@<eax>(int FramePointer@<ebp>, _DWORD *CookiePointer, int EstablisherFrame, unsigned int TargetLevel)
{
  int result; // eax@2
  unsigned int v5; // esi@2
  int v6; // esi@5
  int v7; // ebx@5
  int v8; // [sp-8h] [bp-28h]@1
  signed int (__cdecl *v9)(int, int, int, _DWORD *); // [sp-4h] [bp-24h]@1
  unsigned int v10; // [sp+0h] [bp-20h]@1
  unsigned int v11; // [sp+4h] [bp-1Ch]@1
  int v12; // [sp+8h] [bp-18h]@1
  _DWORD *v13; // [sp+Ch] [bp-14h]@1

  v13 = CookiePointer;
  v12 = EstablisherFrame;
  v11 = TargetLevel;
  v9 = unwind_handler4;
  v10 = (unsigned int)&v8 ^ __security_cookie;
  while ( 1 )
  {
    result = EstablisherFrame;
    v5 = *(_DWORD *)(EstablisherFrame + 12);
    if ( v5 == 0xFFFFFFFE || TargetLevel != 0xFFFFFFFE && v5 <= TargetLevel ) // >
      break;
    v6 = 3 * v5;
    v7 = (*CookiePointer ^ *(_DWORD *)(EstablisherFrame + 8)) + 4 * v6 + 16;
    *(_DWORD *)(EstablisherFrame + 12) = *(_DWORD *)((*CookiePointer ^ *(_DWORD *)(EstablisherFrame + 8)) + 4 * v6 + 0x10);
    if ( !*(_DWORD *)(v7 + 4) )
    {
      _NLG_Notify(*(_DWORD *)(v7 + 8), FramePointer, 0x101);
      _NLG_Call(*(int (**)(void))(v7 + 8));     // 调用对应的Handler
    }
  }
  return result;
}
```

这样上述函数就比较容易理解了，就是对给定的异常信息节点EstablisherFrame，逐一遍历其中的ScopeTable表项，并调用对应的ExecuteFunc。

这样对于异常处理的分析就结束了，可能有地方理解不对，欢迎提出。后面发现有什么新的内容，再添加吧。

**未总结内容**

1. “ExceptionNestedException 和 ExceptionCollidedUnwind”
2. SafeSEH的缓解措施

**参考文章**

1. [SEH分析笔记（X86篇）](http://boxcounter.com/technique/2011-10-19-seh-x86/)
2. [Windows异常处理](https://dbglife.github.io/2017/06/14/Win32-SEH-Internals.html)

**修订历史**

* 2017-08-01 16:51:23		完成文章


By Andy @2017-08-01 16:51:23



