---
title: "glusterfs命令"
author: 一张狗
lastmod: 2019-07-06 08:07:40
date: 2018-08-16 14:21:46
tags: []
---


添加peer：`gluster peer probe HOSTNAME`

添加brick：`gluster volume add-brick VOLNAME NEW_BRICK`

例子：`gluster volume add-brick test-volume server4:/exp4`

添加的时候增加replica：`gluster volume add-brick replica 3 test-volume server4:/exp4`

查看volume：`gluster volume info`

查看peer：`gluster peer status`

删除peer：`gluster peer detach 10.240.0.123`

删除的时候如果报错peer detach: failed: Brick(s) with the peer 10.240.0.123 exist in cluster那么就是这个peer还被用在volume中，需要手动移除：`gluster volume remove-brick glusterfs 10.240.0.123:/mnt/storage/glusterfs force`

如果报错：volume remove-brick commit force: failed: Removing bricks from replicate configuration is not allowed without reducing replica count explicitly那就是需要降低replica或者增加新的brick进去才能删除原来的brick：

- 减少replica：`gluster volume remove-brick glusterfs replica 1 10.240.0.123:/mnt/storage/glusterfs`
- 增加brick：`gluster volume add-brick annoy_build nodeA:/home/annoy_build nodeB:/data/annoy_build force`

删除volume：`gluster volume delete`


gluster在一台机器上开启volume：

`gluster volume create drobo hydro1:/bricks/1 hydro1:/bricks/2 hydro1:/bricks/3 hydro1:/bricks/4`


