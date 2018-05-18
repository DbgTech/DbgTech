---
title: Windows C 时间函数
date: 2017-07-17 17:15:33
tags:
- time
- C/C++
categories:
- 笔记
---

平时写代码总会时不时碰到使用当前时间的问题，在Windows上有很多中获取时间的方法，他们之间有什么区别呢？这里做一个简要总结（可能还不全，后面看到再进行补充）。

### 时间函数 ###

**time()/_time32()/_time64()**

上述三个函数是C运行时库中提供的时间函数。三个函数的返回值是自1970年1月1日午夜到现在的秒数；即从1970-01-01 00:00:00开始算起，到当前这一时刻的总秒数。所以这三个函数的精度就是秒了。其中time()内直接调用的_time32()函数，两个函数相当于一个函数。

time()返回值为time_t（64位系列\__time64_t）。time_t＝long int，它的范围从`1970－1－1 0:0:0`到`2038－1－18 19:14:07`；\__time64_t＝\__int64，它范围从`1970－1－1 0:0:0`到`3000－12－31 23:59:59`，也就是说现在使用这两个函数的代码在对应的年份将出现溢出错误。再者，两个函数出错时，返回-1。有一些代码规范会要求判断这个返回值。

``` time
#include "stdafx.h"
#include <stdlib.h>
#include <time.h>

int _tmain(int argc, _TCHAR* argv[])
{
	time_t ttime = time(NULL);
	__time64_t ttime64 = _time64(NULL);

	if (ttime != -1)
		printf("_time32(): %d\n", ttime);

	if (ttime64 != -1)
		printf("_time64(): %ld\n", ttime64);

	system("pause");
	return 0;
}
```
<!-- more -->
这个两个返回值是Coordinated Universal Time，而非本地时间。要想获取当前地区的时间，还需要通过函数进行转换。可以通过调用localtime将time_t所表示的CUT时间转换为本地时间（我们是+8区，比CUT多8个小时）并转成struct tm类型，该类型的各数据成员分别表示年月日时分秒。struct tm的结构如下：

使用struct tm结构要注意，tm_year从1900年计数，tm_mon月份从0计数。再者，对于struct tm *，由于是栈对象的直接使用，所以一次只能使用一个，即即使ttime1和ttime2不是一个时间，得到的ptm1也是等于ptm2的！
```
struct tm *ptm1=_localtime(&ttime1);
struct tm *ptm2=_localtime(&ttime2);
```

```
struct tm {
    int tm_sec;     /* seconds after the minute - [0,61] */
    int tm_min;     /* minutes after the hour - [0,59] */
    int tm_hour;    /* hours since midnight - [0,23] */
    int tm_mday;    /* day of the month - [1,31] */
    int tm_mon;     /* months since January - [0,11] */
    int tm_year;    /* years since 1900 */
    int tm_wday;    /* days since Sunday - [0,6] */
    int tm_yday;    /* days since January 1 - [0,365] */
    int tm_isdst;   /* daylight savings time flag */
};
```

需要注意的是这里获取到的结构体的年份是从1900年开始计算的，不是和time_t中保存的从1970年开始计算相同。

另外一对函数gmtime()/_gmtime64()，它们是将time_t转成格林尼治时间。各个字段的意思和获取本地时间相同，这里不再多解释，给出一个使用它们的例子，如下代码段。从图1的运行结果可看到本地时间和格林尼治时间差了8个小时，即我们处于北京时区所在的东八区的结果。

```
#include <stdlib.h>
#include <time.h>

int _tmain(int argc, _TCHAR* argv[])
{
	time_t ttime = time(NULL);
	__time64_t ttime64 = _time64(NULL);

	if (ttime != -1)
		printf("_time32(): %d\n", ttime);

	if (ttime64 != -1)
		printf("_time64(): %ld\n", ttime64);

	struct tm *ptm = localtime(&ttime);
	struct tm *ptm64 = _localtime64(&ttime64);

	printf("localtime(): %04d-%02d-%02d %02d:%02d:%02d yday: %d wday: %d isdst: %d\n",
		ptm->tm_year + 1900, ptm->tm_mon, ptm->tm_mday, ptm->tm_hour, ptm->tm_min, ptm->tm_sec,
		ptm->tm_yday, ptm->tm_wday, ptm->tm_isdst);

	struct tm *gmptm = gmtime(&ttime);
	struct tm *gmptm64 = _gmtime64(&ttime);
	printf("gmtime(): %04d-%02d-%02d %02d:%02d:%02d yday: %d wday: %d isdst: %d\n",
		gmptm->tm_year + 1900, gmptm->tm_mon, gmptm->tm_mday, gmptm->tm_hour, gmptm->tm_min, gmptm->tm_sec,
		gmptm->tm_yday, gmptm->tm_wday, gmptm->tm_isdst);

	system("pause");
	return 0;
}
```
<div align="center">
![图1 gmtime运行结果](/img/2017-07-17-Windows-C-Time-Functions-localtime-gmtime.jpg)
</div>

