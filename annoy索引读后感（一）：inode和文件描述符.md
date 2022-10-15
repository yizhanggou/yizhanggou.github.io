---
title: "annoy索引读后感（一）：inode和文件描述符"
author: 一张狗
lastmod: 2019-07-06 08:45:28
date: 2018-07-31 14:14:36
tags: []
---



## 一、概念

一个操作系统需要编个目录来说明每个文件在什么地方，有什么属性，及大小等，这目录是操作系统需要的，用来找文件或叫管理文件。许多操作系统都用到这个概念，如linux， 某些嵌入式文件系统等。当然，对某个系统来说，有许多i节点。所以对i节点本身也是要进行管理的。

在linux中，内核通过inode来找到每个文件，但一个文件可以被许多用户同时打开或一个用户同时打开多次。这就有一个问题，如何管理文件的当前位移量，因为可能每个用户打开文件后进行的操作都不一样，这样文件位移量也不同，当然还有其他的一些问题。所以linux又搞了一个文件描述符（file descriptor）这个东西，来分别为每一个用户服务。每个用户每次打开一个文件，就产生一个文件描述符，多次打开就产生多个文件描述符，一一对应，不管是同一个用户，还是多个用户。该文件描述符就记录了当前打开的文件的偏移量等数据。所以一个i节点可以有0个或多个文件描述符。多个文件描述符可以对应一个i节点。
```
struct inode {
　　struct list_headi_hash;
　　struct list_headi_list;
　　struct list_headi_dentry;
　　struct list_headi_dirty_buffers;
　　unsigned longi_ino; /*每一个inode都有一个序号，经由super block结构和其序号，我们可以很轻易的找到这个inode。*/
　　atomic_t i_count; /*在Kernel里，很多的结构都会记录其reference count，以确保如果某个结构正在使用，它不会被不小心释放掉，i_count就是其reference count。*/
　　kdev_t i_dev; /* inode所在的device代码 */
　　umode_t i_mode; /* inode的权限 */
　　nlink_t i_nlink; /* hard link的个数 */
　　uid_t i_uid; /* inode拥有者的id */
　　gid_t i_gid; /* inode所属的群组id */
　　kdev_t i_rdev; /* 如果inode代表的是device的话，那此字段将记录device的代码 */
　　off_t i_size; /* inode所代表的档案大小 */
　　time_t i_atime; /* inode最近一次的存取时间 */
　　time_t i_mtime; /* inode最近一次的修改时间 */
　　time_t i_ctime; /* inode的产生时间 */
　　unsigned long i_blksize; /* inode在做IO时的区块大小 */
　　unsigned long i_blocks; /* inode所使用的block数，一个block为512 byte*/
　　unsigned long i_version; /* 版本号码 */
　　unsigned short i_bytes;
　　struct semaphore i_sem;
　　struct rw_semaphore i_truncate_sem;
　　struct semaphore i_zombie;
　　struct inode_operations *i_op;
　　struct file_operations *i_fop;/* former ->i_op->default_file_ops */
　　struct super_block *i_sb; /* inode所属档案系统的super block */
　　wait_queue_head_t i_wait;
　　struct file_lock *i_flock; /* 用来做file lock */
　　struct address_space *i_mapping;
　　struct address_space i_data;
　　struct dquot *i_dquot [MAXQUOTAS];
　　/* These three should probably be a union */
　　struct pipe_inode_info *i_pipe;
　　struct block_device *i_bdev;
　　struct char_device *i_cdev;
　　unsigned longi_dnotify_mask; /* Directory notify events */
　　struct dnotify_struct *i_dnotify; /* for directory notifications */
　　unsigned long i_state; /* inode目前的状态，可以是I_DIRTY，I_LOCK和 I_FREEING的OR组合 */
　　unsigned int i_flags; /* 记录此inode的参数 */
　　unsigned char i_sock; /* 用来记录此inode是否为socket */
　　atomic_t i_write count;
　　unsigned int i_attr_flags; /* 用来记录此inode的属性参数 */
　　__u32 i_generation;
　　union {
　　struct minix_inode_info minix_i;
　　struct ext2_inode_info ext2_i;
　　struct ext3_inode_info ext3_i;
　　struct hpfs_inode_info hpfs_i;
　　struct ntfs_inode_info ntfs_i;
　　struct msdos_inode_info msdos_i;
　　struct umsdos_inode_info umsdos_i;
　　struct iso_inode_info isofs_i;
　　struct sysv_inode_info sysv_i;
　　struct affs_inode_info affs_i;
　　struct ufs_inode_info ufs_i;
　　struct efs_inode_info efs_i;
　　struct romfs_inode_info romfs_i;
　　struct shmem_inode_info shmem_i;
　　struct coda_inode_info coda_i;
　　struct smb_inode_info smbfs_i;
　　struct hfs_inode_info hfs_i;
　　struct adfs_inode_info adfs_i;
　　struct qnx4_inode_info qnx4_i;
　　struct reiserfs_inode_info reiserfs_i;
　　struct bfs_inode_info bfs_i;
　　struct udf_inode_info udf_i;
　　struct ncp_inode_info ncpfs_i;
　　struct proc_inode_info proc_i;
　　struct socketsocket_i;
　　struct usbdev_inode_info usbdev_i;
　　struct jffs2_inode_infojffs2_i;
　　void *generic_ip;
　　} u;
　　};
```

