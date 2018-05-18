---
title: Universal Windows Platform 介绍
date: 2017-07-03 17:15:33
tags:
- Win10
- UWP
categories:
- 笔记
---

UWP是Universal Windows Platform的简称，即通用Windows平台。为了弄懂UWP到底是个什么东西，回溯了N年前的资料，弄懂了前后的发展关系，才逐渐明白了它是个什么东西。这一篇简单把这个过程总结一下，给其他对UWP以及之前一些概念比较迷糊的朋友一点启发。文章的内容是从网络上查找资料且整理后串起来的，有些说法不一定对。如果发现不正确的地方可以帮忙指出，对文章修改一下，避免误导后来者。

### 前世今生 ###

**.NET**，简单说是微软对抗Java而研发的编程框架。先说一下Windows DNA，英文全称为Windows Distributed interNet Application architecture，分布式应用程序开发框架，主要基于COM/COM+等建立。它是Win2000时期的开发框架，被称为新一代计算解决方案的框架。还有部分原因是Java这种基于虚拟机的跨平台模型的成功，促使微软开发自己的跨平台的编程框架，就是.Net框架。.Net框架致力于敏捷软件开发（Agile software development）、快速应用开发（Rapid application development）、平台无关性和网络透明化的软件开发平台。以通用语言运行库（Common Language Runtime）为基础，支持多种语言（C#、F#、VB.NET、C\+\+、Python等）的开发。更多关于.NET的内容，可以找一下相关的资料。

上面最后提到了通用语言运行库，即Common Language Runtime，下面看一下这是个什么东西。

**CLR**，通用语言运行平台（Common Language Runtime，简称CLR）。它是微软为.NET的虚拟机所选用的名称，即微软对通用语言架构（CLI）的实现版本，定义了一个代码运行的环境。CLR运行一种称为通用中间语言（IL）的字节码，这个是微软的通用中间语言的实现版本。开发人员使用高级编程语言撰写程序，编译器将代码编译成微软的中间语言(MSIL)。CLR内置有即时编译器，运行的时候CLR会将MSIL码转换为操作系统的原生码（Native code）。

**CLI**，通用语言基础架构（Common Language Infrastructure，简称CLI）。它是一个开放的技术规范，由微软联合惠普以及英特尔于2000年向ECMA倡议。通用语言基础架构定义了构成.NET Framework基础结构的可执行码以及代码的运行时环境的规范，定义了一个语言无关的跨体系结构的运行环境，这使得开发者可以用规范内定义的各种高级语言来开发软件，并且无需修正即可将软件运行在不同的计算机体系结构上。通用语言运行时（CLR）则是微软对此标准（CLI）的实现。
<!-- more -->
从上面的描述中可以看出，CLI是.NET框架的一部分，可以认为是从.NET框架中抽取出来的技术规范。CLR是.NET的虚拟机，认为是CLI的一个实现。这样就理清楚这三个物件的关系了。现在问题来了，.NET框架使用C实现的，它支持C\+\+，C#，VB.NET等，可是C\+\+是编译型语言，需要编译成二进制的（默认方式）。微软怎么办呢？于是就有了Managed C\+\+了。

**C\+\+托管扩展**，英文为Managed Extensions for C\+\+。它是对C\+\+的一个属性和关键字的扩展，以用于在微软的.NET Framework进行编程，也经常被称为托管C\+\+。托管C\+\+并非独立存在的编程语言，而仅仅是微软对C\+\+的一个语法扩展，允许C\+\+程序员在.NET框架和CLR的基础上进行托管编程。与C#和Visual Basic.NET相比，其主要优点是旧代码可以比较快地移植到新的平台上，而且即使不完全重写代码，也可以通过互操作在同一个模块中无缝集成托管和非托管代码，从新的.Net框架中获益。

托管C\+\+并非编程语言，那么微软肯定是不能甘心的，于是从Visual Studio 2005开始支持了一门新的计算机语言C\+\+/CLI。

