---
title: "linux中查看网卡mac地址的各种姿势"
author: 一张狗
lastmod: 2018-09-09 18:00:40
date: 2018-09-07 17:46:25
tags: []
---


1. ifconfig -a 其中 HWaddr字段就是mac地址

2. cat /sys/<span class="keyword">class</span>/net/eth0/address 查看eth0的mac地址

3. cat /proc/net/arp 查看连接到本机的远端ip的mac地址

4. 程序中使用SIOCGIFHWADDR的ioctl命令获取mac地址

和ifcfg-eth0中device和mac地址的eth0对应，mac地址也要对应


