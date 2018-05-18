---
title: ReactOS 编译以及虚拟机运行
date: 2017-06-16 18:15:33
tags:
- ReactOS
- 编译
categories:
- ReactOS
---

前面简单介绍了一下ReactOS系统，且将它作为不了解ReactOS的朋友初步了解用吧！其实不用多说，想要了解Windows运行机制的，都知道这个项目。那这篇呢就简单说说ReactOS源代码下载；如何编译源代码成为可安装CD映像；如何安装ReactOS系统或安装过程中遇到的坑；最后说一下我认为的重中之重，如何用Windbg调试ReactOS的内核。废话少叙，马上开始！

### 源码下载 ###

从2005年1月1日开始，源码就被放到了Subversion(SVN)仓库。所以要想下载ReactOS源码，首先要安装一下SVN客户端程序（安装SVN客户端这活就略过了，没用过SVN的朋友自己问度娘吧，灰常简单！）。安装完SVN之后，下面就是Check源码了。从ReactOS的Wiki上可以找到ReactOS代码所在的SVN仓库的地址如下：

```
svn://svn.reactos.org/reactos
```

简单说一下ReactOS的SVN仓库目录内容，如图 1 所示，branches目录中可以认为是曾经使用过的分支，用于试验性的代码；tags目录中为已经发布的系统版本的代码；vendor目录为ReactOS使用过的一些开源项目；最后说trunk目录，该目录下是正在开发中的代码，其中的reactos目录中就是我们要下载的代码了。直接使用SVN将该`trunk/reactos`分支源码克隆下来即可。

<!-- more -->

<div align="center">
![图1 ReactOS源代码SVN仓库目录](/img/2017-06-16-ReactOS-Compile-and-run-in-vm-SVN-Directory.jpg)
</div>

> 注：此处使用的代码分支为开发分支，也即不断被更新的代码分支。它是可能存在bug，不能保证稳定性；但是它可以保证你获得的都是ReactOS最新的代码。当然这里也可以获取已经发布的版本的代码，他们位于tags目录下。比如ReactOS官网上推荐的最新版本为0.4.5，在tags目录下可以发现ReactOS-0.4.5目录，它就是对应的源代码了。

### 编译 ###

下载了ReactOS源代码之后，下一步就是编译它了，当然了ReactOS Wiki中对ReactOS系统的编译有明确说明。鉴于英语水平很渣，造成了其中的部分东西理解不了，需要亲自尝试，得出对的结论放到这个地方来（英语高手可以略过此部分内容，直接参考Wiki上的原汁原味内容）。

ReactOS项目还是蛮贴心的啦！专门构建一个应用程序，将编译中用到的程序全部打包进来了，只需要下载编译环境包，并安装即可。ReactOS作为一个Windows兼容系统，ReactOS项目组具然提供了Linux平台上的编译环境包，吃一大惊啊！

