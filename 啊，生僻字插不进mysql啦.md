---
title: "啊，生僻字插不进mysql啦"
author: 一张狗
lastmod: 2019-07-06 08:50:06
date: 2018-07-24 15:02:03
tags: []
---



## 一、问题

卤煮用filebeat和logstash来作为日志收集的组件，通过jdbc插入到mysql中（因为后面需要做模型训练的语料所以这里插入mysql而不是ES等其他组件），遇到个问题，生僻字不能支持，报错Incorrect string value。


## 二、原因

查了一下，mysql的utf8编码最多是三个字节，如果要用到四个字节需要用utf8mb4。所有大于0xffff的字符，全都在UTF编码表的辅助平面内（域辅助平台对应的是基础平面，简称BMP）

解决方法：

1. 数据库表中存表情符号的字段的字符集需要设置成utf8mb4
    - mysql的字符集设置可以指定数据库级别的字符集、字符序。
    - https://dev.mysql.com/doc/refman/5.6/en/charset-unicode-utf8mb4.html
2. 需要在插入语句前加一句set names utf8mb4，或者在init sql的时候加上;
    - 如果没有加上那么插入的时候会报错：Caused by: java.sql.SQLException: Incorrect string value: ‘\xF0\x9F\x98\x81’ for column ‘data’ at row 1
4. jdbc的connection_string后面加上connection_string => “jdbc:mysql://127.0.0.1:3306/db?useUnicode=true&characterEncoding=UTF-8&allowMultiQueries=true”
    - 加上characterEncoding=UTF-8是保证中文没有乱码，如果没有加上则插入到数据库中的是乱码
    - 如果没有加上allowMultiQueries则会报错
    `Caused by: com.mysql.jdbc.exceptions.jdbc4.MySQLSyntaxErrorException: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ‘insert into t select ‘????” at line 1`
        - allowMultiQueries Allow the use of ‘;’ to delimit multiple queries during one statement (true/false), defaults to ‘false’, and does not affect the addBatch() and executeBatch() methods, which instead rely on rewriteBatchStatements. Default: false Since version: 3.1.1
    - characterEncoding不能设置为utf8mb4，会报错驱动不支持：
        - 11:15:52.231 [[main]-pipeline-manager] ERROR com.zaxxer.hikari.pool.HikariPool – HikariPool-1 – Exception during pool initialization.
        - java.sql.SQLException: Unsupported character encoding ‘utf8mb4’.

4. 有疑问？
    - 为什么显式执行了set names utf8mb4，却仍需要在jdbc url中显式指定characterEncoding=UTF-8?
    因为若不显式指定characterEncoding为UTF-8的话,默认的字符集为cp1252（因为character_set_server=latin1），这个时候生僻字的四个字节会被掰成两个字节存储。




## 三、怎么验证

mysql登陆后set names utf8mb4;


