---
title: "kubernetes集群使用Ceph"
author: 一张狗
lastmod: 2019-07-06 09:37:13
date: 2018-06-01 12:53:15
tags: []
---


https://blog.csdn.net/aixiaoyang168/article/details/78999851

经实验，在一个node上多个Pod是可以以ReadWrite模式挂载同一个CephRBD，但是跨node则不行，会提示image xxx is locked by other nodes。而我们的应用场景是需要多个node挂载一个ceph的，在我们的应用场景需要使用CephFS。

使用cephfs的场景：创建一个fs，挂载的时候指定path。


kubernetes使用CephFS的两种方式：

1.直接通过pod挂载

```
apiVersion: v1
kind: Pod
metadata:
name: cephfs2
spec:
containers:
- name: cephfs-rw
image: busybox
command: ["sleep", "60000"]
volumeMounts:
- mountPath: "/mnt/cephfs"
name: cephfs
volumes:
- name: cephfs
cephfs:
monitors:
- '&lt;your_etcd_ip&gt;:6789'
user: admin
secretRef:
name: ceph-secret
readOnly: false
```

2.通过创建pv、pvc挂载

在ceph集群上找到key：

`[cephd@<your_ceph_machine> ~]$ ceph auth get-key client.admin | base64
QVFBNEhnNWJpQmN1RWhBQUhWSmJKZTVtOG9jWUdkNmlYMnA5dmc9PQ==`



创建secret：

```
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
data:
  key: QVFBNEhnNWJpQmN1RWhBQUhWSmJKZTVtOG9jWUdkNmlYMnA5dmc9PQ==
```



PV：

```
apiVersion: v1
kind: PersistentVolume
metadata:
name: cephfs-pv
spec:
capacity:
storage: 1Gi
accessModes:
– ReadWriteMany
cephfs:
monitors:
– <your_etcd_ip>:6789

path: /sns
user: admin
secretRef:
name: ceph-secret
readOnly: false
persistentVolumeReclaimPolicy: Recycle
```

PVC:
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
name: cephfs-pv-claim
spec:
accessModes:
– ReadWriteMany
resources:
requests:
storage: 1Gi
```
创建POD:

```
apiVersion: v1
kind: Pod
metadata:
labels:
test: cephfs-pvc-pod
name: cephfs-pv-pod1
spec:
containers:
– name: cephfs-pv-busybox1
image: busybox
command: [“sleep”, “60000”]
volumeMounts:
– mountPath: “/mnt/cephfs”
name: cephfs-vol1
readOnly: false
volumes:
– name: cephfs-vol1
persistentVolumeClaim:
claimName: cephfs-pv-claim
```

遇到的问题：

1.映射到内核的时候报错RBD image feature set mismatch

http://blog.51cto.com/hipzz/1888048

–image-format format-id

format-id取值为1或2，默认为 2。

format 1 – 新建 rbd 映像时使用最初的格式。此格式兼容所有版本的 librbd 和内核模块，但是不支持较新的功能，像克隆。

format 2 – 使用第二版 rbd 格式， librbd 和 3.11 版以上内核模块才支持（除非是分拆的模块）。此格式增加了克隆支持，使得扩展更容易，还允许以后增加新功能。

**解决方案1：**

更改为格式1，重新映射。

注意：需要重新建立镜像。

[root@ceph1 ~]# rbd create block1 –image-format 1 –size 1024  
 rbd: image format 1 is deprecated  
 [root@ceph1 ~]# rbd ls  
 block1  
 block  
 [root@ceph1 ~]# rbd map block1  
 /dev/rbd0  
 [root@ceph1 ~]#

d.如上所示，映射正确。

**解决方案2：**

根据官网介绍，新建rbd默认格式2的rbd 块支持如下特性，并且默认全部开启：

–image-feature：

layering: 支持分层

striping: 支持条带化 v2

exclusive-lock: 支持独占锁

object-map: 支持对象映射（依赖 exclusive-lock ）

fast-diff: 快速计算差异（依赖 object-map ）

deep-flatten: 支持快照扁平化操作

journaling: 支持记录 IO 操作（依赖独占锁）

接下来尝试少开启一些特性：

[root@ceph1 ~]# rbd create block2 –image-feature layering –size 1024  
 [root@ceph1 ~]# rbd map block2  
 /dev/rbd1

2.创建pod挂载的时候遇到rbd: map failed executable file not found in $PATH

k8s集群内的节点上需要安装ceph-client:

*yum install ceph*–*common*

3.umount的时候出现target is busy

umount -l xxx

https://www.cnblogs.com/dkblog/archive/2012/07/18/2597192.html

https://blog.csdn.net/u012207077/article/details/21159339

4.如果k8s的node跟ceph集群的node不一样，则需要在k8s的node上部署ceph-common

yum install ceph-common

5.创建pod的时候提示，mount过去的时候提示libceph: bad option

k8s secret 认证 key 需要使用 base64 编码，有可能是secret文件里的key没有base64编码：

在ceph节点上ceph auth <span class="hljs-keyword">get</span>-key client.admin |base64

填到secret文件里面。


6.如果mount fail，则去机器上查看kubelet的日志

7.多用户隔离

https://www.jianshu.com/p/96a34485f0fc

需要用pool，给user指定目录和权限，之后在pv中使用。

8.mount子目录

https://www.spinics.net/lists/ceph-devel/msg34698.html

`mount -t ceph >> 172.24.0.4:6789:/volumes/kubernetes/test1 /tmp/mnt -o >> name=bar,secret=AQA+ln9Yfm6DKhAA10k7QkdkfIAKqmM6xeCsxA==`


9.写入到共享存储的时候提示File Exists

目录权限问题，需要与Dockerfile中指定的USER的权限一样


