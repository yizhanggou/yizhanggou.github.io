---
title: "C++线程同步     双缓存机制加thread-local锁的解决方案"
author: 一张狗
lastmod: 2019-07-06 09:43:03
date: 2018-06-24 18:55:47
tags: []
---


C++并发指南：http://www.cnblogs.com/haippy/p/3346477.html

首先说明几个知识点：

1.C++的内存模型

https://www.cnblogs.com/navono007/p/5746048.html

C++11提供了6种内存模型：

enum memory_order{ 
    memory_order_relaxed, 
    memory_order_consume, 
    memory_order_acquire, 
    memory_order_release, 
    memory_order_acq_rel, 
    memory_order_seq_cst 
}

原子类型的操作可以指定上述6种模型的中的一种，用来控制同步以及对执行序列的约束。从而也引起两个重要的问题：

1.哪些原子类型操作需要使用内存模型？  
 2.内存模型定义了那些同步语义（synchronization ）和执行序列约束（ordering constraints）？

原子操作可分为3大类：

**读操作：**memory_order_acquire, memory_order_consume  
**写操作：**memory_order_release  
**读-修改-写操作：**memory_order_acq_rel, memory_order_seq_cst

未被列入分类的memory_order_relaxed没有定义任何同步语义和顺序一致性约束


## 执行序列约束

C++11中有3种不同类型的同步语义和执行序列约束：

**1. 顺序一致性（Sequential consistency）**：对应的内存模型是memory_order_seq_cst

**2.请求-释放（Acquire-release）**：对应的内存模型是memory_order_consume，memory_order_acquire，memory_order_release，memory_order_acq_rel

**3.松散型（非严格约束。Relaxed）**：对应的内存模型是memory_order_relaxed

下面对上述3种约束做一个大概解释：

Sequential consistency：指明的是在线程间，建立一个全局的执行序列

Acquire-release：在线程间的同一个原子变量的读和写操作上建立一个执行序列

Relaxed：只保证在同一个线程内，同一个原子变量的操作的执行序列不会被重排序（reorder），这种保证也称之为modification order consistency，但是其他线程看到的这些操作的执行序列式不同的。

还有一种consume模式，也就是std::memory_order_consume。这个模式主要是引入了原子变量的数据依赖。

 

2.std::atomic::load：

`T load( std::memory_order order = std::memory_order_seq_cst ) const noexcept;  
 T load( std::memory_order order = std::memory_order_seq_cst ) const volatile noexcept;  `
 原子地加载并放回原子变量的当前值。按照 order 的值影响内存。
```
std::atomic::store  
 void store( T desired, std::memory_order order = std::memory_order_seq_cst ) noexcept;  
 void store( T desired, std::memory_order order = std::memory_order_seq_cst ) volatile noexcept;  
```
 原子地以 desired 替换当前值。按照 order 的值影响内存。

3.thread_local变量

thread_local关键字修饰的变量具有线程周期(thread duration)，这些变量(或者说对象）在线程开始的时候被生成(allocated)，在线程结束的时候被销毁(deallocated)。并且每 一个线程都拥有一个独立的变量实例。

https://www.cnblogs.com/pop-lar/p/5123014.html

https://blog.csdn.net/jasonchen_gbd/article/details/51367650

 

4.RAII风格

https://zhuanlan.zhihu.com/p/34660259

http://www.voidcn.com/article/p-osbngvxv-ca.html

RAII（Resource Acquisition Is Initialization），使用局部对象来管理资源的技术称为资源获取即初始化；这里的资源主要是指操作系统中有限的东西如内存、网络套接字等等，局部对象是指存储在栈的对象，它的生命周期是由操作系统来管理的，**无需人工介入**;

资源的使用一般经历三个步骤a.获取资源 b.使用资源 c.销毁资源，但是资源的销毁往往是程序员经常忘记的一个环节，所以程序界就想如何在程序员中让资源自动销毁呢？c++之父给出了解决问题的方案：RAII，它充分的利用了C++语言局部对象自动销毁的特性来控制资源的生命周期。

即是说，符合RAII的资源放在一个局部变量中，就可以在作用域结束的时候自动销毁。


5.lock_guard

https://blog.csdn.net/tgxallen/article/details/73522233

类 lock_guard 是互斥封装器，为在作用域块期间占有互斥提供便利 RAII 风格机制。

创建 lock_guard 对象时，它试图接收给定互斥的所有权。控制离开创建 lock_guard 对象的作用域时，销毁 lock_guard 并释放互斥。


6.condition_variable

https://zh.cppreference.com/w/cpp/thread/condition_variable

condition_variable 类是同步原语，能用于阻塞一个线程，或同时阻塞多个线程，直至另一线程修改共享变量（条件）并通知 condition_variable 。



7.pthread_getspecific和pthread_setspecific使用

https://blog.csdn.net/vevenlcf/article/details/77882985

https://www.jianshu.com/p/d52c1ebf808a



8.double buffer和无锁编程

http://wiki.baidu.com/display/RPC/Locality-aware+load+balancing

http://wiki.baidu.com/pages/viewpage.action?pageId=293142706


