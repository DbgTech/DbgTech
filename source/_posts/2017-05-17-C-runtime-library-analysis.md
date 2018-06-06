---
title: C\C++运行时库源代码阅读
date: 2017-05-17 11:33:32
tags:
- C\C++
- 运行时库
categories:
- 调试
---

`C\C++`运行时库主要是提供了`C\C++`运行时使用的一些公用函数，包括参数处理，浮点运算，内存处理，输入输出，国际化，内存的分配与释放管理，数据类型转换，调试，目录控制等等。

这些函数可以静态链接或动态链接库的形式提供给`C\C++`程序使用。静态链接，就是将代码嵌入到EXE或DLL中，编译出来的文件比较大。但是它不需要再依赖动态运行时库，如MSVCR80.dll等；动态链接库，仅仅将库的导出函数信息编译到EXE中，在程序运行时，需要动态加载MSVCR80.dll等运行时库文件。

由于运行时库中包含了众多公用函数，这些公用函数，比如new，需要调用系统的一些函数进行初始化，所以运行时库也是需要初始化的。至于初始化时机，不用说肯定要在用户代码运行之前。否则程序代码调用了new方法，而此时还没有准备好可以分配的内存。那就出现错误了。对于Win32控制台程序，就是要在main()函数执行之前。

VC的做法是在编译器提供模块的入口函数，在模块的入口函数中，先调用CRT的初始化逻辑，然后在调用main()函数。即：

```
CRT入口函数()
{
    CRT初始化
    main()
    CRT清理
}
```

我们以VS2008为例，简单过一下 CRT的初始化与清理：
<!-- more -->
#### 动态链接会使用 crtexe.c中的内容 ####

文件目录为*\Microsoft Visual Studio 9.0\VC\crt\src\crtexe.c
> \* 表示VS2008的安装目录。

对于EXE，根据不同程序类型（`UNICODE/ASCII`，控制台和Win32程序），入口函数不同。如下：

```
#ifdef _WINMAIN_        // Win32程序

#ifdef WPRFLAG          // UNICODE程序
int wWinMainCRTStartup(
#else  /* WPRFLAG */
int WinMainCRTStartup(
#endif  /* WPRFLAG */

#else  /* _WINMAIN_ */  // 控制台程序

#ifdef WPRFLAG          // UNICODE程序
int wmainCRTStartup(
#else  /* WPRFLAG */
int mainCRTStartup(
#endif  /* WPRFLAG */

#endif  /* _WINMAIN_ */
        void
        )
{
        /*
         * The /GS security cookie must be initialized before any exception
         * handling targetting the current image is registered.  No function
         * using exception handling can be called in the current image until
         * after __security_init_cookie has been called.
         */
        __security_init_cookie();

        return __tmainCRTStartup();
}
```

我们的例子CRunTime使用VC2008默认的编译，及UNICODE形式的控制台程序，那么它的起始函数为：

```
_wmainCRTStartup()
```

**1. GS机制初始化**

```
/*
 * The /GS security cookie must be initialized before any exception
 * handling targetting the current image is registered.  No function
 * using exception handling can be called in the current image until
 * after __security_init_cookie has been called.
 */
__security_init_cookie();
```

GS安全cookie必须在任何基于当前映像的异常处理被注册之前初始化。直到该函数被调用，任何使用异常处理的函数都不能调用。

对于32位的GS机制，需要定义一个全局的安全cookie变量，它保存了计算出来的cookie值，用于校验cookie时使用。这个全局变量需要有一个初始值，防止在安全cookie初始化函数调用之前有用到cookie的情况。

```
#define DEFAULT_SECURITY_COOKIE 0xBB40E64E

/*
 * The global security cookie.  This name is known to the compiler.
 * Initialize to a garbage non-zero value just in case we have a buffer overrun
 * in any code that gets run before __security_init_cookie() has a chance to
 * initialize the cookie to the final value.
 */
DECLSPEC_SELECTANY UINT_PTR __security_cookie = DEFAULT_SECURITY_COOKIE;
```

**cookie值的生成：**
&emsp;&emsp;1.取时间值的低32位
&emsp;&emsp;2.异或上时间值的高32位
&emsp;&emsp;3.异或当前线程ID
&emsp;&emsp;4.异或当前进程ID
&emsp;&emsp;5.异或启动机器的毫秒数
&emsp;&emsp;6.异或性能计数值的高32位 和低32位
&emsp;&emsp;7.如果结果值和默认的cookie相同，则将值加1
&emsp;&emsp;8.如果cookie高两字节为0，则将低两字节复制到高两字节

