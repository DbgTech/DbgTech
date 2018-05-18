---
title: 《系统开发》之将引导扇区拷贝到软盘
date: 2017-03-03 09:49:32
tags:
- 操作系统
- 翻译
categories:
- 内核
- 翻译
---

原文地址：[http://www.osdever.net/tutorials/view/copying-your-bootsector-to-a-floppy-disk](http://www.osdever.net/tutorials/view/copying-your-bootsector-to-a-floppy-disk)

*Note:教程中用的partCopy的版本和最新的版本所需要的参数不同*

你已经制作了一个引导扇区，现在想要将它拷贝到软盘中。在做这件事之前，需要确认如下的三件事情：

* 你的引导扇区确实有512字节长
* 你的引导扇区是一0xAA55结尾
* 你的扇区被编译成了平坦模式的二进制

在NASM中完成前面两步，只需要在引导扇区最后添加如下的代码即可：

```
times 512-($-$$)-2 db 0
dw 0AA55h
```

对于最后一步，编译（使用NASM）时使用如下的命令：

```
nasm yourbootsector.asm -f bin -o bootsec.bin
```
<!-- more -->	
现在，让我们将引导扇区拷贝到软盘上。

我假设你使用的是Windows系统。对于Windows系统，需要去John Fine的站点下载它的程序PartCopy。对于Linux，需要使用dd工具（有人想要编写一个dd工具的教程么？）。现在，你需要将引导扇区拷贝到软盘的第一个512字节中。要验证一个引导扇区（在本例子中叫做bootsec.bin），可以使用PartCopy工具：

```
partcopy bootsec.bin 0 200 -f0
```

这会拷贝文件bootsec.bin的0-512字节（PargCopy使用16进制，0x200=512）到磁盘驱动（-f0）中。注意，在-f0之后，我们可以防止一个目标偏移，但是因为引导扇区必须位于软盘的最起始位置，所以我们就可以不用关心目标偏移即可（PartCopy的默认值是0）.

如果你的引导扇区没有bug，那么软盘就是可引导的了。但是有一个事情你需要注意一下，如果你在Windows/Dos/Linux下尝试读取软盘，会被提醒磁盘还没有格式化。这是因为我们仅仅是将引导扇区拷贝到了软盘上。这种方式，我们会腹泻一些重要的信息。

为了不腹泻这些重要信息（Dos引导记录），我们需要增加一些代码到引导扇区中。在引导扇区开始（就是在org 0x7c00之后）添加如下代码：

```
start:
    jmp short begin
    nop
    times 0x3B db 0
    
begin:
    the rest of your code goes here
```

现在，我们就可以进行第二步，将我们的引导扇区拷贝到磁盘上。（如果你使用前面的磁盘，确认要首先格式化它）：

```
partcopy bootsec.bin 0 3 -f0
partcopy bootsec.bin 3e 1c2 -f0 3e
```

到这里，你就有一个引导盘了。一个可引导的磁盘，带有你自己编写的引导扇区。

[下载](http://www.osdever.net/downloads/tuts/pcbootsector.zip)例子的完整代码。

By Andy @ 2017/03/03 10:09:32