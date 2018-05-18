---
title: Windbg的!error命令输出 Unable to get error code text
date: 2017-04-12 11:33:32
tags:
- Windbg
categories:
- 调试
---

对于经常用Windbg调试的人来说，系统函数调用失败，通常会用!gle来看一下GetLastError（）函数返回了什么内容呢？但是这个函数返回的是一个错误码，例如0x00000005，这玩意可是人无法直接识别的（当然了0x00000005这个码倒是可以直接识别，拒绝访问嘛，但不是所有的错误码都能记住的。）。当然了Windbg还是比较人性化的，它立马提供了一个扩展命令，即!error。这个命令可以将错误码转换成“人话”，但是尝试无数次，并且是在换了无数个Windbg版本后尝试了无数次，它的结果始终不变，如下：

```
0:089> !error 0x5
Error code: (Win32) 0x5 (5) - <Unable to get error code text>
```

后面尖括号中的内容是“人话”，意思就是无法获取到错误的文本。换个说法就是!error命令执行有问题，到底有啥问题，它没告诉你！每次看到这段文本就有“一刀捅死”Windbg的冲动。冲动归冲动，还是拿它没办法。好吧，那咱们自己找一下问题根源吧。

!error 命令存在于ext.dll模块中，执行该命令时即调用 ext!error() 函数，调用栈如下所示：

```
0:001> kv
 # ChildEBP RetAddr  Args to Child
00 01f3d8e0 616ca9c7 01f3d904 01f3d900 00000005 ext!FormatAnyStatus+0x1d6 (FPO: [Non-Fpo])
01 01f3d908 616caa87 0059bfe8 00000000 00569ec0 ext!DecodeErrorForMessage+0x23 (FPO: [Non-Fpo])
02 01f3da7c 616cab2b 00000000 616caae0 00569ec0 ext!DecodeError+0x36 (FPO: [Non-Fpo])
03 01f3da98 59d06dd9 00569ec4 01f3db6f 76c9bc84 ext!error+0x4b (FPO: [2,3,4])
04 01f3db14 59d06f8c 00569ec0 01f3dd3c 01f3dc88 dbgeng!ExtensionInfo::CallA+0x255 (FPO: [Non-Fpo])
05 01f3dcb8 59d07024 00569ec0 01f3dd3c 01f3dd48 dbgeng!ExtensionInfo::Call+0xfb (FPO: [Non-Fpo])
06 01f3dce4 59d059ed 01f3dd3c 01f3dd48 00000000 dbgeng!ExtensionInfo::CallAny+0x6c (FPO: [Non-Fpo])
07 01f3e150 59d3ec70 00000000 00000000 00000000 dbgeng!ParseBangCmd+0x46f (FPO: [Non-Fpo])
08 01f3e1cc 59d3faaf 76c985bc 00569ec0 00000001 dbgeng!ProcessCommands+0x816 (FPO: [0,25,4])
09 01f3e22c 59c8e40f 00000000 76c98104 00000000 dbgeng!ProcessCommandsAndCatch+0xad (FPO: [Non-Fpo])
0a 01f3e694 59c8e5e8 0000000a 00000000 76c98148 dbgeng!Execute+0x247 (FPO: [Non-Fpo])
0b 01f3e6d8 002f4bd7 00569ec8 00000001 01f3eab8 dbgeng!DebugClient::ExecuteWide+0x68 (FPO: [Non-Fpo])
0c 01f3ea94 002f5068 00000006 00000008 00000000 windbg!ProcessCommand+0x12f (FPO: [Non-Fpo])
0d 01f3fab0 002f72d1 761fbfe9 00000000 00000000 windbg!ProcessEngineCommands+0xd0 (FPO: [0,1023,4])
0e 01f3faec 74fd336a 00000000 01f3fb38 76ff9902 windbg!EngineLoop+0x571 (FPO: [Non-Fpo])
0f 01f3faf8 76ff9902 00000000 76f56898 00000000 kernel32!BaseThreadInitThunk+0xe (FPO: [Non-Fpo])
10 01f3fb38 76ff98d5 002f6d60 00000000 00000000 ntdll!__RtlUserThreadStart+0x70 (FPO: [Non-Fpo])
11 01f3fb50 00000000 002f6d60 00000000 00000000 ntdll!_RtlUserThreadStart+0x1b (FPO: [Non-Fpo])
```