上述的这些内容，无非是为了让cookie在当前的进程中尽量随机且唯一。

**2. 如下的函数都是在__try/__except防护中完成的。作为应用程序的一个起始异常处理**

首先判断当前代码是否在被调用。

判断 __native_startup_lock 是否为空，如果为空，则将当前栈基址赋值给它。如果有线程在执行本段初始化代码，且线程不是当前线程，则等待。

线程可重入，如果是当前线程，则直接进入。

如果正在初始化，则退出如果还没有初始化，则设置全局状态变量为 __initializing，并且调用 _initterm((_PVFV *)(void *)__xi_a, (_PVFV *)(void *)__xi_z);
```
_initterm(__xc_a, __xc_z)   // 做C++构造函数调用
```

**3. 如果有任何的动态初始化的` __declspec(thread)`的变量，则调用初始化函数。通过定义在`tlsdyn.obj`中的一个回调函数**

`__dyn_tls_init()`，来初始化这些变量。

```
if (__dyn_tls_init_callback != NULL &&
    _IsNonwritableInCurrentImage((PBYTE)&__dyn_tls_init_callback))
{
    __dyn_tls_init_callback(NULL, DLL_THREAD_ATTACH, NULL);
}
```

会通过`func(__xd_a, __xd_z)`来进行初始化。

**4. 获取命令行**

主要是找出 exe之后的命令行内容，方便下面调用 wmain()等函数

**5. 然后就直接调用 wmain()方法**

mainret = main(argc, argv, envp);

#### 静态链接会使用 crt0.c中的内容####

首先，被作为模块入口，并且做CRT初始化和清理的函数位于crt0.c中，路径如下：
`*\Microsoft Visual Studio 9.0\VC\crt\src\crt0.c`

>\* 表示VS2008的安装目录。

我们的例子CRunTime使用VC2008默认的编译，及UNICODE形式的控制台程序，那么它的起始函数为：

```
_tmainCRTStartup()
```

如下为该文件的起始处的说明：

```
/***
*mainCRTStartup(void)
*wmainCRTStartup(void)
*WinMainCRTStartup(void)
*wWinMainCRTStartup(void)
*
*Purpose:
*       These routines do the C runtime initialization, call the appropriate
*       user entry function, and handle termination cleanup.  For a managed
*       app, they then return the exit code back to the calling routine, which
*       is the managed startup code.  For an unmanaged app, they call exit and
*       never return.
*
*       Function:               User entry called:
*       mainCRTStartup          main
*       wmainCRTStartup         wmain
*       WinMainCRTStartup       WinMain
*       wWinMainCRTStartup      wWinMain
*
*Entry:
*
*Exit:
*       Managed app: return value from main() et al, or the exception code if
*                 execution was terminated by the __except guarding the call
*                 to main().
*       Unmanaged app: never return.
*
*******************************************************************************/
```

大致意思是：上面这些函数都是做C运行时初始化的，然后调用合适的用户入口函数，并且处理程序结束时的清理工作。对于托管的程序，他们会将推出代码返回给调用历程，这个代码是托管启动代码。对于未托管的程序，则调用exit，并且不会再返回。

首先以Win32控制台程序为例，中间穿插着Win32应用程序与它的不同的地方：

**1. GS机制初始化**

```
/*
 * The /GS security cookie must be initialized before any exception
 * handling targetting the current image is registered.  No function
 * using exception handling can be called in the current image until
 * after __security_init_cookie has been called.
 */
__security_init_cookie();
```

GS安全cookie必须在任何基于当前映像的异常处理被注册之前初始化。直到该函数被调用，任何使用异常处理的函数都不能调用。

对于32位的GS机制，需要定义一个全局的安全cookie变量，它保存了计算出来的cookie值，用于校验cookie时使用。这个全局变量需要有一个初始值，防止在安全cookie初始化函数调用之前有用到cookie的情况。

