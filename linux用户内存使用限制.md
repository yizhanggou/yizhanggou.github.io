---
title: "linux用户内存使用限制"
author: 一张狗
lastmod: 2019-07-06 08:03:48
date: 2018-08-22 14:16:57
tags: []
---


1. 查看用户使用内存 - `ps -o rss aU qujun |grep -v RSS|awk ‘BEGIN {sum=0}{sum+=$1}END{print sum}’`
2. 限制用户使用内存大小： - ulimit中-m参数，limits.conf里面配置；
- cgroup中定义大小；
3. 具体步骤： - `vim /etc/security/limits.conf最下面一行配置test用户21g的内存使用：- \*\*@test hard rss 21000000\*\*
- (1) 加*号表示对所有用户起作用，加@test表示只对某个名叫test的用户起作用。(2) hard说明是硬上限，你也可以改成soft，也即软上限。(3) rss表示我们限制的是内存的使用量。(4) 21000000(单位KB)表明我们限制的量大概是20GB。
- vim /etc/pam.d/login最下面添加**- \*\*session required /lib/security/pam_limits.so\*\*
- 重新登录用户：\*\*ulimit -a查找max memory\*\*