dentry：

dentry是directory entry的缩写，dentry中包含具体的文件名和指向inode的指针等信息，也就是说通过dentry可以找到对应的inode，再通过inode找到文件存储的block位置。这里我画了一个简单的示例图(图1)，来说明dentry和inode之间的具体关系。

![t_35661_1381231351_830522434](http://yizhanggou.top/imgs/2019/07/t_35661_1381231351_830522434.png)

每一个进程在pcb中保存着一份文件描述符表，而文件描述符就是这个表的索引，这里进程打开/home/wsl/test文件，文件描述符为3，其中文件描述符表项中又有一个指向已打开文件的指针，已打开的文件在内核中用file结构体表示，包括打开的标志位，读写的位置f_pos，引用计数(f_count)以及指向dentry结构体的指针(f_dentry)等信息。为了减少读盘次数，内核都缓存了目录的树状结构，称为dentry cache，这里面每一个节点都是一个dentry结构体【正如前面介绍的，dentry中保存着文件名信息】。dentry结构体中都有一个指针指向inode结构体，因此只要沿着路径各部分的dentry搜索即可找到进程要访问的文件的inode结构体，从而获取文件的inode信息，进行文件的具体操作。

linux系统内部不使用文件名，而是使用inode来识别文件，用户通过文件名打开文件，实际上是首先通过dentry获取文件的inode信息，然后根据读取的inode信息来进行文件的处理。


## 二、各个操作对inode的影响

我们来看下cp和mv对文件inode的影响

> [linux下cp，mv进行动态库覆盖问题分析](https://www.bo56.com/linux%e4%b8%8bcp%ef%bc%8cmv%e8%bf%9b%e8%a1%8c%e5%8a%a8%e6%80%81%e5%ba%93%e8%a6%86%e7%9b%96%e9%97%ae%e9%a2%98%e5%88%86%e6%9e%90/)

```
[work@machine ~]$ touch t1 t2 && ls -i t1 t2 
35743882 t1 35743884 t2 
[work@machine ~]$ cp t1 t2 && ls -i t1 t2 
35743882 t1 35743884 t2//将t1 cp成t2，但t2的inode号和原始的t2保持一致 
[work@machine ~]$ mv t1 t2 && ls -i t2 
35743882 t2//将t1 mv成t2，t2的inode号为原始t1的inode号 
[work@machine ~]$ cp t2 t3 && ls -i t2 t3 
35743882 t2 35743885 t3//cp到一个不存在的文件t3，t3为新的inode号
```

下面分析各个命令：

cp命令：

如果目标不存在，则分配一个新的inode号，如果目标存在，则沿用原来的inode号；​在目录中新建一个dentry，并指向步骤1)中的inode；​把数据复制到block中。

mv命令：

a.如果mv命令的目标和源文件所在的文件系统相同：  
 1）使用新文件名建立dentry  
 2）删除带有原来文件名的dentry； 【该操作对inode表没有影响（除时间戳），对数据的位置也没有影响，不移动任何数据。（即使是mv到一个已经存在的目标文件，新目录项指源文件inode，会先删除目标文件的dentry）】  
 b.如果目标和源文件所在文件系统不相同，就是cp和rm；

rm命令：

1.递减链接计数，从而释放inode号码，这个inode号码可以被重用  
 2.把数据块挂到可用空间列表  
 3.删除目录映射表中的相关行 但是底层数据实际上没有被删除，只是当数据块被另一个文件使用时，原来的数据才会被覆盖  
 简单总结下：  
 - cp命令到一个已经存在的文件，inode号沿用已经存在文件的inode号；  
 - mv命令用新的inode号，也就是mv前的文件的inode号；  
 - rm命令删除的底层数据只有被使用的时候才会被覆盖。


## 三、问题：cp mv so文件的区别？

（strace命令可以跟踪到一个进程产生的系统调用,包括参数，返回值，执行消耗的时间，调试利器）

首先我们用strace查看cp的时候发生了什么：

```
[work@machine ~]$ strace cp t2 t3
execve("/usr/bin/cp", ["cp", "t2", "t3"], [/* 41 vars */]) = 0
………………
geteuid() = 1001
stat("t3", {st_mode=S_IFREG|0664, st_size=0, ...}) = 0
stat("t2", {st_mode=S_IFREG|0664, st_size=0, ...}) = 0
stat("t3", {st_mode=S_IFREG|0664, st_size=0, ...}) = 0
open("t2", O_RDONLY) = 3
fstat(3, {st_mode=S_IFREG|0664, st_size=0, ...}) = 0
open("t3", O_WRONLY|O_TRUNC) = 4
fstat(4, {st_mode=S_IFREG|0664, st_size=0, ...}) = 0
fadvise64(3, 0, 0, POSIX_FADV_SEQUENTIAL) = 0
read(3, "", 65536) = 0
close(4) = 0
close(3) = 0
lseek(0, 0, SEEK_CUR) = -1 ESPIPE (Illegal seek)
close(0) = 0
close(1) = 0
close(2) = 0
exit_group(0) = ?
+++ exited with 0 +++
```

可以看到首先以只读的方式打开t2，然后以以写加截断(O_WRONLY|O_TRUNC)的方式打开t3（O_TRUNC的含义：若文件存在，则长度被截为0，属性不变），然后将t2内容写到t3，然后关闭文件。

mv命令：

```
[work@machine ~]$ strace mv t3 t4
execve("/usr/bin/mv", ["mv", "t3", "t4"], [/* 41 vars */]) = 0
……………………
geteuid() = 1001
ioctl(0, TCGETS, {B38400 opost isig icanon echo ...}) = 0
stat("t4", 0x7fffa18ddff0) = -1 ENOENT (No such file or directory)
lstat("t3", {st_mode=S_IFREG|0664, st_size=0, ...}) = 0
lstat("t4", 0x7fffa18ddca0) = -1 ENOENT (No such file or directory)
rename("t3", "t4") = 0
lseek(0, 0, SEEK_CUR) = -1 ESPIPE (Illegal seek)
close(0) = 0
close(1) = 0
close(2) = 0
exit_group(0) = ?
+++ exited with 0 +++
```

重点是rename这一句。



当用新的so文件去覆盖老的so文件时候：  
 A)如果so里面依赖了外部符号（比如printf），程序会core掉  
 B)如果so里面没有依赖外部符号，so部分代码可以正常运行



**总结：**

整理完这四部分，回到最开始的问题“为什么cp新的so文件替换老的so，程序会core掉的根本原因是什么？”，现在串联起来总结如下。  
 1. cp new.so old.so，文件的inode号没有改变，dentry找到是新的so，但是cp过程中会把老的so截断为0，这时程序再次进行加载的时候，如果需要的文件偏移大于新的so的地址范围会生成buserror导致程序core掉，或者由于全局符号表没有更新，动态库依赖的外部函数无法解析，会产生sigsegv从而导致程序core掉，当然也有一定的可能性程序继续执行，但是十分危险。  
 2. mv new.so old.so，文件的inode号会发生改变，但老的so的inode号依旧存在，这时程序必须停止重启服务才能继续使用新的so，否则程序继续执行，使用的还是老的so，所以程序不会core掉。


