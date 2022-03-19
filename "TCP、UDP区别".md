---
title: "TCP、UDP区别"
author: 一张狗
lastmod: 2018-05-22 00:46:49
date: 2010-04-28 11:18:49
tags: []
---


TCP是面向连接的，UDP是无连接的。即是说，TCP传输之前要建立连接，UDP不需要。

TCP是面向字节流的，UDP是面向报文的。

TCP是可靠的，UDP不提供可靠交付。TCP传输的数据，不丢失，不重复，无差错，按序到达。

TCP首部开销20字节，UDP8字节。

TCP逻辑通信信道是全双工的可靠信道，UDP是不可靠信道。

 

三次握手：

四次分手：

https://blog.csdn.net/xifeijian/article/details/12777187

滑动窗口


