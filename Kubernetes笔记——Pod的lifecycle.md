---
title: "Kubernetes笔记——Pod的lifecycle"
author: 一张狗
lastmod: 2019-07-06 10:06:25
date: 2018-05-09 20:45:52
tags: []
---


参考 https://blog.openshift.com/kubernetes-pods-life/

创建Pod时，不管是从deployment还是statefulset还是其他方式，Pod的状态有以下几种：

- Pending：etcd已经保存了Pod的状态，但是这个Pod还没有被scheduled，还没拉取image
- Running：Pod已经schedule到node上，kubelet已经创建了所有的容器
- Succeeded：Pod中所有容器被终止，不再重启
- Failed：Pod中所有容器被终止，至少有一个容器是因为失败而终止
- Unknown：API server与Pod的通信中断，通常是因为与kubelet的通信中断

当你kubelet get po 的时候，可能会看到除了以上五个状态之外的状态，这时候需要kubelet describe po xxx 来获取po创建的具体Event。

![loap](/imgs/2019/07/loap.png)

创建Pod的时候按照时间顺序执行以下事件：

1. 启动[infra container](https://www.ianlewis.org/en/almighty-pause-container)，创建namespace
2. 所有的pod上初始化 [init container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
3. 容器和post start hook同时启动
4. 存活探针(livenessProbe)和就绪探针(readinessProbe)启动
5. pod被kill后，pre stop hook启动

最佳实践：

1. init container时做一些启动容器的准备工作，比如拉取数据、建数据库表、等待其他server启动等。可以设置多个init container，所有操作成功后容器才会启动
2. 永远需要livenessProbe和readinessProbe，前者用于让deployment决定何时需要重启，或者滚动升级的时候是否成功，后者用于决定该Pod何时开始可以接收流量。如果不设置，那么有可能丢失流量
3. 在postStart和preStop hook中正确的初始化或者关闭容器，比如如果不能修改代码但是需要清数据库。当你使用service的时候，需要一段时间让kube-proxy、endpoint完成工作、去掉iptables等，因此正在处理的请求可能会受容器停止的影响。这时候需要在这里处理优雅关机。
4. 当你想知道容器为什么终止的话，你可以把debug信息写入/dev/termination-log中，然后通过kubectl describe po xxx 来获取信息。通过terminationMessagePath可以修改位置，或者在spec中加上terminationMessagePolicy

配置preStop、postStart：https://support.huaweicloud.com/usermanual-cce/cce_01_0105.html


