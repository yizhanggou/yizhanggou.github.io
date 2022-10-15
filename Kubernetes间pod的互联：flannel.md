---
title: "Kubernetes间pod的互联：flannel"
author: 一张狗
lastmod: 2019-07-06 08:35:07
date: 2018-08-01 14:38:34
tags: []
---


参考
https://tonybai.com/2017/01/17/understanding-flannel-network-for-kubernetes/
https://jiayi.space/post/kubernetescong-ru-men-dao-fang-qi-3-wang-luo-yuan-li


## 一、创建flannel后主机网络长什么样

首先我们来看两张flannel官方的图：
![flannel](http://yizhanggou.top/imgs/2019/07/flannel.png)

卤煮实际操作中应该是下面这样：注意docker0替换为了cni0：

![1048291-20180510001013735-420599402](http://yizhanggou.top/imgs/2019/07/1048291-20180510001013735-420599402.png)

![](http://yizhanggou.top/wp-content/uploads/2018/08/flat-flannel-network.png)



## 二、一个包是怎么从pod1到pod3的

Flannel网络给每个node分配一个网段，比如图中的172.16.99.0和172.16.57.0，这个网段是在flannel启动时候指定的，在图中就是172.16.0.0。

Flannel启动的时候在每个node上会创建一个名为flannel.1的类型为vxlan的网络设备，

一旦flanneld启动，它将从etcd中读取配置，并请求获取一个subnet lease(租约)，有效期目前是24hrs，并且监视etcd的数据更新。flanneld一旦获取subnet租约、配置完backend，它会将一些信息写入/run/flannel/subnet.env文件。

比如两个node：

node1：
```
[work@node1 ~]$ cat /run/flannel/subnet.env
FLANNEL_NETWORK=172.16.0.0/16
FLANNEL_SUBNET=172.16.99.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```
node2：
```
[root@node2 ~]# cat /run/flannel/subnet.env
FLANNEL_NETWORK=172.16.0.0/16
FLANNEL_SUBNET=172.16.57.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```
之后在这个node上启动的pod都从subnet中分配ip。

然后我们看一下这两个node上的路由信息：
```
[work@node1 ~]$ ip route
.......
172.16.0.0/24 dev cni0 proto kernel scope link src 10.244.0.1
172.16.99.0/16 dev flannel.1
.......
[root@yq01-sys-rpm0457e855c ~]# ip route
......
172.16.0.0/16 dev flannel.1
172.16.57.0/24 dev cni0 proto kernel scope link src 10.244.1.1
......
```
我们来看一下从pod1发出的ping pod3的包是怎么到达对端的：

***1.包从pod1发出去docker0***

我们来看一下pod1中的路由信息：
```
[root@pod1 ~]# ip route
default via 172.16.99.1 dev eth0
172.16.99.0/24 dev eth0 proto kernel scope link src 172.16.99.8
172.16.0.0/16 via 172.16.99.1 dev eth0
```
因为目标ip（172.16.57.15）并不在直连网络中，此数据包通过default路由出去。default路由是图中的docker0，相当于docker0 bridge以“三层的工作模式”直接接收到来自容器的数据包(而并非从bridge的二层端口接收)。

***2.包发给docker0后又转到flannel.1***

包发给docker0之后，docker0发现目标地址是172.17.57.15，发现目标并不是自己，那么要查找一下要发给谁，根据master node上的路由规则，发给了flannel.1

***3.包发给flannel.1之后***

flannel.1收到后发现目的也不是自己，也需要沿着网络协议栈往下流动，在二层时需要封二层以太包，填写目的mac地址，这时候需要发一个arp包，而：

master node:
`cat /proc/sys/net/ipv4/neigh/flannel.1/app_solicit`
3
因为这个内核参数配置，linux kernel引发一个”L3 MISS”事件并将arp请求发到用户空间的flanned程序。

flanned程序收到”L3 MISS”内核事件以及arp请求(who is 172.16.57.15)后，并不会向外网发送arp request，而是尝试从etcd查找该地址匹配的子网的vtep信息。

flanned从etcd中找到匹配子网的信息：
```
subnet: 172.16.57.0/24
public ip: {minion node local ip}
VtepMAC: d6:51:2e:80:5c:69
```
而这个mac就是minion node上的flannel.1 设备的mac地址。

接下来，flanned将查询到的信息放入master node host的arp cache表中。

flanneld完成这项工作后，linux kernel就可以在arp table中找到 172.16.57.15对应的mac地址并封装二层以太包了：

***4.包发给min node的flannel之前：kernel的vxlan封包***

我们现在要把包从master node发给minion node，需要再次封包，这个任务在backend为vxlan的flannel network中由linux kernel来完成。

flannel.1为vxlan设备，linux kernel可以自动识别，并将上面的packet进行vxlan封包处理。这个过程中，kernel需要知道对端的public IP，如果不在node上的fdb(forwarding database)以获得上面对端vtep设备（已经从arp table中查到其mac地址：d6:51:2e:80:5c:69）所在的node地址，那么就通过查询etcd（kernel会向用户空间的flanned程序发起”L2 MISS”事件，flanned查询etcd）并注册到fdd中：

master node:

`bridge fdb show dev flannel.1|grep d6:51:2e:80:5c:69
d6:51:2e:80:5c:69 dst {minion node local ip} self permanent`
然后把包封好从eth0发出去：

***5.包发到minion node后：kernel的vxlan拆包***

minion node上的eth0接收到上述vxlan包，kernel将识别出这是一个vxlan包，于是拆包后将packet转给minion node上的vtep（flannel.1）。minion node上的flannel.1再将这个数据包转到minion node上的docker0，继而由docker0传输到Pod3的某个容器里。


## 三、问题

***1.什么叫平坦的网络***

在平坦的flannel network中，每个pod都会被分配唯一的ip地址，且每个k8s node的subnet各不重叠，没有交集。不过这样的subnet分配模型也有一定弊端，那就是可能存在ip浪费：一个node上有200多个flannel ip地址(xxx.xxx.xxx.xxx/24)，如果仅仅启动了几个Pod，那么其余ip就处于闲置状态。

***2.我们在创建cni的时候到底在创建什么***

flannel为Pod分配ip有不同的实现方式，Kubernetes推荐的是基于CNI，另一种是直接与docker结合。

container network interface，Container Network Interface (CNI) 最早是由CoreOS发起的容器网络规范，是Kubernetes网络插件的基础。其基本思想为：Container Runtime在创建容器时，先创建好network namespace，然后调用CNI插件为这个netns配置网络。

使用CNI后，容器的IP分配就变成了如下步骤：
kubelet 先创建pause容器生成network namespace
调用网络CNI driver
CNI driver 根据配置调用具体的cni 插件
cni 插件给pause 容器配置网络
pod 中其他的容器都使用 pause 容器的网络

这时候Pod就直接以cni0作为了自己的网关，而不是docker默认的docker0。所以使用docker inspect查看某个pause容器时，是看不到它的网络信息的。
    - CNI中，docker0的ip与Pod无关，Pod总是生成的时候才去动态的申请自己的IP，而CNM模式下，Pod的网段在docker engine启动时就已经决定。
    - CNI只是一个网络接口规范，各种功能都由插件实现，flannel只是插件的一种


