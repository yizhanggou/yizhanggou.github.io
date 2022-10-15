---
title: "Kubernetes笔记——service的负载均衡"
author: 一张狗
lastmod: 2019-07-06 09:59:27
date: 2018-05-11 18:11:07
tags: []
---


参考：https://toutiao.io/posts/lyrg6q/preview
https://blog.csdn.net/WaltonWang/article/details/55236300
https://kubernetes.io/cn/docs/concepts/services-networking/service/#ip-%E5%92%8C-vip
调试：http://docs.kubernetes.org.cn/733.html#i-6


# 零、太长不看版本

1. kube-dns配置本身的po的ip到每个容器的/etc/resolv.conf 
```
search default.svc.cluster.local svc.cluster.local cluster.local baidu.com
nameserver 10.96.0.10
options rotate
options ndots:5
[work@your_machine us-k8s]$ kubectl get svc -nkube-system
NAME CLUSTER-IP EXTERNAL-IP PORT(S) AGE
kube-dns 10.96.0.10 <none> 53/UDP,53/TCP 27d
```
2. kubedns监控Kubernetes中Service和Endpoint的变化，把DNS信息保存在内存结构中，po中每次通过service访问其他pod都在这里获取域名相对应的svc ip
3. 通过svc ip，就可以访问pod内的服务，具体是kube-proxy通过iptables实现的


# 一、Kubernetes的pod间互联怎么做？

问题：我们在访问pod的时候需要用到pod的ip，但是当滚动升级时k8s是用新pod替换旧pod，那么pod就会产生迁移，ip是会变的，这时候就引入了service这个概念，service通过暴露VIP，把请求路由到后端pod的ip。但是我们怎么获取service的VIP呢？这时候就需要用到kube-dns，kube-dns暴露一个静态IP作为每个pod的DNS的配置，pod内部在访问Service的域名的时候，通过kube-dns获取该Service的VIP。在访问该Service的VIP的时候，kube-proxy通过userspace或者iptables模式把请求转发到到具体pod的ip。这两者的区别如下：

userspace和iptables这两者的区别主要在于处理请求转发的主题是哪个。

userspace模式下，request转发到kube-proxy监听的接口，kube-proxy终止该连接，并重新建立一个到后端service的新连接，然后转发request到后端并返回response给本地进程。userspace的好处是，因为请求是应用创建的，所以如果后端fail了可以重新建立一个连接尝试不同后端。

iptables模式下，添加iptables rule以对request进行转发，这比userspace高效，因为这样就不需要把request从内核转移到kube-proxy然后再返回到内核，主要的缺点是调试比较麻烦。iptables模式下，负载分配方式是随机选择的。

总结：userspace模式下，kube-proxy自身作为代理，iptables模式下，kube-proxy通过配置iptables rule进行转发。


# 二、iptables模式下kubedns是怎么做转发的？

再来看一下iptables模式下request是怎么转发到端的：

1 service是如何选中需要转发到哪些pod的：

service会虚拟出一个VIP，在它销毁之前保持该VIP不变，通过对VIP的访问，以代理的方式负载到pod上。

2 在创建pod的时候带上label：

```
template:
  metadata:
    labels:
    name: semansearch1
[work@your_machine etcd-k8s]$ kubectl get deployment
NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
autoqaserver   4         4         4            4           3d
``` 
```
[work@your_machine etcd-k8s]$ kubectl get po -o wide
NAME READY STATUS RESTARTS AGE IP NODE
autoqaserver-851494763-dw4w7 1/1 Running 0 5d 10.244.0.83 nodeA
autoqaserver-851494763-k6qxz 1/1 Running 0 6d 10.244.1.15 nodeB
autoqaserver-851494763-n314m 1/1 Running 0 6d 10.244.0.74 nodeA
autoqaserver-851494763-qdkx9 1/1 Running 0 6d 10.244.1.17 nodeB
```

3 在创建service的时候带上selector：
```
spec:  
 selector:  
 name: semansearch1
```
service通过标签查找到pod。

```
[work@your_machine etcd-k8s]$ kubectl get service -o wide
NAME CLUSTER-IP EXTERNAL-IP PORT(S) AGE SELECTOR
autoqaserver 10.99.129.154 <none> 8461/TCP 18d name=autoqaserver
```

4 service与pod的映射关系是怎么维护的？

service与pod的映射关系其实是通过endpoint来维护的，service创建的时候，如果有selector标签，则会同时创建一个endpoint维护暴露的pod的ip。service会定期检查pod的状态，并将结果更新到endpoint上。
```
[work@your_machine etcd-k8s]$ kubectl get ep
NAME ENDPOINTS AGE
autoqaserver 10.244.0.74:8461,10.244.0.83:8461,10.244.1.15:8461 + 1 more... 18d
```

