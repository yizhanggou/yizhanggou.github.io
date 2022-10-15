---
title: "kubernetes相关——network的ip耗尽"
author: 一张狗
lastmod: 2019-07-06 07:30:02
date: 2018-12-18 15:46:55
tags: []
---


问题描述：客户的机器经常断电重启，到某一次之后就显示所有pod都Completed状态，describe的时候信息是：”Failed to setup network for pod \”xxxxxx\” using network plugins \”cni\”: no IP addresses available in network: cbr0; Skipping pod”，

解决方式：
```
cd /var/lib/cni/networks/cbr0
for hash in $(tail -n +1 * | grep '^[A-Za-z0-9]*$' | cut -c 1-8); do if [ -z $(docker ps -a | grep $hash | awk '{print $1}') ]; then grep -irl $hash ./; fi; done | xargs rm
kubectl delete po –all –nkube-system
```

问题产生原因：

起服务的时候配置了网段是10.244.0.0/24，ls -la /var/lib/cni/networks/cbr0 | wc -l 发现已经分配了255个ip，但是kubectl get all –all-namespaces | wc -l其实没有用到那么多ip，这说明已经分配的ip因为某些原因没有回收，这个已经在1.6版本的k8s上fix了。

reference：

- https://stackoverflow.com/questions/42450386/kubernetes-frequently-gets-error-adding-network-no-ip-addresses-available-in
- https://github.com/kubernetes/kubernetes/issues/34278#issuecomment-254686727


