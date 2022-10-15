---
title: "ceph命令"
author: 一张狗
lastmod: 2018-08-15 13:44:32
date: 2018-08-15 13:34:24
tags: []
---


罗列ceph上的systemd unit节点：

systemctl status ceph\*.service ceph\*.target

启动一个ceph节点上的所有守护程序

systemctl start ceph.target

停止一个ceph节点上的所有守护程序

systemctl stop ceph\*.service ceph\*.target

按类型启动所有守护程序

systemctl start ceph-osd.target systemctl start ceph-mon.target systemctl start ceph-mds.target

按类型停止所有守护程序

systemctl stop ceph-mon\*.service ceph-mon.target systemctl stop ceph-osd\*.service ceph-osd.target systemctl stop ceph-mds\*.service ceph-mds.target

启动单个进程

systemctl start ceph-osd@{id} systemctl start ceph-mon@{hostname} systemctl start ceph-mds@{hostname}

例如：

systemctl start ceph-osd@1 systemctl start ceph-mon@ceph-server systemctl start ceph-mds@ceph-server

停止单个进程

systemctl stop ceph-osd@{id} systemctl stop ceph-mon@{hostname} systemctl stop ceph-mds@{hostname}

例如

systemctl stop ceph-osd@1 systemctl stop ceph-mon@ceph-server systemctl stop ceph-mds@ceph-server

http://docs.ceph.org.cn/rados/operations/operating/