所以，如果我们需要访问的是外部服务，或者pod不存在标签，或者不同namespace等情况，那么我们就手动创建endpoint和service就行。

5.kube-proxy是如何使用iptables做到服务代理的

5.1请求先到prerouting：

```
[root@your_machine ~]# iptables -t nat -nL PREROUTING
Chain PREROUTING (policy ACCEPT)
target prot opt source destination
KUBE-SERVICES all -- 0.0.0.0/0 0.0.0.0/0 /* kubernetes service portals */
DOCKER all -- 0.0.0.0/0 0.0.0.0/0 ADDRTYPE match dst-type LOCAL
```

5.2 我们再看看KUBE-SERVICES 链：vip是10.99.129.154的那个转发链

```
[root@your_machine ~]# iptables -t nat -nL KUBE-SERVICES
Chain KUBE-SERVICES (2 references)
target prot opt source destination
KUBE-SVC-TCOU7JCQXEZGVUNU udp -- 0.0.0.0/0 10.96.0.10 /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
KUBE-SVC-WUGHIENKODVIRRO4 tcp -- 0.0.0.0/0 10.99.129.154 /* default/autoqaserver:autoqaserver-port cluster IP */ tcp dpt:8461
KUBE-NODEPORTS all -- 0.0.0.0/0 0.0.0.0/0 /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
```
5.3找到之后查看这个Service链：

```
[root@your_machine ~]# iptables -t nat -nL KUBE-SVC-WUGHIENKODVIRRO4
Chain KUBE-SVC-WUGHIENKODVIRRO4 (1 references)
target prot opt source destination
KUBE-SEP-UQRUI2LVT2KA3QBW all -- 0.0.0.0/0 0.0.0.0/0 /* default/autoqaserver:autoqaserver-port */ statistic mode random probability 0.25000000000
KUBE-SEP-XZQLQOJNQE4MT6N2 all -- 0.0.0.0/0 0.0.0.0/0 /* default/autoqaserver:autoqaserver-port */ statistic mode random probability 0.33332999982
KUBE-SEP-F2MXXC5YCUZVZADI all -- 0.0.0.0/0 0.0.0.0/0 /* default/autoqaserver:autoqaserver-port */ statistic mode random probability 0.50000000000
KUBE-SEP-6HAPOU5OENUHH2L7 all -- 0.0.0.0/0 0.0.0.0/0 /* default/autoqaserver:autoqaserver-port */
```

我们可以看到这个Service链上有四个endpoint，以及每个endpoint的负载策略。

这里解说一下iptables的规则：这里四个调用链是从上往下匹配的，所有的调用链都是均衡的（1/4）：

当到达第一个的时候，是 1/4 的概率；  
 到达第二个的时候，是 3/4 * 1/3 = 1/4 的概率；  
 到达第三个的时候，是 3/4 * 2/3 * 1/2 = 1/4 的概率；  
 到达第四个的时候，是 3/4 * 1/3 * 1/2 * 1= 1/4 的概率；

这里附上kube-proxy选择权重的代码：

```
// Now write loadbalancing & DNAT rules.
	n := len(endpointChains)
	for i, endpointChain := range endpointChains {
		// Balancing rules in the per-service chain.
		args := []string{
			"-A", string(svcChain),
			"-m", "comment", "--comment", svcName.String(),
		}
		if i < (n - 1) {
			// Each rule is a probabilistic match.
			args = append(args,
				"-m", "statistic",
				"--mode", "random",
				"--probability", fmt.Sprintf("%0.5f", 1.0/float64(n-i)))
		}
```

5.4抽取一个endpoint来看：

```
[root@your_machine ~]# iptables -t nat -nL KUBE-SEP-UQRUI2LVT2KA3QBW
Chain KUBE-SEP-UQRUI2LVT2KA3QBW (1 references)
target prot opt source destination
KUBE-MARK-MASQ all -- 10.244.0.74 0.0.0.0/0 /* default/autoqaserver:autoqaserver-port */
DNAT tcp -- 0.0.0.0/0 0.0.0.0/0 /* default/autoqaserver:autoqaserver-port */ tcp to:10.244.0.74:8461
```

