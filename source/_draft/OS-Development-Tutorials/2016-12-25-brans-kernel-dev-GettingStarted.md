---
title: 《内核开发》之开始
date: 2016-12-25 09:33:32
tags:
- 操作系统
- 翻译
categories:
- 内核
- 翻译
---

### 开始 ###

内核开发是一个长期写代码的过程，也是一个调试各种系统组件的长期过程。这个过程是一个非常烦人的任务，然而你并不需要大规模的工具集写内核。本内核开发教程主要解决使用GRUB加载内核到内存。GRUB需要直接加载一个保护模式的二进制映像：这个映像就是我们即将要构建的内核了。

学习本教程，你最少需要C编程语言的基础知识。如果X86汇编器的知识就再好不过了，X86汇编可以操作处理器内部的指定寄存器。这就是说，你的工具集至少需要一个C编译器，用于生成32位代码；32位的链接器，链接器可以用于生成32位的X86输出。

对于硬件，必须准备一个带有386或更新的处理器（比如386，486，5X86，6X986，奔腾，赛扬等等）。最好的是在你的开发机器旁边就有第二台计算机作为测试机器。如果你无力再购买一台计算机用作测试，或者桌面上没有地方放另外一台机器了，你或许可以使用虚拟机软件，或你也可以使用你的开发机器作为测试机（尽管这可能会导致开发时间增长）。要准备好随时重启机器，因为你要测试和调试内核。
<!-- more -->
### 测试机的硬件要求 ###

&emsp;&emsp;1. IBM兼容PC
&emsp;&emsp;2. 386或更新的的处理器
&emsp;&emsp;3. 4M的RAM
&emsp;&emsp;4. 兼容VGA的显示器
&emsp;&emsp;5. 键盘
&emsp;&emsp;6. 软驱
&emsp;&emsp;（是的，就是这样，测试机上甚至不需要硬盘）

### 开发机的推荐的硬件要求 ###
&emsp;&emsp;1. IBM兼容PC
&emsp;&emsp;2. 奔腾2或K6 300MHz的CPU
&emsp;&emsp;3. 32M的RAM
&emsp;&emsp;4. 兼容VGA的显示器
&emsp;&emsp;5. 键盘
&emsp;&emsp;6. 软驱
&emsp;&emsp;7. 硬盘，足够的空间存储开发工具，存储文档和源代码
&emsp;&emsp;8. Windows操作系统，或更喜欢的Unix（Linux，FreeBSD）
&emsp;&emsp;9. 联网可以查看文档
&emsp;&emsp;（极力推荐鼠标）

### 工具集 ###

##### 编译器 #####

Gnu C编译器（GCC）[Unix]
DJGPP (GCC for DOS/Windows) [Windows]

##### 汇编器 #####

Netwide Assembler (NASM) [Unix/Windows]

##### 虚拟机 #####

&emsp;&emsp;1）VMWare Workstation 4.0.5 [Linux/Windows NT/2000/XP]
&emsp;&emsp;2）Microsoft VirtualPC [Windows NT/2000/XP]
&emsp;&emsp;3）Bochs [Unix/Windows]

By Andy @ 2016/12/26 10:20:17 