---
title: Bochs内置调试器使用
date: 2017-09-03 09:52:23
tags:
- Bochs
- ReactOS
categories:
- 翻译
- 笔记
---

Bochs虚拟机就不过多介绍了，官网上说的非常清楚。Bochs是一款模拟形式的虚拟机，区别于VMWare等虚拟机。但是Bochs有一个内置调试器，可以用来调试系统内核，尤其调试系统内核未初始化内置调试之前的系统引导阶段代码逻辑。下面将Bochs官网的使用Bochs内置调试器的教程简单翻译一下。

原文翻译如下：

#### 使用Bochs内置调试器 ####

在编译Bochs时可以添加编译条件，在Bochs内部编译像GDB一样的命令行调试器，使用这个调试器可以设置断点，单步执行指令，或其他的有用的调试功能。如果有其他的调试命令你觉得对调试器非常有帮助，告诉我，我尽可能实现它。

>注意：这一部分介绍了如何开启和使用Bochs命令行调试器。如果想要使用图形形式的前端请参考调试器GUI部分如何开启功能。

要是用调试器，在编译Bochs时需要使用`--enable-debugger`和`--enable-disasm`编译标志进行编译配置。例如：
```
./configure --enable-debugger --enable-disasm
```

> 注意：编译时必须使用flex 2.5.4或更高版本。我听说2.5.2版本无法正常工作。

第一次启动Bochs时，会看到如下的命令行提示符。

```
bochs:1>
```

在命令行提示符中就可以输入如下的这些调试命令了。
<!--more-->
**1. 执行控制（Execution Control）**

| 命令 | 解释 |
| :----| :----- |
| c   | 继续执行 |
| cont| 继续执行 |
| continue| 继续执行 |
| s     [count]|执行count条指令，如果不指定参数，默认值为1|
| step  [count]|同上|
| s     [cpu] [count]|对于对称多处理器结构模拟，在cpu上执行count条指令，count的默认值为1|
| step  [cpu] [count]|同上 |
| s     all [count]|对于对称多处理器结构模拟，所有cpu上都执行count条指令，count的默认值为1|
| step  all [count]|同上 |
| Ctrl-C|停止执行，返回到命令行提示符|
| Ctrl-D|如果在空行上执行，则退出调试器|
| q|退出调试器，继续执行|
| quit|同上|
| exit|同上|

**2. 断点（BreakPoints）**

> 注意: 下面的格式符'seg','off'，和'addr'，没有设置它们的基数。（调试中最好使用十六进制）
> hexidecimal:    0xcdef0123
> decimal:        123456789
> octal:          01234567

| 命令 | 解释 |
| :----|:----|
| vbreak seg:off | 设置虚拟地址指令断点 |
| vb     seg:off | 同上 |
| lbreak addr | 在线性地址指令上设置断点 |
| lb     addr | 同上 |
| pbreak [*] addr | 在物理地址上设置断点 |
| pb     [*] addr | * 符号是兼容GDB命令，为可选参数 |
| break  [*] addr | 同上 |
| b      [*] addr | 同上 |
| info break | 显示当前所有断点状态 |
| bpe    n | 开启断点 |
| bpd    n | 关闭断点 |
| delete n | 删除断点 |
| del    n | 同上 |
| d      n | 同上 |

**3. 内存观察点（Memory WatchPoints）**

| 命令 | 解释 |
| :----|:--- |
|watch read  addr | 在物理地址addr上插入读观察点 |
|watch r     addr| 同上|
|watch write addr| 在屋里地址addr上插入一个写观察点 |
|watch w     addr| 同上|
|watch | 显示当前内存观察点的显示状态 |
|watch stop  | 当遇到观察点时，停止模拟执行（默认）|
|watch continue | 在遇到观察点时，不要停止模拟执行 |
|unwatch addr | 移除指定物理地址上的观察点 |
|unwatch| 移除所有的观察点 |
|trace-mem on/off| 开启/关闭内存访问追踪 |

