---
title: 《系统开发》之引导扇区使用GRUB
date: 2017-03-03 20:36:32
tags:
- 操作系统
- 翻译
categories:
- 系统
- 翻译
---

原文地址：[http://www.osdever.net/tutorials/view/using-grub](http://www.osdever.net/tutorials/view/using-grub)

GRUB的主页可以在 [http://www.gnu.org/software/grub/grub.en.html](http://www.gnu.org/software/grub/grub.en.html) 找到。本篇教程是由Chris Giese编写，在他的站点上也可以找到该文章。


### 参考 ###

这些信息来自于我使用GRUB的经验，从如下的邮件中：

>* Subject: Re: generic bootloader question
>* Newsgroups: alt.os.development
>* From: "Marv" 
>* Date: Sat, 7 Apr 2001 23:35:20 +0100
>* References: 
>* Message-ID: 

>* Subject: Re: Grub multiboot example
>* Newsgroups: alt.os.development
>* From: "Marv" 
>* Date: Mon, 4 Jun 2001 17:21:17 +0100
>* References: 
>* Message-ID: 

>* Subject: Re: Grub multiboot example
>* Newsgroups: alt.os.development
>* From: "Mike Wimpy" 
>* Date: Thu, 7 Jun 2001 22:17:51 -0700
>* References:      
>* Message-ID: 

>* Subject: Re: grub coff (solved it!)
>* Newsgroups: alt.os.development
>* From: "Mark & Candice White" 
>* Date: Sun, 16 Sep 2001 10:57:34 GMT
>* References: 
>* Message-ID: 
<!-- more -->
### 获取GRUB ###

源代码：

[ftp://alpha.gnu.org/gnu/grub/grub-0.90.tar.gz](ftp://alpha.gnu.org/gnu/grub/grub-0.90.tar.gz)

二进制：

[ftp://alpha.gnu.org/gnu/grub/grub-0.90-i386-pc.tar.gz](ftp://alpha.gnu.org/gnu/grub/grub-0.90-i386-pc.tar.gz)

DOS和Windows用户需要PARTCOPY或RAWRITE：

[http://www.execpc.com/~geezer/johnfine/index.htm#zero](http://www.execpc.com/~geezer/johnfine/index.htm#zero)
[http://uranus.it.swin.edu.au/~jn/linux/rawwrite.htm](http://uranus.it.swin.edu.au/~jn/linux/rawwrite.htm)
[http://www.tux.org/pub/dos/rawrite/](http://www.tux.org/pub/dos/rawrite/)

### 编译GRUB ###

UNIX

```
	configure ; make
```
	
(... 我认为这样就可以 ... )

DOS和Windows

(ha，忘却它吧)

### 没有文件系统情况下在软盘上安装GRUB ###

1.获取GRUB二进制（文件 stage1 和 stage2）。

2.将文件“stage1”和“stage2”两个文件结合成一个文件：

&emsp;&emsp;（DOS）copy /b stage1 + stage2 boot

&emsp;&emsp;（UNIX）cat stage1 stage2 \> boot

3.直接将文件“boot”写入到软盘中：

&emsp;&emsp;（DOS）rawrite boot a:

&emsp;&emsp;&emsp;&emsp; -OR-

&emsp;&emsp;（DOS）partcopy boot 0 168000 -f0

&emsp;&emsp;（UNIX）cat boot \> /dev/fd0

PARTCOPY会给一个错误信息，因为文件“boot”比0x168000字节短很多，但是这也是OK的。

### 在一个带有文件系统的软盘上安装GRUB ###

1.制作一个没有文件系统的可启动的GRUB软盘，像上面描述一样

2.将文件“stage1”和“stage2”拷贝到第二个磁盘上，用GRUB可识别的文件系统格式化的磁盘上。要使用GRUB的“setup”命令，这些文件必须要被保存在子文件夹“/boot/grub”中：

（DOS）mkdir a:\boot

（DOS）mkdir a:\boot\grub

（DOS）copy stage1 a:\boot\grub

（DOS）copy stage2 a:\boot\grub


（UNIX）mount /dev/fd0 /mnt

（UNIX）mkdir /mnt/boot

（UNIX）cp stage1 /mnt/boot/grub

（UNIX）cp stage2 mnt/boot/grub

3.在GRUB被安装到磁碟#2上之后，文件“stage2”不能修改，删除，碎片整理或是删除。如果它被修改了，那么这个磁盘就不能够再进行引导了。为了防止文件被修改，将文件设置为只读模式：

（DOS）attrib +r +s stage2

（UNIX）chmod a-w stage2

上面的DOS命令使得“stage2”编程一个系统文件，并且是只读。它需要防止被整理掉。

注意：文件“stage1”会拷贝到引导扇区。如果这个文件在安装完GRUB后被移动或删除，软盘仍然是可引导的。

4.从没有文件系统的磁盘使用GRUB引导计算机。在GRUB提示时，弹出这张软盘，并且插入格式化的软盘（带有文件系统和“stage1”“stage2”文件，是在“/boot/grub”目录）。

5a.如果文件“stage1”和“stage2”被保存在软盘2的“/boot/grub”目录，可以通过在软盘2上输入如下命令安装GRUB：

```
    setup (fd0)
```

这条命令等价于如下的命令

```
install /boot/grub/stage1 d (fd0) /boot/grub/stage2 p /boot/brub/menu.lst
```

5b.如果文件“stage1”和“stage2”保存在了其他的地方，比如在子目录“/foo”下，可以输入如下的命令安装GRUB。

```
install=(fd0)/foo/stage1 (fd0) (fd0)/foo/stage2 0x8000 p (fd0)/foo/menu.lst
```

软盘2（带有文件系统的软盘）现在可以引导了。
xxx - 从软盘2引导，拷贝新的或修改“stage2”，重新执行“setup”或“install”？这样会好么？（xxx - GRUB不是一个shell命令，它不能够拷贝文件或是列举目录 - 它可以么？）

```
xxx - install syntax:
    0x8000
    This value gets embedded into the bootsector of the floppy to indicate the address that stage2 should be loaded into memory.
    
    p
    Modifies stage2, to report to the kernel the partition that stage 2 was found on (I think).
    
    (fd0)/boot/grub/menu.lst
    Modifies stage2, and tells it where to load the menu.lst (bootmenu) configuration file from.
```

### 编写一个多引导内核 ###

##### 多引导头 #####

无论文件格式是什么样，你编写的内核必须有一个多引导头。这个头包括如下几点：

&emsp;&emsp;1.必须是4字节对齐的边界

&emsp;&emsp;2.必须出现在内核文件的第一个8K内

Note：这是一个地址位于.text区段的第一个8K，而不必非得是文件开头的8K内。

##### ELF内核 #####

GRUB可以直接理解ELF文件格式。如果你的内核是ELF格式，你可以简单地使用如下的多引导头：

```
MULTIBOOT_PAGE_ALIGN   equ 1
```

（xxx - 这个服务器很容易打开，一些人应该将这些工具拷贝到本地一份。）

建议先编译一个普通的COFF内核，然后按照如下命令处理：

```
objcopy-elf -0 elf32-i386 krnl.cof krnl.elf
```

&emsp;&emsp;如果这么做失败了，可以通过使用 aout kludge 参数编译GRUB，使它可以加载COFF内核。这在多引导头结尾使用额外的成员，像这样：

```
MULTIBOOT_PAGE_ALIGN   equ 1
...
int main(multiboot_info_t *boot_info)
{       
    if(boot_info->flags & 2)
    {       
        kprintf("the command line is:\n'%s'\n", (char *)boot_info->cmdline); 
    }
    ...

```

### 制作一个引导菜单（文件“menu.lst”） ###

```
Example 1:
    # Entry 0:
    title   WildMagnolia
    root(fd0)
    kernel  /boot/kernel.elf
    module  /boot/mod_a
    module  /boot/mod_b
    
Example 2:
    #
    # Sample boot menu configuration file
    #
    
    #  default - boot the first entry.
    default 0
    
    # if have problem - boot the second entry.
    fallback 1
    
    # after 30 sec boot default.
    timeout 30
    
    # GNU Hurd
    title  GNU/Hurd
    root   (hd0,0)
    kernel /boot/gnumach.gz root=hd0s1
    module /boot/serverboot.gz
    
    # Linux - boot ot second HDD
    title  GNU/Linux
    kernel (hd1,0)/vmlinuz root=/dev/hdb1
    
    # booting Mach - get kernel from floppy
    title  Utah Mach4 multiboot
    root   (hd0,2)
    pause  Insert the diskette now	
```

By Andy @ 2017/03/02 20:36:32 