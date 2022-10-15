---
title: "threading"
author: 一张狗
lastmod: 2019-07-06 11:36:21
date: 2016-05-24 17:08:38
tags: []
---


进程是资源管理的最小单元；而线程是程序执行的最小单元。

一个进程的组成实体可以分为两大部分：线程集合和资源集合。进程中的线程是动态的对象；代表了进程指令的执行。资源，包括地址空间、打开的文件、用户信息等等，由进程内的线程共享。线程有自己的私有数据：程序计数器，栈空间以及寄存器。


为什么linux是轻量级进程？

linux所谓的线程其实是**与其他进程共享资源**的进程。  
 为什么说是轻量级？在于**它只有一个最小的执行上下文和调度程序所需的统计信息**。

用户级线程和内核级线程的区别：https://blog.csdn.net/gatieme/article/details/51892437

https://blog.csdn.net/gatieme/article/details/51482122

https://blog.csdn.net/gatieme/article/details/51892437

https://blog.csdn.net/gatieme/article/details/51481863

https://blog.csdn.net/gatieme/article/details/51482122