从上述的调用栈，可以看到Windbg执行扩展命令的过程。dbgeng!ExtensionInfo类用于处理扩展命令，并调用对应的扩展模块。栈中可以看到对 ext!error()函数的调用。其后依次解析错误，将错误转换成可读字符串，最终调用到FormatAnyStatus()，它的作用就是将错误码转换成对应字符串。当然出现问题的也是它了。
Windbg调试虽然强大，但是一堆汇编，想要了解函数的大致流程，用它就费劲了。那么该是其他工具出场的时候了，IDA Pro，大名鼎鼎的反汇编工具。反汇编之后，找到FormatAnyStatus()函数，很快就找到了问题根源。结合IDA F5的结果，以及调试过程，确定了出问题的代码在该函数后半截，所以给出FormatAnyStatus() 的后半截函数，如下:
<!-- more -->
```
CHAR *FormatAnyStatus(unsigned int a1@<ecx>, __int32 a2, void *a3, int *a4, char **a5)
{

    ......

    v12 = FormatMessageA(v16 | 0x200, v11, v6, 0, v7, 0x400u, 0);
LABEL_34:
    if ( v11 && v15 )
        FreeLibrary(v11);
    if ( !v12 )
        goto LABEL_48;
    for ( i = v7; *i; ++i )
    {
        if ( !_isprint((unsigned __int8)*i) )
            *i = 0x20;
    }

    do
    {
        if ( !_isspace((unsigned __int8)v7[v12 - 1]) )
            break;
        v7[--v12] = 0;
    }
    while ( v12 );

    result = v7;
    if ( !v12 )
LABEL_48:
        result = "<Unable to get error code text>";
    return result;
}
```

对于实验命令 !error 0x5，0x5对应的错误信息为 Win32 - "拒绝访问。"，在上述函数中调用 FormatMessageA()后确实返回了字符串"拒绝访问。"，如下图所示调试结果。但是由于后面的代码中会判断字符串是否是可打印的，即调用_isprint()函数，它的返回一直为0，所有的字符都被修改为了 0x20，即空格。再之后的代码do{}while()则将空格替换成结束符'\0'中。将缓存中所有的空格替换成了0，并且长度也减为0了，最终由于没有返回"有效内容"，返回字符串被赋值为 “ < Unable to get error code text > ”。

<div aligin="center">
![函数调用](/img/2017-04-12-windbg-error-outputinfo.jpg)
</div>

```
0:003> r
eax=0000000c ebx=00000000 ecx=773d2fdd edx=00357014 esi=00000005 edi=57c57078
eip=57b8ef54 esp=026dd820 ebp=026dd840 iopl=0         nv up ei pl zr na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
ext!FormatAnyStatus+0x1ae:
57b8ef54 8bf0            mov     esi,eax
0:003> db 57c57078 
57c57078  be dc be f8 b7 c3 ce ca-a1 a3 0d 0a 00 00 00 00  ................
57c57088  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
57c57098  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
57c570a8  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
57c570b8  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
```

结果的结果，就出现了Windbg中看到的结果。正如文章开始处所显示的!error 0x5的内容。

```
0:089> !error 0x5
Error code: (Win32) 0x5 (5) - <Unable to get error code text>
```

从网上查了一下，说是和语言环境有关，从上述的调试结果，也可知和语言相关。通过Windbg的.locale命令设置了语言环境，结果依旧。暂时也没有解决办法了。还是老实从网上下载一个ErrorLookup程序吧，或者换到English版本的操作系统？或者是下载一个英语语言包？不折腾了，晓得原因即可。

**修订历史**

* 2017-04-12 12:30:17		完成文章

By Andy @ 2017/04/12 12:30:17
