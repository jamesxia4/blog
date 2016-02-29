---
title: JVM学习笔记
date: 2016-01-19 16:34:10
tags:
---

##运行时数据区域

线程隔离: 
	虚拟机栈 VM Stack
		线程私有
		每个方法在执行的时候都会创建一个栈帧，放局部变量表，操作数，方法出口，动态链接
		常说的"栈"指的就是这里面的局部变量表
	
        本地方法栈 Native Method Stack
		
        程序计数器 Program Counter Register
		JAVA虚拟机的多线程是通过线程轮流切换并分配处理器执行时间实现的
		所以在任何一个确定的时刻，一个处理器都只会执行一条线程中的指令
		为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器。

		如果执行的是Java方法，PC记录的是正在执行的虚拟机字节码指令的地址
 		如果是Native方法，这里是Undefined
		
		这里不会出现OOM Error

线程共享数据区:
	方法区  Method Area
	堆  	Heap


##To be continued