**4. 操作内存（Manipulating Memory）**

| 命令 | 解释 |
| :----|:--- |
|x  /nuf addr| 在线性地址addr处检查内存内容 |
|xp /nuf addr | 在物理地址 addr处查看内存内容|
|setpmem addr datasize val | 在内存位置addr处 设置datasize大小内存，值为 val |
|writemem | 从指定线性地址dump一个字节的内存内容 到一个文件中|
|crc  addr1  addr2| 显示物理地址范围 addr1..addr2之间内容的 CRC值|

x/xp两个命令的参数解释如下：

```
n 指定要显示的内存单元的数量
u 显示的内存单元的大小，如下参数之一
	b 单个字节
	h 半个字(2 字节)
	w 一个字(4 字节)
	g 一个大字 (8 字节)
	注意：这些不是典型的Intel的类型大小命名，但是它们和GDB惯例一致。
f 打印的格式。如下类型之一：
	x 按照十六进制形式打印
	d 按照十进制形式打印
	u 以无符号的10进制打印
	o 按照八进制形式打印
	t 按照二进制行是打印
```
n，f，和u三个参数是可选参数。u和f默认是使用上一次执行命令时的值，如果没有指定u使用w，f使用x。n当前默认是1.如果这些参数都没有给出，则不应该使用斜线。addr参数也是可选参数。如果没有指定addr参数，则使用下一个地址（即上一个x命令执行时指定的地址n+1）。

**5. 查看信息命令（Info commands）**

| 命令 | 解释 |
| :----|:--- |
|r/reg/regs/registers | 列举CPU整数寄存器和它们的内容 |
|fp/fpu | 列举FPU寄存器和它们的内容 |
|mmx | 列举哦MMX寄存器和它们的内容 |
|sse/xmm| 列举SSE寄存器和它们的内容 |
|ymm | 列举所有AVX寄存器和它们的内容 |
|sreg | 列举段寄存器和它们的内容 |
|dreg | 列举调试寄存器和它们的内容 |
|creg | 列举控制寄存器和它们的内容 |
|info cpu | 列举哦CPU所有的寄存器以及它们的内容 |
|info eflags| 显示解析的EFLAGS寄存器 |
|info break| 显示当前所有断点状态 |
|info tab | 显示分页地址转换 |
|info device | 显示指定设备的状态 |

**6. 操作CPU寄存器（Manipulating CPU Registers）**

| 命令 | 解释 |
| :----|:--- |
| set reg = expr| 修改CPU寄存器值为expr。当前只有通用寄存器和指令寄存器支持修改。不能够修改标志寄存器，段寄存器，浮点寄存器和SIMD寄存器。|
| registers| 列举CPU寄存器以及它们的值 |
| regs| 同上 |
| reg | 同上 |
| r   | 同上 |

```
示例：
	set eax = 2+2/2
	set esi = 2*eax+ebx
```

**7. 反汇编命令 (Disassembly commands)**

| 命令 | 解释 |
| :----|:--- |
|disassemble start end | 在给定的线性地址范围内反汇编指令，包含start处指令，不包含end处指令。使用"set $disassemble_size ="告诉调试器设定的段大小。如果只想反汇编第一条指令，则可以不给出end，或将end值设置小于start即可。|
|disassemble switch-mode | 在Intel和 AT&T两种汇编风格之间切换 |
|disassemble size = n | 设定调试器执行反汇编命令时使用的段大小，使用0，16，32。值0意思是使用当前的CS段寄存器，默认值是0.|
|set $auto_disassemble = n | 如果n=1，每次停止执行时就反汇编当前的指令，默认值为0。当前CPU上下文的段大小用于反汇编，所以变量"disassemble size"就被忽略了。|
| set disassemble on  | 同'set $auto_disassemble = 1'命令 |
| set disassemble off  | 同'set $auto_disassemble = 0'命令 |

