---
title: "nvidia-k8s-device-plugin源码解析"
author: 一张狗
lastmod: 2019-07-06 03:52:47
date: 2019-04-19 12:11:48
tags: []
---

Kubernetes 提供了Device Plugin 的机制，用于异构设备的管理场景。原理是会为每个特殊节点上启动一个针对某个设备的DevicePlugin pod， 这个pod需要启动grpc服务， 给kubelet提供一系列接口。DevicePlugin 注册一个 socket 文件到 **/var/lib/kubelet/device-plugins/** 目录下，kubelet 通过这个目录下的socket文件向对应的 Device plugin 发送grpc请求。

具体流程如下：
- nvidia-container-runtime-hook ：安装该hook，把docker的default-runtime修改为nvidia-container-runtime-hook。这个hook会在创建容器前先hook住，检查是否需要GPU（通过检查是否有NVIDIA_VISIBLE_DEVICES ），如果需要则调用libnvidia-container暴露GPU，把设备和驱动mount到容器内。 ![k8s](http://yizhanggou.top/imgs/2019/07/pasted-image-0-27-2.png)
- K8s通过device-plugin - 上报节点GPU数量
- 分配GPU：把GPU需求转换为NVIDIA_VISIBLE_DEVICES注入容器，并将设备和驱动映射到容器中![](http://yizhanggou.top/wp-content/uploads/2019/04/17233726_BX9Y-e1555654501454.jpg)


参考：
- https://www.twblogs.net/a/5bde55f42b717720b51be0c4
- https://my.oschina.net/jxcdwangtao/blog/1797047
- https://blog.csdn.net/s812289480/article/details/83588320