```
#define DEFAULT_SECURITY_COOKIE 0xBB40E64E

/*
 * The global security cookie.  This name is known to the compiler.
 * Initialize to a garbage non-zero value just in case we have a buffer overrun
 * in any code that gets run before __security_init_cookie() has a chance to
 * initialize the cookie to the final value.
 */
DECLSPEC_SELECTANY UINT_PTR __security_cookie = DEFAULT_SECURITY_COOKIE;
```

**cookie值的生成:**
&emsp;&emsp;1.取时间值的低32位
&emsp;&emsp;2.异或上时间值的高32位
&emsp;&emsp;3.异或当前线程ID
&emsp;&emsp;4.异或当前进程ID
&emsp;&emsp;5.异或启动机器的毫秒数
&emsp;&emsp;6.异或性能计数值的高32位 和低32位
&emsp;&emsp;7.如果结果值和默认的cookie相同，则将值加1
&emsp;&emsp;8.如果cookie高两字节为0，则将低两字节复制到高两字节

&emsp;&emsp;上述的这些内容，无非是为了让cookie在当前的进程中尽量随机且唯一。

**2. 判断是否为托管应用程序**

```
/*
 * Determine if this is a managed application
 */
managedapp = check_managed_app();
```

确定应用程序是否是托管程序。PE文件目录表中`IMAGE_DIRECTORY_ENTRY_COM_DESCRIPTOR 14 COM Runtime descriptor`存在，且其存在虚地址。

**3. `_heap_init(1)`初始化堆**

```
/***
*_heap_init() - Initialize the heap
*
*Purpose:
*       Setup the initial C library heap.
*
*       NOTES:
*       (1) This routine should only be called once!
*       (2) This routine must be called before any other heap requests.
*
*Entry:
*       <void>
*Exit:
*       Returns 1 if successful, 0 otherwise.
*
*Exceptions:
*       If heap cannot be initialized, the program will be terminated
*       with a fatal runtime error.
*
*******************************************************************************/
```

源码中堆函数的描述：
目的是建立初始的C库堆(这个函数应该只被调用一次)。其次，函数应该在其他的堆请求之前调用。如果堆不能初始化，应用程序以严重的运行时错误而被终止。首先创建一个 big-block堆，如果创建失败，则程序直接退出。

对于X86程序，调用`__heap_select()`函数选择一个堆类型，该函数会返回`__V6_HEAP`，`__V5_HEAP`或`__SYSTEM_HEAP` 三种类型之一。对于使用静态运行时库，只会使用系统堆。对于使用CRT动态库时，通过 GetEnvironmentVariableA 获取环境变量中对堆的配置`__HEAP_ENV_STRING`。

```
#define __HEAP_ENV_STRING       "__MSVCRT_HEAP_SELECT"
```

如果该字段对应的值为 __GLOBAL_HEAP_SELECTOR，则为获取到环境堆选择字符串。

```
#define __GLOBAL_HEAP_SELECTOR  "__GLOBAL_HEAP_SELECTED"
```

否则获取从环境变量串中查找应用程序名称，环境堆选择字符串中查找 ','字符，该字符串后的数字为选择的堆类型：

```
#define __SYSTEM_HEAP           1
#define __V5_HEAP               2
#define __V6_HEAP               3
```

对于`__V6_HEAP`类型的堆,调用`__sbh_heap_init()`初始化小块堆。该函数中，在`_crtheap`上创建了一个`__sbh_pHeaderList`分别初始化了门限值，Header列表大小等值。
具体的内容后面研究堆时再详细解释。

对于`__V5_HEAP`类型的堆（老版本的运行时库堆），通过调用`__old_sbh_new_region()`函数.

```
/***
*__old_sbh_region_t * __old_sbh_new_region() - get a region for the small-block heap
*
*Purpose:
*       Creates and adds a new region for the small-block heap. First, a
*       descriptor (__old_sbh_region_t) is obtained for the new region. Next,
*       VirtualAlloc() is used to reserved an address space of size
*       _OLD_PAGES_PER_REGION * _OLD_PAGESIZE, and the first _PAGES_PER_COMMITTMENT
*       pages are committed.
*
*       Note that if __old_small_block_heap is available (i.e., the p_pages_begin
*       field is _OLD_NO_PAGES), it becomes the descriptor for the new regions. This is
*       basically the small-block heap initialization.
*
*Entry:
*       No arguments.
*
*Exit:
*       If successful, a pointer to the descriptor for the new region is
*       returned. Otherwise, NULL is returned.
*
*******************************************************************************/
```

