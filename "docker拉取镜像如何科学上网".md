---
title: "docker拉取镜像如何科学上网"
author: 一张狗
lastmod: 2019-07-06 08:52:07
date: 2018-07-20 17:32:07
tags: []
---



## why？

因为众所周知的原因，在国内访问docker的官方镜像仓库gcr.io会连接不上，我们需要用其他镜像仓库来拉取镜像。这里就涉及到Mirror和Private Registry的区别：


## what？

Private Registry是企业或个人的私有镜像仓库，保存着自己内部的镜像。Mirror是一种代理中转服务。在使用 Private Registry 时，需要在 Docker Pull 或 Dockerfile 中直接键入 Private Registry 的地址，而使用 Mirror 服务，只需要在 Docker 守护进程（Daemon）的配置文件中加入 Mirror 参数，即可在全局范围内透明的访问官方的 Docker Hub。

简单来说，Mirror就是官方仓库的CDN，Mirror只会缓存曾经使用过的镜像。


## how？

### <span style="color: #ff0000;">*方法一、修改镜像仓库*</span>

### 步骤一：配置镜像仓库

在/etc/sysconfig/docker中的OPTIONS中加上：

#### 选择一、官方的中国镜像仓库

--registry-mirror=https://registry.docker-cn.com

配置之后重启服务：service docker restart

#### 选择二、网易的

--registry-mirror=http://hub-mirror.c.163.com

配置之后重启服务：service docker restart

#### 选择三、ustc的镜像

--registry-mirror=https://docker.mirrors.ustc.edu.cn

配置之后重启服务：service docker restart

#### 选择四、DaoCloud

需要注册，注册后

`curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://{your_id}.m.daocloud.io`

实际修改了 /usr/lib/systemd/system/docker.service：

`ExecStart=/usr/bin/docker-current daemon --registry-mirror=http://{your_id}.m.daocloud.io\`

之后需要重启`systemctl enable docker; systemctl daemon-reload ; systemctl restart docker`

#### 选择五、alicloud

注册为阿里云的用户，还得加入开发者平台。

修改/usr/lib/systemd/system/docker.service

`ExecStart=</usr/bin/docker-current daemon --registry-mirror=https://{your_id}.mirror.aliyuncs.com\`

之后需要重启 `systemctl enable docker; systemctl daemon-reload ; systemctl restart docker`

#### 选择六、个人私人仓库推荐

anjia ： https://anjia0532.github.io/2017/11/15/gcr-io-image-mirror/


### *方法二、搜索镜像*

在https://hub.docker.com上搜索之后拉取。这里有很多官方的镜像，也有一些第三方开源的。

修改/etc/sysconfig/docker：加上

`ADD_REGISTRY='–add-registry registry.hub.docker.com'`

然后重启docker服务。


