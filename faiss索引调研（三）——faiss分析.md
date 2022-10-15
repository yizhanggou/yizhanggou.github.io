---
title: "faiss索引调研（三）——faiss分析"
author: 一张狗
lastmod: 2019-07-06 04:55:19
date: 2019-02-21 11:07:38
tags: []
---



# 一、简介

faiss是FaceBook 2017年开源的一个用于高效相似性搜索和密集向量聚类的库，能对海量数据进行高效的相似问搜索和密集向量聚类。

http://houjie13.com/articles/2018/06/21/1529587425820.html
http://houjie13.com/articles/2018/06/25/1529933223485.html
faiss的faq：https://github.com/facebookresearch/faiss/wiki/Troubleshooting#slow-brute-force-search-with-openblas
多线程的限制：https://github.com/facebookresearch/faiss/wiki/Threads-and-asynchronous-calls#performance-of-search
faiss里面几种索引的简介：https://zhuanlan.zhihu.com/c_159623040
faiss源码剖析https://blog.csdn.net/dramer110/article/details/84001605


# 二、安装

1. 安装openblas（一个矩阵运算库） - yum install gcc-gfortran  
 git clone https://github.com/xianyi/OpenBLAS.git  
 cd OpenBLAS  
 make FC=gfortran  
 make install （openblas安装到/opt/下面）  
 ln -s /opt/OpenBLAS/lib/libopenblas.so /usr/lib/libblas.so.3  
 ln -s /opt/OpenBLAS/lib/liblapack.so.3 /usr/lib/liblapack.so.3  
 .bashrc加入:export LD_LIBRARY_PATH=/opt/OpenBLAS/lib:$LD_LIBRARY_PATH
2. 安装faiss - git clone https://github.com/facebookresearch/faiss.git  
 cd faiss  
 cp example_makefiles/makefile.inc.Linux makefile.inc  
 ./configure –with-blas=/usr/lib/libblas.so.3  
 (测试：make misc/test_blas然后运行./misc/test_blas)  
 make  
 make install （安装到/usr/local/lib 和 /usr/local/include/faiss）  
 make py（安装到./python/faiss，把这个faiss文件夹拷贝到你的python的/lib/python2.7/下面）


# 三、faiss的几种基本索引

faiss包含了较多索引方式（详见https://github.com/facebookresearch/faiss/wiki/Faiss-indexes），这里介绍一下tutorial/python目录下使用的三种：

- IndexFlatL2 - 基于brute-force计算向量的L2距离，就是暴搜。检索速度慢，适用于小数据量。
- IndexIVF - 加快索引的方式之一，与暴搜对比就是需要train，把向量空间下的数据切割为Voronoi细胞，检索只对向量所在细胞和周围细胞进行检索。
- IndexIVFPQ - 基于Product Quantizer对高维向量进行压缩，降低内存占用。


## 3.1 IndexFlatL2

直接上例子：faiss处理固定维度的向量的集合，维度通常为几十到几百，这些集合可以存储在矩阵中，行主存储。

```
import numpy as np d = 64 # 维度
nb = 100000 # 数据库大小
nq = 10000 # 要搜索的query
np.random.seed(1234) # 确定种子，使随机数可重现
xb = np.random.random((nb, d)).astype('float32') 
xb[:, 0] += np.arange(nb) / 1000. # 每一行的第一个列增加一个等差数列的对应项数
xq = np.random.random((nq, d)).astype('float32') 
xq[:, 0] += np.arange(nq) / 1000. 
print(xq.shape) # (10000, 64)
print(xb.shape) # (100000, 64)
import faiss # make faiss available 
index = faiss.IndexFlatL2(d) # 构建FlatL2索引
print(index.is_trained) 
print(index.ntotal) 
index.add(xb) # 向索引中添加向量。add操作如果没有提供id，则使用向量序号作为id。
print(index.ntotal) k = 4 # 搜索多少个临近向量
D, I = index.search(xb[:5], k) # 用xb的前五行本身自己搜索自己，完整性检查，用于测试
print("I=") 
print(I) 
#I=
#[[ ***0*** 393 363 78 924]
# [ ***1*** 555 277 364 617]<
# [ ***2*** 304 101 13 801]<
# [ ***3*** 173 18 182 484]
# [ ***4*** 288 370 531 178]] 
# I输出类似于上面，每行对应着相应向量的搜索结果。k为多少就有多少列，distance低的排在前面。
# 可以看到前五行的第一列确实是0~4
print("D=") 
print(D) 
#[[***0.*** 7.1751733 7.207629 7.2511625]
# [***0.*** 6.3235645 6.684581 6.7999454]
# [***0.*** 5.7964087 6.391736 7.2815123]
# [***0.*** 7.2779055 7.5279865 7.6628466]
# [***0.*** 6.7638035 7.2951202 7.3688145]]
# 可以看到第一行第一列都是0，意思是向量与自己本身的距离为0
D, I = index.search(xq, k) # 搜索
print(I[:5]) # 最初五个向量查询的结果 
print(I[-5:]) # 最后五个向量查询的结果
```

*当建立和训练完索引时，可以对索引执行两个操作： **add** 和 **search** 。

将元素添加到索引。我们还可以输出索引的两个状态变量：  
***is_trained*** 表示索引是否需要训练的布尔值，  
***ntotal*** 索引中向量的数量。

IndexFlat搜索的结果是精确的，可以作为评估其他几个索引的测试准确性的标准。


