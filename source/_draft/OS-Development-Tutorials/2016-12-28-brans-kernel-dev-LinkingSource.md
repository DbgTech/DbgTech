---
title: 《内核开发》之基本内核
date: 2017-02-28 14:33:32
tags:
- 操作系统
- 翻译
categories:
- 内核
- 翻译
---

### 创建Main函数，链接C源码 ###

在通常的C编程实践中，函数main()是普通应用程序的入口点。尽量为了保持你普通应用程序的编程实践经验，同时也让你熟悉内核开发，本教程在C代码中依旧保持main()函数作为入口点。如果你急得本教程之前的部分，我们尽量保证最少地使用汇编代码。在这一部分最后，需要进入到'start.asm'添加中断服务例程来调用C函数。

这一部分教程，编写一个'main.c'文件，以及一个头文件，用来包含一些通用的函数原型：'system.h'。'main.c'也会包含函数main()，它将作为我们的C程序的入口点。作为内核开发的规则，我们通常不能够从main()函数返回。许多操作系统用main()来初始化内核和子系统，加载Shell应用程序，最后main()函数会进入idle循环。空闲循环是用于多任务操作系统中，没有任务执行时运行。如下是一个带有基础main()函数的的例子'main.c'，也包括在下面部分教程中要用到的函数体。
<!-- more -->
``` C
#include < system.h >

/* You will need to code these up yourself!  */
unsigned char *memcpy(unsigned char *dest, const unsigned char *src, int count)
{
/* Add code here to copy 'count' bytes of data from 'src' to      'dest', finally return 'dest' */
}

unsigned char *memset(unsigned char *dest, unsigned char val, int count)
{
/* Add code here to set 'count' bytes in 'dest' to 'val'. Again, return 'dest' */
}

unsigned short *memsetw(unsigned short *dest, unsigned short val, int count)
{
/* Same as above, but this time, we're working with a 16-bit 'val' and dest pointer. Your code can be an exact copy of the above, provided that your local variables if any, are unsigned short */
}

int strlen(const char *str)
{
/* This loops through character array 'str', returning how many characters it needs to check before it finds a 0. In simple words, it returns the length in bytes of a string */
}

/* We will use this later on for reading from the I/O ports to get data from devices such as the keyboard. We are using what is called 'inline assembly' in these routines to actually do the work */
unsigned char inportb (unsigned short _port)
{
	unsigned char rv;
	__asm__ __volatile__ ("inb %1, %0" : "=a" (rv) : "dN" (_port));
	return rv;
}

/* We will use this to write to I/O ports to send bytes to devices. This will be used in the next tutorial for changing the textmode cursor position. Again, we use some inline assembly for the stuff that simply cannot be done in C */
void outportb (unsigned short _port, unsigned char _data)
{
	__asm__ __volatile__ ("outb %1, %0" : : "dN" (_port), "a" (_data));
}

/* This is a very simple main() function. All it does is sit in an infinite loop. This will be like our 'idle' loop */
void main()
{
    /* You would add commands after here */

    /* ...and leave this loop in. There is an endless loop in 'start.asm' also, if you accidentally delete this next line */
    for (;;);
}
```

编译main.c之前，需要在'start.asm'中添加两行。需要让NASM知道main()函数是一个外部的文件提供，并且需要从汇编文件中调用main()函数。打开'start.asm'，寻找'stublet:'行。在该行之后，添加如下的内容：

```
extern _main
call _main
```

等一下，看上面的代码，为什么需要在main之前添加下划线呢，但是在C语言中我们将它声明成了'main'了啊？gcc编译器在编译文件时，会在所有的函数和变量名字前添加一个下划线。因此，为了能够从我们的汇编代码中访问到函数或变量，如果函数是在C源码文件中声明，则必须在函数名字前添加一个下划线。

这其实对于'按样编译'已经足够好，但是哦我们仍然需要'system.h'。仅仅需要创建一个空白文本文件，命名为'system.h'。将所有的函数原型，memcpy，memset，memsetw，strlen，inportb，以及outportb等函数添加到文件中。使用宏来防止反复包含include文件或'header'文件，可以使用#ifdef，#define，以及#endif等小技巧。下面会将这个文件包含到每一个C源码文件中。这样就定义了内核中所用到的每一个函数。这样就可以在方便时扩展这个库，添加一些需要的所有的东西。很明显文件应该如下样子：

``` C
#ifndef __SYSTEM_H
#define __SYSTEM_H

/* MAIN.C */
extern unsigned char *memcpy(unsigned char *dest, const unsigned char *src, int count);
extern unsigned char *memset(unsigned char *dest, unsigned char val, int count);
extern unsigned short *memsetw(unsigned short *dest, unsigned short val, int count);
extern int strlen(const char *str);
extern unsigned char inportb (unsigned short _port);
extern void outportb (unsigned short _port, unsigned char _data);

#endif
```

下面，我们给出如何编译这个文件。打开前面教程中创建的'build.bat'文件，加入如下行用于编译'main.c'文件。请注意，我们会假设'system.h'是在内核源码目录下的一个'include'目录中。这个命令执行编译器'gcc'。在众多的传入参数中，‘-Wall’参数会给出代码中的警告。‘-nostdinc’和‘-fno-builtin’意思是不是用C标准库函数。‘-I./include’的意思是告诉编译器我们的头文件是在当前目录下的‘include’目录中。‘-c’参数告诉gcc仅仅编译，而不需要连接！记得前一篇教程中，‘-o main.o’是编译器要输出的文件。最后一个参数就是我们的源码文件‘main.c’。简言之，使用由于内核的选项编译‘main.c’为‘main.o’。

如果你了解一点汇编知识，这个文件中的汇编代码将会非常明了。随着代码的执行，当前这个文件会加载起一个8Kb的栈，然后进入一个无限的循环中。这个栈是一小段的内存，但是它是用于C语言中函数调用时存储和传递参数。它也用于保存你声明的用于函数内部的局部变量。任何其他的全局变量保存在数据和BSS段。在mboot和stublet之间的几行代码组成了一个特殊的标记，GRUB会用它验证最终输出的二进制文件时即将被加载的内核。不要过于纠结于理解多引导头。（汇编文件：'start.asm'）


> 右键点击批处理文件，然后选择‘编辑’，来编辑它

```
gcc -Wall -O -fstrength-reduce -fomit-frame-pointer -finline-functions -nostdinc -fno-builtin -I./include -c -o main.o main.c
```

不要忘记我们在‘build.bat’中遗留下的指令！要将‘main.o’添加到obj文件列表中，让链接器将他链接到你创建的内核中。最后，如果你对于附加函数，比如memcpy等的添加有问题，可以参考‘main.c’的[源码](http://www.osdever.net/bkerndev/Sources/main.c)。

By Andy @ 2017/02/28 10:00:17