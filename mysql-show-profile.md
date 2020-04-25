---
layout: post
comments: true
title: mysql profile 学习笔记
date: 2018-11-24 12:55:54
tags:
- mysql
categories:
- mysql
---

### mysql profile 简介

`show profile` 和 `show profiles`语句，可以用来显示SQL语句执行期间各种资源的使用信息。

> 注意：
> 这些语句从版本`5.6.7`开始就不建议使用了，并且可能在下一个release版本中删除。后续可以通过`Performance Schema`来替代这些语句。具体可以参考:[Profiling Using Performance Schema](https://dev.mysql.com/doc/refman/5.6/en/performance-schema-query-profiling.html)。

### show profiles 

`show profiles` 展示了服务器最近收到的语句列表。 列表的大小可以通过调整`profiling_history_size`来调整，默认大小是`15`，最大值是`100`。如果设置为0，则和关闭profile功能是一样的。

举个例子：

<!-- more -->

```sql
mysql> show profiles;
+----------+------------+---------------------------------------------------------------------------+
| Query_ID | Duration   | Query                                                                     |
+----------+------------+---------------------------------------------------------------------------+
|        1 | 0.00020000 | SET global profiling = 1                                                  |
|        2 | 0.00054500 | SELECT * FROM employees WHERE first_name='Mary' AND last_name LIKE '%man' |
|        3 | 0.00046600 | SELECT * FROM employees WHERE first_name='Mary' AND last_name LIKE '%man' |
|        4 | 0.00063500 | SELECT * FROM employees WHERE first_name='Mary' AND last_name LIKE '%man' |
+----------+------------+---------------------------------------------------------------------------+
4 rows in set, 1 warning (0.00 sec)
```

### show profile

#### profile 语法

```sql
SHOW PROFILE [type [, type] ... ]
    [FOR QUERY n]
    [LIMIT row_count [OFFSET offset]]

type: {
    ALL
  | BLOCK IO
  | CONTEXT SWITCHES
  | CPU
  | IPC
  | MEMORY
  | PAGE FAULTS
  | SOURCE
  | SWAPS
}
```

`show profile` 可以显示一个SQL语句的详细信息。如果不带`for query n`字句，输出的结果是最近一条语句的相关信息。如果包含`for query n`字句，则`show profile`显示语句`n`的相关信息。 这里的`n`是`show profiles`数据结果里面`Query_ID`列的值。

默认情况下`show profile`只显示`Status`和`Duration`列。`Status`列的值和`SHOW PROCESSLIST`输出结果里面的列`Status`的值类似，但有一点小的区别，具体可以查看[Section 8.14, “Examining Thread Information”](https://dev.mysql.com/doc/refman/5.6/en/thread-information.html)

可以通过指定`type`值来获取你关心的信息。

- ALL 展示所有信息
- BLOCK IO 展示阻塞的输入输出操作的个数
- CONTEXT SWITCHES 展示主动和被动的上下文切换次数。
- CPU 展会用户和系统的CPU使用次数。
- IPC 展示消息发送和接收的次数
- MEMORY 暂时没有实现
- PAGE FAULTS 展示发生缺页错误次数。
- SOURCE 展示源代码中的函数名称，以及函数发生的文件的名称和行号
- SWAPS 展示发生内存`swap`的次数。

### 查看并开启profile

#### 查看

```sql
mysql> select @@profiling;
+-------------+
| @@profiling |
+-------------+
|           0 |
+-------------+
```

上面的语句执行结果展示了是否开启了profile功能。默认是关闭的。

#### 开启

```sql
SET profiling = 1;
or
SET global profiling = 1;
```

### 例子

可以通过执行一些sql语句，然后通过profile来查看这些语句的执行情况。

```sql
mysql> SELECT * FROM employees WHERE first_name='Mary' AND last_name LIKE '%man';
mysql> SELECT * FROM employees WHERE first_name='Mary' AND last_name LIKE '%man';
mysql> SELECT * FROM employees WHERE first_name='Mary' AND last_name LIKE '%man';
```

执行profile分析

```sql
mysql> SHOW PROFILE CPU FOR QUERY 7;
+----------------+----------+----------+------------+
| Status         | Duration | CPU_user | CPU_system |
+----------------+----------+----------+------------+
| starting       | 0.003667 | 0.000094 |   0.000977 |
| query end      | 0.000015 | 0.000009 |   0.000007 |
| closing tables | 0.000007 | 0.000004 |   0.000001 |
| freeing items  | 0.000019 | 0.000009 |   0.000010 |
| cleaning up    | 0.000032 | 0.000020 |   0.000013 |
+----------------+----------+----------+------------+
5 rows in set, 1 warning (0.00 sec)
```


