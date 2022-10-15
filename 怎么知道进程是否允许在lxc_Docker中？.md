---
title: "怎么知道进程是否允许在lxc_Docker中？"
author: 一张狗
lastmod: 2019-07-06 08:01:21
date: 2018-08-29 11:21:52
tags: []
---


https://stackoverflow.com/questions/20010199/how-to-determine-if-a-process-runs-inside-lxc-docker

判断是否在docker中：

The most reliable way is to check /proc/1/cgroup. It will tell you the control groups of the init process, and when you are *not* in a container, that will be / for all hierarchies. When you are *inside* a container, you will see the name of the anchor point; which, with LXC/Docker containers, will be something like /lxc/ or /docker/ respectively.

