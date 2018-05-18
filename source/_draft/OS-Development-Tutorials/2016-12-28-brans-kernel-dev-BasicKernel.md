---
title: 《内核开发》之基本内核
date: 2016-12-28 09:33:32
tags:
- 操作系统
- 翻译
categories:
- 内核
- 翻译
---

### 基本内核 ###

教程的这一部分，我们研究一点汇编器知识，学习创建连接器脚本的基础知识以及为什么使用连接器脚本，最后会学习如何使用批处理文件自动化汇编，编译，和连接最基本的保护模式内核。请注意这一点上，教程假设你已经在Windows系统或基于DOS的平台上安装了NASM和DJGPP。我们也假设你已经对X86汇编语言有基本知识。

##### 内核入口 #####

内核的入口点是bootloader调用内核时首先执行的一段代码。这一块代码通常都是使用汇编语言编写的，因为有一些东西，比如设置新的栈或加载新的GDT，IDT或段寄存器，都是无法使用C语言能够完成的功能。在一些刚刚开始的内核，当然也包括其他几个大的内核，更加专业的内核，会将所有的汇编代码放到这一个文件中，将其他的代码放到几个C源码文件中。

如果你了解一点汇编知识，这个文件中的汇编代码将会非常明了。随着代码的执行，当前这个文件会加载起一个8Kb的栈，然后进入一个无限的循环中。这个栈是一小段的内存，但是它是用于C语言中函数调用时存储和传递参数。它也用于保存你声明的用于函数内部的局部变量。任何其他的全局变量保存在数据和BSS段。在mboot和stublet之间的几行代码组成了一个特殊的标记，GRUB会用它验证最终输出的二进制文件时即将被加载的内核。不要过于纠结于理解多引导头。（汇编文件：'start.asm'）
<!-- more -->
```
; This is the kernel's entry point. We could either call main here,
; or we can use this to setup the stack or other nice stuff, like
; perhaps setting up the GDT and segments. Please note that interrupts
; are disabled at this point: More on interrupts later!
[BITS 32]
global start
start:
    mov esp, _sys_stack     ; This points the stack to our new stack area
    jmp stublet

; This part MUST be 4byte aligned, so we solve that issue using 'ALIGN 4'
ALIGN 4
mboot:
    ; Multiboot macros to make a few lines later more readable
    MULTIBOOT_PAGE_ALIGN	equ 1<<0
    MULTIBOOT_MEMORY_INFO	equ 1<<1
    MULTIBOOT_AOUT_KLUDGE	equ 1<<16
    MULTIBOOT_HEADER_MAGIC	equ 0x1BADB002
    MULTIBOOT_HEADER_FLAGS	equ MULTIBOOT_PAGE_ALIGN | MULTIBOOT_MEMORY_INFO | MULTIBOOT_AOUT_KLUDGE
    MULTIBOOT_CHECKSUM	equ -(MULTIBOOT_HEADER_MAGIC + MULTIBOOT_HEADER_FLAGS)

    EXTERN code, bss, end

    ; This is the GRUB Multiboot header. A boot signature
    dd MULTIBOOT_HEADER_MAGIC
    dd MULTIBOOT_HEADER_FLAGS
    dd MULTIBOOT_CHECKSUM

    ; AOUT kludge - must be physical addresses. Make a note of these:
    ; The linker script fills in the data for these ones!
    dd mboot
    dd code
    dd bss
    dd end
    dd start

; This is an endless loop here. Make a note of this: Later on, we
; will insert an 'extern _main', followed by 'call _main', right
; before the 'jmp $'.
stublet:
    jmp $

; Shortly we will add code for loading the GDT right here!

; In just a few pages in this tutorial, we will add our Interrupt
; Service Routines (ISRs) right here!

; Here is the definition of our BSS section. Right now, we'll use
; it just to store the stack. Remember that a stack actually grows
; downwards, so we declare the size of the data before declaring
; the identifier '_sys_stack'
SECTION .bss
	resb 8192               ; This reserves 8KBytes of memory here
_sys_stack:
```

### 链接器脚本 ###

