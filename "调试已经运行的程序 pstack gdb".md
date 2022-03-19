---
title: "调试已经运行的程序 pstack gdb"
author: 一张狗
lastmod: 2018-08-06 14:40:13
date: 2018-08-06 11:28:48
tags: []
---


http://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/strace.html

http://www.cnblogs.com/xybaby/p/8025435.html#_label_3

1. pstack 查看堆栈信息
2. gdb 1. gdb调试正在运行的程序 1. gdb
2. attach <pid>
3. bt
2. gdb也可以调试core文件 1. gdb <program> <core dump file>
3. strace 1. strace -T -tt -e trace=all -p <pid>
4. other


