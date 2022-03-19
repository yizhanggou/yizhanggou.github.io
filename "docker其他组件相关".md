---
title: "docker其他组件相关"
author: 一张狗
lastmod: 2018-02-05 23:02:06
date: 2018-02-05 22:24:39
tags: ["b'docker'"]
---


Docker Machine：安装Docker环境的工具，可以通过命令快速创建带有docker环境的虚拟机，在虚拟机内可以快速部署docker容器。这使得在不同平台下部署docker变得容易；

Docker Compose：允许用户通过一个模板文件来定义一组相关联的应用容器为一个项目。用于快速在集群中部署分布式应用；也可以用于Docker启动运行脚本的集合，可以把启动容器时带的参数都集中管理在一个yml文件中；

Docker Swarm：Docker原生的集群管理工具，可以方便快速创建集群，也可以结合Compose快速创建集群；

Kubernetes：比较热门的容器集群编排管理工具。Google根据其在Linux上容器管理经验，改造到docker管理上。


