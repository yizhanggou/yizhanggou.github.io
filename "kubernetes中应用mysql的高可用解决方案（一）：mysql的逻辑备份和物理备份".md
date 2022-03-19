---
title: "kubernetes中应用mysql的高可用解决方案（一）：mysql的逻辑备份和物理备份"
author: 一张狗
lastmod: 2018-07-27 16:53:23
date: 2018-07-27 16:50:51
tags: []
---



## 一、概念

逻辑备份是备份执行的sql语句，在恢复的时候执行sql命令，移植性强，通常用mysqldump。

物理备份是备份的数据文件，不具备移植性，比较形象的比喻就是cp，但是不只是cp。

一般来说，物理备份恢复速度比较快，占用空间比较大，逻辑备份速度比较慢，占用空间比较小。逻辑备份的恢复成本高。


## 二、逻辑备份

逻辑备份是备份sql语句，在恢复的时候执行备份的sql语句实现数据库数据的重现。

**1）mysqldump**

mysqldump是采用SQL级别的备份机制，他将数据表导成SQL脚本文件，是最常用的逻辑备份方法。

mysqldump对innodb存储引擎支持热备，innodb支持事务，我们可以基于事务通过mysqldump对数据库进行热备。

mysqldump对myisam存储引擎只支持温备，通过mysqldump对使用myisam存储引擎的表进行备份时，最多只能实现温备，因为在备份时会对备份的表请求锁，当备份完成后，锁会被释放。


## 三、物理备份

物理备份就是备份数据文件了，比较形象点就是cp下数据文件，但真正备份的时候自然不是的cp这么简单。

**1）使用 xtrabackup 工具**

是一个用来备份 MySQL数据库的开源工具。

主要特点：

- 在线热备份。可以备份innodb和myisam。innodb主要应用recovery原理。myisam直接拷贝文件。
- 支持流备份。可以备份到disk，tape和reomot host。–stream=tar ./ | ssh user@remotehost cat “>” /backup/dir/
- 支持增量备份。可以利用lsn和基础备份目录来进行增量备份。
- 支持记录slave上的master log和master position信息。
- 支持多个进程同时热备份，xtrabackup的稳定性还是挺好的。

**2）LVM**

特点：热备、支持所有基于本地磁盘的存储引擎、快速备份、低开销、容易保持完整性、快速恢复等。

**3）cp + tar**

使用直接拷贝数据库文件的方式进行打包备份，需要注意的是执行步骤：锁表、备份、解表。

恢复也很简单，直接拷贝到之前的数据库文件的存放目录即可。

注意：对于Innodb引擎的表来说，还需要备份日志文件，即ib_logfile*文件。因为当Innodb表损坏时，就可以依靠这些日志文件来恢复。

**4）mysqlhotcopy**

mysqlhotcopy是一个perl程序，是lock tables、flush tables 和cp或scp来快速备份数据库。

它是备份数据库或单个表的最快的途径，但它只能运行在数据库文件（包括数据表文件、数据文件、索引文件）所在的机器上。

mysqlhotcopy只能用于备份MyISAM。

**5）使用mysql主从复制**

mysql的复制是指将主数据库的DDL和DML操作通过二进制文件（bin-log）传送到从服务器上，然后在从服务器上对这些日志做重新执行的操作，从而使得从服务器和主服务器保持数据的同步。


