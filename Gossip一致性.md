---
title: "Gossip一致性"
author: 一张狗
lastmod: 2019-07-06 09:44:23
date: 2018-05-21 22:23:06
tags: []
---


https://www.jianshu.com/p/3aa9a109072c

顾名思义，Gossip协议本身类似于新闻的传播方式，

Gossip Protocol可以分为Push-based和Pull-based两种。Push-based Gossip Protocol的具体工作流程如下：

- 网络中的某个节点随机的选择其他b个节点作为传输对象。
- 该节点向其选中的b个节点传输相应的信息
- 接收到信息的节点重复完成相同的工作

Pull-based Gossip Protol的协议过程刚好相反：

- 某个节点v随机的选择b个节点询问有没有最新的信息
- 收到请求的节点回复节点v其最近未收到的信息


