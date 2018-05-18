---
title: 《系统开发》之将引导扇区加载到硬盘上
date: 2017-03-03 20:32:32
tags:
- 操作系统
- 翻译
categories:
- 内核
- 翻译
---

原文地址：[http://www.osdever.net/tutorials/view/copying-your-bootsector-to-a-hard-drive](http://www.osdever.net/tutorials/view/copying-your-bootsector-to-a-hard-drive)

首先，如果你还没有阅读[《将引导扇区拷贝到你的软盘》](http://www.osdever.net/tutorials/view/copying-your-bootsector-to-a-floppy-disk)教程，你应该先阅读一下它。

对于安装到硬盘上的引导扇区的要求和软盘上的引导扇区的要求类似：

* 引导扇区应该是确定的512字节长
* 引导扇区应该以0xAA55结尾
* 引导扇区要被编译为一个平坦模式的二进制 

现在，BIOS（基本输入输出系统）只能看到第一个硬盘上的引导扇区（多数的BIOS是这样的，尽管一些BIOS允许你选择要查看的硬盘），所以我们需要将引导扇区拷贝到第一个硬盘上。在Linux，第一个硬盘是/dev/hda，在Partcopy中则是h0。

### 在Windows平台上使用PartCopy拷贝 ###

这和用PartCopy将引导扇区拷贝到软盘上一模一样（在本例中引导扇区被称为bootsec.bin）：

```
partcopy bootsec.bin 0 200 -h0
```

我们从第一硬盘的最起始位置（第一硬盘用-h0指定）开始，一直拷贝到512字节（512字节=0x200）。

在拷贝之前，你可能会想要备份一下老的引导扇区，以便后面复制损坏后可以修复。下面的命令将老的引导扇区保存到oldboot.bin文件中：

```
partcopy -h0 0 200 oldboot.bin
```
<!-- more -->
### 在Linux平台下使用dd拷贝 ###

首先，引导扇区被称为bootsec.bin。在Linux平台下，第一个硬盘是/dev/hda。我们想要将我们的引导扇区当作一个512字节的块写入，bs=512设置了一块的大小，count=1指定了我们仅仅想要写一个块。

```
dd if=bootsec.bin of=/dev/hda bs=512 count=1
```

在写入引导扇区之前，你可能会要备份老的引导的扇区，这样在写入失败时可以重新修复它。如下的命令将老的引导扇区保存到oldboot.bin文件中：

```	
dd if=/dev/hda of=oldboot.bin bs=512 count=1
```

By Andy @ 2017/03/03 20:36:32 