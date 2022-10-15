---
title: "centos7上安装glustefs以及挂载到k8s中遇到的一些问题"
author: 一张狗
lastmod: 2019-07-06 08:08:45
date: 2018-08-16 10:55:44
tags: []
---



## 1.glusterfs启动的时候报错glusterd.service: control process exited, code=exited status=1


### 启动报错：

Redirecting to /bin/systemctl status  glusterd.service
glusterd.service – GlusterFS, a clustered file-system server
Loaded: loaded (/usr/lib/systemd/system/glusterd.service; enabled)
Active: failed (Result: exit-code) since Fri 2014-09-26 01:47:54 EDT; 5min ago
Process: 25116 ExecStart=/usr/sbin/glusterd -p /run/glusterd.pid (code=exited, status=1/FAILURE)

Sep 26 01:47:54 datasvr1 systemd[1]: glusterd.service: control process exited, code=exited status=1
Sep 26 01:47:54 datasvr1 systemd[1]: Failed to start GlusterFS, a clustered file-system server.
Sep 26 01:47:54 datasvr1 systemd[1]: Unit glusterd.service entered failed state.


### 原因：

glusterfs工作目录的问题


### 解决方法：

打开glusterfs配置文件(/etc/glusterfs/glusterd.vol)

volume management  
 type mgmt/glusterd

#working-directory的位置为/var/lib/glusterfsd  
 option working-directory /var/lib/glusterfsd

option transport-type socket,rdma

option transport.socket.keepalive-time 10  
 option transport.socket.keepalive-interval 2  
 option transport.socket.read-fail-log off  
 end-volume
 
清空working-directory指定目录下的内容。


## 2.挂载的时候报错version libssl.so.10 not defined in file libssl.so.10 with link time reference

首先查看openssl的版本，尝试更新 yum -y update openssl

如果不行，则查看libssl.so.10是哪个：ldconfig -p | grep libssl

解决问题的时候发现libssl.so.10跟可以运行的环境的libssl.so.10不一样，openssl都升级到了1.0.2g，但是能用的那个系统是软连接到了libssl.so.1.0.2k

尝试软连过去，问题解决！


