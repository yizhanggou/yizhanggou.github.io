---
title: "nginx负载均衡和重试机制"
author: 一张狗
lastmod: 2019-07-06 09:48:33
date: 2018-05-15 11:13:39
tags: []
---


https://blog.csdn.net/bigtree_3721/article/details/72792141

http://www.suyunyou.com/aid4.html

**nginx配置web容器实现自动剔除异常(失败)的服务**

具体内容如下：

```
upstream my_server {
    	server   127.0.0.1:8080 weight=1 max_fails=1 fail_timeout=60s;
    	server   127.0.0.1:8088 weight=1 max_fails=1 fail_timeout=60s;
        server   127.0.0.1:8086 weight=7 max_fails=1 fail_timeout=60s down;
    	server   127.0.0.1:8087 weight=7 max_fails=1 fail_timeout=60s backup;
}
```

说明：

上面配置的是同一台服务器，负载了2个tomcat容器，端口分别为8080和8088。请求失败1次后，服务剔除60s（在60s内该服务不会被请求到）。

**参数说明：**

- down 表示单前的server暂时不参与负载
- weight 默认为1.weight越大，负载的权重就越大。
- max_fails ：允许请求失败的次数默认为1.当超过最大次数时，返回proxy_next_upstream 模块定义的错误
- fail_timeout:max_fails次失败后，暂停的时间。
- backup： 其它所有的非backup机器down或者忙的时候，请求backup机器。所以这台机器压力会最轻。

https://blog.csdn.net/mj158518/article/details/49847119

现在对外服务的网站，很少只使用一个服务节点，而是部署多台服务器，上层通过一定机制保证容错和负载均衡。

nginx就是常用的一种HTTP和反向代理服务器，支持容错和负载均衡。

nginx的重试机制就是容错的一种。

在nginx的配置文件中，proxy_next_upstream项定义了什么情况下进行重试，官网文档中给出的说明如下：

1. Syntax: proxy_next_upstream error | timeout | invalid_header | http_500 | http_502 | http_503 | http_504 | http_403 | http_404 | off …;
2. Default:    proxy_next_upstream error timeout;
3. Context:    http, server, location

默认情况下，当请求服务器发生错误或超时时，会尝试到下一台服务器。

还有一个参数影响了重试的次数：proxy_next_upstream_tries，官方文档中给出的说明如下：

1. Syntax: proxy_next_upstream_tries number;
2. Default:    proxy_next_upstream_tries 0;
3. Context:    http, server, location
4. This directive appeared in version 1.7.5.

该配置决定了最多重试多少次，0表示不限制。

不了解这个机制，在日常开发web服务的时候，就可能会踩坑。

比如有这么一个场景：一个用于导入数据的web页面，上传一个excel，通过读取、处理excel，向数据库中插入数据，处理时间较长（如1分钟），且为同步操作（即处理完成后才返回结果）。暂且不论这种方式的好坏，若nginx配置的响应等待时间（proxy_read_timeout）为30秒，就会触发超时重试，将请求又打到另一台。如果处理中没有考虑到重复数据的场景，就会发生数据多次重复插入！（当然，这种场景，内网可以通过机器名访问该服务器进行操作，就可以绕过nginx了，不过外网就没办法了。）

同理，在处理POST请求的时候也需要注意类似的问题。网上有一篇讨论如何阻止POST请求的超时重试，感兴趣的可以看看。


