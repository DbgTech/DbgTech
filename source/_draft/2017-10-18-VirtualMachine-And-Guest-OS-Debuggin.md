---
title: 运行操作系统的“虚拟机”们
date: 2017-10-18 20:01:23
tags:
- 虚拟机
- 启动过程
categories:
- 虚拟机
---

在平时工作中会用到虚拟机用于调试程序在不同平台的运行，用的最多的当属VMVware和Virtual Box。最近看了一点操作系统原理相关的东西，即ReactOS操作系统的源码。单看源码没啥意思，关键是要跑起来，最好是能够有控制地“跑”起来，那就要调试。市面上这些虚拟机很多如何用尚且不完全知道，更何况要拿它调试系统，于是乎就有了此篇，用于总结常用到的几种“虚拟机”。之所以加上引号，是因为有两个根本算不上所谓的虚拟机，顶多算是硬件模拟器，它们就是Bochs和Qemu。

#### 虚拟机们的对比 ####

几种虚拟机的整体特点对比，如下表。

|  |Cost/License|	Method	| Debugging	| Configuration |
|-------|-----------|---------------------------|-----------------------------------------|
|Bochs | Free / LGPL | Full emulation (slow)|Yes, built-in|	Command line, script file, interactive menus|
|QEMU  | Free / GPL | Emulation/dynamic translation | Yes, via GDB stub | Command line (optional GUI)|
|VMWare | No |	Virtualization | Yes, via GDB stub | GUI, command line (optional) |
|VirtualBox |Free / mixed |	Virtualization | Yes, built-in | GUI, command line (optional) |
|Virtual PC | Free | Virtualization (on PC), Emulation (on Mac)| No | GUI, command line (optional)|
|VMWare Virtual Server 2 | Free | Virtualization | Yes, via GDB stub | Web interface, non-free Windows client (VI3)|
|Microsoft Hyper-V	| Free | Virtualization, Emulation on legacy devices | Yes, via WinDBG | GUI, command line (PowerShell)|

总体来讲VirtualBox和VMWare提供了最丰富的功能集，并且具有非常好性能。Bochs目前来讲最慢，但是这是因为它是完全模拟执行，这也使得它的准确性最好。并没有哪一个虚拟机比其他的更好，这个对比仅仅是指出他们之间的差别。

支持安装的宿主平台

|                      | Windows	   | Linux (x86)  | Mac OS X	                   | Others                 |
|----------------------|---------------|--------------|--------------------------------|------------------------|
|Bochs	               |Yes (binaries) |Yes (binaries)|Yes (must compile source code)  |Others (by source code) |
|QEMU	               |Yes	           |Yes	          |Yes	                           |PowerPC and others (by source code)|
|VirtualBox	           |Yes	           |Yes	          |Yes	                           |Solaris |
|Microsoft Virtual PC  |Yes*	       |No	          |Maybe (yes for PowerPCs, no for Intel Macs) |* requires AMD-VT or Intel VM support|
|VMWare Virtual Server 2 |Yes	       |Yes	          |No	                           |No |
|Microsoft Hyper-V	   |Yes	           |No	          |No	                           |No |


支持客户系统

|                      |x86-32	|x86-64	|Others |
|----------------------|--------|-------|-------|
|Bochs	               |Yes	    |Yes	| No |
|QEMU	               |Yes	    |Yes	| Yes: ARM, SPARC, MIPS, MIPS64, m68k, PowerPC|
|VirtualBox	           |Yes	    |Yes	| No|
|Microsoft Virtual PC  |Yes	    |No	    | No|
|VMWare Virtual Server 2 |Yes   |Yes	| No|
|Microsoft Hyper-V	   |Yes	    |Yes    | No|


Supported Disk Image Formats

This chart shows the file formats for an emulated hard disk. The emulators usually support only a flat image for a floppy and an ISO image file for CD-ROMs.

                        Flat	Concatenated	Sparse/Stackable	Journaling	Growing	VMWare format
Bochs	                Yes	    Yes	            Yes	                Yes	        Yes	    Yes
QEMU	                Yes	    No	            Yes	                No	        No	    Yes
VirtualBox	            Yes	    No	            No	                No	        Yes	    Yes
Microsoft Virtual PC	Yes	    No	            Yes	                No	        Yes	    No

////////////////////////////////////////////////
VMWare


Virtual Box




Virtual PC


Parallels Desktop 
    Mac OS平台上的虚拟机解决方案

Bochs

Qemu

SkyEye


**参考文章**

* 模拟器对比 [http://wiki.osdev.org/Emulator_Comparison](http://wiki.osdev.org/Emulator_Comparison)
* Bochs文档 [http://bochs.sourceforge.net/doc/docbook/](http://bochs.sourceforge.net/doc/docbook/)
* Qemu文档 [https://wiki.qemu.org/Main_Page](https://wiki.qemu.org/Main_Page)

**修订历史**

* 2017-10-19 11:21:23		完成文章

By Andy @2017-10-19 11:21:23
