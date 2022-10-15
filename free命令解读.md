---
title: "free命令解读"
author: 一张狗
lastmod: 2019-07-06 07:59:58
date: 2018-09-09 18:07:53
tags: []
---


太长不看版本：

[root@node ~]# free -m total used free shared buff/cache available Mem: 128656 1050 83213 194 44392 126530 Swap: 0 0

***buffer***：缓冲区，程序写到硬盘的数据会先放到buffer中，然后再由硬盘读取出来。解决内存写和写硬盘速度不一致；
***cache***：缓存，系统读取文件的时候会把文件缓存在内存中，程序结束了也不释放。解决硬盘读与内存速度不一致问题；
***swap***：系统负载太大的时候，内存被用完，swap把一部分硬盘空间当内存用；
对程序来说buffer和cache是想要用的时候可以立即使用的内存，所以对于程序来说可用的内存空间有free+buffer/cache；应用程序实际使用的内存是used – buffers/cache；理论上可以被使用的内存是free + buffers/cached；

1. free命令显示系统使用和空闲的内存情况，包括物理内存、交互区内存(swap)和内核缓冲区内存。共享内存将被忽略；
2. 参数： - -b 　以Byte为单位显示内存使用情况。  
 -k 　以KB为单位显示内存使用情况。  
 -m 　以MB为单位显示内存使用情况。  
 -g 以GB为单位显示内存使用情况。  
 -o 　不显示缓冲区调节列。  
 -s<间隔秒数> 　持续观察内存使用状况。  
 -t 　显示内存总和列。  
 -V 　显示版本信息。
3. [root@machine ~]# free -g total used free shared buffers cached Mem: 157 119 37 0 0 82 -/+ buffers/cache: 35 121 Swap: 0 0 0
4. -/+ buffers/cache的意思相当于：  
 -buffers/cache 的内存数：1397032 (等于第1行的 used – buffers – cached)  
 +buffers/cache 的内存数: 2752124 (等于第1行的 free + buffers + cached)可见-buffers/cache反映的是被程序实实在在吃掉的内存，而+buffers/cache反映的是可以挪用的内存总数。第三行单独针对交换分区, 就不用再说了.为了提高磁盘存取效率, Linux做了一些精心的设计, 除了对dentry进行缓存(用于VFS,加速文件路径名到inode的转换), 还采取了两种主要Cache方式：Buffer Cache和Page Cache。前者针对磁盘块的读写，后者针对文件inode的读写。这些Cache有效缩短了 I/O系统调用(比如read,write,getdents)的时间。
5. cache和buffer的区别：http://www.cnblogs.com/peida/archive/2012/12/25/2831814.html，https://www.cnblogs.com/chenpingzhao/p/5161844.html 
#### 1、page cahe和buffer cache**
Page cache实际上是针对文件系统的，是文件的缓存，在文件层面上的数据会缓存到page cache。文件的逻辑层需要映射到实际的物理磁盘，这种映射关系由文件系统来完成。当page cache的数据需要刷新时，page cache中的数据交给buffer cache，但是这种处理在2.6版本的内核之后就变的很简单了，没有真正意义上的cache操作。

Buffer cache是针对磁盘块的缓存，也就是在没有文件系统的情况下，直接对磁盘进行操作的数据会缓存到buffer cache中，例如，文件系统的元数据都会缓存到buffer cache中。  
 简单说来，page cache用来缓存文件数据，buffer cache用来缓存磁盘数据。在有文件系统的情况下，对文件操作，那么数据会缓存到page cache，如果直接采用dd等工具对磁盘进行读写，那么数据会缓存到buffer cache。

补充一点，在文件系统层每个设备都会分配一个def_blk_ops的文件操作方法，这是设备的操作方法，在每个设备的inode下面会存在一个radix tree，这个radix tree下面将会放置缓存数据的page页。这个page的数量将会在top程序的buffer一栏中显示。如果设备做了文件系统，那么会生成一个inode，这个inode会分配ext3_ops之类的操作方法，这些方法是文件系统的方法，在这个inode下面同样存在一个radix tree，这里会缓存文件的page页，缓存页的数量在top程序的cache一栏进行统计。从上面的分析可以看出，2.6内核中的buffer cache和page cache在处理上是保持一致的，但是存在概念上的差别，page cache针对文件的cache，buffer是针对磁盘块数据的cache，仅此而已。

#### 2、cache 和 buffer的区别

A buffer is something that has yet to be “written” to disk. A cache is something that has been “read” from the disk and stored for later use ; 对于共享内存（Shared memory），主要用于在UNIX 环境下不同进程之间共享数据，是进程间通信的一种方法，一般的应用程序不会申请使用共享内存