当然了，对于我这样一个Linux白痴来说，只能使用Windows上的编译环境包了，这里给出当前最新的Windows平台下的 [RosBE](http://sourceforge.net/projects/reactos/files/RosBE-Windows/i386/2.1.4/RosBE-2.1.4.exe/download)，点击即可下载。下载完毕后，直接安装即可（Windows上的安装够傻瓜，不需多说吧！）。安装过程中，会有一个界面让确定ReactOS源码的目录，默认被放到了C盘的用户目录了，如果前面已经下载了ReactOS，则修改一下这个源码目录，如图 2 所示（我的源码目录），这样在后面启动了编译命令行，直接就进入了源码目录了。

<div align="center">
![图2 RosBE源码目录](/img/2017-06-16-ReactOS-Compile-and-run-in-vm-RosBE-src-directory.jpg)
</div>

安装完之后，在Windows的开始目录就可以找到编译环境了。这里就不给出具体的图了，朋友们自己安装完了，自己查看吧！输入help可以查看其中可以执行的命令。

对于默认的编译，非常简单！启动RosBE命令行，切换目录到源码目录（如果之前没有正确设置源码目录的话），按照如下的命令，依次执行即可。其中，默认配置生成的编译文件目录为output-MinGW-i386，要进入该目录进行对应的编译；ninja all执行时间会长一点，耐心等待即可。

```
configure.cmd			// 完成配置
cd output-MinGW-i386

ninja all				// 编译所有文件

ninja bootcd			// 生成系统安装CD映像
```

从上述的编译环境生成过程中我们可以看到，默认的编译器是GNU编译器，即gcc等。那么它是不生成用于Windbg调试用的PDB文件的，怎么办呢？答案是研究ReactOS Wiki，它上面说是可以使用Windbg调试ReactOS系统的，那么一定有编译出PDB文件的方法啊，这样才能用Windbg进行调试！

寻寻觅觅，寻寻觅觅，找到Wiki中的"[调试](https://reactos.org/wiki/Debugging)"一节，有Windbg小节的说明。说得倒是很明确：要完全利用Windbg的优势，需要用MSVC编译ReactOS得到PDB符号文件才可以。对于MSVC编译，这是默认的调试风格（笔者注：这句话不懂啥意思）。如果你想要使用gcc编译，需要用WINKD选项设置为TRUE（或者可以在配置后使用CMake-GUI设置，然后在重新配置，或者编辑options.cmake文件的默认值）（笔者注：云里雾里，不知道该怎么设置）。另外一种可用的方法是用WINKD=TRUE选项编译的ntoskrnl.exe和kdcom.dll替换掉默认编译的这两个文件。也可以用Windows 2003的kdcom.dll替换掉ReactOS的该文件。上述的最后一种方法用Windows 2003的kdcom.dll替换ReactOS的kdcom.dll文件，这种方式之前试过，也确实可以调试，但是问题还是很多的，关键没有pdb啊。又找到Wiki中"[Windbg的使用说明](https://reactos.org/wiki/WinDBG)"，依旧没有发现有效方法。再看ReactOS编译一节的内容吧，再次看到了"Optional: Set Up Visual Studio"，内容也比较简单，如果想要用VS编译ReactOS，需要获取支持的版本，VS2010或之后更新版本。要使用VS编译，打开合适的VS命令行提示程序，依照基于GCC构建ReactOS的方法即可（再之后的内容是可能碰到的问题，不再翻译了）。看到这里，恍了一大悟啊！直接用VS的编译环境编译即可！！！

哎，说多了都是泪！英语好何至于如此？鼓捣了两次，每次都花了一整天时间！就是这么简单的指导，理解了分分钟搞定啊！虽然说看文章是可以这么搞，但是具体怎么样呢？还是要实践出真知啊！

机器上装了VS2015英文版编译器，直接使用它的VS Command prompt，按照上面默认的编译方式进行（这里我将编译目录换了一个，避免与原来的冲突，使用output-MSVS-i386，方法是首先创建该目录，然后切换到目录执行configure.cmd），按照官方说法就直接可以编译成功了。但是我还是遇到了问题啊！其实是国际化的问题，用的是英文的VS2015，它编译文件按照ReactOS内部设置，会编译多国语言，那么就要用到多国语言资源，包括头文件。编译过程中就弹出了错误，在包含`reactos\base\setup\usetup\lang`目录下的某个文件时，出现了不认识的字符，其实就是多国语言造成的。为了简单起见，将其他的语言都注掉，只留下一个英语。此处涉及到两个文件有`build.ninja`和`rules.ninja`，从中搜索`-DLANGUAGE_EN_US`，然后将除它之外的其他的语言都去掉。

> 此处最好是将其他的都去掉，包括中文。我没有去掉中文，后面碰到了RC.exe编译资源文件是找不到一些内容。我怀疑还是和编译器是英文有关。无奈只剩下了一个英文

搞完了上述的问题，就直接编译通过了，出现了我们要的bootcd.iso。

略曲折啊！最主要的原因是比较笨T-T（让我悲伤十分钟）。如果有小伙伴用前面设置WINKD的方法搞出来了，请分享给我，让我好好学习一下。

### 安装虚拟机中系统 ###

前面编译出来用于安装系统的bootcd.iso了，那么接下来就是将它安装一下，看一下实际效果喽！我使用的VMWare 12，当然也可以用其他的虚拟机，这里就以VMWare为例啦！

这里我也碰到了一个坑，后面没有仔细研究，只简单说一下！我一开始创建的虚拟机是Windows 2003，然后使用bootcd.iso安装系统，装到要格式化磁盘时，就提醒找不到磁盘，确定后就重启了，反复试验了好几次，都是一个问题，具体原因就没分析了。如果那个小伙伴知道，请您告诉我！

有了上面教训，于是乎就重新创建了一个XP系统，professional版本（其他的没试验）。安装过程就很顺利了，和Windows XP的安装非常类似，此处不再多说。给一个安装完之后的效果图，如图 3.

<div align="center">
![图3 ReactOS界面](/img/2017-06-16-ReactOS-Compile-and-run-in-vm-ReactOS-Face.jpg)
</div>

### 调试ReactOS的内核 ###

安装完虚拟机，体验了ReactOS系统，和Win2003非常像，但是界面效果还是差好多的说。但是的但是是这已经非常牛X了，好么！

怎么调试呢？人家都说了可以兼容Windbg调试啦，那就是和Windows的Windbg调试一样啦！此处省略一千字，自己脑补......

关键地方省略了（超级欠揍有木有？？）！^_^

开玩笑！开玩笑！关键是和Windows的调试一样的设置，问度娘，度娘会告诉你一大票，虽然不那么精准，还是够用了！这里说几个可能遇到的问题吧。

* 首先是要关掉ReactOS系统，在VMWare 12中才能给ReactOS虚拟机创建串口
* 其次VMWare X开始，默认第一个串口都是被共享打印机占用的，所以只能用COM2
	此处需要在ReactOS系统中将C盘下的freeldr.ini中的ReactOS Debug节区中的调试端口设置为COM2
* 最后就是启动Windbg调试器时使用管理员权限（Win7及其之后系统）

这样就应该没有什么问题了，可以直接调试ReactOS系统了。下面给出我最终调试ReactOS的一张图，见图4

<div align="center">
![图4 ReactOS调试系统调试图](/img/2017-06-16-ReactOS-Compile-and-run-in-vm-kernel-windbg-dbg.jpg)
</div>

从图中可以看到nt模块的符号是私有符号，且路径指向了我们的编译路径中的msvc_pdb目录下的ntoskrnl.pdb，和Windows很像有木有！！！

接下来就是调试系统机制，阅读源码啦！开端还算顺利，加油！！！

#### 补充：####

** 1. 源码分支更新**
2017年10月3日ReactOS代码管理从SVN集中式管理切换到GIT分布式管理，原来的SVN代码分支也不再更新。

ReactOS托管在github上的页面地址如下：

```
https://github.com/reactos/reactos
```

在github上的GIT分支如下所示，使用GIT客户端下载即可：

```
https://github.com/reactos/reactos.git
```

** 2. 设置系统的语言 **

前面有关于英文版VS2015无法编译通过的问题的修改，其实有一个更方便的办法就是在构建output-*目录之前先修改要编译的系统使用的语言，在执行`configure.cmd`时就构建出编译为指定语言的环境，这样就不会出现要编译所有语言版本而VS2015英文版无法识别非英系语言的问题。

修改ReactOS源码中的内容位于`*\reactos\sdk\cmake\localization.cmake`文件中，修改为如下内容：

```
if(NOT DEFINED I18N_LANG)
    set(I18N_LANG all)
endif()
// 修改为如下的内容，只编译英文内容
if(NOT DEFINED I18N_LANG)
    set(I18N_LANG "en-US")
endif()
```

** 参考文档 **

1. ReactOS Wiki [https://www.reactos.org/wiki/Welcome_to_the_ReactOS_Development_Wiki](https://www.reactos.org/wiki/Welcome_to_the_ReactOS_Development_Wiki)
2. https://www.reactos.org/

** 修订历史 **

* 2017-06-18 20:12:25 完成文章
* 2017-10-16 11:12:25 添加ReactOS新的源码分支信息

By Andy @2017-06-18 20:12:25

<div align="center">
**声明**
*“博客原创文章遵守《[知识共享许可协议](https://creativecommons.org/licenses/)》，转载文章请标明[出处](https://dbglife.github.io/)”*
</div>