** GetSystemTime()/GetLocalTime() **

两个函数获取的是SYSTEMTIME结构，该结构体的内容代表系统时间。与time_t的转化函数类似，`void GetSystemTime( LPSYSTEMTIME lpSystemTime);`函数返回结构体中代表标准格林尼治时间；`void GetLocalTime(LPSYSTEMTIME lpSystemTime);`函数返回值代表的是本地的系统时间。其中Win32中SYSTEMTIME数据结构定义如下。在下面代码中给出了两个函数使用方法，见代码块。

```
SYSTEMTIME STRUCT{
    WORD wYear;		// 年
    WORD wMonth;	// 月
    WORD wDayOfWeek;//  星期，0=星期日，1=星期一...
    WORD wDay;		// 日
    WORD wHour;		// 时
    WORD wMinute;	// 分
    WORD wSecond;	// 秒
    WORD wMilliseconds; // 毫秒
};
```

```
#include <Windows.h>

int _tmain(int argc, _TCHAR* argv[])
{
	SYSTEMTIME systemTime = {};
	SYSTEMTIME localTime = {};
	GetSystemTime(&systemTime);
	GetLocalTime(&localTime);
	printf("GetSystemTime(): %04d-%02d-%02d %02d:%02d:%02d.%d wday: %d\n",
		systemTime.wYear, systemTime.wMonth, systemTime.wDay, 
		systemTime.wHour, systemTime.wMinute, systemTime.wSecond, systemTime.wMilliseconds,
        systemTime.wDayOfWeek);
	printf("GetLocalTime(): %04d-%02d-%02d %02d:%02d:%02d.%d wday: %d\n",
		localTime.wYear, localTime.wMonth, localTime.wDay, 
		localTime.wHour, localTime.wMinute, localTime.wSecond, systemTime.wMilliseconds,
        localTime.wDayOfWeek);

	system("pause");
	return 0;
}
```

** GetFileTime() **

文件时间是一个64位的值，代表了从1601年1月1日12:00开始到现在的时间计数，单位是100纳秒。当文件被创建，访问和写入时，系统会记录文件时间。GetFileTime()会返回输出三个FILETIME结构变量，包含了文件或目录的日期和时间信息。FILETIME结构与SYSTEMTIME结构可以通过调用系统函数相互转换。函数使用如下例所示：

```
// GetFileTime函数原型
BOOL WINAPI GetFileTime(
  _In_      HANDLE     hFile,
  _Out_opt_ LPFILETIME lpCreationTime,
  _Out_opt_ LPFILETIME lpLastAccessTime,
  _Out_opt_ LPFILETIME lpLastWriteTime
);

//
#include <windows.h>
#include <tchar.h>
#include <strsafe.h>

// GetLastWriteTime - Retrieves the last-write time and converts
//                    the time to a string
//
// Return value - TRUE if successful, FALSE otherwise
// hFile      - Valid file handle
// lpszString - Pointer to buffer to receive string
BOOL GetLastWriteTime(HANDLE hFile, LPTSTR lpszString, DWORD dwSize)
{
	FILETIME ftCreate, ftAccess, ftWrite;
	SYSTEMTIME stUTC, stLocal;
	DWORD dwRet;

	// Retrieve the file times for the file.
	if (!GetFileTime(hFile, &ftCreate, &ftAccess, &ftWrite))
		return FALSE;

	// Convert the last-write time to local time.
	FileTimeToSystemTime(&ftWrite, &stUTC);
	SystemTimeToTzSpecificLocalTime(NULL, &stUTC, &stLocal);

	// Build a string showing the date and time.
	dwRet = StringCchPrintf(lpszString, dwSize, 
		TEXT("%02d/%02d/%d  %02d:%02d"),
		stLocal.wMonth, stLocal.wDay, stLocal.wYear,
		stLocal.wHour, stLocal.wMinute);

	if( S_OK == dwRet )
		return TRUE;
	else
		return FALSE;
}

int _tmain(int argc, TCHAR *argv[])
{
	HANDLE hFile;
	TCHAR szBuf[MAX_PATH];

	hFile = CreateFile(_T("C:\\Users\\Administrator\\Desktop\\Temp.txt"), GENERIC_READ, FILE_SHARE_READ, NULL, OPEN_EXISTING, 0, NULL);
	if(hFile == INVALID_HANDLE_VALUE)
	{
		printf("CreateFile failed with %d\n", GetLastError());
		return 0;
	}
	if(GetLastWriteTime( hFile, szBuf, MAX_PATH ))
		_tprintf(TEXT("Last write time is: %s\n"), szBuf);

	CloseHandle(hFile);
}
```

