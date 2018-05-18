---
title: Windows中的定时器
date: 2017-08-18 20:52:23
tags:
- Windbg
- 定时器
categories:
- Windbg
---

在Windows的编程中一直使用定时器，但是对于定时器的具体实现原理没有做过详细总结。为了不糊涂到底，这一篇简单总结一下Windows下的各种定时器。对于每一类定时器，先介绍使用方法，再总结它的实现原理。通过查资料，大致有如下三种定时器，再看到其他的再添加进来。下面先详细总结一下这三种定时器。

#### SetTimer / KillTimer ####

这类定时器是用的最多的一类，针对窗口设置定时器，到时间后触发定时器响应函数，处理定时器函数的逻辑。两个函数的原型如下代码块所示：

```
UINT_PTR WINAPI SetTimer(
  _In_opt_ HWND      hWnd,			// 关联的窗口
  _In_     UINT_PTR  nIDEvent,		// 定时器ID，用于标识设置的定时器
  _In_     UINT      uElapse,	 	// 定时器时间间隔
  _In_opt_ TIMERPROC lpTimerFunc  	// 定时器过程函数
);

BOOL WINAPI KillTimer(_In_opt_ HWND hWnd, _In_ UINT_PTR uIDEvent);
```

如下代码列举出了如何使用SetTimer/KillTimer函数（里面省略了WinMain函数，以及主窗口的注册与创建）。这里可以看到在窗口过程函数的`IDM_ABOUT`消息中（本来是弹出About窗口用的）中设置了定时器，设置的定时器窗口句柄即本窗口句柄；定时器的ID设置为`TIMER_TEST`，即2017，代表一个定时器；时间间隔设置为5s；最后一个参数为定时器过程，这里分别尝试了两种，一种是设置最后一个参数，一种是将最后一个参数设置为NULL。对于给出定时器响应函数的时候，会直接调用定时器函数；否则会将定时器消息发送给窗口过程，例子中用OnTimer做为`WM_TIMER`消息的相应函数。如果设置了定时器函数，那么定时器消息就不会被发送到窗口过程中，即调用不到OnTimer函数，关于这点，具体的原因后面总结到实现原理时会有说明。

```
#define TIMER_TEST 2017

LRESULT OnTimer(HWND hWnd, WPARAM wParam, LPARAM lParam)
{
	if (TIMER_TEST == wParam)
	{
		MessageBoxA(hWnd, "From OnTimer()", "OK", MB_OK);
		KillTimer(hWnd, TIMER_TEST);
	}

	return 0;
}

VOID CALLBACK TimerProc(HWND hWnd, UINT uMsg, UINT_PTR idEvent, DWORD dwTime)
{
	if (TIMER_TEST == idEvent)
	{
		MessageBoxA(hWnd, "From TimerProc()", "OK", MB_OK);
		KillTimer(hWnd, TIMER_TEST);
	}

	return ;
}

LRESULT CALLBACK WndProc(HWND hWnd, UINT message, WPARAM wParam, LPARAM lParam)
{
	int wmId, wmEvent;
	PAINTSTRUCT ps;
	HDC hdc;

	switch (message)
	{
	case WM_COMMAND:
		wmId    = LOWORD(wParam);
		wmEvent = HIWORD(wParam);
		// Parse the menu selections:
		switch (wmId)
		{
		case IDM_ABOUT:
			SetTimer(hWnd, TIMER_TEST, 5 * 1000, TimerProc);
            //SetTimer(hWnd, TIMER_TEST, 5 * 1000, NULL);
			break;
		case IDM_EXIT:
			DestroyWindow(hWnd);
			break;
		default:
			return DefWindowProc(hWnd, message, wParam, lParam);
		}
		break;
	case WM_TIMER:
		OnTimer(hWnd, wParam, lParam);
		break;
	case WM_PAINT:
		hdc = BeginPaint(hWnd, &ps);
		// TODO: Add any drawing code here...
		EndPaint(hWnd, &ps);
		break;
	case WM_DESTROY:
		PostQuitMessage(0);
		break;
	default:
		return DefWindowProc(hWnd, message, wParam, lParam);
	}
	return 0;
}
```

