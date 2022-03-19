---
title: "kubernetes跨namespace访问服务"
author: 一张狗
lastmod: 2019-07-06 08:21:55
date: 2018-08-07 10:43:26
tags: []
---


问题：

namespaceB的应用想要访问部署在namespaceA的service。

解决方案：

在namespaceB中创建一个service，不使用selector，使用type=ExternalName的方式，externalName定义成为指向namespaceA中的service。
```
apiVersion: v1 
kind: Service 
metadata: 
  name: etcd-production 
  spec: ports: 
  - name: port1 
    port: 2379 
    targetPort: 2379 
  - name: port2 
    port: 4001 
    targetPort: 4001 
    sessionAffinity: None 
    type: ExternalName 
    externalName: etcd-production.kube-system.svc.cluster.local
```

验证：在容器内`nslookup etcd-production.kube-system.svc.cluster.local`