Cache：高速缓存，是位于CPU与主内存间的一种容量较小但速度很高的存储器。由于CPU的速度远高于主内存，CPU直接从内存中存取数据要等待一定时间周期，Cache中保存着CPU刚用过或循环使用的一部分数据，当CPU再次使用该部分数据时可从Cache中直接调用，这样就减少了CPU的等待时间，提高了系统的效率。Cache又分为一级Cache（L1 Cache）和二级Cache（L2 Cache），L1 Cache集成在CPU内部，L2 Cache早期一般是焊在主板上，现在也都集成在CPU内部，常见的容量有256KB或512KB L2 Cache

它是根据程序的局部性原理而设计的，就是cpu执行的指令和访问的数据往往在集中的某一块，所以把这块内容放入cache后，cpu就不用在访问内存了，这就提高了访问速度。当然若cache中没有cpu所需要的内容，还是要访问内存的

**查看CPU的 L1、L2、L3**

```
[root@AY1301180424258d59678 ~]
# ll /sys/devices/system/cpu/cpu0/cache/
total 0
drwxr-xr-x 2 root root 0 Jan 26 22:49 index0 
#一级cache中的data和instruction cache
drwxr-xr-x 2 root root 0 Jan 26 22:49 index1 
#一级cache中的data和instruction cache
drwxr-xr-x 2 root root 0 Jan 26 22:49 index2 
#二级cache，共享的
drwxr-xr-x 2 root root 0 Jan 26 22:49 index3 
#三级cache，共享的 
```
Buffer：缓冲区，一个用于存储速度不同步的设备或优先级不同的设备之间传输数据的区域。通过缓冲区，可以使进程之间的相互等待变少，从而使从速度慢的设备读入数据时，速度快的设备的操作进程不发生间断。

#### 3、Free中的buffer和cache （它们都是占用内存）基于内存的**  
**

buffer ：作为buffer cache的内存，是块设备的读写缓冲区

cache：作为page cache的内存， 文件系统的cache

如果 cache 的值很大，说明cache住的文件数很多。如果频繁访问到的文件都能被cache住，那么磁盘的读IO 必会非常小

**如何释放Cache Memory**
```
To free pagecache:
echo 1 > /proc/sys/vm/drop_caches
To free dentries and inodes:
echo 2 > /proc/sys/vm/drop_caches
To free pagecache, dentries and inodes:
echo 3 > /proc/sys/vm/drop_caches
#注意，释放前最好sync一下，防止丢失数据，但是一般情况下没
```
#### 4、总结

#### cached是cpu与内存间的，buffer是内存与磁盘间的，都是为了解决速度不对等的问题

- 缓存（cached）是把读取过的数据保存起来，重新读取时若命中（找到需要的数据）就不要去读硬盘了，若没有命中就读硬盘。其中的数据会根据读取频率进行组织，把最频繁读取的内容放在最容易找到的位置，把不再读的内容不断往后排，直至从中删除
- 缓冲（buffers）是根据磁盘的读写设计的，把分散的写操作集中进行，减少磁盘碎片和硬盘的反复寻道，从而提高系统性能。linux有一个守护进程定期 清空缓冲内容（即写入磁盘），也可以通过sync命令手动清空缓冲。举个例子吧：我这里有一个ext2的U盘，我往里面cp一个3M的MP3，但U盘的灯 没有跳动，过了一会儿（或者手动输入sync）U盘的灯就跳动起来了。卸载设备时会清空缓冲，所以有些时候卸载一个设备时要等上几秒钟
- 修改/etc/sysctl.conf中的vm.swappiness右边的数字可以在下次开机时调节swap使用策略。该数字范围是0～100，数字越大越倾向于使用swap。默认为60，可以改一下试试。–两者都是RAM中的数据

<div>**buffer是即将要被写入磁盘的，而cache是被从磁盘中读出来的**</div><div></div><div>- buffer是由各种进程分配的，被用在如输入队列等方面。一个简单的例子如某个进程要求有多个字段读入，在所有字段被读入完整之前，进程把先前读入的字段放在buffer中保存
- cache经常被用在磁盘的I/O请求上，如果有多个进程都要访问某个文件，于是该文件便被做成cache以方便下次被访问，这样可提高系统性能
- Buffer Cachebuffer cache，又称bcache，其中文名称为缓冲器高速缓冲存储器，简称缓冲器高缓。另外，buffer cache按照其工作原理，又被称为块高缓

</div>


