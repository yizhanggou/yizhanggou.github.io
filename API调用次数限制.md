---
title: "API调用次数限制"
author: 一张狗
lastmod: 2019-07-06 09:41:30
date: 2018-06-12 15:51:38
tags: []
---


https://blog.csdn.net/SunnyYoona/article/details/51228456

https://blog.csdn.net/chdhust/article/details/51471202


## 一、令牌桶算法

令牌桶算法最初来自于计算机网络，为了限制网络流出流量以减少拥堵情况，使流量以比较均匀的方式向外发送，用于计算机网络的网络流量整形和速率限制。

大小固定的令牌桶以恒定速率源源不断的产生令牌，令牌数最大是桶数，每次请求都消耗一个令牌。传送到令牌桶的数据包根据数据大小消耗不同的令牌数。如果令牌桶中有足够的令牌则可以发送，如果没有则不允许发送。因此，如果突发门限被合理地配置并且令牌桶中有足够的令牌，那么流量就可以以峰值速率发送。

对于当前没有获得足够令牌的流量，有三种处理方法：（1）它们可以被丢弃；（2）它们可以排放在队列中以便当令牌桶中累积了足够多的令牌时再传输；（3）它们可以继续发送，但需要做特殊标记，网络过载的时候将这些特殊标记的包丢弃。

![20160423213519927](/imgs/2019/07/20160423213519927.png)


## 二、漏桶算法

http://blog.51cto.com/leyew/860302

漏桶算法可以理解为一个会漏水的桶，水是网络上的流量，桶本身具有一个恒定的速率往下滴水，上方时快时慢的有水滴进桶中，当桶满了的时候，上方的水就无法加入了桶中了。

此条件下如何处理上方欲留下的水，则有了下面两种常见的方式。
Traffic Shaping和Traffic Policing
在桶满水之后，常见的两种处理方式为：
1. 暂时拦截住上方水的向下流动，等待桶中的一部分水漏走后，再放行上方水。
2. 溢出的上方水直接抛弃。
将水看作网络通信中数据包的抽象，则
方式1起到的效果称为Traffic Shaping，
方式2起到的效果称为Traffic Policing。
由此可见，Traffic Shaping的核心理念是“等待”，Traffic Policing的核心理念是“丢弃”。它们是两种常见的流速控制方法。

![223552670](/imgs/2019/07/223552670.jpg)

漏桶算法能够强行限制数据的传输速率，而令牌桶算法在能够限制数据的平均传输速率外，还允许某种程度的突发传输。


## 三、使用redis实现API调用次数

http://redisdoc.com/string/incr.html

http://www.cnblogs.com/iforever/p/5796902.html

http://vinoyang.com/2015/08/23/redis-incr-implement-rate-limit/

第一种：使用INCR和EXPIRE

伪代码：

```
current = GET(key)

if current !=NULL and current >= max_request_time:

error “too many request”

else:

       INCR(key,1)
       if value == 1:
          EXPIRE(value,1)
```

这里会有一个race condition，即是如果在INCR后没有执行EXPIRE，则这个key会泄漏，符合该key的请求在到达max_request_time后将永远不能继续请求。
第二种：使用RPUSH
```
current = LLEN(key)
if current >max_request_time:
    error “too many requests per second”
else:
    if EXISTS(key) == false:
       MULTI
          RPUSH(key,key)
          EXPIRE(key,1)
       EXEC
    else:
       RPUSHX(key,key)
```
rpushx命令只在key存在时才会将值加入list。这里仍然有个race condition，如果在我们检查EXISTS的时候返回false，但是执行MULTI的时候这个key存在了，那么这里会丢失一次API调用，这是可以接受的，比第一种情况好太多。


## 四、C++库

qLibc实现了一个令牌桶算法，线程安全。


