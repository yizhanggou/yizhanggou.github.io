---
title: "python调用出现[Errno_5]_Input_output_error的问题"
author: 一张狗
lastmod: 2018-03-14 15:11:31
date: 2018-03-14 15:11:31
tags: []
---


现象：

gearman调用python脚本，出现不能调用的问题。

原因分析：

A模块调用B脚本，如果B脚本里面有print语句，当A模块的终端关闭的时候，B的print就没有地方输出了，这时候需要重新部署A。

总结：

当出现[Errno 5] Input/output error的问题的时候，一定是print报错。

http://blog.csdn.net/appleyuchi/article/details/78761800


