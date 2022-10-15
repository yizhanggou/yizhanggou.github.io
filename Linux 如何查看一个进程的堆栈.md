---
title: "Linux 如何查看一个进程的堆栈"
author: 一张狗
lastmod: 2019-07-06 10:16:57
date: 2018-03-15 15:39:13
tags: []
---



# 1.用pstack

第一种：pstack 进程ID

第二种，使用gdb 然后attach 进程ID，然后再使用命令 thread apply all bt

两种方法都可以列出进程所有的线程的当前的调用栈。

不过，使用gdb的方法，还可以查看某些信息，例如局部变量，指针等。

不过，如果只看调用栈的话，pstack还是很方便的。

https://www.ibm.com/developerworks/cn/linux/l-cn-deadlock/


# 2.用gstack

gstack -打印正在运行的进程的堆栈跟踪

使用方法：

gstack PID

描述

gstack连接到命令行中pid的活动进程

打印执行堆栈跟踪。如果ELF符号存在于二进制(usu -)中

如果你没有运行条带(1)，那么这个例子就会被打印出来

同样如此。

如果进程是线程组的一部分，那么gstack将打印出一个堆栈

对组中的每个线程进行跟踪。


