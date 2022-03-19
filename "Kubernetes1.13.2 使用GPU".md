---
title: "Kubernetes1.13.2 使用GPU"
author: 一张狗
lastmod: 2019-07-06 07:19:25
date: 2019-01-21 10:16:36
tags: []
---


1. 首先安装nvidia驱动，nvidia-smi有输出即安装成功
2. 安装docker，版本最新就行，当前装的是1.13.1
3. 安装nvidia-docker，https://github.com/NVIDIA/nvidia-docker 
    - Centos7使用的是nvidia-container-runtime-hook
    - hook和nvidia-docker的区别：https://xigang.github.io/2018/11/08/nvidia-container-runtime/，简单来讲就是nvidia-docker1实现了docker client的封装，并且在容器启动的时候将GPU device和libraries挂载到容器中，而第二代主要是通过修改json替换原来的runc为NVIDIA Container runtime，然后在NVIDIA Container runtime中，在原有的runc上加一个hook，用于调用libnvidia-container库，该库提供一个库和一个简单的CLI工具，用于使得容器可以调用NVIDIA GPU。
6. 安装kubernetes，当前安装的是1.13.2 1. 
```
 docker pull jicki/kube-proxy:v1.13.2  
 docker pull jicki/kube-controller-manager:v1.13.2  
 docker pull jicki/kube-scheduler:v1.13.2  
 docker pull jicki/kube-apiserver:v1.13.2  
 docker pull jicki/coredns:1.2.6  
 docker pull jicki/cluster-proportional-autoscaler-amd64:1.3.0  
 docker pull jicki/kubernetes-dashboard-amd64:v1.10.0  
 docker pull jicki/etcd:3.2.24  
 docker pull jicki/node:v3.1.3  
 docker pull jicki/ctl:v3.1.3  
 docker pull jicki/kube-controllers:v3.1.3  
 docker pull jicki/cni:v3.1.3  
 docker pull jicki/pause:3.1  
 docker pull jicki/pause-amd64:3.1  
 docker pull quay.io/coreos/flannel:v0.10.0-arm  
 docker pull quay.io/coreos/flannel:v0.10.0-ppc64le  
 docker pull quay.io/coreos/flannel:v0.10.0-s390x
 docker tag jicki/kube-proxy:v1.13.2 k8s.gcr.io/kube-proxy:v1.13.2  
 docker tag jicki/kube-controller-manager:v1.13.2 k8s.gcr.io/kube-controller-manager:v1.13.2  
 docker tag jicki/kube-scheduler:v1.13.2 k8s.gcr.io/kube-scheduler:v1.13.2  
 docker tag jicki/kube-apiserver:v1.13.2 k8s.gcr.io/kube-apiserver:v1.13.2  
 docker tag jicki/coredns:1.2.6 k8s.gcr.io/coredns:1.2.6  
 docker tag jicki/cluster-proportional-autoscaler-amd64:1.3.0 k8s.gcr.io/cluster-proportional-autoscaler-amd64:1.3.0  
 docker tag jicki/kubernetes-dashboard-amd64:v1.10.0 k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0  
 docker tag jicki/etcd:3.2.24 k8s.gcr.io/etcd:3.2.24  
 docker tag jicki/node:v3.1.3 k8s.gcr.io/node:v3.1.3  
 docker tag jicki/ctl:v3.1.3 k8s.gcr.io/ctl:v3.1.3  
 docker tag jicki/kube-controllers:v3.1.3 k8s.gcr.io/kube-controllers:v3.1.3  
 docker tag jicki/cni:v3.1.3 k8s.gcr.io/cni:v3.1.3  
 docker tag jicki/pause:3.1 k8s.gcr.io/pause:3.1  
 docker tag jicki/pause-amd64:3.1 k8s.gcr.io/pause-amd64:3.1
 docker rmi jicki/kube-proxy:v1.13.2  
 docker rmi jicki/kube-controller-manager:v1.13.2  
 docker rmi jicki/kube-scheduler:v1.13.2  
 docker rmi jicki/kube-apiserver:v1.13.2  
 docker rmi jicki/coredns:1.2.6  
 docker rmi jicki/cluster-proportional-autoscaler-amd64:1.3.0  
 docker rmi jicki/kubernetes-dashboard-amd64:v1.10.0  
 docker rmi jicki/etcd:3.2.24  
 docker rmi jicki/node:v3.1.3  
 docker rmi jicki/ctl:v3.1.3  
 docker rmi jicki/kube-controllers:v3.1.3  
 docker rmi jicki/cni:v3.1.3  
 docker rmi jicki/pause:3.1  
 docker rmi jicki/pause-amd64:3.1
```
7. 搭建集群 
    - 开启kubectl服务：`systemctl enable kubelet.service`
    - `kubeadm init –kubernetes-version=v1.13.2 –pod-network-cidr=10.244.0.0/16 –apiserver-advertise-address=<your_ip>`
    - apply flannel： 
        - mkdir -p ~/k8s/  
        - cd ~/k8s  
        - wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml  
        - kubectl apply -f kube-flannel.yml
        - 有可能识别不到网卡，需要手动添加iface=eth0![2018-11-27-095235-1-1](http://yizhanggou.top/imgs/2019/07/2018-11-27-095235-1-1.png)
8. master加入调度：`kubectl taint nodes node1 node-role.kubernetes.io/master-`
9. 安装NVIDIA device plugin，插件以daemonset部署，通过label筛选出有GPU的集群: `kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v1.12/nvidia-device-plugin.yml`
10. 测试：
```
apiVersion: v1 
kind: Pod 
metadata: 
    name: test-gpu 
    spec: volumes: 
    - hostPath: path: /usr/lib64/nvidia 
    name: lib containers: 
    - env: 
        - name: TEST value: "GPU" 
    imagePullPolicy: Always 
    name: gpu-container-1 
    image: tensorflow/tensorflow:latest-gpu 
    resources: 
        limits: nvidia.com/gpu: 1 
        volumeMounts: 
        - mountPath: /usr/local/nvidia/lib64 name: lib
```
12. 注意： 教程中说要设置docker json加上default runtime的，都不需要，最新版本的docker已经有了。
13. 很多教程中都说要设置kubectl的参数Accelerators=true。其实不需要，因为1.13.2已经莫任务true了。设置了之后启动的时候报错unrecognized feature gate: Accelerators。因为这个标签已经在1.11之后已经废弃了。
14. 问题：
    - 跑任务的时候报错CUDA_ERROR_NO_DEVICE，网上查了之后有两种原因：一种是CUDA_VISIBLE_DEVICES设置的问题，一种是cuda驱动没装好。后面发现是因为程序内设置了CUDA_VISIBLE_DEVICES=1，CUDA_VISIBLE_DEVICES的意思是是编号为几的GPU对程序可见。k8s调度的时候会随机选择/dev/下面的任意一张设备挂载在镜像内，如果挂载的不是CUDA_VISIBLE_DEVICES指定的那个就会失败。通过修改CUDA_VISIBLE_DEVICES为0,1,2,3就可以运行了：![5669fdcee75dfcc8eda142960](http://yizhanggou.top/imgs/2019/07/5669fdcee75dfcc8eda142960.png)
    - 跑job的时候希望能够使用job的自动清理功能：spec.ttlSecondsAfterFinished，但是跑的时候发现并没有生效，kubectl get job xxx -o json的时候也发现没有设置上去。后面发现是需要开启TTLAfterFinished这个feature-gates： 
    - kubectl需要加feature-gate：
        - vim /etc/sysconfig/kubelet
        - KUBELET_EXTRA_ARGS=–feature-gates=TTLAfterFinished=true
        - systemctl daemon-reload
        - systemctl restart kubelet.service
    - apiserver、controller-manager、scheduler都需要加–feature-gates=TTLAfterFinished=true：(kubeadm 将会为 API server、controller manager 和 scheduler 生成 Kubernetes 的静态 Pod manifest 文件，并将这些文件放入 /etc/kubernetes/manifests 中)
        - cd /etc/kubernetes/manifests
        - 编辑kube-apiserver.yaml、kube-controller-manager.yaml、kube-scheduler.yaml添加flag
        - 重启：kubectl get po -nkube-system delete掉相应的pod*</div>
15. 参考：
     - https://bluesmilery.github.io/blogs/afcb1072/
     - https://www.kubernetes.org.cn/4956.html
     - http://likakuli.com/post/2018/09/06/kubernetes_gpu/
     - https://k8smeetup.github.io/docs/admin/kubeadm/