从函数的说明上，它会为小块堆获取一个区。即创建和增加一个区，并且要创建一个描述符，来描述这个区块。这块的内容，也在后面研究堆时，再详述。`__V5_HEAP`这种老式的堆分配也只会在动态库中才会存在。

> ULONG HeapType = 2;
> HeapSetInformation(_crtheap, HeapCompatibilityInformation, &HeapType, sizeof(HeapType));
> 对于X64的系统，开启低碎片堆。8字节开销的堆通常比16字节开销的堆有更好的性能。
> 尤其对于需要执行大量小块分配的应用程序。

**4. 初始化的多线程：`_mtinit()`**

先看一下函数的说明：

```
/****
*_mtinit() - Init multi-thread data bases
*
*Purpose:
*       (1) Call _mtinitlocks to create/open all lock semaphores.
*       (2) Allocate a TLS index to hold pointers to per-thread data
*           structure.
*
*       NOTES:
*       (1) Only to be called ONCE at startup
*       (2) Must be called BEFORE any mthread requests are made
*
*Entry:
*       <NONE>
*Exit:
*       returns FALSE on failure
*
*Uses:
*       <any registers may be modified at init time>
*
*Exceptions:
*
*******************************************************************************/
```

初始化多线程数据基础。两个目的，一个是调用多线程初始化锁，创建并打开用于锁的信号量。第二个是分配TLS索引值，用于保存每一个线程数据的指针。等待加载Kernel32.dll模块，获取FlsAlloc/FlsGetValue/FlsSetValue/FlsFree。如果失败了，则获取TlsAlloc/TlsGetValue/TlsSetValue/TlsFree等函数。分配用于存储FlsGetValue函数的指针。初始化全局函数指针，分别是如下几个：

```
heap_handler()
initcrit()
invarg()
purevirt()
rand_s()
winsig()
winxfltr()
eh_hooks()
aexit_rtn()
```

初始化多线程锁，`_locktable[]`组分配条目，标记为lkPrealloc。将lock指向lclcritsects数组的CS结构变量分配一个tls索引，用于维护每一个线程数据的指针。为当前线程创建一个每个线程都有的数据结构。`_tiddata`同时初始化这个结构体，并将`_tid`设置为当前线程ID，`_thandle`设置为 -1。

**5. 如果开启了 运行时检查 功能，则要初始化该功能**

未找到对应源码，后续补充！

**6. `_ioinit()`进行低级I/O初始化**

首先获取启动信息，如果获取失败，就返回。分配一个ioinfo结构体数组，用`__pioinfo[0]`指向它，并以此进行初始化。如果有继承的句柄，则处理继承的文件句柄信息。这块的信息在`StartupInfo.cbReserved2/StartupInfo.lpReserved` 两个成员中然后遍历继承的句柄，将继承到的句柄。依次分配`ioinfo`结构体，填入到`__pioinfo[]`数组中，从下标1开始。

如果没有继承标准输入，输出，错误文件句柄，则尝试从系统中直接获取。并且将它们设置到适当的osfile域中。对于继承的标准句柄，要保证他们是具有文本模式的。最后，将支持的句柄数量设置到`_nhandle`变量中

**7. 获取命令行和环境变量信息**

`GetCommandLineT()/GetEnvironmentStringT()` 这两个函数都是对系统API的封装

`_tsetargv()/_tsetenvp()`   // 代码中无实际作用

**8. C 初始化`_cinit()`**

```
/***
*_cinit - C initialization
*
*Purpose:
*       This routine performs the shared DOS and Windows initialization.
*       The following order of initialization must be preserved -
*
*       1.  Check for devices for file handles 0 - 2
*       2.  Integer divide interrupt vector setup
*       3.  General C initializer routines
*
*Entry:
*       No parameters: Called from __crtstart and assumes data
*       set up correctly there.
*
*Exit:
*       Initializes C runtime data.
*       Returns 0 if all .CRT$XI internal initializations succeeded, else
*       the _RT_* fatal error code encountered.
*
*Exceptions:
*
*******************************************************************************/
```

这个函数执行共享的DOS和Windows的初始化。必须按照如下的序列进行初始化。
&emsp;&emsp;1.校验文件句柄 0 - 2
&emsp;&emsp;2.整数除法中断向量设置
&emsp;&emsp;3.通用的C初始化函数调用

