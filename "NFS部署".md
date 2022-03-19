---
title: "NFS部署"
author: 一张狗
lastmod: 2019-07-06 09:33:31
date: 2018-06-14 12:04:49
tags: []
---



## 一、部署

### 服务端
```
chkconfig nfs on  
 chkconfig rpcbind on  
 service rpcbind start  
 service nfs start  
 mkdir -p <your_folder>  
 vim /etc/exports  
 <your_folder> 10.133.0.0/16(rw,async,no_root_squash)  
 exportfs -a
```
### 客户端:
```
chkconfig nfs on  
 chkconfig rpcbind on  
 service rpcbind start  
 service nfs start  
 mkdir -p <your_local_folder>  
 mount -t nfs server_ip:<your_server_folder> <your_local_folder>  
 查看：df -h
```
> [CentOS下的NFS安装、配置、启动及mount挂载方法](http://www.oicto.com/unix-nfs-mount/)
 
 （测试soft： `mount -t nfs -o soft,intr,timeo=30,retry=3` ）


