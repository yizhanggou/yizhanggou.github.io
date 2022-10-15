---
title: "python multiprocess 抛 This platform lacks a functioning sem_open implementation, therefore, the required synchronization primitives needed will not "
author: 一张狗
lastmod: 2018-06-26 14:29:00
date: 2018-06-26 14:29:00
tags: []
---


原因：没有挂载/dev/shm;python安装时需要写入东西，才能开启sem_open  
 解决办法：（需要root权限；以下命令root账号执行）

1. 修改/etc/fstab 增加：tmpfs /dev/shm tmpfs defaults 0 0
2. mount /dev/shm
3. chmod 777 /dev/shm
4. 重装python （必须重装，没办法，安装时检查该设备是否存在，存在则可以使用sem_open）


