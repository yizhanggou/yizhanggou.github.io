---
title: "redis集群迁移"
author: 一张狗
lastmod: 2019-07-06 11:51:36
date: 2018-02-12 17:48:06
tags: []
---


### redis集群中机器增删改的文章

1. 要fail掉一个主需要一半以上主都投票通过才可以
2. 安装ruby、gem redis - gem redis 4.0.1 有兼容性问题，如果遇到则需要更改版本gem uninstall redis-4.0.1gem install redis -v 3.3.3
3. 实战 
    - 起一个集群，假设在同一台机器上从8110-8115 
    - redis解压到redis目录下，新建一个目录为dm-cluster。修改redis.conf内的pidfile的目录
    - 起redis：注意修改目录结构 
    ```
    cd dm-cluster/8110  
    ../../redis-3.2.8/src/redis-server ./redis.confcd ..  
    cd 8111  
    ../../redis-3.2.8/src/redis-server ./redis.confcd ..  
    cd 8112  
    ../../redis-3.2.8/src/redis-server ./redis.confcd ..  
    cd 8113  
    ../../redis-3.2.8/src/redis-server ./redis.confcd ..  
    cd 8114  
    ../../redis-3.2.8/src/redis-server ./redis.conf
    cd ..  
    cd 8115  
    ../../redis-3.2.8/src/redis-server ./redis.conf
    ```

4. - - 停redis为： - ../bin/redis-cli -p 8110 shutdown nosave  
 ../bin/redis-cli -p 8111 shutdown nosave  
 ../bin/redis-cli -p 8112 shutdown nosave  
 ../bin/redis-cli -p 8113 shutdown nosave  
 ../bin/redis-cli -p 8114 shutdown nosave  
 ../bin/redis-cli -p 8115 shutdown nosave

5. - - 建立集群： `./redis-3.2.8/src/redis-trib.rb create –replicas 1 127.0.0.1:8110 127.0.0.1:8111 127.0.0.1:8112 127.0.0.1:8113 127.0.0.1:8114 127.0.0.1:8115`

6. 起新加入的redis实例，假设这里用9110主9111从
    - 加入集群：主：`./redis-3.2.8/src/redis-trib.rb add-node 127.0.0.1:9110 127.0.0.1:8115`
    - 从：slave身份加入集群：`./redis-3.2.8/src/redis-trib.rb add-node –slave –master-id 83896826cb37a47461e4649124cfd48133f40bbd 127.0.0.1:9111 127.0.0.1:8115</div>`
    - 查看集群状态`./redis-3.2.8/src/redis-trib.rb check 127.0.0.1:9110`
    - 找到要被迁移的实例id以及slots number(比如5461个 slots)
    - reshard哈希槽：`./redis-3.2.8/src/redis-trib.rb reshard 127.0.0.1:8115`按照提示输入要接收的nodeid（目标redis实例），要reshard的哈希槽数目，要被分的nodeid（被分slots的redis实例）期间如果有不可用，则设置该slot为stable：比如从8113中移动，提示有些slot状态不对，则登录入8113的客户端，设置其为stable：注意，这里会丢失数据cluster setslot 8 stable或者用`fix./redis-3.2.8/src/redis-trib.rb fix 127.0.0.1:9110`或者直接`import cluster setslot16383importing6f5cd78ee644c1df9756fc11b3595403f51216cc`
    - 删除原节点：`./redis-3.2.8/src/redis-trib.rb del-node b0cf8e18e13ef0b61e7946b157dd08b1e368235d 127.0.0.1:8115`
-

Tips：可以调用rebalance让redis自己调整整个集群的slot分配状况
Cluster nodes 显示：[节点id] [ip:端口] [标志(master、myself、salve)] [(- 或者主节id)] [ping发送的毫秒UNIX时间，0表示没有ping] [pong接收的unix毫秒时间戳] [配置-epoch] [连接状态] [槽点]
在线迁移参考：http://blog.51cto.com/13544424/2058280
redis集群在建的时候如果用127.0.0.1搭建集群，则其他机器不能访问，解决方法是：修改nodes.conf，把127.0.0.1改为本机ip再重启。（kill掉重启）
