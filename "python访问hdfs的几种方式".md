---
title: "python访问hdfs的几种方式"
author: 一张狗
lastmod: 2019-07-06 07:22:57
date: 2018-12-25 15:03:32
tags: []
---



# 零、搭建

快速搭建单节点的方法：https://blog.csdn.net/yu616568/article/details/42780383

# 一、python库
## hdfs库

pip install hdfs

只可以使用hdfs的http端口（通常是50070），不支持rpc端口（9000或8020）

需要在启动hdfs节点的时候配置：

![](http://yizhanggou.top/imgs/2019/07/210934_vvtP_2704218.png)

使用也很方便：

from hdfs import * fs = InsecureClient(hdfs_url, root=hdfs_root, user=hdfs_proxy,timeout=hdfs_timeout) fs_folders_list = fs.list(hdfs_root)

2.2.2的文档：https://media.readthedocs.org/pdf/hdfscli/latest/hdfscli.pdf



## snakebite库

O’Reilly的书Hadoop with Python推荐的库，可以用rpc端口，但是只可以下载，（截止到20181225）<span style="color: #ff0000;">没有上传的接口</span>

使用：返回的是一个generator
```
from snakebite.client import Client client = Client("localhost", 8020, use_trash=False) for x in client.ls(['/']): 
print x
```

snakebite的github：https://github.com/spotify/snakebite


## pyhdfs库

pyhdfs是对libhdfs的python封装库，

装起来很麻烦，pip install python-hdfs但是还是不能访问，是需要有libhds，看了网上要装的东西挺多的，

https://pypi.org/project/PyHDFS/

各种安装方法:https://blog.csdn.net/w894607785/article/details/50205857


## libhdfs库

libhdfs 是HDFS的底层C函数库, 由hadoop官方提供。

安装：pip install hdfs3，还需要安装libhdfs，如果是Centos必须使用源码编译安装，没折腾出来。

使用：from hdfs3 import HDFileSystem hdfs = HDFileSystem(host='localhost', port=8020)

文档：https://hdfs3.readthedocs.io/en/latest/install.html


## pyarrow库

安装pyhdfs的时候发现的一个库，可以用rpc端口。

调用java来访问hdfs的库，需要装jre、hadoop客户端：

export JAVA_HOME=/home/work/java/jdk1.8.0_60  
 export PATH=$JAVA_HOME/jre/bin:$PATH  
 export HADOOP_HOME=/home/work/hadoop-2.6.0-cdh5.13.1  
 export HADOOP_LIBEXEC_DIR=${HADOOP_HOME}/libexec  
 export HADOOP_USER_NAME=<your_user_name>

因为Clouder CDH3B3开始后hadoop.job.ugi不再生效，所以装客户端的时候需要配置环境变量HADOOP_USER_NAME来修改访问用户名

使用:

import pyarrow as pa fs = pa.hdfs.connect(hdfs_url, int(hdfs_port), user=hdfs_user, kerb_ticket=None) fs.ls(hdfs_root)

带了Kerberous的话得看一下：https://zhuanlan.zhihu.com/p/43887793

使用参考文档：  
 访问：https://arrow.apache.org/docs/python/filesystems.html#hadoop-file-system-hdfs  
 接口：https://arrow.apache.org/docs/python/generated/pyarrow.HdfsFile.html  
 https://blog.csdn.net/zyb378747350/article/details/79020787


# 二、调用shell

用os.system(cmd)调用

![](http://yizhanggou.top/imgs/2019/07/20170414142412448.png)

参考：http://wesmckinney.com/blog/python-hdfs-interfaces/这篇文章说的比较全面

三、其他

安装hadoop客户端的时候一定要下一个与服务端版本匹配的，修改etc/hadoop/core-site.xml的fs.default.name为服务端即可。