`_FPinit`调用，浮点数包初始化。（如果有的话）。

`_initp_misc_cfltcvt_tab()`将数组的指针进行编码，具体作用待查。

`_initterm_e(__xi_a, __xi_z)`执行注册在PE文件数据区的初始化函数。
`_initerm( __xc_a, __xc_z)`执行C++的初四化函数，全局对象的构造函数。
这块和编译器相关，不是特别确定，但是从各个资料，以及调试结果看是对的。具体编译器的处理逻辑是怎样的，这个就有点深入了，能力不及。待提高^_^......

其实`__xi_a`和`__xi_z`是两个全局变量，如下所示
```
_CRTALLOC(".CRT$XIA") _PIFV __xi_a[];
_CRTALLOC(".CRT$XIZ") _PIFV __xi_z[];    /* C initializers */
_CRTALLOC(".CRT$XCA") _PVFV __xc_a[];
_CRTALLOC(".CRT$XCZ") _PVFV __xc_z[];    /* C++ initializers */
```
作为一个区域的开始于结束。编译器会将需要初始化的变量的函数放入两个变量之间。在`_initterm_e()`方法中会遍历这个区间内的所有指针，只要指针不为空，则将它当作函数调用。即完成对应的变量的初始化。对于C++的初始化代码类似，比如全局对象，会将对象的初始化函数放入到`__xc_a`和`__xc_z`之间，在程序CRT初始化时，就会调用到全局对象的初始化函数。同时在初始化函数中，会将对象的析构函数通过 atexit调用，注册到程序退出的回调函数列表中。

```
int __cdecl _initterm_e (_PIFV * pfbegin, _PIFV * pfend)
{
        int ret = 0;
        /*
         * walk the table of function pointers from the bottom up, until
         * the end is encountered.  Do not skip the first entry.  The initial
         * value of pfbegin points to the first valid entry.  Do not try to
         * execute what pfend points to.  Only entries before pfend are valid.
         */
        while ( pfbegin < pfend  && ret == 0) // >
        {
            /*
             * if current table entry is non-NULL, call thru it.
             */
            if ( *pfbegin != NULL )
                ret = (**pfbegin)();
            ++pfbegin;
        }

        return ret;
}
```

如果有任何的动态初始化的`__declspec(thread)`的变量，则调用初始化函数。通过定义在tlsdyn.obj中的一个回调函数`__dyn_tls_init()`，来初始化这些变量。

```
if (__dyn_tls_init_callback != NULL &&
    _IsNonwritableInCurrentImage((PBYTE)&__dyn_tls_init_callback))
{
	__dyn_tls_init_callback(NULL, DLL_THREAD_ATTACH, NULL);
}
```

**9. 调用WinMain/main等函数**

对于Win32程序，处理命令行，过滤出命令行内容，传递给WinMain

对于控制台程序，直接调用main

最后对于托管程序，需要将main返回值 调用exit，而非托管程序，直接退出程序。


> 在`_tWinMain()/_tmain()`函数调用时，会加一个`__try/__except` 的代码防护
> &emsp;&emsp;这块内容在异常过滤的地方补充一下。

#### Win32程序与Console程序的一个不同点 ####

对于Win32应用程序，需要获取启动信息，在 gs的cookie值初始化完了，要获取一下Win32程序的启动信息。GetStartupInfo() 获取启动信息，包括界面实现的位置，窗口显示状态等。

```
typedef struct _STARTUPINFOA {
    DWORD   cb;
    LPSTR   lpReserved;
    LPSTR   lpDesktop;
    LPSTR   lpTitle;
    DWORD   dwX;
    DWORD   dwY;
    DWORD   dwXSize;
    DWORD   dwYSize;
    DWORD   dwXCountChars;
    DWORD   dwYCountChars;
    DWORD   dwFillAttribute;
    DWORD   dwFlags;
    WORD    wShowWindow;
    WORD    cbReserved2;
    LPBYTE  lpReserved2;
    HANDLE  hStdInput;
    HANDLE  hStdOutput;
    HANDLE  hStdError;
} STARTUPINFOA, *LPSTARTUPINFOA;
```

** 参考文章 **

1. 《软件调试》
2. VS2008运行时库源码

** 修订历史 **

* 2017-05-17 18:33:32	完成博文

By Andy @2017-05-17 18:33:32