**C\+\+/CLI（C++/CLR）**，C\+\+/CLI（CLI: Common Language Infrastructure），有时也被称为C\+\+/CLR，具体区别根据上述的CLI/CLR区分吧。C\+\+/CLI是一门由微软设计，用来代替C\+\+托管扩展（Managed C\+\+）的语言，它在兼容原有的C\+\+标准的同时，重新简化了托管代码扩展的语法，提供了更好的代码可读性。对C\+\+/CLI简单描述，C\+\+/CLI是一门独立的语言，而不是像C\+\+托管扩展一样是C\+\+的超集（C\+\+托管扩展有一些不标准的关键字如\_\_gc和\_\_value）。所以，C\+\+/CLI对于这些语法有较大的改变，尤其是去除了一些意义不明确的关键字，增加了一些对.NET的特性的语言级别的支持。

到此为止，看到的是微软从Native代码转换为跨平台的虚拟机运行方式，似乎.NET即将接管所有东西。但是可能微软发现这个玩意并不那么好用吧，于是自己又搞了一套新的Windows API，即WinRT。WinRT是一套基于COM的C\+\+ API。

**WinRT**，Windows Runtime。它是Windows 8中的一种跨平台应用程序架构。WinRT支持的开发语言包括C\+\+（包括C\+\+/CX）和托管语言C#和VB.NET，还有JavaScript。WinRT应用程序原生支持x86架构和ARM架构，同时为了更好的安全性和稳定性，支持运行在沙盒环境中。由于依赖于一些增强的COM组件，WinRT本质上是一个基于COM的API。正因为其COM风格的基础，WinRT可以像COM那样轻松地实现多种语言代码之间的交互联系，不过本质上是**非托管**的本地API。API的类型定义存储在以”.winmd"为后缀的元数据文件中，格式编码遵循ECMA 335的定义，和.Net使用的文件格式一样，不过稍有改进。使用统一的元数据格式相比于P/Invoke，可以大幅减少WinRT调用.NET程序时的开销，同时拥有更简单的语法。

既然WinRT是基于COM的，那么使用现有的C\+\+编程（老版本C\+\+）当然也是可以的了。为了方便使用老版本语言进行编程，微软提供了一套类似于ATL的库，即WTL。在新的称为Windows Runtime C\+\+ Template Library(WRL)的模板类库的帮助下（就好似ATL之于COM一样)，规范的C\+\+代码（遵循COM化规范）也同样可以用在使用了WinRT组件的程序里。不过MSDN建议使用C\+\+/CX而不是WRL。那么什么是C\+\+/CX呢？

**C\+\+/CX**, C\+\+/CX (Visual C\+\+ Component Extensions，Visual C++ 组件扩展)。它是微软的C\+\+编译器对C\+\+的扩展，使得程序员可以比较方便地编写面向Windows Runtime(WinRT)的程序。这个语言规范引入了一系列语法和类库抽象，以对C\+\+程序员来说比较自然的方式暴露了基于组件对象模型（COM)的WinRT编程范型的接口。这个语言扩展从C\+\+/CLI借用了语法，但它是面向WinRT和原生码而不是通用语言运行库和托管代码。

再到后来应该就是Win10发布了（跨过了Win9，不知道Win9会出一个什么C++的东西，直接被略过了）。Win10一出来，微软的野心就慢慢付诸实践了。直接出了一个玩意叫UWP。

**UWP**，通用Windows平台（Universal Windows Platform，简称UWP）。它是微软创建并在Windows 10中首次引入的一个同性质应用程序架构平台。此软件平台的目的是帮助发展Metro样式的应用程序，便于软件可以在Windows 10和Windows 10 Mobile上运行且无需重新编写。它支持使用C\+\+、C#、VB.NET或XAML开发的Windows应用程序。API采用C\+\+实现，并支持C\+\+、VB.NET、C#和JavaScript。UWP在Windows Server 2012和Windows 8中作为一个Windows Runtime平台的扩展被首次引入，允许开发者创建可潜在运行于多种类型设备的应用程序。UWP程序需要在Visual Studio 2015上进行原生开发。Win8.1/WP8.1的旧版本Metro应用程序需要修改迁移到UWP程序。对于Android和iOS的程序，前者将会允许直接运行于Windows，而iOS则是通过开源工具IslandWood将Object-C代码转换为Visual Studio项目，从而可以运行在Windows Phone上。