可以看到nat链如下:
![k8s-svc-2](http://yizhanggou.top/imgs/2019/07/k8s-svc-2.png)

 

6.那么，如果我们需要在集群外部访问，Service是如何暴露服务的呢？

service的配置文件中，spec.type就是用来设置服务暴露的方式：

- ClusterIP：默认方式，提供一个虚拟IP供集群内部访问
- NodePort：使用主机的网络地址，开一个端口供外部访问
- LoadBalancer：通过外部负载均衡，通常需要云服务商提供
- ExternalName：这个也是在集群内发布服务用的，需要借助KubeDNS(version >= 1.7)的支持，就是用KubeDNS将该service和ExternalName做一个Map，KubeDNS返回一个CNAME记录（比如foo.bar.example.com）

7.那么到底是谁负责修改iptables的呢？

https://www.jianshu.com/p/bbb673e79c3e

![1786811-d09d01237bfe8ecc](http://yizhanggou.top/imgs/2019/07/1786811-d09d01237bfe8ecc.png)

- **apiserver** 用户通过kubectl命令向apiserver发送创建service的命令，apiserver接收到请求以后将数据存储到etcd中。
- **kube-proxy** kubernetes的每个节点中都有一个叫做kube-proxy的进程，这个进程负责感知service，pod的变化，并将变化的信息写入本地的iptables中。
- **iptables** 使用NAT等技术将virtualIP的流量转至endpoint中。

# 三、kubedns的原理？

参考：http://open.tenxcloud.com/d/16-kube-dns
https://segmentfault.com/a/1190000007342180
https://jimmysong.io/posts/configuring-kubernetes-kube-dns/

![](http://yizhanggou.top/imgs/2019/07/前世今生2.png)

kube-dns包括三个组件：

kubedns：请求API Server监控Kubernetes中Service和Endpoint的变化，把DNS信息保存在内存结构中。

dnsmasq：DNS缓存，以加快查找。

execheathz：对容器的健康检查监控。

kubedns使用树状结构保存域名节点，叶子节点包含Entries，非叶子结点保存ChildNode。kubedns监视Service资源，对于Add、Delete、Update调用起回调函数，回调函数中处理保存在内存结构中树状的DNS信息。


# 四、负载均衡

四层负载均衡和七层负载均衡的区别？

https://www.cnblogs.com/kevingrace/p/6137881.html

https://www.jianshu.com/p/fa937b8e6712

所谓四层负载均衡，也就是主要通过报文中的目标地址和端口，再加上负载均衡设备设置的服务器选择方式，决定最终选择的内部服务器。

所谓七层负载均衡，也称为“内容交换”，也就是主要通过报文中的真正有意义的应用层内容，再加上负载均衡设备设置的服务器选择方式，决定最终选择的内部服务器。


# 五、Ingress

https://cloud.tencent.com/developer/article/1010572

https://cloud.tencent.com/developer/article/1010572

1.ingress是什么？

Ingress 使用开源的反向代理负载均衡器来实现对外暴漏服务，比如 Nginx、Apache、Haproxy等。Nginx Ingress 一般有三个组件组成：

- Nginx 反向代理负载均衡器
- Ingress Controller Ingress Controller 可以理解为控制器，它通过不断的跟 Kubernetes API 交互，实时获取后端 Service、Pod 等的变化，比如新增、删除等，然后结合 Ingress 定义的规则生成配置，然后动态更新上边的 Nginx 负载均衡器，并刷新使配置生效，来达到服务自动发现的作用。
- Ingress Ingress 则是定义规则，通过它定义某个域名的请求过来之后转发到集群中指定的 Service。它可以通过 Yaml 文件定义，可以给一个或多个 Service 定义一个或多个 Ingress 规则。


# 六、Traefik

https://cloud.tencent.com/developer/article/1010575

在 Kubernetes 中使用 nginx 作为前端负载均衡，通过 Ingress Controller 不断的跟 Kubernetes API 交互，实时获取后端 Service、Pod 等的变化，然后动态更新 Nginx 配置，并刷新使配置生效，来达到服务自动发现的目的，而 Traefik 本身设计的就能够实时跟 Kubernetes API 交互，感知后端 Service、Pod 等的变化，自动更新配置并热重载。大体上差不多，但是 Traefik 更快速更方便，同时支持更多的特性，使反向代理、负载均衡更直接更高效。

![plpq33jl1z](/imgs/2019/07/plpq33jl1z.png)

https://mritd.me/2016/12/06/try-traefik-on-kubernetes/#13ingress

https://jiajunhuang.com/articles/2017_10_24-traefik.md.html

在 Kubernetes v1.0 版本，Service 是 “4层”（TCP/UDP over IP）概念。 在 Kubernetes v1.1 版本，新增了 Ingress API（beta 版），用来表示 “7层”（HTTP）服务。

由于微服务架构以及 Docker 技术和 kubernetes 编排工具最近几年才开始逐渐流行，所以一开始的反向代理服务器比如 nginx、apache 并未提供其支持，毕竟他们也不是先知；所以才会出现 Ingress Controller 这种东西来做 kubernetes 和前端负载均衡器如 nginx 之间做衔接；即 Ingress Controller 的存在就是为了能跟 kubernetes 交互，又能写 nginx 配置，还能 reload 它，这是一种折中方案；而最近开始出现的 traefik 天生就是提供了对 kubernetes 的支持，也就是说 traefik 本身就能跟 kubernetes API 交互，感知后端变化，因此可以得知: 在使用 traefik 时，Ingress Controller 已经无用了！
