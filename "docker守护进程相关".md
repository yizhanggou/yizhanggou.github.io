---
title: "docker守护进程相关"
author: 一张狗
lastmod: 2019-07-06 11:55:58
date: 2018-02-12 15:57:22
tags: []
---



## 一、docker守护进程

1. docker的守护进程的层级结构如下图：  
![](http://wiki.baidu.com/download/attachments/452613111/docker-after111.png?version=1&modificationDate=1515478701000&api=v2)  
 三者关系：Docker Engine 负责镜像管理，将镜像交付到 containerd 运行，containerd 使用 runC 来运行容器。Docker Engine和client交互，管理镜像和容器。启动时指定runC实现containerd守护进程，主要用于镜像管理和容器执行，向上为dockerd提供grpc接口，向下通过containerd-shim结合runC，使得引擎可以独立升级，避免之前Docker Daemon升级会导致所有容器不可用的问题。独立负责容器运行时和生命周期（如创建、启动、停止、中止、信号处理、删除等），其他一些如镜像构建、卷管理、日志等由dockerd守护进程的其他模块处理。containerd 只需管理容器，包括容器的启动、停止、暂停和销毁。鉴于容器运行时与引擎隔离，Engine 能够重启和升级，无需重启容器。
实例：  
2. 启动docker之后：（42720是docker守护进程dockerd）

```
[root@your_machine my_container]# pstree -l -a -A 42720
dockerd-current --add-runtime docker-runc=/usr/libexec/docker/docker-runc-current --default-runtime=docker-runc --exec-opt native.cgroupdriver=systemd --userland-proxy-path=/usr/libexec/docker/docker-proxy-current --graph=/home/work/docker --selinux-enabled --log-driver=journald --signature-verification=false
|-docker-containe -l unix:///var/run/docker/libcontainerd/docker-containerd.sock --shim docker-containerd-shim --metrics-interval=0 --start-timeout 2m --state-dir /var/run/docker/libcontainerd/containerd --runtime docker-runc --runtime-args --systemd-cgroup=true
[root@your_machine my_container]# ps -ef |grep 42720
root 16284 12092 0 14:30 pts/4 00:00:00 grep --color=auto 42720
root 42720 1 1 2017 ? 03:24:29 /usr/bin/dockerd-current --add-runtime docker-runc=/usr/libexec/docker/docker-runc-current --default-runtime=docker-runc --exec-opt native.cgroupdriver=systemd --userland-proxy-path=/usr/libexec/docker/docker-proxy-current --graph=/home/work/docker --selinux-enabled --log-driver=journald --signature-verification=false
root 42729 42720 0 2017 ? 00:03:00 /usr/bin/docker-containerd-current -l unix:///var/run/docker/libcontainerd/docker-containerd.sock --shim docker-containerd-shim --metrics-interval=0 --start-timeout 2m --state-dir /var/run/docker/libcontainerd/containerd --runtime docker-runc --runtime-args --systemd-cgroup=true
root 44419 42720 0 2017 ? 00:00:00 xz -d -c -q
```
/usr/bin/dockerd-current是dockerd守护进程，启动时指定runC实现为/usr/libexec/docker/docker-runc-current

/usr/bin/docker-containerd-current是docker-containerd守护进程

3 启动一个容器后：

```
[work@your_machine ~]$ docker run -itd docker.io/centos /bin/bash
c22307e49cc135640e3d72b69244d92aedcef1a400b0080f44d6cfca5a77b04b
[work@your_machine ~]$ docker ps
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
c22307e49cc1 docker.io/centos "/bin/bash" 2 seconds ago Up 1 seconds stoic_liskov
[root@your_machine my_container]# pstree -l -a -A 42720
dockerd-current --add-runtime docker-runc=/usr/libexec/docker/docker-runc-current --default-runtime=docker-runc --exec-opt native.cgroupdriver=systemd --userland-proxy-path=/usr/libexec/docker/docker-proxy-current --graph=/home/work/docker --selinux-enabled --log-driver=journald --signature-verification=false
|-docker-containe -l unix:///var/run/docker/libcontainerd/docker-containerd.sock --shim docker-containerd-shim --metrics-interval=0 --start-timeout 2m --state-dir /var/run/docker/libcontainerd/containerd --runtime docker-runc --runtime-args --systemd-cgroup=true
| |-docker-containe c22307e49cc135640e3d72b69244d92aedcef1a400b0080f44d6cfca5a77b04b /var/run/docker/libcontainerd/c22307e49cc135640e3d72b69244d92aedcef1a400b0080f44d6cfca5a77b04b /usr/libexec/docker/docker-runc-current
| | |-bash
[root@your_machine my_container]# ps -ef |grep 42729

root 17195 42729 0 14:51 ? 00:00:00 /usr/bin/docker-containerd-shim-current c22307e49cc135640e3d72b69244d92aedcef1a400b0080f44d6cfca5a77b04b /var/run/docker/libcontainerd/c22307e49cc135640e3d72b69244d92aedcef1a400b0080f44d6cfca5a77b04b /usr/libexec/docker/docker-runc-current
root 17320 12092 0 14:53 pts/4 00:00:00 grep --color=auto 42729
root 42729 42720 0 2017 ? 00:03:00 /usr/bin/docker-containerd-current -l unix:///var/run/docker/libcontainerd/docker-containerd.sock --shim docker-containerd-shim --metrics-interval=0 --start-timeout 2m --state-dir /var/run/docker/libcontainerd/containerd --runtime docker-runc --runtime-args --systemd-cgroup=true

```

可以看到，启动一个容器后，containerd进程创建了一个docker-containerd-shim（/usr/bin/docker-containerd-shim-current）进程，参数c22307e49cc135640e3d72b69244d92aedcef1a400b0080f44d6cfca5a77b04b是启动容器的id，参数/var/run/docker/libcontainerd/c22307e49cc135640e3d72b69244d92aedcef1a400b0080f44d6cfca5a77b04b是和容器相关的目录，其中包括容器配置、标准输入、标准输出：
```
[root@your_machine my_container]# ll /var/run/docker/libcontainerd/c22307e49cc135640e3d72b69244d92aedcef1a400b0080f44d6cfca5a77b04b
total 20
-rw-r--r--. 1 root root 20441 Jan 9 14:51 config.json
prwx------. 1 root root 0 Jan 9 14:51 init-stdin
prwx------. 1 root root 0 Jan 9 14:51 init-stdout
```

这个容器的json文件：[config.json](http://wiki.baidu.com/download/attachments/452613111/config.json?version=2&modificationDate=1515482480000&api=v2)

最后一行的bash进程是容器中正在运行的进程，因为PID隔离所以在宿主机上看到的pid和在容器内看到的PID不一样：

```
[root@your_machine my_container]# ps -ef |grep 17195
root 17195 42729 0 14:51 ? 00:00:00 /usr/bin/docker-containerd-shim-current c22307e49cc135640e3d72b69244d92aedcef1a400b0080f44d6cfca5a77b04b /var/run/docker/libcontainerd/c22307e49cc135640e3d72b69244d92aedcef1a400b0080f44d6cfca5a77b04b /usr/libexec/docker/docker-runc-current
root 17211 17195 0 14:51 pts/3 00:00:00 /bin/bash
[work@your_machine ~]$ docker attach c22307e49cc1
[root@c22307e49cc1 /]# ps
PID TTY TIME CMD
1 ? 00:00:00 bash
15 ? 00:00:00 ps
```

4. runC  
 runC实现了容器启停，资源隔离等功能。下面只用runC来启停容器：3.1 把docker respoity上的busybox解压到本地：
```
mkdir my_container
cd my_container
mkdir rootfs
docker export $(docker create busybox) | tar -C rootfs -xvf -
[root@your_machine work]# tree -d my_container/
my_container/
`-- rootfs
|-- bin
|-- dev
| |-- pts
| `-- shm
|-- etc
|-- home
|-- proc
|-- root
|-- sys
|-- tmp
|-- usr
| `-- sbin
`-- var
|-- spool
| `-- mail
`-- www

17 directories
```

5. 然后调用runc生成配置文件，主要包含容器挂载信息、平台信息、进程信息等容器启动依赖的所有数据：

生成配置文件：

```
[root@your_machine my_container]# /usr/libexec/docker/docker-runc-current spec
[root@your_machine my_container]# ll
total 8
-rw-r--r--. 1 root root 2342 Jan 9 15:19 config.json
drwxrwxr-x. 12 work work 4096 Jan 8 23:14 rootfs
```

</div></div>[config.json](http://wiki.baidu.com/download/attachments/452613111/config.json?version=2&modificationDate=1515482480000&api=v2)


启动容器：

```
[root@your_machine my_container]# /usr/libexec/docker/docker-runc-current run busybox2
/ # pwd
/
/ # ll
sh: ll: not found
/ # ls
bin dev etc home proc root sys tmp usr var
```