** GetTickCount64()/GetTickCount() **

两个函数的原型如下，从原型可以看出，它们返回一个数字。返回值为从启动系统到现在的毫秒数。

```
DWORD GetTickCount(void);
ULONGLONG WINAPI GetTickCount64(void);
```

计数机器启动至今毫秒数，对于一般精度已经足够表示了。但是毕竟还是有范围的，要注意：
1. DWORD最多只能计数49.7天，如果Windows这么长时间后还未重启，计数失效。可以换用64位函数
2. 这两个函数都是依赖于系统定时器，典型的定时器的精度在10毫秒到16毫秒之间，所以精度有限

这两个函数比较简单，这里就不给出着两个函数的用法了。

** QueryPerformanceCounter() **

确切说这个函数不是用于获取日期时间的函数，它是用于计数/计时的函数。微软的文档解释是获取性能计数的当前值，这个计数值是一个高精度的时间戳（精度小于1ms），可以用于测量时间间隔。函数使用非常简单，传参为`LARGE_INTEGER`，传出值就是性能计数。

```
LARGE_INTEGER li={0};
QueryPerformanceCounter(&li);	// li.QuadPart返回的是机器计数！
```

下面给出一个完整的用于检测耗时代码的实际运行时间的代码，在耗时操作前后分别调用QueryPerformanceCounter，然后计算耗时。注意这里获取的是一个Tick值，需要调用另外一个函数QueryPerformanceFrequency获取性能计数频率。

```
#include <Windows.h>

int _tmain(int argc, _TCHAR* argv[])
{
	LARGE_INTEGER StartingTime, EndingTime, ElapsedMicroseconds;
	LARGE_INTEGER Frequency;

	QueryPerformanceFrequency(&Frequency);
	QueryPerformanceCounter(&StartingTime);

	// Activity to be timed
	Sleep(10);	// 睡眠10毫秒

	QueryPerformanceCounter(&EndingTime);
	ElapsedMicroseconds.QuadPart = EndingTime.QuadPart - StartingTime.QuadPart;


	//
	// We now have the elapsed number of ticks, along with the
	// number of ticks-per-second. We use these values
	// to convert to the number of elapsed microseconds.
	// To guard against loss-of-precision, we convert
	// to microseconds *before* dividing by ticks-per-second.
	//
	ElapsedMicroseconds.QuadPart *= 1000000;
	ElapsedMicroseconds.QuadPart /= Frequency.QuadPart;

	printf("Time span: %ld", ElapsedMicroseconds.QuadPart);

	system("pause");
	return 0;
}
```

### 时间计算函数 ###

这一部分总结一点有关时间计算和转换的函数，用于今后使用中的示例。

**获取时间差**
```
double difftime(time_t tm1,time_t tm2);
```

**构建时间值**

从一个确切的日期时间值构建一个time_t类型值。
```
time_t mktime(struct tm*ptm);
__time64_t _mktime64(struct tm*ptm);
```

**拼装字符串**

用于tm结构表示的日期时间值转换为可输出字符串，不需要像前面的例子那样自己组织printf的格式。

