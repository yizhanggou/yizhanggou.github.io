---
title: "nginx reload的时候发生了什么"
author: 一张狗
lastmod: 2019-07-06 08:53:55
date: 2018-07-17 20:56:33
tags: []
---



## 一、问题

我们在修改nginx配置之后经常需要nginx reload以加载新的配置文件，nginx会处理完旧的请求然后加载新的配置文件。那么nginx reload的时候到底发生了什么呢？


## 二、过程

nginx工作过程中，会起一个master进程，多个worker进程，worker进程负责具体的http等工作，master进程进行控制。
![wps_clip_image7114](http://yizhanggou.top/imgs/2019/07/wps_clip_image7114.png)

当reload的时候，发生了以下事件：

1. master进程检查配置的正确性，如果不正确则不reload，nginx按照原配置工作。
2. 如果正确，则nginx启动新的worker，采用新的配置文件。
3. nginx将新的请求分配给新的worker。
4. nginx等以前的worker处理完旧的请求，关闭以前的woker。
5. 重复上面过程，知道全部旧的worker进程都被关闭掉


## 三、需要注意的点

1. 如果有大量请求，则有可能请求丢失。 - 在某一时刻worker进程是之前的二倍，导致系统资源的下降，多个worker之间会进行锁的竞争。
2. 如果worker子进程比较多则会耗费内存。 - 如果老的worker和后端的机器保持长连接，则在一段时间内nginx的worker进程个数会随着reload的次数成倍数增加。


## 四、代码分析

```
//ngx_reconfigure标志位为1，重新读取配置文件
//nginx不会让原来的worker子进程再重新读取配置文件，其策略是重新初始化ngx_cycle_t结构体，
//用它来读取新的额配置文件再创建新的额worker子进程，销毁旧的worker子进程
if (ngx_reconfigure) {
    ngx_reconfigure = 0;

    //ngx_new_binary标志位为1，平滑升级Nginx
    if (ngx_new_binary) {
    ngx_start_worker_processes(cycle, ccf->worker_processes, NGX_PROCESS_RESPAWN);
    ngx_start_cache_manager_processes(cycle, 0);
    ngx_noaccepting = 0;
    continue;
}

ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reconfiguring");

//初始化ngx_cycle_t结构体
cycle = ngx_init_cycle(cycle);
if (cycle == NULL) {
    cycle = (ngx_cycle_t *) ngx_cycle;
    continue;
}

ngx_cycle = cycle;
ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
//创建新的worker子进程
ngx_start_worker_processes(cycle, ccf->worker_processes,NGX_PROCESS_JUST_RESPAWN);
ngx_start_cache_manager_processes(cycle, 1);

/* allow new processes to start */
ngx_msleep(100);

live = 1;
//向所有子进程发送QUIT信号
    ngx_signal_worker_processes(cycle,
    ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
}
```
