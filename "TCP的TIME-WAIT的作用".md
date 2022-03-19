---
title: "TCP的TIME-WAIT的作用"
author: 一张狗
lastmod: 2019-07-06 12:46:09
date: 2017-08-07 19:35:31
tags: []
---


TIME-WAIT通常取2MSL，MSL是Maximum Segment Lifetime，报文最大生存时间，是任何报文在网络的最大生存时间，超过则丢弃。IP头有个TTL域，没经过一个路由器就减一，直到为0则直接丢弃，同时发送ICMP报文通知源主机。协议规定MSL为2分钟。
![wKiom1cd6_mwEZr2AACU62IiAp4333](http://yizhanggou.top/imgs/2019/07/wKiom1cd6_mwEZr2AACU62IiAp4333.png)

TIME-WAIT作用：

1.保证断开连接。比如被动方没有收到主动方最后一个ACK，那么会重新发送FIN，如果这时候主动方已经关闭端口，那么会回复RST包，RST包返回被动方后会影响进程。

这就是TIME-WAIT为什么要设置为2MSL的原因，这段时间内没有收到FIN包就认为被动分已经收到了主动方发送的ACK。

2.防止这次连接的网络包被下一次连接接收到了，引起混乱。老的连接包可能会被新的连接接收到，那么2MSL就保证了老的连接包已经在网络上消失了。