**8. 指令跟踪（Instruction tracing）**

| 命令 | 解释 |
| :----|:--- |
| trace on | 反汇编每一条执行的指令。引起异常的指令都没有真正执行，因此也不会被跟踪 |
|trace off | 关闭指令跟踪功能 |

** 9. 指令功能（Instrumentation） **

要使用Bochs中的指令功能，在编译时就要设置支持它。你应该在"instrument/"目录下构建自定义的指令库到单独目录中。在配置中设置想要使用的指令库，用`--enable-instrumentation`选项指定。默认的库包含了一组桩，和如下的设置等效：
```
./configure [...] --enable-instrumentation
./configure [...] --enable-instrumentation="instrument/stubs"
```
自定义的库要创建一个独立的目录，例如"instrument/myinstrument"，将"instrument/stubs"目录拷贝进去，然后使用如下的指令设定：
```
./configure [...] --enable-instrumentation="instrument/myinstrument"
```

指令命令：
```
  instrument [command]    用[command]调用BX_INSTR_DEBUG_CMD指令回调
```

** 10. 其他命令（Other Commands） **

|   命令 | 解释 |
| :-----|:----- |
| ptime | 打印当前的时间（从开始模拟到现在的ticks）|
|sb delta | 在未来执行中插入一个时间断点“delta”指令("delta"是一个64位的整数，跟着字母"L"，例如1000L |
|sba time | 在时间“time”处插入一个时间断点("time"同上书delta|
| print-stack [num words] | 打印num words个栈顶端的16-bit字。"num words"默认是16。只在保护模式下栈的段寄存器基址是0的情况下才有效。|
|modebp | 触发CPU模式转换断点 |
|ldsym [global] filename [offset] | 从文件'filename'加载符号。如果添加了全局关键字，在所有的上下文中（即使那个上下文没有加载这些符号）符号都可见。Offset(默认值为0) 被添加到每一个符号条目。符号仅仅不加载当前正在执行的上下文。符号文件由零个或多行的"%x %s"格式的组成。 |

单独介绍show 命令
```
show [string]
```
开启显示符号信息
```
  show 		 - 显示当前的模式
  show mode     - 当处理器切换模式时显示
  show int      - 当中断发生时显示
  show call     - 当调用发生时显示
  show ret      - 当返回发生时显示
  show off      - 关闭符号信息
  show dbg-all  - 开启所有的显示标记
  show dbg-none - 关闭所有的显示标记
```

** 11. Bochs调试器GUI（The Bochs debugger gui） **

在Windows和GDK2中可以使用Bochs命令行调试器的图形界面的前端。要或私用GUI调试器，在编译时就需要用默认的调试器开关和参数标记`--enable-debugger-gui`进行配置。例如:

```
  ./configure --enable-debugger --enable-disasm --enable-debugger-gui
```

在运行时，为了使用GUI代替命令行需要使用gui_debug值添加进display_library选项参数中。如下的例子显示了如何用“X”GUI使用GUID调试器。:
```
display_library: x, options="gui_debug"
```

GUI调试器由一个菜单栏，按钮栏和一些子窗口组成，这些子窗口可以显示CPU寄存器，反汇编输出，内存dump和内部的调试器输出。同时也有一个可以输入命令的命令行提示符可用。

GUI调试器的大部分的设置被保存在INI文件中，在调试器下次运行时可以自动恢复它。

#### 调试示例 ####

**1.启动方法**

Windows上关联了EXE，直接双击配置文件（*.bxrc）即可运行。或者另外在一种方法，从开始菜单或桌面启动bochs.exe，然后从加载中选择配置文件，启动即可。

Windows上要调试，则要使用`bochsdbg.exe`打开配置文件，并启动。



**修订历史**

* 2017-09-03 11:21:23		完成文章

By Andy @2017-09-03 11:21:23