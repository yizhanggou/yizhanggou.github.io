---
title: "Lambda架构 vs Kappa架构"
author: 一张狗
lastmod: 2019-07-06 10:52:29
date: 2018-03-15 13:59:20
tags: []
---


参考：http://blog.csdn.net/brucesea/article/details/45937875#t5
http://blog.csdn.net/Post_Yuan/article/details/52241252
http://blog.csdn.net/lvsaixia/article/details/51778487
https://zhuanlan.zhihu.com/p/20510974
http://www.raincent.com/content-85-7857-1.html
http://www.bijishequ.com/detail/249233
https://www.jianshu.com/p/d391fe9c7976
http://www.raincent.com/content-85-7857-1.html

随着DT时代的到来，大数据处理的需求越来越多，这里简单介绍一下通用的两套大数据处理计算框架，Lambda和演进的Kappa架构。

Lamdba架构

为了满足大数据系统高容错、低延时和可扩展的需求，Lambda整合在线实时和离线计算。

数据系统的本质就是数据+查询，查询就是Query=**Function(All Data)**

有一类称为Monoid特性的函数应用非常广泛。Monoid的概念来源于范畴学（Category Theory），其一个重要特性是满足结合律。如整数的加法就满足Monoid特性：

> **(a+b)+c=a+(b+c)**

不满足Monoid特性的函数很多时候可以转化成多个满足Monoid特性的函数的运算。如多个数的平均值Avg函数，多个平均值没法直接通过结合来得到最终的平均值，但是可以拆成分母除以分子，分母和分子都是整数的加法，从而满足Monoid特性。

Monoid的结合律特性在分布式计算中极其重要，满足**Monoid特性意味着我们可以将计算分解到多台机器并行运算**，然后再结合各自的部分运算结果得到最终结果。同时也意味着部分运算结果可以储存下来被别的运算共享利用（如果该运算也包含相同的部分子运算），从而减少重复运算的工作量。

大数据处理框架要解决的核心问题是：如何实时地在任意大数据集上进行查询？

当数据量太大的时候，实时变得不可达，这时候就需要拆分计算，Lambda架构由三个组件组成：Batch Layer，Speed Layer和Serving Layer。这三个组件的功能分别如下：


## Batch Layer

功能主要有两点：

- 存储数据集
- 在数据集上预先计算查询函数，构建查询所对应的View

我们把针对查询预先计算并保存的结果称为View，View是Lamba架构的一个核心概念，它是针对查询的优化，通过View即可以快速得到查询结果。

![20150523220607925](http://yizhanggou.top/imgs/2019/07/20150523220607925.png)

**对View的理解：**  
 View是一个和业务关联性比较大的概念，View的创建需要从业务自身的需求出发。一个通用的数据库查询系统，查询对应的函数千变万化，不可能穷举。但是如果从业务自身的需求出发，可以发现业务所需要的查询常常是有限的。Batch Layer需要做的一件重要的工作就是根据业务的需求，考察可能需要的各种查询，根据查询定义其在数据集上对应的Views。

## Speed Layer

Batch Layer可以很好的处理离线数据，但有很多场景数据不断实时生成，并且需要实时查询处理。Speed Layer正是用来处理增量的实时数据。

Speed Layer和Batch Layer比较类似，对数据进行计算并生成Realtime View，其主要区别在于：

- Speed Layer处理的数据是最近的增量数据流，Batch Layer处理的全体数据集
- Speed Layer为了效率，接收到新数据时不断更新Realtime View，而Batch Layer根据全体离线数据集直接得到Batch View。

## Serving Layer

Lambda架构的Serving Layer用于响应用户的查询请求，合并Batch View和Realtime View中的结果数据集到最终的数据集。

这儿涉及到数据如何合并的问题。前面我们讨论了查询函数的Monoid性质，如果查询函数满足Monoid性质，即满足结合率，只需要简单的合并Batch View和Realtime View中的结果数据集即可。否则的话，可以把查询函数转换成多个满足Monoid性质的查询函数的运算，单独对每个满足Monoid性质的查询函数进行Batch View和Realtime View中的结果数据集合并，然后再计算得到最终的结果数据集。另外也可以根据业务自身的特性，运用业务自身的规则来对Batch View和Realtime View中的结果数据集合并。

![20150523221307488](http://yizhanggou.top/imgs/2019/07/20150523221307488.png)

下图给出了Lambda架构中各个层常用的组件。数据流存储可选用基于不可变日志的分布式消息系统Kafka；Batch Layer数据集的存储可选用Hadoop的HDFS，或者是阿里云的ODPS；Batch View的预计算可以选用MapReduce或Spark；Batch View自身结果数据的存储可使用MySQL（查询少量的最近结果数据），或HBase（查询大量的历史结果数据）。Speed Layer增量数据的处理可选用Storm或Spark Streaming；Realtime View增量结果数据集为了满足实时更新的效率，可选用Redis等内存NoSQL。

![20150523220916124](http://yizhanggou.top/imgs/2019/07/20150523220916124.png)
or：
![20160628202924389](http://yizhanggou.top/imgs/2019/07/20160628202924389.png)

总结：

- Batch Layer : 存储数据集，在数据集上预先计算查询函数，并构建查询所对应的View。Batch Layer可以很好的处理离线数据，但有很多场景数据是不断实时生成且需要实时查询处理，对于这情况， Speed Layer更为适合。
- Speed Layer : Batch Layer处理的是全体数据集，而Speed Layer处理的是最近的增量数据流。Speed Layer为了效率，在接收到新的数据后会不断更新Real-time View，而Batch Layer是根据全体离线数据集直接得到Batch View。
- Serving Layer : Serving Layer用于合并Batch View和Real-time View中的结果数据集到最终数据集。

改进：Kappa

Lambda 架构的一个很明显的问题是需要维护两套分别跑在批处理和实时计算系统上面的代码，而且这两套代码得产出一模一样的结果。因此对于设计这类系统的人来讲，要面对的问题是：为什么我们不能改进流计算系统让它能处理这些问题？为什么不能让流系统来解决数据全量处理的问题？流计算天然的分布式特性注定其扩展性比较好，能否加大并发量来处理海量的历史数据？基于种种问题的考虑，Jay提出了Kappa这种替代方案。
![20160818153828373](http://yizhanggou.top/imgs/2019/07/20160818153828373.png)

那如何用流计算系统对全量数据进行重新计算，步骤如下：

1、用Kafka或类似的分布式队列保存数据，需要几天数据量就保存几天。

2、当需要全量计算时，重新起一个流计算实例，从头开始读取数据进行处理，并输出到一个结果存储中。

3、当新的实例完成后，停止老的流计算实例，并把老的一引起结果删除。

一个典型的Kappa架构如下，

![20160818162617279](http://yizhanggou.top/imgs/2019/07/20160818162617279.png)
