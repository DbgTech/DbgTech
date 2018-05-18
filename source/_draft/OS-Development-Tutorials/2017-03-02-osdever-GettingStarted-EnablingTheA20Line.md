---
title: 《系统开发》之开始开启A20地址线
date: 2017-03-02 14:00:17
tags:
- 操作系统
- 翻译
categories:
- 系统
- 翻译
---

原文地址：[http://www.osdever.net/tutorials/view/enabling-the-a20-line](http://www.osdever.net/tutorials/view/enabling-the-a20-line)

Okay，这一篇其实算不上一个教程。下面在汇编中开启A20地址线的例子中，注释已经非常明确了。
<!-- more -->
``` ASM
;;
;; enableA20.s (adapted from Visopsys OS-loader)
;;
;; Copyright (c) 2000, J. Andrew McLaughlin
;; You're free to use this code in any manner you like, as long as this
;; notice is included (and you give credit where it is due), and as long
;; as you understand and accept that it comes with NO WARRANTY OF ANY KIND.
;; Contact me at jamesamc@yahoo.com about any bugs or problems.
;;
;; 你可以自已地使用这个代码，只要添加本声明即可。
;;
    
enableA20:
	;; This subroutine will enable the A20 address line in the keyboard
	;; controller.  Takes no arguments.  Returns 0 in EAX on success, 
	;; -1 on failure.  Written for use in 16-bit code, see lines marked
	;; with 32-BIT for use in 32-bit code.
	;; 
	;; 这个子过程会用键盘控制器开启A20地址线。 没有参数，用EAX寄存器返回0表示成功
	;; 返回-1表示失败。使用16位代码编写，如果看到了标有32-BIT的标记，后面用32位代码完成

	pusha

	;; Make sure interrupts are disabled  关中断
	cli

	;; Keep a counter so that we can make up to 5 attempts to turn
	;; on A20 if necessary 计数器，后面我们将五次尝试开启A20地址线
	mov CX, 5

	.startAttempt1:		
	;; Wait for the controller to be ready for a command 等待控制器准备好
	.commandWait1:
	xor AX, AX
	in AL, 64h
	bt AX, 1
	jc .commandWait1

	;; Tell the controller we want to read the current status.
	;; Send the command D0h: read output port. 告诉控制器想要读取当前状态，D0H命令
	mov AL, 0D0h
	out 64h, AL

	;; Wait for the controller to be ready with a byte of data 等待控制器准备数据
	.dataWait1:
	xor AX, AX
	in AL, 64h
	bt AX, 0
	jnc .dataWait1

	;; Read the current port status from port 60h  从60h端口读取状态数据
	xor AX, AX
	in AL, 60h

	;; Save the current value of (E)AX		保存当前EAX值
	push AX			; 16-BIT
	;; push EAX		; 32-BIT

	;; Wait for the controller to be ready for a command 等待控制器
	.commandWait2:
	in AL, 64h
	bt AX, 1
	jc .commandWait2

	;; Tell the controller we want to write the status byte again 告诉控制器写状态字节
	mov AL, 0D1h
	out 64h, AL	

	;; Wait for the controller to be ready for the data
	.commandWait3:
	xor AX, AX
	in AL, 64h
	bt AX, 1
	jc .commandWait3

	;; Write the new value to port 60h.  Remember we saved the old
	;; value on the stack 写新的值到端口60h，老的值被压入栈
	pop AX			; 16-BIT
	;; pop EAX		; 32-BIT

	;; Turn on the A20 enable bit 开启A20开启位
	or AL, 00000010b
	out 60h, AL

	;; Finally, we will attempt to read back the A20 status
	;; to ensure it was enabled. 最后，尝试读取A20状态，确保它开启

	;; Wait for the controller to be ready for a command 等待控制器准备好接受命令
	.commandWait4:
	xor AX, AX
	in AL, 64h
	bt AX, 1
	jc .commandWait4

	;; Send the command D0h: read output port.	发送D0h命令，
	mov AL, 0D0h
	out 64h, AL	

	;; Wait for the controller to be ready with a byte of data 等待控制器准备好读取数据
	.dataWait2:
	xor AX, AX
	in AL, 64h
	bt AX, 0
	jnc .dataWait2

	;; Read the current port status from port 60h  从60h端口读取当前的端口状态
	xor AX, AX
	in AL, 60h

	;; Is A20 enabled?   判断A20地址线是否开启
	bt AX, 1

	;; Check the result.  If carry is on, A20 is on.
	jc .success

	;; Should we retry the operation?  If the counter value in ECX
	;; has not reached zero, we will retry 是否达到尝试次数上线 6
	loop .startAttempt1


	;; Well, our initial attempt to set A20 has failed.  Now we will
	;; try a backup method (which is supposedly not supported on many
	;; chipsets, but which seems to be the only method that works on
	;; other chipsets).  初始的尝试失败了，则尝试一个备用的方法。


	;; Keep a counter so that we can make up to 5 attempts to turn
	;; on A20 if necessary
	mov CX, 5

	.startAttempt2:
	;; Wait for the keyboard to be ready for another command 等待键盘准备好处理其他的命令
	.commandWait6:
	xor AX, AX
	in AL, 64h
	bt AX, 1
	jc .commandWait6

	;; Tell the controller we want to turn on A20  发送DFh命令
	mov AL, 0DFh
	out 64h, AL

	;; Again, we will attempt to read back the A20 status
	;; to ensure it was enabled.  读取A20状态，确认开启

	;; Wait for the controller to be ready for a command
	.commandWait7:
	xor AX, AX
	in AL, 64h
	bt AX, 1
	jc .commandWait7

	;; Send the command D0h: read output port.
	mov AL, 0D0h
	out 64h, AL	

	;; Wait for the controller to be ready with a byte of data
	.dataWait3:
	xor AX, AX
	in AL, 64h
	bt AX, 0
	jnc .dataWait3

	;; Read the current port status from port 60h
	xor AX, AX
	in AL, 60h

	;; Is A20 enabled?	判断状态值  是否开启了A20
	bt AX, 1

	;; Check the result.  If carry is on, A20 is on, but we might warn
	;; that we had to use this alternate method
	jc .warn

	;; Should we retry the operation?  If the counter value in ECX
	;; has not reached zero, we will retry
	loop .startAttempt2


	;; OK, we weren't able to set the A20 address line.  Do you want
	;; to put an error message here?
	jmp .fail


	.warn:
	;; Here you may or may not want to print a warning message about
	;; the fact that we had to use the nonstandard alternate enabling
	;; method

	.success:
	sti
	popa
	xor EAX, EAX
	ret

	.fail:
	sti
	popa
	mov EAX, -1
	ret
```

By Andy @ 2017/03/02 14:00:17 