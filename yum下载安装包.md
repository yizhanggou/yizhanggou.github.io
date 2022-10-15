---
title: "yum下载安装包"
author: 一张狗
lastmod: 2019-07-06 08:10:01
date: 2018-08-16 10:52:46
tags: []
---


1. 查看依赖关系 - `yum deplist libcurl`
- 或者已经安装的可以`ldd <your_bin>` 找到后tar就行
2. 下载指定版本 - `yum -y install yum-utils`
- 找到可提供的版本 - `yum list kernel-debuginfo –showduplicates`
- 如果直接install是`yum install kernel-debuginfo-2.6.32-220.el6`
- 如果要下载则看下一步
- 下载单个rpm包 
    - yumdownloader <package-name> 
    - 比如`yumdownloader tree.x86_64 1.5.3-3.el6`
- 前面是包，后面是版本号
- 指定目录和把依赖的包也下载下来：`yumdownloader –resolve –destdir=/home/work/linhongyun/glusterfs-package/ centos-release-gluster41-1.0-3.el7.centos.noarch`
- 下载rpm包和依赖关系包 - `yum install -y yum-plugin-downloadonly.noarch`
- `yum -y install –downloadonly –downloaddir=/soft/ openssh.x86_64  5.3p1-118.1.el6_8`
3. 安装离线包 - `yum localinstall -y <your_folder>*.rpm`


