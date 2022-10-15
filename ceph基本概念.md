---
title: "ceph基本概念"
author: 一张狗
lastmod: 2019-07-06 09:38:31
date: 2018-06-11 11:37:41
tags: []
---


参考：http://www.cnblogs.com/kevingrace/p/8430213.html

https://tobegit3hub1.gitbooks.io/ceph_from_scratch/content/introduction/introduction.html

https://blog.csdn.net/aixiaoyang168/article/details/78825850

![ceph_all_component](http://yizhanggou.top/imgs/2019/07/ceph_all_component.png)

1）OSDs: Ceph的OSD守护进程（OSD）存储数据，处理数据复制，恢复，回填，重新调整，并通过检查其它Ceph OSD守护程序作为一个心跳 向Ceph的监视器报告一些检测信息。Ceph的存储集群需要至少2个OSD守护进程来保持一个 active + clean状态.（Ceph默认制作2个备份，但可以调整它）

2）Monitors:Ceph的监控保持集群状态映射，包括OSD(守护进程)映射,分组(PG)映射，和CRUSH映射。 Ceph 保持一个在Ceph监视器, Ceph OSD 守护进程和 PG的每个状态改变的历史（称之为“epoch”）。

3）MDS: MDS是Ceph的元数据服务器，代表存储元数据的Ceph文件系统（即Ceph的块设备和Ceph的对象存储不使用MDS）。Ceph的元数据服务器使用POSIX文件系统，用户可以执行基本命令如 ls, find,等，并且不需要在Ceph的存储集群上造成巨大的负载。

Ceph把客户端的数据以对象的形式存储到了存储池里。利用CRUSH算法，Ceph可以计算出安置组所包含的对象，并能进一步计算出Ceph OSD集合所存储的安置组。CRUSH算法能够使Ceph存储集群拥有动态改变大小、再平衡和数据恢复的能力。

**Ceph提供了3种使用场景**

- 分布式文件系统CephFS。多个客户端mount CephFS到本地，CephFS遵循POSIX接口，使用体验类似于ext4等本地文件系统。类似于其他分布式文件系统，各个CephFS客户端共享同一命名空间。

- RadosGW（rgw）对象存储。rgw使用场景类似于Amazon S3，据个人理解也类似于七牛云存储。

- 块设备rbd（Rados Block Device）。Ceph提供虚拟的rbd块设备，用户像使用SATA盘那样的物理块设备一样使用rbd。rbd的使用是排他的，每个rbd块设备是用户私有的，相对的，CephFS的使用方式是共享的。虚拟化和云计算的发展正当盛年，IaaS结合rbd块设备这一使用方式有如干柴遇烈火，因此rbd是Ceph社区开发的重心之一。本文也主要从rbd的视角来了解Ceph。


CephFS和CephRBD的区别：

- cephfs是一个文件系统，跟NFS差不多，是通过网络共享的文件系统。
- rbd是一个块设备，更像是一个通过网络共享的硬盘映射。
- 使用场景：如果你需要在多个机器中共享一堆文件，则使用cephfs。如果你想要存储磁盘映像，被虚拟机使用，则用rbd。如果你想与亚马逊的S3兼容，则使用radosgw。