上述的介绍中，UWP平台是对Win8上的WinRT的扩展，目的是让一个程序可以无修改地运行在其他安装Windows系统的设备中。既然是WinRT的扩展，那么C\+\+/CX这个Visual C\+\+组件扩展就自然地用来编写UWP程序了。但是很多人可能不习惯C\+\+/CX的语法，于是微软就搞了一个开源的库，即C\+\+/WinRT。使用这个库，就可以用标准的C++语法编写UWP程序了，而不需要用C\+\+/CX这个扩展。详细内容可以参考微软的[Github开源库](https://github.com/microsoft/cppwinrt)，它里面有详细的介绍，这里不再过多介绍。

这样就将UWP及其相关内容的“前世今生”做了简单梳理，下面介绍几个相关的内容一些细节了。零篇，而非第一个内容，那应该是学习一下最新标准的C\+\+，即C\+\+ 14。接下来，第一个要说的就是WinRT，第二个是C\+\+/CX，最后要说一下C\+\+/WinRT。最终目的是了解UWP程序的编写要学习那些涉及到的内容。一个C\+\+程序员，要使用标准C\+\+编程，所以还是要说一下C\+\+/WinRT，实现以标准C\+\+编写UWP的目的。

### 新标准C\+\+ ###

**移动构造和右值引用**，在C++中有函数返回对象和传参传递对象的情况，这两种情况中都会调用类的拷贝构造函数进行对象复制。对于具有指针的类，在拷贝构造函数中需要对指针做处理以实现“深拷贝”，防止默认的浅拷贝造成函数指针重复释放。即便如此，拷贝构造函数的代价还是很大的，一方面需要动态申请内存，另一方面还需要进行内存块的复制。

新的标准中给出了一种新的构造函数，即移动构造函数。一图胜千言，见图二移动构造函数如何做。

<div align="center">
![图1 拷贝构造和移动构造](/img/2017-07-03-Universal-Windows-Platform-Win8APP-Move-Constructor.jpg)
</div>

为了实现上述的构造函数，需要搞一个新的传参形式（传值，传引用都已经被使用了），即右值引用（T &&）。被移作它用的将亡对象通常适合用右值引用，比如返回右值引用T&&的函数返回值，std::move的返回值或者转换为T&&的类型转换函数返回值。

**auto和decltype**，在之前的C\+\+中所有的变量使用之前必须进行定义，且要以确定类型进行定义。C/C\+\+就被称为了静态类型的语言。静态类型和动态类型的主要区别在于对变量进行类型检查的时间点。所谓静态类型，类型检查主要发生在编译阶段；而对于动态类型，类型检查主要发生在运行阶段。而动态类型的实现主要归功于类型推导。

其实C\+\+也可以使用类型推导（在之前的模板中其实就有类型推导的影子）。C++11中将auto关键字重新定义了，用于类型推导中。在下面的代码中，使用auto关键字要求编译器对name变量的类型进行自动推导。编译器可以用初始化值"word.\n"来确定name变量应该是char*类型。

```
#include <iostream>
using namespace std;
int main(){
	auto name = "word.\n";
    cout << "hello, " << name;
}
```

其实根据上述的描述可知，C++是在编译阶段推导出对应变量的类型。那么问题来了，对于函数传参，未初始化变量等是无法使用auto关键字的。至于原因也可知一二，编译期间编译器并无法得知函数参数的类型，同时对于未初始化变量编译器也无法推导出它的类型，因此编译势必会出错。

另外一个好玩的关键字就是decltype了。根据类型推导，从给auto定义变量初始化的值可以推导出该变量应该使用的类型。那么从一个已经定义的变量，同样可以推得它的类型了。得到了某个变量的类型，那么就可以拿它继续定义变量。如下的代码可以看出一二。

```
#include <typeinfo>
#include <iostream>
using namespace std;
int main(){
	int i;
    decltype(i) j = 0;
    cout << typeid(j).name() << endl;

    float a;
    double b;
    decltype(a + b) c;
    cout << typeid(c).name() << endl;
}
```

其实从应用上可以看出，auto和decltype都相当于“占位符”，在编译器真正编译时将变量应有类型推到出来，替换进去。简化代码编写，加快开发速度，方便代码书写。

**智能指针**，之前版本中有智能指针auto_ptr，但它有一些缺点，C++11中被废弃了。添加了新的智能指针，unique_ptr, shared_ptr以及weak_ptr。看过Chrome源码的对这几个关键字不会陌生。从字面上，unique_ptr表示该指针只被本智能指针拥有，其他智能指针不能拥有该指针值。一旦转移了指针归属，之前的智能指针对象对指针引用将出错；可以认为它是删除了拷贝构造函数而添加移动构造函数的类。而shared_ptr则是共享指针，允许多个智能指针对象同时保存同一个指针，即采用了引用计数方式管理对象。

对于weak_ptr可以指向shared_ptr指向的对象，但是不拥有该对象。weak_ptr的lock成员可以返回一个shared_ptr对象；但是在所指向内存释放掉了，lock返回值便是空了，即nullptr。

说到nullptr，简单说一下空指针的问题。C++11中引入了这个关键词，用它来代表空指针。主要是为了解决之前空指针定义混乱，造成语义不明确的问题，见下面代码段。

```
#include <stdio.h>

void f(char* c){
	printf("invoke f(char*)\n");
}

void f(int i){
	printf("invoke f(int)\n");
}

int main(){
	f(0);
    f(NULL);		// 有些编译器上编译有问题
    f((char*)0);
}
```

**lambda函数**，lambda概念是“函数式编程”的基础，现在很多面向对象编程和命令式编程都引入了lambda支持，C++11中也引入了对它的支持。先看一个代码片段。

```
int main(){
	int girls = 3, boys = 4;
    auto totalChild = [](int x, int y)->int{ return x + y; };
    return totalChild(girls, boys);
}
```

lambda表达式的语法如下：

```
[capture](parameter) mutable ->return-type {statement};
```

[capture]: 捕捉列表，出现在表达式开始处。用于捕捉上下文中的变量用于表达式中。
(parameter)： 参数列表，与普通函数参数列表相同，如果不传参写为空。
mutable： 默认情况下，lambda函数为const函数，mutable可以取消常量性。使用该修饰符，参数列表不可省略。
->return-type： 返回类型，用追踪返回类型形式声明函数的返回值。
{statement} ： 正常的函数语句

对于捕捉列表，有如下几种情况：

* [var] 表示值传递方式捕捉变量var
* [=] 表示值传递方式捕捉所有父作用域内的变量，包括this
* [&var] 表示引用方式传递捕捉变量var
* [&] 表示引用方式传递捕捉所有父作用域的变量，包括this
* [this] 表示值传递方式捕捉当前的this指针

### WinRT ###

根据上述的介绍，大概知道WinRT是个什么东西了。说得容易理解一点，微软最早在搞DNA，结果发现Sun在搞虚拟机，就掉头搞了一套.NET虚拟机，同时搞了个C#专注于虚拟机方式跨平台程序设计。等到后面，微软几乎将它自己所用到的语言都搞进.NET中去了，越来越发现这个玩意不能一统天下，需要想新的对策。在Win8的发行中，添加了一个新的API层即WinRT，全称Windows Runtime。

WinRT是一个新的API集合，按照微软的设计，具有以下一些特性（优点）：

* 实现了Metro UI规范的UI库
* 为Windows开发人员提供一个简单的UI编程模型，你不需要学习Win32API的那些复杂的API了
* 可以使用XAML-base的UI系统编写界面
* API都设计成了异步的调用（其实是调用时间超过50ms的API被搞成了异步调用）
* 和.NET一样是个沙箱的API，自成体系，用于创建AppStore上的应用程序
* API的元数据格式是ECMA335，和.NET一样的标准。

WinRT包装的新用户界面系统，和Win32API一样是COM的上层。微软官网文档给出了一个如下图的架构，从这个架构图上可以看出来微软给予Metro程序何种地位。Metro程序和Desktop程序在内核服务之上是平分天下的分布，且WinRT应用程序模型直接置于Windows Kernel之上了。但事实上不应该是这样的架构，或者说在Intel X86平台上不应该是这样的布局。从现有的进程中模块加载，以及WinRT的编程中可用的API中可以发现，一部分Win32的API可以用于Metro程序，同时在线程创建，进程创建等都是最终调用到ntdll的NtCreateThread等。

<div align="center">
![图2 Win8 WinRT架构图](/img/2017-07-03-Universal-Windows-Platform-Win8APP-Old.jpg)
</div>

对于WinRT不做过多的说明了，如果以后有机会可以对WinRT程序进行调试剖析，将它与Win32程序做一个对比。从而可以从创建，运行，权限等方面对有一个相对较全面的理解。

最后给出几个教程，WinRT原理，以及Metro程序编写教程，更加详细的内容可以参考一下这些教程：

* Turning to the past to power Windows’ future: An in-depth look at WinRT
    https://arstechnica.com/features/2012/10/windows-8-and-winrt-everything-old-is-new-again/

* Windows Runtime(WinRT) 揭秘
    http://www.cnblogs.com/shanyou/archive/2011/09/17/2179699.html

* Win8 Metro APP简要教程
    http://blog.csdn.net/column/details/win8.html?&page=2

正常编程微软建议使用C\+\+/CX，但是还是提供了一套类似ATL的编程库WRL，以方便使用标准C\+\+进行编程。WRL，全称为Windows Runtime C\+\+ Template Library，即Windows运行时C++模板库。之前简单说过，库的具体内容也不再介绍，有兴趣的可以参考一下WRL的文档。WRL在MSDN上的地址为[https://msdn.microsoft.com/en-us/library/hh438466(v=VS.110).aspx](https://msdn.microsoft.com/en-us/library/hh438466(v=VS.110).aspx)。

### C\+\+/CX ###

微软官网的`C++/CX`文档地址是[https://docs.microsoft.com/zh-cn/cpp/cppcx/visual-c-language-reference-c-cx](https://docs.microsoft.com/zh-cn/cpp/cppcx/visual-c-language-reference-c-cx)。这一节内容按照该文档的内容，摘抄一些重要点放到此处。

`C++/CX`只是标准C\+\+语言的扩展，目的是用尽可能接近标准C\+\+的"方言"构建Windows App和WinRT组件；使用`C++/CX`以Native代码形式编写UWP App和WinRT组件可以更容易与C#，VB，JS等交互；再者简化编写WinRT的App程序（与使用WRL相比）。编译中设置`/ZW`选项，它用来编译UWP App或WinRT组件。WinRT的声明定义在WinRT元数据文件中（.winmd），要使用WinRT声明指定#using或使用/FU编译选项。创建UWP工程后，VS会默认设置上述编译选项，并且对所有的WinRT库添加引用。

有几个类型需要注意下，如下表中所述：

|概念    | 标准C\+\+ |         C\+\+/CX          |            注释                          |
|-------|-----------|---------------------------|-----------------------------------------|
|基础类型|C\+\+基础类型|C\+\+/CX基础类型定义于WinRT中|编译器会隐式地将`C++/CX`类型映射为标准C\+\+类型|
|       | enum{}    | enum class{} <p>Or</p>enum struct{}| 32位枚举类型                           |
|       |（not apply）| Platform::Object^        |C++视角的WinRT类型系统中引用计数基础对象      |
|       |std::wstring L"..."| Platform::String&| Platform::String^是一个基于计数的非可变，Unicode序列，代表文本|
|指针    |指向对象 * Or std::shared_ptr|对象句柄 ^(hat) T^ identifier|所有的WinRT类使用指向对象句柄修饰符生命。访问对象的成员使用`->`符号。hat修饰符意思是指向WinRT对象，自动执行引用计数。更准确一点，对象句柄声明编译器应该插入代码自动管理对象的引用计数，如果引用计数归零，应该自动销毁对象。|
|引用    |对象的引用（&）:<p></p>T & identifier|追踪引用（%）：<p></p> T % identifier|仅仅Windows运行时类型可以使用追踪引用修饰符生命。对象的成员可以通过`.`访问。追踪引用意味着引用一个自动计数的WinRT对象。也即需要编译器插入代码管理对象引用计数|
|动态类型声明| new  |   ref new    |  分配一个WinRT对象，然后返回给指向对象的句柄                  |
|数组声明 | T identifier [] <p> Or</p>std::array identifier| Array< T ^> ^ identifier(size) <p> or </p> WriteOnlyArray< T^ >identifier(size)| 声明一维可修改数组或只写数组。数组本身也是一个引用计数对象，也需要使用指向对象句柄修饰符|
|类声明 |	class 标识符 {} Or struct 标识符 {}| 	ref class 标识符 {} Or ref struct 标识符 {}| class声明一个运行时类，默认私有访问权限 struct声明一个默认公有访问权限|
|结构声明| struct 标识符 {} 这是一个纯数据结构体|value class 标识符{} Or value struct 标识符{}| class声明一个POD结构体，默认私有访问权限，value class可以代表Win元数据，但是标准C++不行。struct声明一个默认公有访问权限的POD结构体|
| 接口声明| 仅仅包含虚拟方法的抽象类 | interface class 标识符{} Or interface struct 标识符{} | 声明一个默认私有访问权限的接口。 struct 声明一个默认公有访问权限的接口 |
| 委派   | std::function | public delegate return-type delegate-type-identifier ( [ parameters ] ); |	声明一个对象，可以像函数调用一样被调用 |
|事件    | (Does not apply) |	event delegate-type-identifier event-identifier ; <p></p>delegate-type-identifier delegate-identifier = ref newdelegate-type-identifier( this[, parameters]); <p></p>event-identifier += delegate-identifier ; <p>-or-</p> EventRegistrationToken token-identifier = obj.event-identifier+=delegate-identifier; <p>-or-</p>  auto token-identifier = obj. event-identifier::add(delegate-identifier); obj.event-identifier -= token-identifier ;<p>-or-</p>  obj.event-identifier ::remove( token-identifier ); | 声明一个事件对象，保存事件句柄的集合，当事件发生时这些委托将被调用。创建事件处理器。 增加事件处理器。增加事件处理器返回时间token。如果想要显示删除事件处理器，需要保存事件token以备后用。移除事件处理器。|
| 属性    | (Does not apply) | property T identifier; <p></p>property T identifier [ index ]; <p></p>property T default[ index ]; | 在类或对象成员函数上声明一个属性 |

C\+\+/CX只是形式上像C\+\+/CLI，而实际编译是不一样的。C\+\+/CX编写的代码编译后是Native的形式，而非运行于虚拟机中的字节码。与CLI语言不同的是，C\+\+/CX编程中不需要区分哪些对象是垃圾回收的，而那些对象需要自己管理生命周期。使用C\+\+/CX定义的基础类型，编译器就负责插入代码管理对象生命周期，而不像WRL一样，需要自己管理对象的生命周期，简化编程。

在实际编程中，使用C\+\+/CX编写代码和使用WRL编写代码的区别就是编译器会帮C\+\+/CX生成ABI接口，处理异常以及返回值等。而在WRL中，这些需要程序员自己根据COM要求进行构建。

**^（hat）符号**，该字符从C/+/+\CLI中借鉴，但是意义全不完全相同。它在C\+\+/CX中代表了智能指针类型，有两个作用：第一个就是自动管理WinRT对象生命周期，第二个就是提供自动类型转换能力，简化WinRT对象使用。

对于第一个自动管理对象生命周期无需多述，如同CComPtr类一样，在创建对象时自动调用AddRef增加计数，在退出作用域时自动调用Release减少引用计数，最后当引用计数到达0时，则将对象销毁！

WinRT中是要实现跨语言间的调用的，对于WinRT对象的引用都是对接口的引用。再者它的引用类型并非透明公开，在内存中的结构也未明确定义，C#的应用类型实现与C\+\+的引用类型实现并不相同，无法直接对WinRT类型进行转换。为了实现类型转换，需要使用IUnknow接口的QueryInterface，它可被认为是独立于语言的动态转换。所以对于同一个对象的引用，要想调用不同的接口，就需要用该函数进行接口转换，查询到对应的接口再进行调用。

^的好处在于编译器已经通过winmd文件查找到运行时类型以及其对应的接口，就可以在自动生成类型转换代码，从而简化程序员的工作。

使用hat的另外一个更重要的好处是统一使用，即对于ABI接口函数，它的参数只能是原始的指针类型，但是依然可以将hat定义的引用类型传递进去，编译器会完成T^到T*的转化。

**值类型value和引用类型ref**，在类声明中有ref和value的区分，其实微软的文档中两者被区分成Class declaration和Structure declaration。ref class/ref struct定义的是运行时类，而value class/value struct定义的是POD结构体。ref class是引用类型，支持多态，在堆上分配空间，效率低下，需要额外操作引用计数。而对于value形式的是在栈上分配空间，效率高，但不支持多态。

**public类**，在C\+\+中是不存在public定义类的情况，而在C\+\+/CX中对于准备对外公开的类，需要定义为public。只有用ref和value声明的类才可以设定为public。

```
public ref class Object(){};
public value class Object(){};
```

还有如下的几点：
1. 标准C\+\+方式的类不能被使用在public类的public属性成员中。（这个可以理解为public的要进入metadat文件中，标准C\+\+类不能进入）
2. public ref类型类的public属性成员中，除非使用property，否则不能声明类成员变量
3. public value类型的类的public属性成员，不能声明类成员变量以外的成员，如函数。

其中的property用于定义公有类的public属性，没有property修饰的共有成员变量编译长出错。实际是为成员添加了get()/set()函数。类似下面代码：

```
ref class SomeObj{
public:
	property int propertyX;
}

// 编译为如下形式：
ref class SomeObj{
public:
	property int propertyX{
    	int get() {return mX;};
        void set(int x) {mX = x;}
    }
private:
	int mX;
}
```

**委托delegate和事件event**，对于这两个内容，直接Copy一下网上的内容。delegate声明的EventHandler其实功能就类似于函数指针，在WinRTObj类中我们定义了EventHandler^类型的成员变量，并在构造函数中初始化，new EventHandler时的参数是一个Lambda表达式的匿名函数，所以在FireEvent中调用eHandler(x)时，其实就会去运行这个匿名函数。

```
delegate void EventHandler(int x);
ref class WinRTObj sealed
{
public:
     WinRTObj() {
          eHandler = ref new EventHandler([this](int x) {
               this->mX = x;
          });
     }
     void FireEvent(int x) {
          eHandler(x);
     }
     EventHandler^ eHandler;
private:
     int mX;
};
```

对于event的作用，看下面的代码：

```
delegate void EventHandler(int x);
ref class WinRTObj sealed
{
public:
     WinRTObj() {
          eHandler += ref new EventHandler([this](int x) {
               this->mX = x;
          });

          eHandler += ref new EventHandler([this](int x) {
               this->mX = x;
          });

          eHandler.add(ref new EventHandler([this](int x) {
               this->mX = x;
          }));

          ...
     }
     void FireEvent(int x) {
          eHandler(x);
     }
     event EventHandler^ eHandler;
private:
     int mX;
};
```

从代码上看，即可用`+=`来为eHandler添加“指针”，又可以用.add（）方法添加“指针”，当然还可以用remove()方法删除已经绑定的委托。其实用event关键字之后，eHandler被当作一个容器了。

这里将C\+\+/CX的一些感觉比较重要的点简单记录了一下，还有更为详细的内容，参考MSDN文档吧，毕竟比较权威。

### C\+\+/WinRT ###

WinRT API比较容易在托管语言比如C#来使用，对于本地代码C\+\+开发者，或者使用复杂的COM代码，或者使用VC++组件扩展（即C\+\+/CX）。这个语言扩展使得C\+\+能够理解描述WinRT对象的元数据，提供WinRT对象的自动引用计数等（前面的分析知道，这些都是通过编译器的实现的，由编译器插入额外的代码来完成复杂的COM需求功能），但是这些不是标准C\+\+语言。

微软提供了一个C\+\+/WinRT库，用于将标准C\+\+映射到Windows运行时库，这些内容被写进了头文件中。这样就允许程序员直接编写标准C\+\+编译器识别的代码来使用WinRT的API。也就是说C\+\+/WinRT被设计为C\+\+开发者提供了使用现代Windows API的通道。下面给出一个Hello World例子，来看一下C\+\+/WinRT和C\+\+/CX编写出来的代码的区别。

```
// C++/WinRT 编写代码
struct App : ApplicationT<App>
{
    void OnLaunched(LaunchActivatedEventArgs const &)
    {
        TextBlock block;

        block.FontFamily(FontFamily(L"Segoe UI Semibold"));
        block.FontSize(72.0);
        block.Foreground(SolidColorBrush(Colors::Orange()));
        block.VerticalAlignment(VerticalAlignment::Center);
        block.TextAlignment(TextAlignment::Center);
        block.Text(L"Hello World!");

        Window window = Window::Current();
        window.Content(block);
        window.Activate();
    }
}

// C++/CX编写出来代码
ref class App sealed : public Application
{
    protected:
        virtual void OnLaunched(LaunchActivatedEventArgs^ e) override
        {
            TextBlock^ block = ref new TextBlock();
            block->FontFamily = ref new FontFamily("Segoe UI Semibold");
            block->FontSize = 72.0;
            block->Foreground = ref new SolidColorBrush(Colors::Orange);
            block->VerticalAlignment = VerticalAlignment::Center;
            block->TextAlignment = TextAlignment::Center;
            block->Text = "Hello World!";

            Window^ window = Window::Current;
            window->Content = block;
            window->Activate();
        }
};
```

上述两段代码等价，可见C\+\+/WinRT代码和标准C\+\+写法没什么差异。

上述的内容摘自微软的官方博客。C\+\+/WinRT的具体使用方法在Github中有介绍，这里不再过多介绍。下面给出C\+\+/WinRT相关微软官方的的链接。

Standard C\+\+ and Windows Runtime(C\+\+/WinRT):
	[https://blogs.windows.com/buildingapps/2016/11/28/standard-c-windows-runtime-cwinrt](https://blogs.windows.com/buildingapps/2016/11/28/standard-c-windows-runtime-cwinrt)

C\+\+/WinRT的Github链接：
	[https://github.com/microsoft/cppwinrt](https://github.com/microsoft/cppwinrt)

** 参考：**

1. 维基百科
2. MSDN文档

**修订历史：**

* 2017-07-07 19:30:32 完成博文
* 2017-07-17 17:44:14 修改描述不当句子


By Andy @2017-07-06 20:42


