---
title: Windows wow64几个机制分析
date: 2017-08-16 20:52:23
tags:
- Windbg
- Wow64
categories:
- Windbg
- 调试
---

现在X64的系统越来越多，而应用程序更新为64位的速度远远没有那么快，再者X64系统本身也支持32位代码的运行，所以还是有大量程序为32位但是运行在X64系统中。那64位的Windows操作系统是如何做到运行32位程序的呢？就是所谓的WoW64技术，全称为Windows 32-bit on Windows 64-bit，简称WoW64。

有大量的32位程序运行于64位系统，遇到WoW64的问题总会有些不知如何下手，这一篇就将碰到的几个WoW64相关问题做分析，简单总结一下。今后碰到其他的相关问题，分析后逐一再进行补充。

#### WoW64简介 ####

WoW64（**W**indows 32-bit **o**n **W**indows **64**-bit）是Windows操作系统上的子系统，提供了Windows 64位系统上运行32位应用程序的能力。WoW64旨在解决32位与64位Windows中的许多差异，特别是Windows本身的结构变化。

WoW64子系统有轻量级兼容层组成，其所在的64位Windows版本上有类似的接口，目的是创建一个32位环境，为未修改的32位应用程序在64位系统上运行提供一个必须的接口。主要是通过三个动态库实现，这三个DLL和64位的Ntdll.dll是WoW64上32位进程加载的仅有4个64位模块：

* Wow64.dll  提供了核心模拟架构，通往Windows NT内核的入口函数，转换32位与64位调用，包括指针和调用栈操作。
* WoW64Win.dll 提供了Win32k.sys入口函数的thunks。
* WoW64CPU.dll 负责解决进程从32位切换到64位模式，提供主处理器功能抽象的接口。

进程启动时，WoW64.dll加载X86版本的ntdll.dll，并执行它的初始化代码，加载必要的32位的DLL。几乎所有的32位DLL是32位Windows上未修改的模块。但是有一部分模块被修改了，因为这部分模块与64位组件共享内存。用户地址空间之上的32位限制被系统保留。与X86上的系统调用序列不同，WoW64上的32位模块做系统调用是被重新构建过的，使用一套自定义的调用序列。这套调用序列位于用户模式，它的调用过程并不比付出太大代价。如果检测出是自定义调用序列，WoW64 CPU会转换回本地64位模式，并且调用Wow64.dll。转接是在用户模式完成，这样减少对64位内核的影响，

因为是模拟32位程序运行环境，其中会有一些问题。首先是注册表和文件系统的重定向，32位程序访问的HKLM\Software被重定位为HKLM\Software\Wow64Node。操作系统使用的%SystemRoot%\system32目录存放的是64位可执行文件；32位应用程序使用的系统可执行文件被放到了%SystemRoot%\SysWoW64目录下；以及Program Files目录，32位程序存放路径为Program File(x86)，64位程序存放路径才是Program Files。这些对于应用程序来说都是透明的，是由WoW64子层进行转换。

同时应用程序也有兼容性问题，32位驱动和有组件插入到64位组件进程（如Explorer.exe）内存空间的32位应用程序无法在64位平台上执行。更早期的16位程序也无法运行，因为X64的CPU上不支持VM86模式。

关于效率问题，在X86-X64 CPU上指令模拟是由芯片完成，和真正的32位程序执行差别不大；但是在Itanium处理器上，指令模拟更多由软件模拟效率相对低一些。API thunk开销相比于NT内核调用本身微乎其微。虚拟内存大小，对于页粒度位8Kb的Itanium处理器来说，这个开销是比较显著的。

WoW64为每一个线程分配一个额外的64位的栈。

IsWow64Process() 用于32位程序判断是否运行于WoW64，GetNativeSystemInfo()获取更多CPU相关的信息。

#### WoW64切换 ####



#### WoW64进程异常分发 ####








































**未总结内容**

1. “ExceptionNestedException 和 ExceptionCollidedUnwind”

**参考文章**

* WoW64 Wiki	[https://zh.wikipedia.org/wiki/WoW64](https://zh.wikipedia.org/wiki/WoW64)
* WoW64的切换细节   [http://advdbg.org/blogs/advdbg_system/articles/5495.aspx](http://advdbg.org/blogs/advdbg_system/articles/5495.aspx)
* WoW进程的异常分发过程   [http://advdbg.org/blogs/advdbg_system/articles/5884.aspx](http://advdbg.org/blogs/advdbg_system/articles/5884.aspx)


**修订历史**

* 2017-08-16 11:21:23		完成文章

By Andy @2017-08-16 20:13:23