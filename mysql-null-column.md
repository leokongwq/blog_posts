---
layout: post
comments: true
title: mysql中字段究竟该不该为null
date: 2019-01-05 09:54:48
tags:
- mysql
categories:
---

### 为什么会有许多表的字段设置为null？

1. 开发中常用的建表工具创建表时字段默认可以为null。
2. 开发人员不能正确区分null和not null的区别，以为默认null可以节省空间。
3. 默认为null，在插入数据时可以少很多判断

针对这些问题，下面就彻底搞清楚字段该不该为null。    

### null 字段可以节省空间吗？


#### NULL

MySQL官方文档有如下的描述：

> For NULL tables, NULL columns require additional space in the row to record whether their values are NULL. Each NULL column takes one bit extra, rounded up to the nearest byte.

从上面的描述中可以知道，在MyISAM中`NULL`字段并不能完全的节省空间。

<!-- more -->

#### InnoDB

InnoDB 表的行格式

{% asset_img mysql_innodb_row_format.png %}

平时使用COMPACT格式的行较多，正对该格式，文档有如下的说明。

> The variable-length part of the record header contains a bit vector for indicating NULL columns. If the number of columns in the index that can be NULL is N, the bit vector occupies CEILING(N/8) bytes. (For example, if there are anywhere from 9 to 16 columns that can be NULL, the bit vector uses two bytes.) Columns that are NULL do not occupy space other than the bit in this vector. The variable-length part of the header also contains the lengths of variable-length columns. Each length takes one or two bytes, depending on the maximum length of the column. If all columns in the index are NOT NULL and have a fixed length, the record header has no variable-length part.

从上面的内容可以知道，NULL列除不能节省空间，反而会增加空间。

MySQL考虑的是越小的行大小，在固定大小的内存中就可以更多的缓存行数据，提升性能。如果字段有值就存，没值就不存，默认值也不是保存行数据里面的。

### null 字段带来的问题

1. NULL值到非NULL的更新无法做到原地更新，更容易发生页分裂，从而影响性能。
2. NULL值在`timestamp`类型下容易出问题，特别是没有启用参数`explicit_defaults_for_timestamp=true`具体见文章最后的：参考资料
3. `NOT IN`、`!=` 等负向条件查询在有`NULL`值的情况下返回永远为空结果，查询容易出错

1，NOT IN子查询在有NULL值的情况下返回永远为空结果，查询容易出错

```sql NOT IN子查询在有NULL值的情况下返回永远为空结果，查询容易出错
create table table_2 (
	 `id` INT (11) NOT NULL,
	 user_name varchar(20) NOT NULL
)


create table table_3 (
	 `id` INT (11) NOT NULL,
	 user_name varchar(20)
)

insert into table_2 values (4,"zhaoliu_2_1"),(2,"lisi_2_1"),(3,"wangmazi_2_1"),(1,"zhangsan_2"),(2,"lisi_2_2"),(4,"zhaoliu_2_2"),(3,"wangmazi_2_2")

insert into table_3 values (1,"zhaoliu_2_1"),(2, null)

select user_name from table_2 where user_name not in (select user_name from table_3 where id!=1)
```

2，单列索引不存null值，复合索引不存全为null的值，如果列允许为null，可能会得到“不符合预期”的结果集如果name允许为null，索引不存储null值，结果集中不会包含这些记录。所以，请使用not null约束以及默认值。

```sql
select * from table_3 where name != 'zhaoliu_2_1'
```

3，如果在两个字段进行拼接：比如题号+分数，首先要各字段进行非null判断，否则只要任意一个字段为空都会造成拼接的结果为null。

```sql
select CONCAT("1", null) from dual; -- 执行结果为null。
```

4，如果有 Null column 存在的情况下，count(Null column)需要格外注意，null 值不会参与统计。

```sql
// 下面的语句返回 2， 但是数据库里面有4条记录
select count(user_name) from table_3;
```

5， null 字段的判断 需要使用 `is null` 或 `is not null`

### NULL字段和索引

#### 索引长度 key_len

key_len 的计算规则和三个因素有关：数据类型、字符编码、是否为 NULL 

key_len 62 == 20*3（utf8 3字节） + 2 （存储 varchar 变长字符长度 2字节，定长字段无需额外的字节）

key_len 83 == 20*4（utf8mb4 4字节） +  1 (是否为 Null 的标识) + 2 （存储 varchar 变长字符长度 2字节，定长字段无需额外的字节）

### null字段唯一索引

在可以为NULL的字段上也是可以建立唯一索引的，但需要注意的是：唯一索引想要防止重复记录的功能就失效了。官方文档描述如下：

> A UNIQUE index creates a constraint such that all values in the index must be distinct. An error occurs if you try to add a new row with a key value that matches an existing row. For all engines, a UNIQUE index permits multiple NULL values for columns that can contain NULL.

### null字段普通索引

在可以为null的字段上建立普通索引，则所有索引字段为null的索引记录都是排列在一起的。

### 参考资料

https://my.oschina.net/leejun2005/blog/1342985
https://dev.mysql.com/doc/refman/5.6/en/column-count-limit.html
https://dev.mysql.com/doc/refman/5.6/en/innodb-row-format.html
https://dev.mysql.com/doc/refman/5.6/en/is-null-optimization.html
https://www.jianshu.com/p/d7d364745173
https://dev.mysql.com/doc/refman/5.5/en/problems-with-null.html
http://mysql.taobao.org/monthly/2018/01/04/


