---
title: "Kubernetes笔记——容器的就绪检查和健康检查"
author: 一张狗
lastmod: 2019-07-06 08:19:17
date: 2018-08-14 13:53:38
tags: []
---


K8S的应用程序健康检查分为livenessProbe和readinessProbe，两者相似，但两者存在着一些区别。

livenessProbe在服务运行过程中检查应用程序是否运行正常，不正常将杀掉进程；而readness Probe是用于检测应用程序启动完成后是否准备好对外提供服务，不正常继续检测，直到返回成功为止。

# livenessProbe

许多应用程序经过长时间运行，最终过渡到无法运行的状态，除了重启，无法恢复。通常情况下，K8S会发现应用程序已经终止，然后重启应用程序/pod。

有时应用程序可能因为某些原因（后端服务故障等）导致暂时无法对外提供服务，但应用软件没有终止，导致K8S无法隔离有故障的pod，调用者可能会访问到有故障的pod，导致业务不稳定。K8S提供livenessProbe来检测应用程序是否正常运行，并且对相应状况进行相应的补救措施。


# readinessProbe

在没有配置readinessProbe的资源对象中，pod中的容器启动完成后，就认为pod中的应用程序可以对外提供服务，该pod就会加入相对应的service，对外提供服务。但有时一些应用程序启动后，需要较长时间的加载才能对外服务，如果这时对外提供服务，执行结果必然无法达到预期效果，影响用于体验。

比如使用tomcat的应用程序来说，并不是简单地说tomcat启动成功就可以对外提供服务的，还需要等待spring容器初始化，数据库连接没连上等等。对于spring boot应用，默认的actuator带有/health接口，可以用来进行启动成功的判断。


# 检测方式

## exec-命令

在用户容器内执行一次命令，如果命令执行的退出码为0，则认为应用程序正常运行，其他任务应用程序运行不正常。

``` 
 livenessProbe:  
 exec:  
 command:  
 – cat  
 – /home/laizy/test/hostpath/healthy  
```


## TCPSocket

将会尝试打开一个用户容器的Socket连接（就是IP地址：端口）。如果能够建立这条连接，则认为应用程序正常运行，否则认为应用程序运行不正常。

``` 
 livenessProbe:  
 tcpSocket:  
 port: 8080  
```

## HTTPGet

调用容器内Web应用的web hook，如果返回的HTTP状态码在200和399之间，则认为应用程序正常运行，否则认为应用程序运行不正常。每进行一次HTTP健康检查都会访问一次指定的URL。

```
 httpGet: #通过httpget检查健康，返回200-399之间，则认为容器正常  
 path: / #URI地址  
 port: 80 #端口号  
 #host: 127.0.0.1 #主机地址  
 scheme: HTTP #支持的协议，http或者https  
 httpHeaders：’’ #自定义请求的header  
```



# 参数说明

initialDelaySeconds：容器启动后第一次执行探测是需要等待多少秒。

periodSeconds：执行探测的频率。默认是10秒，最小1秒。

timeoutSeconds：探测超时时间。默认1秒，最小1秒。

successThreshold：探测失败后，最少连续探测成功多少次才被认定为成功。默认是1。对于liveness必须是1。最小值是1。

failureThreshold：探测成功后，最少连续探测失败多少次才被认定为失败。默认是3。最小值是1。

