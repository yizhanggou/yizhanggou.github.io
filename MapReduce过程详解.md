---
title: "MapReduce过程详解"
author: 一张狗
lastmod: 2019-07-06 09:27:15
date: 2018-07-16 12:05:48
tags: []
---

![041657025158483](/imgs/2019/07/041657025158483.png)

MapReduce的过程可以分为Sort, Partition, Shuffle, Combine, Merge等子阶段。

- 第一步 Map - 把输入文件的内容分解，如果是WordCount则分解为单词和1。
- 第二步 Partition - 有时候会有多个Reducer，Partition就是提前对输入进行处理，根据将来的Reducer进行分区. 到时候Reducer处理的时候， 只需要处理分给自己的数据就可以了。
- 主要的分区方法就是按照Key 的不同，把数据分开，其中很重要的一点就是要保证Key的唯一性， 因为将来做Reduce的时候有可能是在不同的节点上做的， 如果一个Key同时存在于两个节点上， Reduce的结果就会出问题， 所以很常见的Partition方法就是哈希。
- 第三步 Sort - 每个partition里面的条目都按照Key的顺序做了排序。
- 第四步 Combine - Combine 其实可以理解为一个mini Reduce 过程， 它发生在前面Map的输出结果之后， 目的就是在结果送到Reducer之前先对其进行一次计算， 以减少文件的大小， 方便后面的传输。
- 就是把自己Partition里面的数据先进行一次排重计数。
- 第五步 Copy - Reducer节点通过http的方式向各个mapper节点下载属于自己分区的数据。
- 第六步 Merge - Reducer节点对不同Mapper拉过来的数据进行合并。
- 第七步 Reduce - 根据每个文件中的内容最后做一次统计。

 

MapReduce最主要的三个过程是：

1. **Map**:数据输入,做初步的处理,输出形式的中间结果；
2. **Shuffle**:按照partition、key对中间结果进行排序合并,输出给reduce线程；
3. **Reduce**:对相同key的输入进行最终的处理,并将结果写入到文件中。

![mapreduce](/imgs/2019/07/mapreduce.png)

http://matt33.com/2016/03/02/hadoop-shuffle/

https://changsiyuan.github.io/2015/04/01/2015-4-1-mapreduce/