首先从SetTimer函数开始，函数从User32.dll导出，而SetTimer函数对应到User32.dll中的函数即NtUserSetTimer()，如下代码从user32.dll的反汇编代码中拷贝（wow64），它简单处理参数，直接调用进了内核中（同Ntdll.dll中的stub函数）。

```
.text:7DC679FB ; UINT_PTR __stdcall NtUserSetTimer(HWND hWnd, UINT_PTR nIDEvent, UINT uElapse, TIMERPROC lpTimerFunc)
.text:7DC679FB                 public _NtUserSetTimer@16
.text:7DC679FB _NtUserSetTimer@16 proc near            ; CODE XREF: xxxLBoxCtlKeyInput(x,x,x)+14DCAp
.text:7DC679FB                                         ; xxxLBoxCtlCharInput(x,x,x)+249p ...
.text:7DC679FB
.text:7DC679FB hWnd            = dword ptr  4
.text:7DC679FB nIDEvent        = dword ptr  8
.text:7DC679FB uElapse         = dword ptr  0Ch
.text:7DC679FB lpTimerFunc     = dword ptr  10h
.text:7DC679FB
.text:7DC679FB                 mov     eax, 1018h
.text:7DC67A00                 mov     ecx, 0
.text:7DC67A05                 lea     edx, [esp+hWnd]
.text:7DC67A09                 call    large dword ptr fs:0C0h
.text:7DC67A10                 add     esp, 4
.text:7DC67A13                 retn    10h
```

这里以Win2000的源码作为参考，分析一下定时器的原理。Win2000源码中的NtUserSetTimer函数如下，它比较简单。首先根据hwnd参数查找当前进程的句柄表，获取该句柄对应的内核结构体，从而获取它对应的tagWND结构体指针。其次判断定时器的时间间隔是否小于10ms，如果小于10ms，则强制改为10ms，由于太小的时间粒度使得内核消耗太多时间在定时器消息处理上；再者，NT的定时器精度依赖于多媒体定时器，可能也不会太小；如果定时器的超时时间值设置太小，会造成定时器无法准确工作，有一些定时器消息被CPU“吃掉”了（来不及处理）。

```
UINT_PTR NtUserSetTimer(
    IN HWND hwnd,
    IN UINT_PTR nIDEvent,
    IN UINT wElapse,
    IN TIMERPROC pTimerFunc)
{
    PWND pwnd;
    BEGINATOMICRECV(UINT_PTR, 0);

    ValidateHWNDOPT(pwnd, hwnd);

    /*
     * If we let apps set a timer granularity less then 10 the app
     * spend too long processing timer messages.  Some WOW apps like
     * Paradox in WinStone use zero to effectively get the minimal
     * timer value which was ~55ms in Win 3.1.  We also step this
     * value up for 32 bit apps because the NT timer resolution
     * can very depending if the multimedia timers have turned up
     * the resolution.  If they have NT apps that specify a low value
     * will not work properly because they will eat the CPU processing
     * WM_TIMER messages
     */
    if (wElapse < 10) {
        RIPMSG1(RIP_WARNING, "SetTimer: timeout value was %ld set to 10",
                wElapse);
        wElapse = 10;
    }

    retval = _SetTimer(
            pwnd,
            nIDEvent,
            wElapse,
            (TIMERPROC_PWND)pTimerFunc);

    TRACE("NtUserSetTimer");
    ENDATOMICRECV();
}
```

NtUserSetTimer函数中，使用了一个宏定义ValidateHWNDOPT，用于验证hwnd的有效性，同时根据hwnd获取tagWND结构体指针。tagWND结构中保存了窗口所属的线程信息。紧接着将简单转调`_SetTimer`函数，下面看一下`_SetTimer`函数的逻辑。从参数也可以看出，第一个参数即前面根据HWND查找到的tagWND结构体的指针。对于pwnd非空的情况，要判断tagWND结构体中存储的线程是否和当前线程一致。如果不一致，则不允许设置定时器（即跨线程是无法设置其他线程窗口的定时器的）。