## 3.2 IndexIVFFlat

对于暴搜来说，海量数据搜索速度太慢，那么需要预训练把向量都聚类。这里使用IndexIVFFlat来加快搜索速度。IndexIVFFlat是faiss的倒排索引，把数据构成的向量空间切割为Voronoi细胞，每个向量落入其中一个Voronoi细胞中。在搜索时，只有查询x所在细胞中包含的数据库向量y与少数几个相邻查询向量进行比较。

训练的时候还需要有一个量化器，用于决定以什么方式将向量分配给Voronoi细胞。每个细胞由一个质心定义，找到一个向量所在的Voronoi细胞包括在质心集中找到该向量的最近邻居。

搜索方法有两个参数：  
***nlist*** 划分Voronoi细胞的数量  
***nprobe*** 执行搜索访问的单元格数(不包括nlist)，该参数**调整结果速度和准确度之间折中的一种方式。如果设置nprobe=nlist则结果与暴搜一致。**

上代码：
```
import numpy as np d = 64 # dimension 
nb = 100000 # database size 
nq = 10000 # nb of queries 
np.random.seed(1234) # make reproducible 
xb = np.random.random((nb, d)).astype('float32') 
xb[:, 0] += np.arange(nb) / 1000. 
xq = np.random.random((nq, d)).astype('float32') 
xq[:, 0] += np.arange(nq) / 1000. 
import faiss 
nlist = 100 
k = 4 
quantizer = faiss.IndexFlatL2(d) # 内部的索引方式
index = faiss.IndexIVFFlat(quantizer, d, nlist, faiss.METRIC_L2) # here we specify METRIC_L2, by default it performs inner-product search 
print("before train") 
assert not index.is_trained 
index.train(xb) 
assert index.is_trained 
print("before add") 
index.add(xb) # add may be a bit slower as well 
D, I = index.search(xq, k) # actual search print(I[-5:]) # neighbors of the 5 last queries 
index.nprobe = 10 # default nprobe is 1, try a few more 
D, I = index.search(xq, k) 
print(I[-5:]) # neighbors of the 5 last queries
```



## 3.3 IndexIVFPQ

上面两种索引都是存储的完整向量，下面介绍一种压缩向量的方法。IndexIVFPQ基于PQ算法压缩向量。在这种情况下，**由于向量没有精确存储，搜索方法返回的距离也是近似值**。

```
import numpy as np d = 64 # dimension 
nb = 100000 # database size 
nq = 10000 # nb of queries 
np.random.seed(1234) # make reproducible 
xb = np.random.random((nb, d)).astype('float32') 
xb[:, 0] += np.arange(nb) / 1000. 
xq = np.random.random((nq, d)).astype('float32') 
xq[:, 0] += np.arange(nq) / 1000. 
import faiss 
nlist = 100 
m = 8 
k = 4 
quantizer = faiss.IndexFlatL2(d) # 内部的索引方式
index = faiss.IndexIVFPQ(quantizer, d, nlist, m, 8) 
# 每个向量都被编码为8个字节大小
index.train(xb) index.add(xb) 
D, I = index.search(xb[:5], k) 
# sanity check 
print(I) 
print(D) 
#[[ 0 78 714 372]
# [ 1 1063 555 277]
# [ 2 304 134 46]
# [ 3 773 64 8]
# [ 4 288 531 827]]
#[[1.6675376 6.1988335 6.4136653 6.4228306]
# [1.4083313 6.023788 6.025648 6.284443 ]
# [1.6988016 5.592166 6.139589 6.6717234]
# [1.7987373 6.625978 6.7166452 6.865783 ]
# [1.5371588 5.7953157 6.38059 6.4141784]] 
# 可以看到确实搜索到了正确的结果，但是第一行第一列的distance不为零，属于有损压缩。
# 虽然与接下来的几列（其他几个搜索结果）对比还是有几倍的优势。
index.nprobe = 10 # 与以前的方法相比 
D, I = index.search(xq, k) # search print(I[-5:])
```

另外搜索真实查询时，虽然结果大多是错误的(与刚才的IVFFlat进行比较)，但是它们在正确的空间区域，而对于真实数据，情况更好，因为：

- 统一数据很难进行索引，因为没有规律性可以被利用来聚集或降低维度
- 对于自然数据，语义最近邻居往往比不相关的结果更接近。

简化指标结构：  
 由于构建索引可能会变得复杂，因此有一个工厂函数用于接受一个字符串来构造响应的索引。上面的索引可以通过以下简写获得：

index = faiss.index_factory（d，“ IVF100，PQ8 ”）

更换PQ4用Flat得到的IndexFlat。当预处理（PCA）应用于输入向量时，工厂特别有用。例如，预处理的工厂字符串通过PCA投影将矢量减少到32维为：“PCA32,IVF100,Flat”。


# 四、index总结
![5617720-56b8280f2c78029b](http://yizhanggou.top/imgs/2019/07/5617720-56b8280f2c78029b.png)

选择：

需求 | 选择
---- | ----
no build time, high query time, high storage, exact accuracy | Faiss IndexFlat
low build time, med query time, high storage, high accuracy< | Faiss IndexIVFFlat
med build time, low query time, low-med storage, med-high accuracy | Faiss IndexIVFPQ
very high build time, low query time, low-high storage (whether stored as a k-NN graph or raw data), high accuracy | N-Descent by Dong et al. (e.g., nmslib)