链接器是将编译器和汇编器的输出文件链接起来，变成一个二进制文件的工具。二进制文件可以有这么几个格式：Flat，AOUT，COFF，PE和ELF。如果你记得的话，我们从工具集中选择的链接器是一个LD链接器。它是一个带有扩展功能集的非常优秀的多用途链接器。现存的有几个版本的LD可以输出你所希望的任何格式的输出文件。忽略你所选择的格式，在输出文件中通常有三个部分。‘Text’或‘Code’是可执行部分。‘Data’区段的作用是保存代码中的硬编码值，例如你声明了一个变量，并且将它设置为5。这个值5就会被保存到‘Data’区段。最后一部分被称为‘BSS’段，‘BSS’段包含了未初始化的数据，它保存了还没有设置任何值的数据。‘BSS’是一个虚拟的区段：它实际上并不存在于二进制，但是当二进制映像加载时它存在于内存中。

接下来是一个LD链接器脚本文件。在这个脚本文件中有三个主要的关键字：OUTPUT_FORMAT会告诉LD我们想要创建什么样的二进制映像。为了保持简单，我们会生成一个纯粹的二进制影响。ENTRY会告诉链接器什么样的文件要作为列表中的第一个object文件链接。我们想要‘start.asm’编译为‘start.o’，作为第一个object文件被链接，因为那是我们的内核的入口点所在。下一行是“phys”，这不是一个关键字，但是它是一个用于链接脚本的变量。这种情况下，我们用它作为内存中地址的指针：一个指向1Mb的指针，这个地址是我们的二进制文件加载和运行的地址。第三个关键字是SECTIONS。如果你研究了这个链接器脚本，你会发现如果定义了3个主要的区段：'.text','.data'和'.bss'。同时会定义三个变量：‘code’，‘data’，‘bss’。不要对这个定义疑惑：看到的三个变量实际上是起始start.asm文件中的三个变量。ALIGN(4096)确保每一个区段起始于4096字节的边界。在这种情况下，这意味着每一个区段在内存中都会起始于一个独立的‘page’。（链接脚本：‘link.ld’）

```
OUTPUT_FORMAT("binary")
ENTRY(start)
phys = 0x00100000;
SECTIONS
{
    .text phys : AT(phys)
    {
        code = .;
        *(.text)
        *(.rodata)
        . = ALIGN(4096);
    }
    .data : AT(phys + (data - code))
    {
        data = .;
        *(.data)
        . = ALIGN(4096);
    }
    .bss : AT(phys + (bss - code))
    {
        bss = .;
        *(.bss)
        . = ALIGN(4096);
    }
    end = .;
}
```

### 汇编和链接 ###

现在，我们必须汇编‘start.asm’，并且使用上面给出的链接器脚本‘link.ld’，来创建GRUB加载的二进制内核。在Unix中最简单的方式就是创建一个makefile脚本来做这个汇编，编译和链接。但是，大多数人，也包括我自己，更加喜欢用Windows。那么，我们只能创建一个批处理文件。批处理文件就是DOS命令的简单集合，可以使用文件名字作为一个命令执行。更加简单的方法：在Windows下编译内核，仅仅需要双击批处理文件即可。

下面给出了教程中使用的批处理文件。'echo'是一个DOS命令，告诉Windows接下来的文字输出到屏幕上。‘nasm’是我们的汇编器，我们将汇编文件会变成aout格式。因为LD为了解决链接过程中的符号问题，它需要一个已知的格式。这会将start.asm会变成start.o。‘rem’命令意思是‘remark’，这是一个注释，它仅仅显示在批处理文件中，对于计算机来说它什么都不是，不需要做任何动作。‘ld’是我们的链接器。‘-T’参数告诉LD接下来是一个链接器脚本。‘-o’意思是接下来是输出文件。任何其他的参数都被理解为需要链接和处理到一起，创建kernel.bin文件。最后，‘pause’命令会在屏幕上显示“Press a key to continue...”，等待我们按下任何的键，这样就可以看到汇编器或链接器会在屏幕上给出语法错误。(编译批处理文件：‘build.bat’)

```BAT
echo Now assembling, compiling, and linking your kernel:
nasm -f aout -o start.o start.asm
rem Remember this spot here: We will add 'gcc' commands here to compile C sources

rem This links all your files. Remember that as you add *.o files, you need to
rem add them after start.o. If you don't add them at all, they won't be in your kernel!
ld -T link.ld -o kernel.bin start.o
echo Done!

pause
```

By Andy @ 2016/12/26 10:20:17