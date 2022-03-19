---
title: "topK问题"
author: 一张狗
lastmod: 2018-06-10 17:23:30
date: 2014-06-10 16:28:28
tags: []
---


参考：https://blog.csdn.net/suibianshen2012/article/details/52003082

https://cloud.tencent.com/developer/article/1121675

https://blog.csdn.net/boo12355/article/details/11788655

topK问题：在海量数据中找出出现频率前K的数据，或者最大的K个数，都为topK问题。

解题思路：数据量太大装不下内存，考虑使用分治+Trie树/Hash+小堆，先把数据hash到多个小数据集，（如果是找出出现频率topK的数据则）在集合内用Trie或者Hash统计小数据集里面数据出现的频率，然后在小数据集里面用小顶堆求topK，最后所有数据集求topK。

### 第一步：query统计

#### 方法一：直接排序

用归并排序使时间复杂度将为O(NlogN)，然后对数据出现的次数进行计数。

#### 方法二：Hash

维护一个key为被统计数据，value为次数的HashTable，每读取一个数据，如果key在HashTable内则加一，否则插入HashTable并设置value为1。

### 第二步：求topK

#### 方法一：全排序

#### 方法二：部分排序

维护一个k长数组，先初始化读取k个数据进入数据并排序，然后再遍历剩下的数据，每次读取都跟最小一个数字作比较，小于则跳过，大于数组中最小值则再一个个比较数组中数据，找到插入位置后把原先的最小值剔除出数组。最坏的时间复杂度是N*K。

#### 方法三：小顶堆

基于方法二，借助小顶堆我们可以把最坏的时间复杂度将为N*logK。每次都跟根节点比较，如果大于则替换为小顶堆的根节点。

 

PS:位图法用于在大规模数据，数据状态不是很多的情况下判断某个数据是否存在。

 

https://cloud.tencent.com/developer/article/1121675

https://blog.csdn.net/wypblog/article/details/8237956


