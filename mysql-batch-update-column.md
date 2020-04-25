---
layout: post
comments: true
title: mysql批量修改列类型
date: 2017-05-24 14:09:28
tags:
- mysql
- 面试
categories:
- 数据库
---

### 背景

今天在开发一个功能时发现一个MySQL表的字段`extra`，类型是`varchar(128)`，该字段是一个冗余字段，用来保存一些json格式的数据。该功能需要往该`extra`字段添加一个子字段，保存数据时报错了。当然了，是因为字段长度超过了定义的长度。如果是单个表，马上就可以修改该字段的长度，但我们的业务是分表的，手动修改工作量太大了，作为聪明的程序员肯定需要用程序员的思维来解决。

### 方案一

首先想到的方案是：写一段Python脚本，动态构建表名，执行`alter table`语句。还没开始前，想到这样的问题别人肯定遇到过，一定有其它的方案。Google了一下，发现了第二种方案。

<!-- more -->

### 方案二

该方案的核心是：利用mysql中保存的meta信息来构造批量修改语句，并执行这些结果语句。

`information_schema`库是MySQL保存meta信息的库，里面有好多表。例如：保存所有数据库信息的：`SCHEMATA`,保存所有表信息的：`TABLES`和视图的`VIEWS`。其中表`COLUMNS`保存了所有表的字段信息，格式如下：

```sql
CREATE TEMPORARY TABLE `COLUMNS` (
  `TABLE_CATALOG` varchar(512) NOT NULL DEFAULT '',
  `TABLE_SCHEMA` varchar(64) NOT NULL DEFAULT '',
  `TABLE_NAME` varchar(64) NOT NULL DEFAULT '',
  `COLUMN_NAME` varchar(64) NOT NULL DEFAULT '',
  `ORDINAL_POSITION` bigint(21) unsigned NOT NULL DEFAULT '0',
  `COLUMN_DEFAULT` longtext,
  `IS_NULLABLE` varchar(3) NOT NULL DEFAULT '',
  `DATA_TYPE` varchar(64) NOT NULL DEFAULT '',
  `CHARACTER_MAXIMUM_LENGTH` bigint(21) unsigned DEFAULT NULL,
  `CHARACTER_OCTET_LENGTH` bigint(21) unsigned DEFAULT NULL,
  `NUMERIC_PRECISION` bigint(21) unsigned DEFAULT NULL,
  `NUMERIC_SCALE` bigint(21) unsigned DEFAULT NULL,
  `CHARACTER_SET_NAME` varchar(32) DEFAULT NULL,
  `COLLATION_NAME` varchar(32) DEFAULT NULL,
  `COLUMN_TYPE` longtext NOT NULL,
  `COLUMN_KEY` varchar(3) NOT NULL DEFAULT '',
  `EXTRA` varchar(27) NOT NULL DEFAULT '',
  `PRIVILEGES` varchar(80) NOT NULL DEFAULT '',
  `COLUMN_COMMENT` varchar(1024) NOT NULL DEFAULT ''
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
```

这个表就是我们第二种方案的重点，可以查询该表获取需要修改的所有表的所有字段信息，然后利用MySQL的字符串函数构造`alter table`语句。

例如：

```sql
select CONCAT('alter table  ',TABLE_NAME,'  modify  ', COLUMN_NAME, ' varchar(50) ;') from information_schema.COLUMNS where TABLE_SCHEMA='trade' and TABLE_NAME like 'order_%' and COLUMN_NAME = 'extra';
```

### 总结

遇到问题多思考，一定要用程序员的思维解决问题。求助Google不会错。

### 参考

[MySQL 批量修改表字段属性](http://weipengfei.blog.51cto.com/1511707/960493)
 

