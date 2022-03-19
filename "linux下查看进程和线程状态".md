---
title: "linux下查看进程和线程状态"
author: 一张狗
lastmod: 2018-08-15 14:18:06
date: 2018-08-15 14:18:06
tags: []
---


检查 使用 ps -fe |grep programname 查看获得进程的pid,再使用 ps -Lf pid 查看对应进程下的线程数.

查找资料发现可以通过设置 ulimit -s 来增加每进程线程数。 每进程可用线程数 = VIRT上限/stack size   32位x86系统默认的VIRT上限是3G（内存分配的3G+1G方式），64位x86系统默认的VIRT上限是64G

用 ulimit -s 可以查看默认的线程栈大小，一般情况下，这个值是 8M[8192]

查看最大线程数：

cat /proc/sys/kernel/threads-max

 

1.根据进程号进行查询：

# pstree -p 进程号

# top -Hp 进程号

 

**1、** cat /proc/${pid}/status

**2、**pstree -p ${pid}

**3、**top -p ${pid} 再按H 或者直接输入 top -bH -d 3 -p ${pid}

**top -H**  
 手册中说：-H : Threads toggle  
 加上这个选项启动top，top一行显示一个线程。否则，它一行显示一个进程。

**4、ps xH**  
 手册中说：H Show threads as if they were processes  
 这样可以查看所有存在的线程。

**5、ps -mp <PID>**  
 手册中说：m Show threads after processes  
 这样可以查看一个进程起的线程数。

ps -Lf pid|wc

ps -eLf |grep pid|grep -v grep

pstree -p `ps -aux | grep server | awk ‘{print $2}’` | wc -l

===

查看线程

其实[linux](http://lib.csdn.net/base/linux "Linux知识库")没有线程，都是用进程模仿的

1. ps -ef f  
 用树形显示进程和线程，比如说我想找到proftp现在有多少个进程/线程，可以用  
 $ ps -ef f | grep proftpd  
 nobody 23117 1 0 Dec23 ? S 0:00 proftpd:   (accepting   connections)  
 jack 23121 23117 0 Dec23 ? S 7:57 \_ proftpd: jack – ftpsrv:   IDLE  
 jack 28944 23117 0 Dec23 ? S 4:56 \_ proftpd: jack – ftpsrv:   IDLE  
 这样就可以看到proftpd这个进程下面挂了两个线程。  
 在Linux下面好像因为没有真正的线程，是用进程模拟的，有一个是辅助线程，所以真正程序开的线程应该只有一个。

2. pstree -c也可以达到相同的效果  
 $ pstree -c | grep proftpd  
 |-proftpd-+-proftpd  
 | `-proftpd

3. cat /proc/${pid}/status  
 可以查看大致的情况

4.  pstack

有些系统可以用这个东东，可以查看所有线程的堆栈

如何查看进程中各线程的内存占用情况？

用ps aux只能查看到进程，如果进程里面使用了pthread编程，用什么命令才能查询到进程里的线程资源占用？  
 ps aux | grep不就是了

 

查看进程

1. top 命令

top命令查看系统的资源状况

load average表示在过去的一段时间内有多少个进程企图独占CPU

zombie 进程 ：不是异常情况。一个进程从创建到结束在最后那一段时间遍是僵尸。留在内存中等待父进程取的东西便是僵尸。任何程序都有僵尸状态，它占用一点内存资源，仅仅是表象而已不必害怕。如果程序有问题有机会遇见，解决大批量僵尸简单有效的办法是重起。kill是无任何效果的stop模式：与sleep进程应区别，sleep会主动放弃cpu，而stop是被动放弃cpu ，例单步跟踪，stop(暂停)的进程是无法自己回到运行状态的。

cpu states：

nice：让出百分比irq：中断处理占用

idle：空间占用百分比 iowait：输入输出等待(如果它很大说明外存有瓶颈，需要升级硬盘(SCSI))

Mem：内存情况

设计思想：把资源省下来不用便是浪费，如添加内存后free值会不变，buff值会增大。 判断物理内存够不够，看交换分区的使用状态。

交互命令：

[Space]立即刷新显示

[h]显示帮助屏幕

[k] 杀死某进程。你会被提示输入进程 ID 以及要发送给它的信号。 一般的终止进程可以使用15信号;如果不能正常结束那就使用信号9强制结束该进程。默认值是信号15。在安全模式中此命令被屏蔽。

[n] 改变显示的进程数量。你会被提示输入数量。

[u] 按用户排序。

[M] 按内存用量排序。

[o][O] 改变显示项目的顺序。

[P] 根据CPU使用百分比大小进行排序。

[T] 根据时间/累计时间进行排序。

[Ctrl+L] 擦除并且重写屏幕。

[q] 退出程序。

[r] 重新安排一个进程的优先级别。系统提示用户输入需要改变的进程PID以及需要设置的进程优先级值。输入一个正值将使优先级降低，反之则可以使该进程拥有更高的优先权。默认值是10。

[S] 切换到累计模式。

[s] 改变两次刷新之间的延迟时间。系统将提示用户输入新的时间，单位为s。如果有小数，就换算成m s。输入0值则系统将不断刷新，默认值是5 s。需要注意的是如果设置太小的时间，很可能会引起不断刷新，从而根本来不及看清显示的情况，而且系统负载也会大大增加。

缩写含义：

PID每个进程的ID

USER进程所有者的用户名

PRI每个进程的优先级别

NI每个优先级的值

SIZE 进程的代码大小加上数据大小再加上堆栈空间大小的总数，单位是KB RSS 进程占用的物理内存的总数量，单位是KB

SHARE进程使用共享内存的数量

STAT 进程的状态。其中S代表休眠状态;D代表不可中断的休眠状态;R代表运行状态;Z代表僵死状态;T代表停止或跟踪状态

%CPU进程自最近一次刷新以来所占用的CPU时间和总时间的百分比

%MEM进程占用的物理内存占总内存的百分比

TIME进程自启动以来所占用的总CPU时间

CPU CPU标识

COMMAND进程的命令名称

2. ps命令

ps查看当前用户的活动进程，如果加上参数可以显示更多的信息，如-a，显示所有用户的进程

ps ax ：tty值为“?”是守护进程，叫deamon 无终端，大多系统服务是此进程，内核态进程是看不到的

ps axf ：看进程树，以树形方式现实进程列表敲 ，init是1号进程，系统所有进程都是它派生的，杀不掉

ps axm ：会把线程列出来。在[Linux](http://lib.csdn.net/base/linux "Linux知识库")下进程和线程是统一的，是轻量级进程的两种方式。

ps axu ：显示进程的详细状态。

vsz：说此进程一共占用了多大物理内存。

rss：请求常驻内存多少


