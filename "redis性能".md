---
title: "redis性能"
author: 一张狗
lastmod: 2019-07-06 08:05:04
date: 2018-08-17 10:19:31
tags: []
---


https://www.flyml.net/2016/08/20/basic-redis-user-guide/

keys的性能问题：Warning: consider KEYS as a command that should only be used in production environments with extreme care. It may ruin performance when it is executed against large databases. This command is intended for debugging and special operations, such as changing your keyspace layout. Don’t use KEYS in your regular application code. If you’re looking for a way to find keys in a subset of your keyspace, consider using SCAN or sets.

执行keys命令，redis会锁定

http://www.fullstackdevel.com/computer-tec/big-data/798.html
https://redis.io/commands/keys
https://redis.io/commands/scan

内存设置：https://blog.tanteng.me/2016/03/redis-maxmemory/


