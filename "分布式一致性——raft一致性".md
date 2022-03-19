---
title: "分布式一致性——raft一致性"
author: 一张狗
lastmod: 2019-07-06 10:10:07
date: 2018-05-05 15:59:49
tags: []
---


参考：http://www.jdon.com/artichect/raft.html

分布式系统需要解决的问题：CAP理论

C：consistency，一致性，所有数据变动都是一致的。

A：available，好的响应性能。

P：partition tolerance 分区容忍。

定理：任何分布式系统只可以同时满足两个，不可三者得兼。

 

raft中角色有三种：

Leader：处理所有客户端请求，日志复制等，一般只有一个。

Follower：被动处理Leader发来的请求。

Candidate：可以被选为一个新的Leader。


raft分为两个阶段：选举、

选举出来的Leader带领进行正常操作。每个节点都可以成为Candidate，像其他节点发送vote请求，最先得到N/2+1选票的就当选为Leader。Leader当选后向选民发出指令，比如日志复制等，以后通过心跳进行日志复制的通知。如果Leader宕机了，则Follow重新选举为Leader。如果某一时间选举Leader时有多个Leader得到票数一样，则在timeout比如300ms后重新选举，先得到N/2+1的成为Leader，这时候同票的概率大大降低。每个Candidate通常timeout时间不同，那么最短timeout时间的那个就会最先得到大多数投票。


日志复制：

假设Leader已经选出，现在客户端发出一个增加日志的要求，Leader要求Follower遵从他的指令，都将这个新的日志内容追加到他们各自日志中，当**超过半数以上**的Follow都向Leader发出确认后，Follower服务器将日志写入磁盘文件后，确认追加成功，发出Commited Ok，Leader返回添加成功给client。在下一个心跳heartbeat中，Leader会通知**所有**Follwer更新commited 项目。


分区容忍：

如果这时候发生了网络分区，则原先的Leader只能正常访问本分区内的Follower，失去Leader的分区会重新选举一个Leader出来。如果这之后网络故障消失了，因为新的Leader的term比老的Leader的term大，那么原先的Leader会成为Follow，在失联阶段这个老Leader的任何更新都不能算commit，都回滚，接受新的Leader的新的更新。

 

http://www.cnblogs.com/bangerlee/p/5991417.html

为达到更容易理解和实行的目的，Raft将问题分解和具体化：Leader统一处理变更操作请求，一致性协议的作用具化为保证节点间操作日志副本(log replication)一致，以term作为逻辑时钟(logical clock)保证时序，节点运行相同状态机(state machine)<sup>[4]</sup>得到一致结果。Raft协议具体过程如下：

![116770-20161024005549560-244386650](/imgs/2019/07/116770-20161024005549560-244386650.png)

1. Client发起请求，每一条请求包含操作指令
2. 请求交由Leader处理，Leader将操作指令(entry)追加(append)至操作日志，紧接着对Follower发起AppendEntries请求、尝试让操作日志副本在Follower落地
3. 如果Follower多数派(quorum)同意AppendEntries请求，Leader进行commit操作、把指令交由状态机处理
4. 状态机处理完成后将结果返回给Client

指令通过log index(指令id)和term number保证时序，正常情况下Leader、Follower状态机按相同顺序执行指令，得出相同结果、状态一致。

 


