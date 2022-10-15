---
title: "docker镜像构建"
author: 一张狗
lastmod: 2019-07-06 12:54:38
date: 2018-02-05 22:22:15
tags: []
---



## 一、基于联合文件系统

典型的Linux文件系统由bootfs和rootfs两部分组成，bootfs(boot file system)主要包含 bootloader和kernel，bootloader主要是引导加载kernel，当kernel被加载到内存中后 bootfs就被umount了。 rootfs (root file system) 包含的就是典型 Linux 系统中的/dev，/proc，/bin，/etc等标准目录和文件。
![](http://wiki.baidu.com/download/attachments/453038481/docker-filesystem-1.png?version=1&modificationDate=1516021907000&api=v2)</span>

联合文件系统是一种分层的轻量级文件系统，支持将文件系统的修改作为一次提交来一层层叠加，可以对每个目录设定只读(Readonly)、读写(Readwrite)和写(Witeout-able)权限。联合文件系统是docker镜像的基础，镜像可以通过分层继承。下面是docker的文件系统：

![](http://wiki.baidu.com/download/attachments/453038481/image2018-1-14%2019%3A48%3A39.png?version=1&modificationDate=1515930520000&api=v2)

    传统的Linux加载bootfs时会先将rootfs设为read-only，然后在系统自检之后将rootfs从read-only改为read-write，然后我们就可以在rootfs上进行写和读的操作了。但Docker的镜像却不是这样，它在bootfs自检完毕之后并不会把rootfs的read-only改为read-write。而是利用union mount（UnionFS的一种挂载机制）将一个或多个read-only的rootfs加载到之前的read-only的rootfs层之上。在加载了这么多层的rootfs之后，仍然让它看起来只像是一个文件系统，在Docker的体系里把union mount的这些read-only的rootfs叫做Docker的镜像。但是，此时的每一层rootfs都是read-only的，我们此时还不能对其进行操作。当我们创建一个容器时，将Docker镜像进行实例化，系统会在一层或是多层read-only的rootfs之上分配一层空的read-write的rootfs。

    AUFS是Docker最初采用的文件系统，然而因为AUFS不在Linux的内核主干里，所以，对于非Ubuntu的Linux分发包，比如CentOS，就无法使用AUFS作为Docker的文件系统了。此时我们需要使用devicemapper。Device mapper 是Linux 2.6内核中提供的一种从逻辑设备到物理设备的映射框架机制。在该机制下，用户可以很方便的根据自己的需要制定实现存储资源的管理策略。Device mapper 基于Thin-provisioning Snapshot。Snapshot是LVM的一种特性，可以在不中断服务的情况下创建一个虚拟快照，当原始device发生变化的时候，snapshot只对变化的部分做拷贝以用来对原始device重构。当对snapshot进行写操作时，丢弃原始device的拷贝，以后的操作snapshot中为准。Thin-Provisioning是一项利用虚拟化方法减少物理存储部署的技术，简单来说就是服务申请虚拟卷的时候并不分配对应大小的物理卷，当使用的时候如果超过了存储报警阈值再重新调整大小。Thin-provisioning Snapshot结合了这两种技术，允许将多个Snapshot（虚拟设备）挂载到同一个数据卷，在每个虚拟设备发生写操作时触发COW操作，并支持递归，没有深度限制。




## 二、实例：docker build、run时发生了什么

1. 构建镜像（docker build）时发生了什么：  
docker build在执行一条指令时，会根据当前镜像层启动一个容器，Docker会在容器的层级文件系统最上层建立一层空的可读可写层(镜像层的内容对于容器来说是readonly的)，之后Docker容器执行指令，将执行结果写入可读可写层(并更新镜像Json文件)，最后再通过docker commit命令将可读可写层提交为一个新的镜像层。其中镜像Json文件的作用为：
    - 记录镜像的父子关系，以及父子间的差异信息  
    - 弥补镜像本身以及镜像到容器转换所需的额外信息

总的来说，docker build做了以下几件事情：

    1.提取Dockerfile

    2. 根据command，将之后的字符串用对应的数据结构进行接收。

    3. 根据分析的command，在dispatchers.go中选择对应的函数进行处理。比如CMD，会首先更新Config，然后根据Config创建一个容器

    4. 处理完所有的命令，如果需要打标签，则给最后的镜像打上tag

2. 从镜像到容器时（docker run）发生了什么：  
 在从静态的镜像到动态的容器的过程中，docker做了以下事情
    - 将 Docker 镜像的镜像层文件作为 Docker 容器的 rootfs 。
    - 提取 Docker 镜像 json 文件中的动态文件，确定启动进程，并为之配置动态运行环境。

![](http://wiki.baidu.com/download/attachments/453038481/json%20document.jpg?version=1&modificationDate=1516021706000&api=v2)

通过镜像创建容器时，前者的镜像层内容将作为容器的 rootfs；而前者的 json 文件，会由 dockerd 解析，并提取出其中的容器执行入口 CMD 信息，以及容器进程的环境变量 ENV 信息，最终初始化容器进程。容器进程的执行入口来源于镜像提供的 rootfs。当 dockerd 通过镜像运行容器时，首先提取最上层镜像的 json 文件，找到 config 属性中的 Cmd 命令，并在镜像层文件构成的容器 rootfs 中找到相应的执行程序，最终执行，完成容器的启动。

3. 实例以语义检索为例子： Dockerfile为：

```
FROM kb-centos:v11
ENV PATH="/home/work/.jumbo/bin:${PATH}:/home/work/kb_install/gearman-instal/gperf-install/bin/" \
LD_LIBRARY_PATH="${LD_LIBRARY_PATH}:/home/work/kb_install/gearman-instal/boost-install/lib:/home/work/kb_install/gearman-instal/libevent-install/lib:/home/work/kb_install/gearman-instal/libuuid-install/lib"
COPY supervisord.conf /home/work/kb_install/
USER 1001
EXPOSE 8011
CMD ["/home/work/.jumbo/bin/supervisord","-c","/home/work/kb_install/supervisord.conf"]
```

这个镜像的层级：
```
[root@your_machine ~]# docker history ss-test-mount
IMAGE CREATED CREATED BY SIZE COMMENT
387685b9d691 4 days ago /bin/sh -c #(nop) CMD ["/home/work/.jumbo/bi 0 B
50abddaca730 4 days ago /bin/sh -c #(nop) EXPOSE 8011/tcp 0 B
c50a4079a660 4 days ago /bin/sh -c #(nop) USER [1001] 0 B
8b24f7782928 4 days ago /bin/sh -c #(nop) COPY file:766b243af97880801 8.866 kB
3fccd77a8a76 4 days ago /bin/sh -c #(nop) ENV PATH=/home/work/.jumbo 0 B
6cfbce048a17 4 days ago 5.913 GB Imported from -
```

Dockerfile在构建的时候，每一个命令就是一层。通过 docker build  以上 Dockerfile  的时候，会在 kb-centos:v11  镜像基础上，添加五层独立的镜像，依次对应于五条不同的命令。

在构建镜像时候大小不为0的镜像层，对当前的文件系统进行了修改，通常是ADD/COPY或者RUN命令。ADD/COPY层的大小是增加的文件的大小，RUN层的大小是当前运行命令后所增加的文件系统大小。

EXPOSE、USER等几个大小为0的命令，更新了这个镜像的json文件，以EXPOSE这一层为例子：

容器json在./image/devicemapper/imagedb/content/sha256/387685b9d6916cab73c330689be63e7bfbe2589a3a19344f3b1ada414aa28d44

左边是最后一层，右边是倒数第二层

![](http://wiki.baidu.com/download/attachments/453038481/image2018-1-9%2018%3A45%3A48.png?version=1&modificationDate=1515494748000&api=v2)

![](http://wiki.baidu.com/download/attachments/453038481/image2018-1-9%2018%3A44%3A22.png?version=1&modificationDate=1515494662000&api=v2)

可以看出json文件包含了镜像以及其对应的容器的配置信息，其中有docker run时指定的，如CpuShares，Cpuset，也有系统生成的，如parent，正是这个属性记录了继承结构，从而实现了layer的层次关系。


