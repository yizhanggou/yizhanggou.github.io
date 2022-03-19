---
title: "Centos7上安装ceph以及遇到的一些问题"
author: 一张狗
lastmod: 2019-07-06 10:04:35
date: 2018-05-25 10:45:17
tags: []
---


http://www.chinacloud.cn/show.aspx?id=27762&cid=81

https://blog.csdn.net/ywy463726588/article/details/42743923

http://www.cnblogs.com/me115/p/6366374.html


# 一、Ceph架构

Ceph生态系统可以大致划分为四部分：
1.客户端（数据用户）
2.元数据服务器（缓存和同步分布式元数据）
3.对象存储集群（将数据和元数据作为对象存储，执行其他关键职能）
4.集群监视器（执行监视功能）。

块存储、文件存储、对象存储的区别：https://blog.csdn.net/qq_36357820/article/details/74390809
ceph的基本概念：
http://www.d-kai.me/ceph%E7%A7%91%E6%99%AE/
http://www.cnblogs.com/kevingrace/p/8430213.html
# 二、Ceph安装

https://blog.csdn.net/aixiaoyang168/article/details/78788703
https://zhangchenchen.github.io/2017/02/28/install-ceph-with-domestic-yum-source/
        之前在项目中使用glusterfs作为分布式存储，后面公司内有人测试发现gluster的性能在大量读写的情况下会降低，于是打算换成ceph存储集群，这里记录一下安装步骤：

## 1.配置yum源

1. 所有节点上执行：更换为国内163源
```
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak && wget -O/etc/yum.repos.d/CentOS-Base.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo
#更换epel源为中科大的
rpm -Uvh http://mirrors.ustc.edu.cn/centos/7/extras/x86_64/Packages/epel-release-7-9.noarch.rpm
rpm –import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
#更换ceph源为163的
export CEPH_DEPLOY_REPO_URL=http://mirrors.163.com/ceph/rpm-jewel/el7
export CEPH_DEPLOY_GPG_URL=http://mirrors.163.com/ceph/keys/release.asc
yum clean all
yum makecache
```
2. admin节点上执行：yum 配置其他依赖包
```
$ sudo yum install -y yum-utils && sudo yum-config-manager –add-repo https://dl.fedoraproject.org/pub/epel/7/x86_64/ && sudo yum install –nogpgcheck -y epel-release && sudo rpm –import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7 && sudo rm /etc/yum.repos.d/dl.fedoraproject.org*
```
# 添加 Ceph 源
```
$ sudo vim /etc/yum.repos.d/ceph.repo
[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://download.ceph.com/rpm-jewel/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=1
```
## 2.安装配置ceph

本实验使用的centos版本：CentOS Linux release 7.5.1804 (Core)
内核版本：3.10.0-693.el7.x86_64
节点数：3，admin+二个node

### 前置条件

ceph需要使用ssh直连各个节点：

1.配置节点host：把以下三行加入每个节点的/etc/hosts
```
> <admin_ip> admin  
>  <node0_ip> node0  
>  <node1_ip> node1
```
2.每台机器修改hostname
```
> $ cat /etc/hostname  
>  admin  
>  $ cat /etc/hostname  
>  node0  
>  $ cat /etc/hostname  
>  node1
```
修改不了hostname的话，需要把admin、node0、node1换成机器的hostname（去除点）\

3.安装ceph-deploy

ceph-deploy是Ceph提供的配置Ceph集群的工具

yum install ceph-deploy

4.安装ntp校对时间，安装ssh
```
> yum install ntp ntpdate ntp-doc  
>  ntpdate 0.cn.pool.ntp.org  
>  yum install openssh-server
```
5.创建ceph部署用户

ceph-deploy 工具必须以普通用户登录 Ceph 节点，且此用户拥有无密码使用 sudo 的权限，因为它需要在安装软件及配置文件的过程中，不必输入密码。官方建议所有 Ceph 节点上给 ceph-deploy 创建一个特定的用户，而且不要使用 ceph 这个名字。这里为了方便，我们使用 cephd 这个账户作为特定的用户，而且每个节点上（admin-node、node0、node1）上都需要创建该账户，并且拥有 sudo 权限。

所有节点都需要创建cephd用户：
```
> useradd -d /home/cephd -m cephd  
>  passwd cephd  
>  echo “cephd ALL = (root) NOPASSWD:ALL” | sudo tee /etc/sudoers.d/cephd  
>  chmod 0440 /etc/sudoers.d/cephd
```
`ceph-deploy mon create-initial`


## 3.遇到的问题：

1.如果ceph-deploy osd activate提示Permission denied，则需要chmod -R777 osd0 ，chmod -R777 osd1

2.如果ceph-deploy osd activate提示ceph_disk.main.Error: Error: ceph osd start failed: Command ‘[‘/usr/bin/systemctl’, ‘start’, ‘ceph-osd@0′]’ returned non-zero exit status 1，
或者** ERROR: error creating empty object store in /run/local/osd0: (22) Invalid argument
http://blog.51cto.com/michaelkang/1786298
http://tracker.ceph.com/issues/13833

需要再手动chown chmod osd0和osd1目录

3.提示找不到fsid

http://www.voidcn.com/article/p-ejsvedpf-beh.html

No cluster conf found in /etc/ceph with fsid 则需要通过ceph -s获取cluster id，并填写到ceph.conf里面fsid，然后再分发出去：
```
ceph-deploy –overwrite-conf osd prepare <machine0>:<folder0> <machine1>:<folder1>
ceph-deploy –overwrite-conf mon create-initial
ceph-deploy osd activate<machine0>:<folder0> <machine1>:<folder1>
```
4.Your backend filesystem appears to not support attrs large enough to handle the configured max rados name size. You may get unexpected ENAMETOOLONG errors on rados operations or buggy behavior

https://webcache.googleusercontent.com/search?q=cache:5uVYi3kFCC0J:https://tonybai.com/2016/11/07/integrate-kubernetes-with-ceph-rbd/+&cd=7&hl=zh-CN&ct=clnk

ceph去检查当前osd所在的目录的文件系统支持的最大attr value size = 1024（此时我的宿主机的文件系统是ext4）小于osd 默认的osd_max_object_name_len(2048)。解决方法是：

官方不建议采用ext4文件系统作为ceph的后端文件系统，如果采用，那么对于ext4的filesystem，应该在ceph.conf中添加如下配置：
```
osd max object name len = 256
osd max object namespace len = 64
```
修改/etc/ceph/ceph.conf并分发到其他节点，然后重新active。5.安装的时候报No section: ‘ceph’

yum remove ceph-release

可能是之前装一半挂了，必须删除干净再装
```
ceph-deploy purge <admin> <node0> <node1>
ceph-deploy purgedata <admin> <node0> <node1>
ceph-deploy forgetkeys
```

6.如果ceph osd tree发现都是down，则看日志/var/log/ceph/ceph-osd.3.log

如果有ERROR: osd init failed: (36) File name too long，则处理方式同4

7./home下有可能挂载失败

http://bbs.ceph.org.cn/question/683

8.ceph-deploy new <admin_machine>的时候报错No module named ceph_deploy.cli

需要使用python2.7.5


## 3.硬件选择

http://www.cnblogs.com/kevingrace/p/8430161.html

