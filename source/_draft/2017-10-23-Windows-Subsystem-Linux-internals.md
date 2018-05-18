---
title: Windows WSL子系统（Windows Subsystem for Linux）
date: 2017-09-06 13:52:23
tags:
- Windows
- WSL
categories:
- 笔记
---

windows 10在14393之后开始支持Ubuntu子系统WSL，全称Windows Subsystem for Linux（Windows的Linux子系统）。这个子系统并非Hyper-V启动Ubuntu的虚拟机，而是直接的Local系统，由子系统将Linux调用转成Native的API；这个子系统也不是简单cygwin扩展，而是比cygwin更像Linux操作系统。

之前看过老雷的文章《在调试器里看Windows 10的Linux子系统》，这里贴个链接[http://geek.csdn.net/news/detail/200336](http://geek.csdn.net/news/detail/200336)，这篇文章对于Win10 RS2版本中的WSL Beta版做了分析，有兴趣可以看看这篇文章。

在Win10 RS3版本中WSL转正了，添加删除服务中没有Beta标注了，并且支持多个Linux版本。支持的Linux版本包括Ubuntu，OpenSUSE，SLES等。

这一篇简单介绍一些WSL的开启及其安装方法；在后面将微软官方博客关于WSL的原理做简单介绍。

微软的博客地址如下：

[https://blogs.msdn.microsoft.com/wsl/page/2/](https://blogs.msdn.microsoft.com/wsl/page/2/)

#### WSL及其上Ubuntu安装 ####

从Win10的16215.0开始支持Ubuntu通过应用商店安装，从应用商店中搜索Ubuntu，可以看到如下的Ubuntu APP页面，直接安装即可。

<div align="center">
![图1 开启WSL可选组件](/img/2017-10-23-Windows-Subsystem-Linux-internals-appshop-ubuntu-download.jpg)
</div>
<!--more-->
从页面的描述中可以看到，要安装Ubuntu应用则需要开启WSL可选组件功能，这个是从控制面板，启用或关闭Windows功能，如下图2所示列表下方一点就可以看到如下图的WSL功能组件。勾选组件进行WSL组件安装。

<div align="center">
![图2 开启WSL可选组件](/img/2017-10-23-Windows-Subsystem-Linux-internals-open-WSL-component.jpg)
</div>

如果不事先安装WSL组件，在下载完Ubuntu应用，安装过程中会出现如下图3的错误提示，提示内容也明确指出要先启用WSL功能组件。按照上述方法启用组件后，则再次启动Ubuntu App则正常安装完成。

<div align="center">
![图3 未开启WSL组件安装Ubuntu中错误](/img/2017-10-23-Windows-Subsystem-Linux-internals-ubuntu-install-error.jpg)
</div>

安装成功后，就会进入到Linux系统的Bash。这里与Ubuntu安装过程一样，需要设定用户名和密码，但是在正常使用中并不需要输入，如图4所示进入到bash的提示符中。

<div align="center">
![图4 Ubuntu安装成功](/img/2017-10-23-Windows-Subsystem-Linux-internals-install-success.jpg)
</div>

可以看到正常进入到Bash后，这个所谓的Ubuntu子系统涉及到了好多进程。svchost.exe进程启动init进程，作为Linux下的Init进程；ubuntu.exe会启动wsl.exe，它相当于Windows和Linux子系统之间的启动代理；bash进程则是作为独立进程存在。其中init，bash是真正的Linux进程，其他几个进程都是配合Ubuntu的Windows进程。

#### 初步认识 ####

Windows Subsystem for Linux是一个独立的环境子系统，是NT内核固有机制，之前具有OS/2子系统和POSIX子系统，后来Windows发展中删除了。每个子系统通常有一个子系统服务进程和一个内核态驱动组成，Windows子系统则是CSRSS.exe和Win32k.sys。

WSL子系统则是服务进程LxssManager，服务管理器中可以看到这个服务。LxssManager的核心代码是一个DLL，它位于system32\lxss子目录，这个DLL运行在svchost.exe宿主进程中。内核空间运行的Linux系统驱动有lxss.sys和LxCore.sys两个。

Bash执行后，执行它的命令查看根目录等可以发现它们和Windows目录没什么直接关系，查看mnt目录下，可以发现Windows系统原有的几个磁盘被挂在到了这个目录下，如图5所示。从图上可以看到C，D两个盘被挂在mnt目录下，可以查看C盘根目录下的文件。

<div align="center">
![图5 Bash中找Windows原有磁盘](/img/2017-10-23-Windows-Subsystem-Linux-internals-Bash-C-Logical-partion.jpg)
</div>

那么Ubuntu的根目录位于什么地方呢？老雷的文章中指出它在`C:\Users\Test\AppData\Local\lxss`目录下，但是新的版本已经不位于这个目录下，而是移动到如下代码块所示的这个目录了。这个目录可以认为是Ubuntu App的数据存储目录，也即Windows将Ubuntu完全当作了一个应用程序存在。如图6所示在该目录下存在大量熟悉的Linux的目录，如tmp，mnt，home等。

```
C:\Users\XXXX\AppData\Local\Packages\CanonicalGroupLimited.UbuntuonWindows_79rhkp1fndgsc\LocalState\rootfs
```

<div align="center">
![图5 Bash中找Windows原有磁盘](/img/2017-10-23-Windows-Subsystem-Linux-internals-ubuntu-root-dir.jpg)
</div>

** Ubuntu和Windows的互操作性 **

Windows中调用Linux的程序需要带额外的bash，如下命令行。很明显Windows调用Linux程序需要bash进行过渡，也即要将Windows下的bash启动起来，在其中执行命令。

```
C:\temp>bash -c "ls -la"
```

Linux下调用Windows程序则比较简单，前面说过Windows下的磁盘被挂在到了mnt目录下，那么直接通过mnt目录拼接Windows程序的路径执行即可。这里有一个问题是这种调用需要写Windows程序全名，即不可省略后缀名。对于非可执行的文件比如批处理脚本和命令行命令比如dir等，则需要通过cmd.exe间接执行，如下面cmd.exe执行dir命令。

```
$/mnt/c/Windows/System32/notepad.exe

$ /mnt/c/Windows/System32/cmd.exe /C dir
```

#### WSL子系统 ####

** Windows子系统历史 **

从起始时，Windows NT就设计成了支持多个子系统的形式，可以设计其他的子系统（像Win32子系统一样）来给不同平台的应用程序提供编程接口，而不需涉及内核中的实现细节。所以NT发布之初就支持OS/2，Win32，POSIX子系统。

早期的子系统都是在用户层通过提供特定模块实现，这些模块提供POSIX或OS/2等的应用调用的API。所有的应用程序都是PE/COFF执行格式，提供一组库和服务来实现子系统的API，NTDLL执行NT系统调用。当应用程序启动时，则由Loader根据执行文件的头内容，调起正确的子系统来满足应用程序的依赖。

后来的SUA替代了POSIX层，但是SUA依然是用户模式的组件来满足POSIX程序调用。SUA意图鼓励人们将应用程序在不需要大幅修改情况下移植到Windows上。但是由于是Ring3层实现，它对于内核模式的系统调用很难有语义和性能上的改进。这种模式依旧依赖于程序重新编译，所以持续的移植于维护负担太重。

WSL则是一组可以让Linux ELF64二进制直接在Windows上执行的组件集合。它包含了用户层和内核层的组件，主要有如下三部分组成：

* 用户模式会话管理器服务控制Linux实例的生命周期
* Pico进程支持驱动（lxss.sys,lxcore.sys），通过转换Linux系统调用模拟Linux内核
* Pico进程则承载了未修改用户模式Linux程序

在用户模式Linux二进制和Windows内核组件之间的部分就是魔法发生的地方，这层转换使得Pico进程中的Linux二进制程序的系统调用直接进入到Windows内核中。其中lxss.sys和lxcore.sys两个驱动完成了从Linux调用到NT API的转换，完美地模拟了Linux内核。基本的原理如下图6所示。

<div align="center">
![图6 Bash中找Windows原有磁盘](/img/2017-10-23-Windows-Subsystem-Linux-internals-lxss-diagram.jpg)
</div>

LXSS管理器服务 —— LXSS管理器服务是Linux子系统驱动的代理，它是Bash.exe调用Linux二进制的途径。负责同步安装和卸载，只允许一个进程执行安装和卸载操作，在安装或卸载执行挂起时会阻塞Linux进程启动。由特定的用户启动的Linux进程都会放入同一个Linux实例中，这个实例仅仅是一个数据结构，保存和追踪了LX进程，线程，和运行时状态。第一次要求启动Linux二进制程序时床架Linux实例。最后一个NT客户端关闭，Linux实例结束。这个实例会包含所有在实例中启动的Linux进程。

Pico进程 —— 作为Drawbridge项目的一部分，Windows内核引入了Pico进程和驱动的概念。Pico进程是OS的进程，但是它并不保存与子系统相关OS服务的数据，比如Win32进程的PEB。此外，Pico进程的系统调用和用户异常都被传递给对应的驱动处理。Pico进程和驱动提供了WSL的基础，通过将可执行的ELF文件加载到Pico进程地址空间，并且在Linux兼容的系统调用层上运行他们。

系统调用 —— WSL通过在Windows NT内核之上虚拟一个Linux内核接口层来执行未修改过的Linux ELF二进制文件。Windows和Linux内核都导出了几百个系统调用供用户层程序使用，但是他们的语义不同，也不会直接相互兼容。比如Linux内核中会到处fork，open，kill等函数，但是NT内核却是等价的NtCreateProcess，NtOpenFile，NtTerminaterProcess。WSL中的内核驱动lxss.sys和lxcore.sys负责处理Linux系统调用请求，并且与Windows NT内核配合完成功能。驱动中并不包含来自Linux内核的代码，完全重新设计的与Linux内核接口兼容的层。真实的Linux环境中，系统调用从用户模式的执行文件上发起时，它是由Liinux内核处理的。在WSL上，ELF进行系统调用时由Win NT内核转发请求给lxcore.sys；lxcore.sys将Linux系统调用转换为等价的Win NT调用，lxcore.sys需要做大量的提升与改进工作以转换成标准NT调用。所以这里就不需要将Windows内核模式驱动映射为直接向Linux程序提供服务。比如Linux中的fork()调用，Windows中并没有直接等价的调用存在。所以当fork被调用时，lxcore.sys先做一些初始化工作来准备拷贝进程，然后调用Win NT的内核创建进程，为新建的进程完全拷贝父进程其他附带的数据。

文件系统 —— 支持WSL的文件系统满足两个目标即可，一方面提供完整的完美的Linux文件系统形式；另外一方面允许和Windows中的磁盘和文件的互操作性。WSL则是提供了一组虚拟的文件系统来模拟真实Linux内核。两个文件系统VolFs和DriveFs用于提供用户系统访问文件。

VolFs —— 它是一个提供了完整的Linux文件系统特征支持的文件系统，包含了通过chmod和chroot可修改的Linux文件权限，指向其他文件的符号链接，Windows文件名不支持的非法字符的文件名支持，大小写敏感等。包括Linux系统目录，应用程序目录（/etc，/usr等）和用户home目录的文件夹都是用了VolFs。Windows应用程序和VolFs中文件的互操作性并没有支持。

DirveFs —— 一个用于和Windows进行互操作的文件系统。它要求所有的文件名字在Windows下必须合法，使用Windows安全性模型，并不支持Linux文件系统所有特征。文件时大小写敏感，用户不能创建大小写不同的相同字符串名字文件。所有固定的Windows卷被挂载到/mnt/c，/mnt/d等，通过这种方法可以访问所有的Windows文件。这样用户可以使用喜欢的VS Code编辑文件，然后在Bash中用开源软件进行操作。

#### Pico进程概述 ####

** Drawbridge **

Pico源于 Drawbridge项目，目标是实现一个轻量级的方法在隔离的环境中运行应用程序，这个隔离环境中带有不同于主机系统的OS依赖（比如XP的应用程序运行在Win10上面）。通常将应用程序和OS运行在虚拟机上即可，但是这会使得资源开销很大。Drawbridge项目的目标是在主机系统（比如Win10）中的一个进程用户层地址空间中运行目标应用和OS（比如XP上的程序）。这和VM比起来开销就小很多了，并且这种方法可以在单台主机（比如Win10）上运行更大量的应用程序，同时提供相同的隔离和兼容保证。可以到Drawbridge项目主页看一下如何实现这样的目标。

在Drawbridge中，library OS是应用的目标OS。为了支持这个区别于主机OS的library OS运行在用户模式，MSR的小伙伴需要主机OS不再管理该进程中的用户地址空间。他们叫这样类型的进程为pico process，意味着在主机OS上它是正常的系统进程的小号版。内核模式的驱动负责支持pico进程，并且充当主机OS和用户层library OS的代理。

当然，比较完善的支持需要核心的Windows内核修改，为pico进程增加大量功能，这样才能达到向公众发布。MSR团队在2013年就将这个想法就提给了内核组，并且都同意作为一个重大的功能添加进内核。事实上，这个方法依赖于与先前的内部讨论的要将子系统重新放回内核以支持将来Windows架构变化的长期策略。对于pico进程的第一支持出现在Win8.1中，但是受限于Drawbridge。Pico进程的支持后来被扩展为其他的Windows功能。

** 最小进程 **

在开始实现官方对Pico进程支持时，决定将抽象层分为两层：

* Minimal Process：这是一个基本类型的进程。如果一个进程标记为minimal进程，那么它等于告诉系统的其余部分不要再管理这个进程了。从Windows内核角度看，它就是一个简单的具有空白的用户模式地址空间的进程。
* Pico process：这是一个minimal进程，它由pico提供者内核模式驱动来管理空白的用户模式地址空间

<div align="center">
![图7 Pico进程](/img/2017-10-23-Windows-Subsystem-Linux-internals-picoProcess.png)
</div>

与传统NT进程不同，创建minimal进程时不会修改该进程的用户空间，不创建线程。内核中设置用户地址空间的位置都被修改跳过：

1. 用户模式二进制 ntdll.dll不会映射进进程
2. 进程环境块PEB不会创建
3. 没有初始线程创建；并且在创建线程时，线程环境块TEB也不会自动创建
4. 共享用户数据区块也不会映射到进程，这块内存正常会被以只读模式映射到用户模式，用于读取功能的系统范围的信息
5. 几个假设进程必须有PEB或TEB的地方被处理为不需要这些值

尽管Windows内核不会显示第管理minimal进程，但是它依然提供所有OS潜在支持的功能，比如线程调度，内存管理等。

这里会好奇，为什么分开说minimal和pico进程？空白minimal进程的想法似乎只对它自己有用，与pico进程需要的相关支持分开。尽管现在并没有什么特定的东西是2013年出现的，但是在Win10中还是有几个场景是直接使用minimal process的：

* Memory Compression: 内存压缩是一个Windows功能，它会压缩没有在使用的内存，以保证内存中可以存储尽可能多的数据。减少了从pagefile中读取和写入的数据量，由此提高Windows性能。Windows内存压缩使用了minimal进程的用户模式地址空间
* Virtualization based Security（VBS）: 使用潜在的虚拟化能力，VBS将重要的用户模式进程的用户地址空间与OS其他部分隔离开，防止这些进程被其他进程破坏，甚至是被内核或内核模式的驱动。任务管理器就是这么一个minimal进程，使用了VBS。

** Pico进程和Provider **

pico进程是一个与pico provider内核驱动相关联的minimal进程。pico提供者需要提供进程用户模式部分所需要的整个内核接口。Windows内核会将所有的系统调用和异常处理传递给pico提供者来处理。这使得pico提供者模拟了一个不同的用户/内核联系，而这个用户/内核联系区别于Windows现在已经提供的环境。

在引导的早期，pico驱动向Windows内核注册为pico提供者，内核和pico提供者会交换一组pico提供者所需要的接口描述表。例如当内核分发一个用户模式的系统调用或异常时，pico提供者提供了要调用的函数指针，内核提供者提供创建pico进程和线程函数指针。

无论pico提供者给用户模式暴露了什么样的行为和抽象，它最终还是要依赖于Wndows内核来做线程调度，内存管理和I/O管理的功能。当然了，Windows内核也需要更新部分功能用于支持新的场景。

** Windows内核的修改 **

后面的文章中会详细介绍Windows内核的修改，但是这里做一个列举：

1. 改进fork支持：Windows内核已经支持fork很长时间了，可以追溯与POSIX和SFU应用支持的版本。但是它并没有暴露给Win32编程模式现在改进了fork的实现，用于满足WSL工作中的部分新的需求。

2. 细粒度的内存管理：Windows通常以64K块管理用户模式地址空间，但是对于pico进程更新到可以管理单页4Kb力度内存.

3. 文件名称大小写敏感： Windows内核和NTFS文件系统很早就支持大小写敏感的文件名称，但是它模式是关闭的，并且没有导出给Win32编程模型。现在允许单个线程有选择地开启大小写敏感的操作，用于支持更广范围的WSL场景。

#### WSL系统调用 ####

通过在Windows NT之上模拟Linux内核，WSL可以执行不经任何修改的ELF64二进制文件。它导出的内核接口之一就是系统调用（syscalls）。这部分介绍在WSL中syscalls如何处理。

** 概述 **

系统调用是内核提供的一套服务，用户模式的程序可以调用它们来访问设备或其他的特权操作。作为例子，nodejs web服务器需要访问磁盘上的文件，处理网络请求，创建进程/线程和其他的一些操作。

系统调用如何处理依赖于操作系统和处理器架构，但是最终它会形成一个用户模式和内核模式交互的层，叫做应用程序二进制接口（ABI）。大多数情况下，做系统调用可以分为三个步骤：

1. 安排参数 - 用户模式将参数和调用号放在ABI指定的位置
2. 特殊指令 - 用户模式使用一个特殊的内核指令转换到内核模式以进行系统函数调用
3. 处理返回 - 在系统调用完成后，内核使用特殊的处理器指令返回到用户模式，用户模式校验返回值

尽管Linux内核和Windows内核的系统调用都遵循上述步骤，但是他们在ABI上是不同的，因此也不相互兼容。即使是Linux和Windows NT内核有相同的ABI，他们导出的系统调用不同，一般情况下并非一一对应。例如Linux内核包含了fork，open，kill等导出函数，而Windows NT内核中与其兼容的函数为NtCreateProcess，NtOpenFile和NtTerminateProcess等。如下部分会深入如何在不同环境下作系统调用的细节处理。

** Linux x86_64上的系统调用机制 **

Linux X86_64上的系统调用约定遵循System V x86_64 ABI定义。举个例子，从C程序中调用getdents64系统调用的方法要使用系统调用包装器：

```
Result = syscall(_NR_getdents64, Fd, Buffer, szieof(Buffer));
```

讨论一下系统调用的约定，它很容易按照前面的3步转换成汇编伪代码如下：

```
1. mov rax, __NR_getdents64
2. mov rdi, Fd
3. mov rsi, Buffer
4. mov rdx, sizeof(Buffer)
5. syscall
6. cmp rax, 0xFFFFFFFFFFFFF001
```

首先看一下参数准备，1-4步根据调用约定将调用参数放入指定寄存器，第5步指定的系统调用指令执行后转向内核模式，最后第6步在用户模式检验返回值。

在5和6步之间是Linux内核处理getdents系统调用的地方。当syscall指令调用后，处理器执行一个环转换，进入内核模式，开始在内核模式下的特定函数中执行，通常是系统调用分发函数。在机器引导时作为内核初始化一部分，内核配置处理器在系统调用指令执行时去执行系统调用分发器。Linux内核在系统调用分发器中做的第一件事是保存用户模式线程寄存器状态，这样当系统调用返回到用户模式是可以继续执行。然后它检查RAX寄存器来确定调用那个系统调用，然后将寄存器的值作为参数传递给getdents函数。一旦getdents完成，内核恢复用户模式寄存器状态，更新rax值保存syscall的返回值，使用另外一个特殊指令（sysret或iretq）通知处理器执行环切换，回到用户模式。

** NT amd64上的系统调用机制 **

NT x64上的调用习惯依据X64的调用约定（https://msdn.microsoft.com/en-us/library/7kcdt6fy.aspx）。看一个例子，当NtQueryDirectoryFile调用时，有如下的最简单的形式，与前面的getdents相比更简单：

```
Status = NtQueryDirectoryFile(Foo, Bar, Baz);
```

真实的NtQueryDirectoryFile API有11个参数，这些参数在这里描述会让例子更加难以理解，它需要将参数压栈。现在看一下按照前面getdents的3个步骤如何一步一步处理为汇编代码，与Linux做个对比：

|Step |  Getdents                   | NtQueryDirectoryFile          |
|-----|-----------------------------|-------------------------------|
|1    | mov rax, __NR_getdents64    | mov rax, #NtQueryDirectoryFile|
|2    | mov rdi, Fd                 | mov rcx, Foo                  |
|3    | mov rsi, Buffer             | mov rdx, Bar                  |
|4    | mov rdx, sizeof(Buffer)     | mov r8, Baz                   |
|5    | syscall                     | syscall                       |
|6    | cmp rax, 0xFFFFFFFFFFFFF001 | test eax, eax                 |

对于准备参数那一步，NT也用rax寄存器保存系统调用号，但是在传递系统调用参数时用的寄存器不同，这主要是因为他们的ABI不同。对于特殊指令一步，可以看到syscall也用在amd64的系统调用中了，这是因为这个指令是x64做系统调用的首选方法。对于最后一步校验返回值，代码有一点不同，这是因为NTSTATUS失败是负数，而Linux失败码会落在特定范围内。

和Linux的内核相似，5和6步之间是NT内核处理NtQueryDirectoryFile系统调用的地方。这一部分与Linux相同，除了一点小例外，即用户模式传递给系统调用的参数保存寄存器不同。NT系统调用依据X64的调用约定，内核不需要保存易失性的寄存器，这些工作是由编译器注入指令完成的易失性寄存器的保存。

** WSL的系统调用机制 **

当看过了NT和Linux的系统调用对比后，发现它们在调用约定上只有少许不同。下面看一下WSL上的系统调用如何做。

WSL上的系统调用的调用约定依据上面描述的Linux习惯，因为进行系统调用的是没有修改过的ELF64二进制文件。WSL包含了内核模式pico驱动（lxss.sys和lxcore.sys），它们负责处理Linux系统调用请求，并与NT内核合作完成功能。驱动没有任何Linux内核代码，但是它是一个Linux内核兼容接口的完整洁净的实现。依据原来的getdents的例子，当syscall指令执行时，NT内核可以通过进程结构中的状态检测到请求来自于pico进程。因为NT内核不知道如何处理来自pico进程的syscall，那么它就保存寄存器状态，然后将调用请求转发到pico驱动。pico驱动通过检查EAX寄存器的值确定调用哪一个系统调用，然后按照Linux调用约定设定的寄存器传递参数。在pico驱动处理了syscall，它会返回到NT内核，NT保存了寄存器状态，然后将返回值放入rax中，调用sysret/iretq指令返回到用户模式。

<div align="center">
![图8 Pico系统调用图示](/img/2017-10-23-Windows-Subsystem-Linux-internals-syscall_graphic.png)
</div>

** WSL调用例子 **

对于有些情况，lxss.sys将Linux的syscall可以直接转换成等价的Windows NT调用。但是另外一些调用，并不能直接转换成NT的系统调用，需要lxss.sys做大量转换工作。这些Linux系统调用会由lxss.sys直接实现。这一部分讨论几个WSL实现的系统调用，以及它们与NT内核的交互。

比如sched_yield系统调用可以直接映射为NT的系统调用。当Linux程序调用sched_yield时，NT内核将请求转交给lxss.sys，然后它直接将请求转发给ZwYieldExecution，它的语义和sched_yield相同。

尽管sched_yield是一个正好可以映射为已存的NT syscall的例子，但是并不是所有的系统调用都有类似的属性，在NT内核上有对应的类似函数。Linux的管道的语义就不同于NT的管道，WSL不能直接使用NT管道来模拟Linux管道的完整功能。WSL直接实现了Linux管道，但是仍然使用NT功功能用于同步原语和数据结构。

最后Linux的Fork在NT内核中没有等价的调用。WSL中发起fork调用时，lxss.sys会做一些准备拷贝进程的初始化工作。然后调用NT API创建进程，创建线程。最后它做一些额外工作，将进程数据完整拷贝一下，恢复新进程的执行。


#### WSL文件系统支持 ####

** 介绍 **

WSL的主要目标就是让用户可以在Linux中处理他们的文件，与Windows机器上的已有文件有完全的互操作性。虚拟机必须使用网络共享或其他的解决方案在主机和客户机之间共享文件，而WSL与此完全不同。WSL可以直接访问Windows磁盘上的文件，并且易于交互。

Windows文件系统和Linux文件系统有本质的不同，这一篇文章就是讲解了WSL如何连接两个世界。

** Linux上的文件系统 **

Linux通过VFS来抽象文件爱你系统操作，VFS为用户模式程序提供了与文件系统交互的接口（通过系统调用，如open，read，chmod，stat等），另一方面文件系统需要实现VFS的这些接口。这种方式就允许多文件系统共存，提供相同的操作和语义，使用VFS可以将所有的文件系统向用户提供统一的接口。

在VFS这个名字空间中，文件系统被挂在不同的目录。例如，典型的Linux系统上，硬盘被挂在到root目录，即“/”。目录例如/dev，/proc，/sys，和/mnt/cdrom都会挂在不同的文件系统，这些文件系统可能在不同的设备上。在Linux使用的文件系统的例子包括ext4，rfs，FAT和其他的一些格式。

VFS通过使用各种数据结构，例如inodes，目录条目和文件，以及文件系统必须实现的回调等来实现各种文件系统操作相关的系统调用。

Inodes —— inode是VFS中使用的关键数据结构。它代表了文件系统对象，例如普通的文件，目录，符号链接等。inode包含了文件类型，大小，权限，最后修改时间和其他的属性等信息。对于许多普通的Linux磁盘文件系统，例如ext4，磁盘上的数据结构用于代表文件元数据，直接对应到Linux内核中使用的inode结构。尽管一个inode代表一个文件，但是它不代表一个文件名。一个文件或许有多个名字，或硬链接，但是只有一个inode。文件系统提供了一个到VFS的查找回调，它用于依据父节点和孩子名字获取一个特定文件的inode。文件系统必须实现其他的一些inode操作，比如chmod，stat，open等。

Directory entries —— VFS使用一个目录条目缓存代表文件系统名字空间。目录条目仅仅存在于内存中，包含一个指向文件的inode指针。例如，如果有路径/home/user/foo，就有目录条目home，user，和foo。每一个都有一个指针指向inode。目录条目用于快速查询，但是如果条目没有在缓存中，就会使用inode查找操作从文件系统中获取这个inode，创建一个新的目录条目。

File Objects —— 如果打开了一个inode，就会创建一个文件对象，它用来保存一些文件信息，例如文件的偏移，文件打开是用于读，写还是读写。文件系统必须提供数个文件操作，例如read，write，sync等。

File descriptors —— 应用程序通过文件描述符引用文件对象。使用数字值，对于一个进程来说唯一，它们用来引用进程打开的文件。文件描述符也可以应用其他类型对象，只要这些对象提供了和文件类似的接口，例如ttys，sockets和pipes。多个文件描述符可以引用相同文件对象，即通过dup系统调用进行复制。

Special file types —— 除了普通的文件和目录，Linux支持很多额外的文件类型。这些包括设备文件，FIFO，sockets，和符号链接。这些文件会影响到路径解析。Symbolic links是一种特殊文件，它们指向一个不同的文件或路径，这种指向是由VFS无缝处理。如果你打开的路径/foo/bar/baz，bar是一个符号链接指向/zed，那么真实打开的文件是/zed/baz。类似地，目录或许被当作另外一个文件系统的挂载点。这种情况下，如果一个路径穿过了这个目录，那么接下来所有的inode操作会直接进入新的文件系统。

特殊和伪文件系统 —— Linux使用一些不从磁盘上读文件的文件爱你系统。TmpFs是用作临时，内存中文件系统，它其中的内容不会持久。ProcFs和SysFs都提供访问进程，设备和驱动内核信息的途径。这些文件系统都没有磁盘，网络或其他设备关联，它们都是由内核虚拟。

** Windows上的文件系统 **

Windows将所有的系统资源抽象为对象。这不仅仅包括文件，也包括线程，共享内存区块和定时器。所有打开文件的请求最终都会通过NT内核的对象管理器，它会将请求通过I/O管理器传递给文件系统驱动。Windows中实现的文件系统驱动接口更加通过，只有少量要求。例如，Windows中没有普通inode结构或其他类似结构，也没有目录条目。相反，文件系统驱动，例如ntfs.sys来负责解析路径，打开文件对象。

Windows中的文件系统通常挂在到驱动符，如C:，D:等，尽管它们在其他的文件系统中也可以被挂在到目录中。这些驱动符号其实就是Win32的组成部分，并不是对象管理器直接处理的东西。对象管理器则维护了一个名字空间，类似于Linux文件系统的名字空间，以\符号为根，系统卷则通过设备对象路径，例如\Device\HarddiskVolume1来代表。

当使用路径"C:\foo\bar"路径打开文件时，Win32 CreateFile将路径转换为NT路径形式，即"\DosDevice\C:\foo\bar"，其中"\DosDevice\C:"是一个符号链接，它指向了例如\Device\HarddiskVolume4。因此真实的完整路径其实是"\Device\HarddiskVolume4\foo\bar"。对象管理器解析路径的每一个组成部分，类似于VFS在Linux中的做法，直到它遇到设备对象。这时候，它将请求转发到I/O管理器，它会创建一个带有剩余路径的IRP，然后IRP就被转发到设备的文件系统驱动上了。

文件对象 —— 打开一个文件时，对象管理器就会为它创建一个文件对象。与文件描述符不同，对象管理器提供了指向文件对象的句柄。句柄可以指向任何对象管理器对象，不单是文件。当调用系统调用NtReadFile时（通常通过Win32 ReadFile函数），I/O管理器再一次创建一个IRP，并发送到文件系统驱动区执行文件对象的请求。因为在NT上没有inodes或其他类似概念，所以多数的文件操作都需要文件对象。

重解析点 —— Windows仅支持两种文件类型：普通文件和目录。文件和目录都可以成为重解析点，解析点是特殊文件，有一个固定的头和一块任意数据。头中包含了一个表示重解析点类型的标识，它只能由文件爱你系统过滤器驱动处理，对于内置的重解析点类型，则只有I/O管理器自己可以解析。

大小写敏感 —— 与Linux不同，Windows文件系统默认是大小写保留，但是并不大小写敏感。实际上，Windows和NTFS支持大小写敏感，只是默认并不开启。

** WSL中的文件系统 **

WSL必须将各种Linux文件系统操作转换成NT内核操作，WSL也要提供一个存放Linux系统文件的地方，并且可以提供Linux文件系统的功能要求，比如Linux权限，符号链接和特殊的文件比如FIFO等。在这还需要提供访问Windows卷的方法；并且要提供特殊的文件系统比如ProcFs。

WSL有一个VFS组件，它可以模拟Linux上的VFS，整个架构如下图所示。

<div align="center">
![图9 WSL文件系统图示](C:\Users\Administrator\Desktop\Windows-WSL\2017-10-23-Windows-Subsystem-Linux-internals-file-system-graphic.png)
</div>

应用程序调用系统调用时，这是由系统调用层来处理，它定义了各种内核入口，比如open，read，chmod，stat等。这些文件相关的系统调用，在系统调用层功能很少，它只是将调用转给VFS。

对于使用路径的操作，像open，stat等，VFS使用目录条目缓存解析路径。如果缓存中没有对应条目存在，VFS调用文件系统的几个插件之一创建一个节点缓存该条目。这些插件提供了像查找，chmod等类似Linux中inode的操作。当打开一个文件时，VFS使用文件系统的inode打开操作创建文件对象，然后返回该对象的描述符。
操作文件描述符的系统调用，如read，write或sync，调用文件系统定义的系统操作。这个系统就和Linux
的行为非常像，所以WSL可以支持相同的语义。

VFS定义了几个文件系统插件：VolFs和DrvFs用来表示磁盘上的文件，其他的则是内存中的文件系统TmpFs，和伪文件系统如ProcFs，SysFs，和CgroupFs。VolFs和DrvFs是Linux文件系统与windows文件系统相结合的地方。它们是WSL如何和磁盘上的文件交互，以及另外两个目的：VolFs设计提供完整的Linux文件系统特征，DrvFs设计提供与Windows的交互。下面详细看一下这些文件系统。

** VolFs **

WSL使用的主要的文件系统就是VolFs。它用于存储Linux系统文件，以及Linux下Home目录的内容。VolFs支持Linux VFS提供的大部分功能，包括Linux权限，符号链接，FIFO，sockets以及设备文件。

VolFs用于挂载VFS根目录，使用%LocalAppData%\lxss\rootfs目录作为后备存储。此外，少数几个额外的VolFs挂载点存在，尤其/root和/home使用%LocalAppData%\lxss\root和%LocalAppData%\lxss\home目录分别进行挂在。使用单独目录进行挂载的原因是，当你卸载了WSL，home目录默认是不会删除的，所以任何存在这个目录的个人文件都会保存下来。

注意所有这些挂载点使用Windows用户文件夹下的目录。每一个Windows用户使用它们自己的WSL环境，并且由此有Linux的root权限，安装应用程序不会影响到其他的Windows用户。

Inode和文件对象 —— 因为Windows并没有相关的inode概念，VolFs必须在inode中保存一个指向Windows文件对象的句柄。当VFS使用查找回调请求一个新的inode时，VolFs使用父inode的句柄，以及孩子的名字执行相关的打开和获取新的inode使用的句柄。这些句柄打开时并没有任何的读写访问权限，仅仅能用于元数据请求。当文件打开时，VolFs创建一个Linux文件对象指向inode。他也重新使用请求的读写权限打开inode的文件句柄，将新的文件句柄保存到文件对象中。这个句柄也用于满足文件操作，像读和谐。

模拟Linux特征 —— 像上面讨论的一样，Linux在几个方面都与Windows的文件系统不同，VolFs必须提供几种Linux功能支持，而这些功能Windows并不支持。大小写敏感是由Windows自己支持。前面提到过，Windows和NTFS实际上支持大小写操作，所以VolFs简单请求对象管理器将路径按照大小写敏感方式处理，从而忽略全局注册表键值控制。

Linux支持几乎所有的字符作为合法的文件名。NT则有比较多的限制，一些字符根本不允许使用，而其他的则具有特殊的含义（比如“:”指示一个可变的数据流）。为了支持所有的Linux文件名，VolFs则转换文件名中的非法字符。

Linux在解链接和重命名上有不同的语义。特别的，即使有打开文件描述符执行文件，它也可以被解链接。类似地，即使文件被打开，它也可以被按照目标重命名操作重写。在Windows中，如果文件请求被删除，那么它就会在最后一个打开句柄被关闭后被删除，但是会保持文件系统中名字可见直到被删除。为了支持Linux的unlink语义，VolFs会在请求删除之前，将解链接文件重名到一个隐藏临时目录。

Linux中的inodes有一组数字属性，这个在Windows中不存在，这组属性包含了它的拥有者，组和文件模式，以及其他的信息。这些属性存储在NTFS存储在与磁盘上文件相关的扩展属性中，如下的信息存储在扩展属性中：

* Mode： 这个包括文件爱你类型（普通，符号，FIFO等）和文件的权限位
* Owner： 拥有该文件的Linux用户和组的用户的ID，组ID
* Device ID：对于设备文件，设备主号和辅号。注意WSL不允许用户在VolFs上创建设备文件
* File times：Linux上的文件访问，修改和变化时间使用了和Windows上不同的格式，所以这些也存在EA中

此外，如果文件有任何文件的信息，这些都被存储在可选的文件数据流中了。注意WSL当前不允许用户修改文件的容量。剩下的inode属性，比如inode号和文件大小都是从NTFS保存的信息中衍生出来。

与Windows的互操作性 —— 尽管VolFs文件被存储在上述指出的Windows目录中，与Windows的互操作并不支持。如果一个文件从Windows中被添加到这些目录，它缺少VolFs目录需要的扩展属性，因此VolFs并不知道如何处理这些文件，只是简单忽略它。许多编辑器在保存已存文件时也会剥离掉EA数据，那么这些文件在WSL中就不可用了。此外，因为VFS缓存目录条目，从Windows中对这些目录的任意修改并不会马上反应出来。

** DrvFs **

为了使用于Windows的互操作性，WSL使用DrvFs文件系统。WSL自动挂在所有支持的文件系统的固定驱动器到/mnt目录，例如/mnt/c，/mnt/d等。当前进NTFS和ReFs卷是支持的。

DrvFs与VolFs使用类似的操作方式。当创建inodes和文件对象时，也会打开Windows文件并保存其句柄。但是与VolFs对比，DrvFs则要遵守Windows的规则（有一点点不一样，看下面）。这里使用Windows权限许可，仅仅合法的NTFS文件允许使用，特殊文件类型如FIFO和sockets并不支持。

DrvFs权限 —— Linux通常使用一个简单的权限模型，文件允许文件拥有者，组或所有人都可以读，写或执行访问。Windows则使用ACL来指定每一个文件和目录的复杂访问规则。（Linux也可以支持ACL，但是现在的WSL并不支持）。

当在DrvFs中打开一个文件时，Windows权限是以基于执行bash.exe的用户的令牌来使用的。因此为了访问C:\Windows下的文件，在bash环境下使用sudo命令是不足以达到目的，sudo可以在WSL下赋予你根权限，但是改变不了你的Windows用户令牌。要达到这样的目的，需要以提升权限的形式启动bash.exe。

为了给用户一些关于他们在一些文件上所具有的权限，DrvFs会校验用户是否有操作某个文件的权限，然后将它们转换为read/write/execute位，这些值在执行“ls -l”命令时可见。然后，这种映射并不总是可以得到一对一的对应。例如，Windows在一些目录下将创建文件和创建子目录的权限分割开了。如果用户具有这两个权限之一，DrvFs就会显示在该目录有些权限，尽管一些操作可能遇到访问拒绝而失败。

依据bash.exe是以权限提升启动与否，你访问一个文件的结果可能不同。在DrvFs中显示的文件权限在提升权限的Bash.exe实例和未提权的实例之间也有区别。当计算了对一个文件的有效访问权限时，DrvFs仅将只读属性读到账户中。Windows中带有只读属性标记的文件会在WSL中显示为没有写权限。chmod可以用于设置只读属性（通过移除所有的写权限，即"chmod a-w some_file"）或清除它（通过添加任何写权限，即“chmod u+w some_file”）。这个行为类似于Linux中的CIFS文件系统，它用于访问Windows的SMB共享。

大小写敏感 —— 因为在Wnidows和NTFS中支持大小写敏感，DrvFs支持大小写敏感文件。这意味着在DrvFs中可以创建两个一样名字，仅仅大小写不同。注意许多Windows此应用程序可能无法处理这种情况，可能不能够打开它们。大小写敏感在卷的根目录是被关闭的，但是它可以在其他任何地方开启。因此为了使用大小写敏感文件，不要试图在/mnt/c等根卷下创建文件，但是在其他的目录下都可以创建。

符号链接 —— 尽管NT支持符号链接，但是WSL中并不能依赖这个功能，因为WSL中创建的符号链接可能指向/proc，而这个目录在Windows下并没有意义。此外，NT要求管理员权限来创建符号链接，因此必须寻找其他的解决方案。

不像VolFs，在DrvFs中并不能依赖于EA来指示一个文件是符号链接。相反，WSL使用新的重解析点类型来代表符号链接。作为结果，这些链接只能在WSL中起作用，并不能被其他的Windows组件解析，例如File Explorer或cmd.exe。注意因为ReFS缺少对重解析点的支持，它也不能支持WSL中的符号链接。在WSL中NTFS现在有全符号链接支持。

与Windows的交互性 —— 不同与VolFs，DrvFs并没有保存任何额外的信息，相反，所有的inode属性是从NT中衍生出来的，通过查询文件属性，有效权限和其他的信息来完成。DrvFs关掉了目录条目缓存，保证它总是正确显示，及时更新信息，即使Windows进程修改了目录的内容。所以，当DrvFs打开文件操作时，就对Windows进程的操作没有限制。DrvFs也使用Windows的文件删除语义，因此如果有打开的文件描述符或Windows进程中的句柄，这个文件就不能解链接。

** ProcFs和SysFs **

和Linux中类似，这些特殊的文件系统并不现实存储在磁盘上的文件，而是保存了内核中关于进程，线程和设备的信息。这些文件动态生成。在一些情况下，这些文件的信息是完全保存在lxcore.sys驱动中的。在另外一些情况下，例如进程的CPU的使用情况，WSL则需要从NT内核中请求这些信息。但是，并没有与Windows文件系统的交互。

** 结论 **

WSL使用VolFs通过模拟完整的Linux行为提供了对Windows文件的访问，满足了内部的Linux文件系统，通过DrvFs提供完全的Windows驱动和文件的访问。本篇写作时，DrvFs已经提供了一些Linux文件系统的功能，例如大小写敏感，符号链接，同时仍然支持与Windows的交互性。未来会继续改进对Linux文件系统功能的支持，不仅仅是在VolFs，也改进DrvFs的功能。


#### Windows和Ubuntu的互操作性 ####

网上有一篇翻译文章，将连接贴到这里[【译】Windows和Ubuntu的互操作性](https://zhuanlan.zhihu.com/p/25432384)

** 介绍 **

这一篇文章介绍WSL所要求的最主要功能的设计与实现，即与Win32应用程序的交互性。自从早些时候在预览版中功能可用时，用户就已经指出了问题，并且给出了他们想要使用WSL的关键场景。有这么多人主动使用WSL，并提供反馈是非常好的体验。我们欢迎所有的bug提交和建议，它们已经是改进Windows未来版本的物价之宝。

互操作性的定义，广义上讲是在同一个shell中混合使用NT和Linux的二进制可执行文件。例如在bash.exe中导航到文件系统，并从当前目录启动NT的图文编辑器。另外一方面就是NT和Linux二进制之间相互重定向输入输出数据。例如使用Linux的grep程序过滤Windows命令行输出。这种互操作性使得WSL可以完成之前看来不可能的功能：从喜欢的Shell中无缝地运行喜欢的Windows和Linux工具。

** WSL中启动进程 **

下面的图描述了各种不同的组件配合起来。在启动进程时，只要很少几个状态需要共享。第一个是当前的工作目录，第二个是代表输出从新进程所去往地方（系统）的标准输入输出和错误的对象。例如，管道输出到另外一个线程，重定向输出到一个文件，或者确保内容出现在正确控制台环境中。如下的部分会详细介绍如何实现共享。

<div align="center">
![图10 互操作性示意图](/img/2017-10-23-Windows-Subsystem-Linux-internals-interop_diagram.png)
</div>

启动/bin/bash进程
```
1. bash.exe使用内部的CreateLxProcess COM API与LXSS管理器服务交互
2. LXSS管理器服务将当前工作目录转换到WSL路径，将bash.exe提供的代表标准输入输出和错误等NT句柄参数打包
3. LXSS管理器通过LxBus发送一个消息到自定义的/init守护进程
4. init守护进程解析消息，解包标准输入输出，错误和控制台等参数，将它们设置到文件描述符的0，1，2和创建tty文件描述符
5. /init forks 和execs /bin/bash二进制文件，回去继续坚挺下一个进程创建消息
```

从WSL实例启动一个NT进程
```
6. /bin/bash forks和execs NT二进制文件，LxCore.sys找到处理Windows PE映像的二进制格式描述符注册信息(binfmt_misc)，调用对应的解析器
7. /init进程（作为binfmt_misc解析器执行）转换当前的工作目录并组织描述符0，1和2（代表了标准输入，输出，错误）参数
8. /init 发送创建NT进程消息到bash.exe
9. bash.exe解包文件描述符，它们代表了NT句柄，然后以代表了标准输入，输出和错误的NT句柄作为参数调用CreatProcess
```

** 水面之下 - LxBus **

为了讨论为什么高层的目标可以实现的机制，需要了解一些WSL进程间的一些通信机制LxBus的细节。LxBusNT和WSL进程之间的通信通道，主要是LXSS服务管理器和/init进程。它是一个像协议一样的socket，可以在NT和WSL进程间发送消息和共享状态。LxBus有很多功能，有可能成为未来博客发布的首选工具，但是我们讨论一些它可以做的事情。

在WSL实例中，LxBus通过打开/dev/lxss设备来访问。这个设备是被锁定在/init守护进程下了，有另外一个/dev/lxssclient设备支持/dev/lxss设备功能的子集。在Windows上，LxBus示通过表示运行的WSL实例的句柄访问。一旦打开了这些设备中的一个文件描述符或句柄，它支持多种ioctls。和我们讨论问题最接近的是register server和connect to server的ioctls。NT或WSL进程想要使用register server的ioctl接受连接，那它需要用LxCore.sys驱动注册一个命名的LxBus服务器。当注册完成时，代表服务器的文件描述符或句柄就会返回给调用者。我们叫这个对象是LxBus服务器端口。可以将register server的ioctl当作是socket中的绑定。一旦服务器被注册，ServerPort对象支持wait for connection的ioctl，这个函数类似于Socket的listen调用

客户端希望连接到注册的服务器时，它会使用“connect to server”的ioctl来连接，参数中指定要链接的服务器的名字。如果服务器正在等待连接，那么就在NT和WSL进程之间建立了连接。NT进程会获取到代表连接的一个句柄，WSL进程获得一个文件描述符。这些都可以引用一个潜在的文件对象叫做LxBuf MessagePort。MessageProts支持读写消息，他们也实现了多个ioctl完成各种功能主要涉及到NT和WSL进程间的资源共享。例如，其中一个ioctl允许NT进程和WSL进程共享句柄。它也能够是得WSL进程的输出重定向回Windows。

** 来自Bash.exe的I/O重定向 **

早期发布WSL时，bash.exe只支持控制台的输入输出。如果启动bash.exe，并且将它作为连接到远端机器的ssh连接会非常完美。但是如果想要将WSL合并进脚本或Wn32程序中调用会变得比较困难。用户很快指出了这一点，在Github上的话题2也指出了这个局限性。到了Windows内测版，就可以将WSL进程的数据导入或导出，并且可以以NT文件作为输入输出。

当通过LxssManager的COM接口启动WSL进程时，需要一些参数。比较明显的参数，像二进制文件路径，当前的工作目录等。其他的参数，包括作为WSL进程的标准输入输出和错误的NT句柄。这些句柄传递给LxssManager服务器，进而与将要创建的WSL进程共享。我们将这些句柄的共享称作参数组织。参数组织，是实现为LxBus MessagePort对象上的一个ioctl，它会接收一个句柄，返回一个唯一标识。这个唯一标识然后和其他的创建进程的参数一起发送给/init守护进程。/Init解析消息，开始解包句柄，用于WSL新进程的输入句柄。解包是作为另外一个MessagePort ioctl实现，返回一个文件描述符。/init进程解包这三个句柄，调用dup2系统调用替代描述符0，1和2。有第四个参数也会被组包，它用于创建WSL实例中的tty设备。从同一个bash.exe启动的所有的WSL进程共享相同的tty设备。一旦解包了所有句柄，/init使用forks和exec系统调用创建子进程。

如果熟悉bash，你可能了解-c参数允许提供一个命令字符串来运行（而不是以交互模式运行bash）。Bash.exe发送命令行参数到/bin/bash ，所以对命令行的调用可以以这种简单方式实现。

<div align="center">
![图11 使用-c执行bash](/img/2017-10-23-Windows-Subsystem-Linux-internals-interop_cmd.png)
</div>

** 在WSL内启动Win32应用程序 **

最近发布的几个Windows预览版现在支持从bash中启动Win32进程。这个功能是Github用户Xilun极其期待的功能。创建了它自己的开源项目cbwin来完成了这个目标。它的功能工作非常好。但是自从WSL接管整个功能，它已经变成了一个完整的解决方案，不再需要WSL中额外的启动二进制文件，或额外的Windows服务。证明了即使去掉加载器影响也可以做到启动Win32程序。

Linux有一个binfmt_misc数据，让用户模式应用程序注册他们可以处理的可执行格式。Java，Mono和Wine是几个使用这个功能注册向Linux内核注册文件类型的例子。注册的字符串由冒号分割，有七个域，其中一些是可选的。为了注册一个binfmt_misc数据，用户模式应用程序将注册字符串写到binfmt_misc注册文件中。（通常位于/proc/sys/fs/binfmt_misc/register）。例如：

```
sudo echo ":WSLInterop:M::MZ::/init:" > /proc/sys/fs/binfmt_misc/register
	WSLInterop  注册的名字，必须唯一
	M           注册类型，或者是Magic字节数组或扩展名
	MZ          WinPE的Magic字节序列
	/init       解析器的完整路径
```

解析器是系统中的一个脚本文件或二进制映像，它知道该如何处理特定类型文件。exec迭代寻找注册的binfmt_misc，查找正要被执行文件所对应的解析器。如果找到了，则调用exec，将解析器的路径放入Argv[0]中，余下的参数依次排列。如果熟悉“Shebang”，即"#!"操作符，经常在bash或Perl脚本中可以看到。Binfmt_misc则更加灵活，不是必须将额外的文本添加到文件头部，这对于二进制文件来说非常好。

如上述的例子中所示，正在使用/init作为binfmt_misc解析器。在WSL中，init是一个多用途的二进制执行文件，它由微软编写，并且以二进制资源形式嵌入到LxssManager.dll中。当init进程启动时，它首先校验pid。如果pid是1，init会以守护模式执行，这个进程就会作为WSL实例的Lxss管理器服务的接口点。如果PID不是1，init会作为binfmt_misc解析器模式运行，它用作启动NT二进制程序。在未来，如果我们开始运行其他的Ubuntu守护进程，我们不会将我们的守护进程替代/sbin/init，而是启动起来作为系统的另外一个守护进程。

<div align="center">
![图12 调用NotePad](/img/2017-10-23-Windows-Subsystem-Linux-internals-interop_notepad.png)
</div>

几个注意点和说明：

* NT进程会和bash.exe一样的权限启动
* NT二进制只能在DrvFs控制的路径下启动
* 如果工作目录在DrvFs挂载的目录中，启动的NT进程的当前的工作目录会被NT进程继承
* 在最初实现中，NT环境变量和WSL环境变量是分离的项目，包括$PATH环境变量。在开发中的一种方式是将NT的%PATH%环境变量与WSL进程共享，因此还在持续调整。
* 第一个尝试的NT命令应该是Windows的dir。与Linux的/bin/ls不同，dir是一个内置的命令，而非应用程序。如果你想要运行dir，你需要执行"/mnt/c/Windows/System32/cmd.exe /c dir"。比较幸运的是bash可以设置命令别名，例如alias dir='/mnt/c/Windows/System32/cmd.exe /c dir'。
* /init当作binfmt_misc解释器，这是当前的实现形式而不是以后的修改方向

** I/O重定向到Win32 **

从bash内部启动NT进程非常棒，但是要支持NT命令行的使用则需要通过一种方式NT二进制可以访问WSL进程中的代表stdin，stdout，stderr的文件描述符。这个要求通过引进一组API将VFS文件描述符参数打包到NT进程中来解决。下面过一下如何实现这个工作。

如果熟悉CreateProcess API，那么就会知道在调用CreateProcess时可以提供自定义的句柄作为标准输入，输出和错误输出。问题是我们得到的不是句柄，而是WSL实例内部的文件描述符。进入VFS文件组包，WSL进程（binfmt_misc解释器）来确定它想要将stdin，stdout，stderr文件句柄导出给NT就昵称。它在LxBus MessagePort上使用特殊的组织VFS文件的ioctl调用，这个调用可以以文件描述符作为参数。LxCore.sys驱动处理这个ioctl，使用提供的文件爱你描述符引用对应的VFS文件对象。这个对象会被加入到MessagePort的"组织VFS文件对象"的列表中。这个ioctl会返回一个唯一的标识，标注了VFS文件对象被编组。WSL进程在一个LxBus消息中通过MessagePort发送这个标识符到NT进程。

当NT进程（bash.exe）接受到这个消息时，它使用解包VFS文件的ioctl将标识符作为参数进行解包。LxCore.sys在编组的文件对象列表中查找这个标识，如果找到就从列表中移除，然后返回一个指向VFS文件爱你对象的NT句柄到调用者。这句柄可以用于CreateProcess API的标准句柄。因为句柄引用一个WSL实例的文件对象，在每一次的读写操作中rundown保护是必要的（保证WSL实例的特定状态在NT读写中不会损坏）。

<div align="center">
![图13 调用NotePad](/img/2017-10-23-Windows-Subsystem-Linux-internals-interop_bash.png)
</div>

** 结论 **

这篇文章总结了Windows的交互性；Windows子系统最高要求的功能。在NT和Linux进程之间的互操作性是允许WSL用户在同一个Shell中混合使用NT和Linux二进制可执行文件的功能，都内在地运行在NT内核上。简单总结一下LxBus，一个内在的IPC机制，是得NT和WSL进程间的通信变得可能。有了这个知识，后面的文章会介绍LxBus如何用于在NT和WSL进程之间共享NT和VFS文件对象。

#### Ubuntu权限 ####

正常安装完Ubuntu之后，会提示设置用户名密码，这个账户是Linux下的root权限账户。这个账户就是Linux下的默认用户，在启动时自动登录。这个账户可以也可以修改密码，使用passwd程序即可。也可以使用windows下的ubuntu.exe程序修改Linux的默认登录用户。

Linux有自己的权限系统，Windows也有它自己的权限系统。当Linux程序访问Linux环境下的资源时，则Linux下的权限系统生效；而Linux程序访问Windows环境下的资源时受限于Windows的权限系统。

Normal（没有提升权限）情况下运行Ubuntu，Linux用Windows登录用户的权限运行。而如果提升了权限或使用admin账户登录Windows，那么Linux则是用带有提升权限或admin的Windows权限运行，这种情况下Linux程序就具有Wnidows上Admin权限的程序一样的允许执行范围。

这里的Windows的允许范围都是独立于Linux实例的允许准则的，即Linux的Root权限仅仅影响Linux环境或文件系统中的用户权限，对于Windows上的资源没有影响。因此Linux进程以root权限执行仅仅允许进程在Linux环境有管理员权限。


到此WSL的基础技术点都已经翻译完了，在微软的官方博客上还有如下几个主题，等后面有时间了再一一翻译填充吧！

1. [WSL反病毒和防火墙兼容性](https://blogs.msdn.microsoft.com/wsl/2016/11/01/wsl-antivirus-and-firewall-compatibility/)
2. [WSL网络](https://blogs.msdn.microsoft.com/wsl/2016/11/08/225/)
3. [WSL测试](https://blogs.msdn.microsoft.com/wsl/2017/04/11/testing-the-windows-subsystem-for-linux/)
4. [WSL上的串口支持](https://blogs.msdn.microsoft.com/wsl/2017/04/14/serial-support-on-the-windows-subsystem-for-linux/)
5. [WSL上的文件系统改进](https://blogs.msdn.microsoft.com/wsl/2017/04/18/file-system-improvements-to-the-windows-subsystem-for-linux/)

后一篇会用调试和逆向的方法将上述的这些原理逐一解析一下对应的代码。


By Andy @2017-11-02 17:56:23