```
UINT_PTR _SetTimer(
    PWND         pwnd,
    UINT_PTR     nIDEvent,
    UINT         dwElapse,
    TIMERPROC_PWND pTimerFunc)
{
    /*
     * Prevent apps from setting a Timer with a window proc to another app
     */
    if (pwnd && (PpiCurrent() != GETPTI(pwnd)->ppi)) {

        RIPERR1(ERROR_ACCESS_DENIED,
                RIP_WARNING,
                "Calling SetTimer with window of another process %lX",
                pwnd);

        return 0;
    }

    return InternalSetTimer(pwnd, nIDEvent, dwElapse, pTimerFunc, 0);
}
```
如果满足条件则进一步调用到InternalSetTimer()方法，如下代码块所示。先看一下源码中对于InternalSetTimer函数的注释，该函数是SetTimer函数的核心。这块用的技术使得定时器响应函数调用有一些延迟（SetTimer的NtSetEvent和RIT唤醒并调用ScanTimers之间的时间）在SetTimer调用和计数器开始计数之间的延迟等。但由于RIT是高优先级的，所以这个不是太大问题。


```
/***************************************************************************\
* InternalSetTimer
*
* This is the guts of SetTimer that actually gets things going.
*
* NOTE (darrinm): Technically there is a bit of latency (the time it takes
* between SetTimer's NtSetEvent and when the RIT wakes up and calls ScanTimers)
* between when SetTimer is called and when the counter starts counting down.
* This is uncool but it should be a very short amount of time because the RIT
* is high-priority.  If it becomes a problem I know how to fix it.
*
* History:
* 15-Nov-1990 DavidPe      Created.
\***************************************************************************/

UINT_PTR InternalSetTimer(PWND pwnd, UINT_PTR nIDEvent, UINT dwElapse, TIMERPROC_PWND pTimerFunc, UINT flags)
{
    LARGE_INTEGER liT = {1, 0};
    PTIMER        ptmr;
    PTHREADINFO   ptiCurrent;

    CheckCritIn();

    /*
     * Assert if someone tries to set a timer after InitiateWin32kCleanup
     * killed the RIT.
     */
    UserAssert(gptiRit != NULL);

    /*
     * 1.0 compatibility weirdness. Also, don't allow negative elapse times
     * because this'll cause ScanTimers() to generate negative elapse times
     * between timers.
     */
     // 不允许有负数时间存在
    if ((dwElapse == 0) || (dwElapse > ELAPSED_MAX))
        dwElapse = 1;

    /*
     * Attempt to first locate the timer, then create a new one
     * if one isn't found. 尝试查找定时器，如果没有找到，则创建一个新的定时器
     */
    if ((ptmr = FindTimer(pwnd, nIDEvent, flags, FALSE)) == NULL) {

        /*
         * Not found.  Create a new one. 创建新的定时器
         */
        ptmr = (PTIMER)HMAllocObject(NULL, NULL, TYPE_TIMER, sizeof(TIMER));
        if (ptmr == NULL) {
            return 0;
        }

        ptmr->spwnd = NULL;

        if (pwnd == NULL) {

            WORD timerIdInitial = cTimerId;

            /*
             * Pick a unique, unused timer ID. 对于pwnd为空，即没有指定HWND，则获取一个唯一的TimerID
             */
            do {

                if (--cTimerId <= TIMERID_MIN)
                    cTimerId = TIMERID_MAX;

                if (cTimerId == timerIdInitial) {

                    /*
                     * Flat out of timers bud.
                     */
                    HMFreeObject(ptmr);
                    return 0;
                }

            } while (FindTimer(NULL, cTimerId, flags, FALSE) != NULL);

            ptmr->nID = (UINT)cTimerId;

        } else {
            ptmr->nID = nIDEvent;
        }

        /*
         * Link the new timer into the front of the list. 将新的定时器链入链表前端，gptmrFirst为空也没关系
         * Handily this works even when gptmrFirst is NULL.
         */
        ptmr->ptmrNext = gptmrFirst;
        ptmr->ptmrPrev = NULL;
        if (gptmrFirst)
            gptmrFirst->ptmrPrev = ptmr;
        gptmrFirst = ptmr;		// 双向链表

    } else {

        /*
         * If this timer was just about to be processed,
         * decrement cTimersReady since we're resetting it.
         */ // 如果找到了定时器，并且要被处理了，则将cTimersReady减1，因为这里设置就相当于重置
        if (ptmr->flags & TMRF_READY)
            DecTimerCount(ptmr->pti);
    }

    /*
     * If pwnd is NULL, create a unique id by
     * using the timer handle.  RIT timers are 'owned' by the RIT pti
     * so they are not deleted when the creating pti dies.
     * 如果pwnd是NULL，使用定时器句柄创建一个唯一的ID，RIT定时器是由RIT pti指向，所以
     * 当pti死掉时，这些定时器并不会被删除掉。
     * We used to record the pti as the pti of the window if one was
     * specified.  This is not what Win 3.1 does and it broke 10862
     * where some merge app was setting the timer on winword's window
     * it it still expected to get the messages not winword.
     *
     * MS Visual C NT was counting on this bug in the NT 3.1 so if
     * a thread sets a timer for a window in another thread in the
     * same process the timer goes off in the thread of the window.
     * You can see this by doing a build in msvcnt and the files being
     * compiled do not show up.
     */
    ptiCurrent = (PTHREADINFO)(W32GetCurrentThread()); /*
                                                        * This will be NULL
                                                        * for a non-GUI thread.
                                                        */

    if (pwnd == NULL) {

        if (flags & TMRF_RIT) {
            ptmr->pti = gptiRit;
        } else {
            ptmr->pti = ptiCurrent;
            UserAssert(ptiCurrent);
        }

    } else {

        /*
         * As enforced in the API wrappers.  We shouldn't get here
         * any other way for an app timer.
         *
         * Always use pti of the window when TMRF_PTIWINDOW is passed in.
         */
        if ((ptiCurrent->TIF_flags & TIF_16BIT) && !(flags & TMRF_PTIWINDOW)) {
            ptmr->pti = ptiCurrent;
            UserAssert(ptiCurrent);
        } else {
            ptmr->pti = GETPTI(pwnd);
        }
    }

    /*
     * Initialize the timer-struct.
     *
     * NOTE: The ptiOptCreator is used to identify a JOURNAL-timer.  We
     *       want to allow these timers to be destroyed when the creator
     *       thread goes away.  For other threads that create timers across
     *       threads, we do not want to destroy these timers when the
     *       creator goes away.  Currently, we're only checking for a
     *       TMRF_RIT.  However, in the future we might want to add this
     *       same check for TMRF_SYSTEM.
     */
    Lock(&(ptmr->spwnd), pwnd);

    ptmr->cmsCountdown  = ptmr->cmsRate = dwElapse;
    ptmr->flags         = flags | TMRF_INIT;
    ptmr->pfn           = pTimerFunc;
    ptmr->ptiOptCreator = (flags & TMRF_RIT ? ptiCurrent : NULL);

    /*
     * Force the RIT to scan timers.
     *
     * N.B. The following code sets the raw input thread timer to expire
     *      at the absolute time 1 which is very far into the past. This
     *      causes the timer to immediately expire before the set timer
     *      call returns.
     */
    if (ptiCurrent == gptiRit) {
        /*
         * Don't let RIT timer loop reset the master timer - we already have.
         */
        gbMasterTimerSet = TRUE;
    }

    UserAssert(gptmrMaster);
    KeSetTimer(gptmrMaster, liT, NULL);

    /*
     * Windows 3.1 returns the timer ID if non-zero, otherwise it returns 1.
     */
    return (ptmr->nID == 0 ? 1 : ptmr->nID);
}
```

#### TimerQueueTimer ####

 CreateTimerQueue / DeleteTimerQueue / CreateTimerQueueTimer / DeleteTimerQueueTimer

timeSetEvent / timeKillEvent

#### CreateWaitableTimer / OpenWaitableTimer ####

  / WaitableTimer




















**未总结内容**

1. 无

**参考文章**

* WoW64 Wiki	[https://zh.wikipedia.org/wiki/WoW64](https://zh.wikipedia.org/wiki/WoW64)
* WoW64的切换细节   [http://advdbg.org/blogs/advdbg_system/articles/5495.aspx](http://advdbg.org/blogs/advdbg_system/articles/5495.aspx)
* WoW进程的异常分发过程   [http://advdbg.org/blogs/advdbg_system/articles/5884.aspx](http://advdbg.org/blogs/advdbg_system/articles/5884.aspx)

**修订历史**

* 2017-08-20 11:21:23		完成文章

By Andy @2017-08-20 20:13:23