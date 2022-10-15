---
title: "分布式一致性——Paxos算法"
author: 一张狗
lastmod: 2019-07-06 09:46:10
date: 2018-05-22 00:43:19
tags: []
---


https://segmentfault.com/a/1190000004474543

https://blog.csdn.net/cloudresearch/article/details/23127985

http://codemacro.com/2014/10/15/explain-poxos/

https://draveness.me/consensus

Paxos对每个节点的并发修改采取编号记录的方式保持一致性，对多个节点的并发修改采取少数服从多数的方式保持一致性。

Paxos协议类似于人类社会的投票过程。

Paxos 协 议中，有一组完全对等的参与节点(称为 accpetor)，这组节点各自就某一事件做出决议，如果某个 决议获得了超过半数节点的同意则生效。Paxos 协议中只要有超过一半的节点正常，就可以工作， 能很好对抗宕机、网络分化等异常情况。

第一阶段：  
 proposer 选择一个提案编号 n 并将 prepare 请求发送给acceptors 中的一个多数派；acceptor 收到 prepare 消息后，如果提案的编号大于它已经回复的所有 prepare 消息，则 acceptor 将自己上次的批准回复给 proposer并承诺不再批准小于 n 的提案。  
 第二阶段：  
 当一个 proposor 收到了多数 acceptors 对 prepare 的回复后，就进入批准阶段。它要向回复 prepare 请求的acceptors 发送 accept 请求，包括编号 n 和根据 P2c 决定的 value（如果根据 P2c 没有决定 value，那么它可以自由决定 value）。在不违背自己向其他 proposer 的承诺的前提下，acceptor 收到 accept 请求后即批准这个请求。