```
size_t strftime(char*strDest,size_t maxsize,const char*format,const struct tm*timeptr);
size_t wcsftime(wchar_t*strDest,size_t maxsize,const wchar_t*format,const struct tm *timeptr);
```

如下的函数将tm结构体内容转换为可输出字符串形式。

```
char * asctime(const struct tm *);
char * ctime(const time_t *);
```


**SYSTEMTIME与time_t互转**

```
/*
** SYSTEMTIME转time_t
*/

time_t systime_to_timet(const SYSTEMTIME& st)
{
    struct tm gm = {st.wSecond, st.wMinute, st.wHour, st.wDay, st.wMonth-1, st.wYear-1900, st.wDayOfWeek, 0, 0};
    return mktime(&gm);
}

/*
**time_t转SYSTEMTIME
*/
SYSTEMTIME Time_tToSystemTime(time_t t)
{
    tm temptm = *localtime(&t);
    SYSTEMTIME st = {1900 + temptm.tm_year,
    				 1 + temptm.tm_mon,
                     temptm.tm_wday,
                     temptm.tm_mday,
                     temptm.tm_hour,
                     temptm.tm_min,
                     temptm.tm_sec,
                     0};
    return st;
}
```

由上可以看出struct tm结构和struct SYSTEMTIME结构的年和月的取值范围是不一样的，前面说过tm结构年份是从1900年开始的，月份是从0开始计算，既有如下的转换关系；此外tm结构体中没有毫秒值。
```
tm.tm_mon = systemtime.wMonth - 1
tm.tm_year = systemtime.wYear - 1900
```


还有一种方法是通过struct FILETIME作为中间量来转换time_t和systemtime

```
/*
**time_t转SYSTEMTIME
*/
SYSTEMTIME TimetToSystemTime(time_t t)
{
    FILETIME ft;
    SYSTEMTIME pst;
    LONGLONG nLL = Int32x32To64(t, 10000000) + 116444736000000000;
    ft.dwLowDateTime = (DWORD)nLL;
    ft.dwHighDateTime = (DWORD)(nLL >> 32);
    FileTimeToSystemTime(&ft, &pst);
    return pst;
}
/*
**SYSTEMTIME转time_t
*/
time_t SystemTimeToTimet(SYSTEMTIME st)
{
    FILETIME ft;
    SystemTimeToFileTime( &st, &ft );
    LONGLONG nLL;
    ULARGE_INTEGER ui;
    ui.LowPart = ft.dwLowDateTime;
    ui.HighPart = ft.dwHighDateTime;
    nLL = (ft.dwHighDateTime << 32) + ft.dwLowDateTime;
    time_t pt = (long)((LONGLONG)(ui.QuadPart - 116444736000000000) / 10000000);
    return pt;
}
```

** SystemTime/LocalTime/FileTime **

这些函数获取的时间值之间的相互转换有如下的几个函数，不再一一给出示例，用到时百度/Google即可。

```
BOOL WINAPI SystemTimeToFileTime(
  _In_  const SYSTEMTIME *lpSystemTime,
  _Out_       LPFILETIME lpFileTime
);

NTSTATUS WINAPI RtlLocalTimeToSystemTime(
  _In_  PLARGE_INTEGER LocalTime,
  _Out_ PLARGE_INTEGER SystemTime
);

BOOL WINAPI FileTimeToLocalFileTime(
  _In_  const FILETIME   *lpFileTime,
  _Out_       LPFILETIME lpLocalFileTime
);

BOOL WINAPI FileTimeToSystemTime(
  _In_  const FILETIME     *lpFileTime,
  _Out_       LPSYSTEMTIME lpSystemTime
);

BOOL WINAPI SystemTimeToFileTime(
  _In_  const SYSTEMTIME *lpSystemTime,
  _Out_       LPFILETIME lpFileTime
);
```

**参考文档**

1. [日常时间的获取、格式化等操作汇总](http://sodino.com/2015/03/15/c-time/)
2. [Windows/Linux下C/C++时间函数全攻略](http://www.cnblogs.com/hicjiajia/archive/2011/01/20/1940165.html)
3. [Time Functions](https://msdn.microsoft.com/zh-cn/library/windows/desktop/ms725473(v=vs.85).aspx)

**修订历史**

* 2017-07-17 21:03:40   完成文章

By Andy @2017-07-17 21:03:40
