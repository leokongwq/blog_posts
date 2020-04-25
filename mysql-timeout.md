---
layout: post
comments: true
title: mysql中重要超时处理机制
date: 2017-07-06 08:42:58
tags:
- mysql
categories:
- 数据库
- mysql
---

### 背景

应用中的一个事务性方法锁住了一条记录(select ... for update), 并在锁住记录后陷入了死循环，由此导致其它事务对该条记录的任何操作都会处于等待中。此问题也可以理解为，一个事务加锁了某些记录，但是长时间不提交，导致大量事务等待超时。那这个超时时间是多少呢？如何控制这个超时时间呢？

<!-- more -->

### wait_timeout

`wait_timeout`是mysql的一个系统变量，可以通过启动时的命令行参数`--wait-timeout`来指定，也可以在运行时动态指定，支持`session`和`Global`。 

类型：integer
单位：秒
默认值：28800 （8个小时）
取值范围: [1 - 31536000]， windows系统的最大值：2147483
含义：MySQL服务端在关闭非活跃连接前等待的秒数。

depending on the type of client (as defined by the CLIENT_INTERACTIVE connect option to mysql_real_connect()). See also interactive_timeout.

在线程启动时，session级别的`wait_timeout`初始值来自global级别的`wait_timeout`值或global级别的`interactive_timeout`的值，这具体取决于客户端类型（是否在调用函数`mysql_real_connect()`时，使用了 `CLIENT_INTERACTIVE`选项）。

### interactive_timeout

`interactive_timeout`也是mysql的一个系统变量，可以通过启动时的命令行参数`--interactive_timeout`来指定，也可以在运行时动态指定，支持`session`和`Global`。 

类型：integer
单位：秒
默认值：28800 （8个小时）
最小值：1
含义：MySQL服务端在关闭交互式连接前等待的秒数。

在关闭交互式连接之前，服务器等待活动的秒数。 交互式客户端被定义为在调用函数`mysql_real_connect()`使用`CLIENT_INTERACTIVE`选项的客户端。其实通常就是Mysql自带的客户端工具`mysql`

### wait_timeout vs interactive_timeout vs rollback

1. 控制连接最大空闲时长的wait_timeout参数。
2. 对于非交互式连接，类似于`jdbc`连接，`wait_timeout`的值继承自服务器端全局变量`wait_timeout`。对于交互式连接，类似于mysql客户单连接，`wait_timeout`的值继承自服务器端全局变量`interactive_timeout`。
3. 当出现这两种超时后，MySQL服务端的会关闭对于的连接，并回滚未提交的事务。

### innodb_lock_wait_timeout

`innodb_lock_wait_timeout`是一个系统变量，可以通过命令行参数`--innodb-lock-wait-timeout`，支持`session`和`global`范围并可以动态修改。

类型：integer
单位：秒
默认值：50
最小值：1
最大值：1073741824
含义：InnoDB事务在放弃前等待行锁的超时时间。在锁等待超时后，会有如下的信息输出：

```
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

当出现锁超时，当前的语句会被回滚（并不是整个事务），如果想要在锁超时后导致整个事务回滚，需要在启动MySQL Server时添加命令行参数`--innodb_rollback_on_timeout`。更多信息参考[InnoDB Error Handling](https://dev.mysql.com/doc/refman/5.7/en/innodb-error-handling.html)

你可以为针对重视交互的应用程序或OLTP系统减少此值，以此来实现快速失败，或将任务插入到队列，等待后续的处理。可以为长时间运行的后端操作增加此值，例如等待其他大型插入或更新操作完成的数据仓库中的转换步骤。

`innodb_lock_wait_timeout` 只适用于InnoDB行锁。 InnoDB中不会发生表锁，并且此超时不适用于等待表锁。

在启用`innodb_deadlock_detect`时，次锁等待超时值不适用于死锁（默认值），因为InnoDB会立即检测到死锁并回滚其中一个导致死锁的事务。 当`innodb_deadlock_detect`被禁用时，发生死锁后，InnoDB依赖`innodb_lock_wait_timeout`进行事务回滚。 请参见第14.5.5.2节[死锁检测和回滚](https://dev.mysql.com/doc/refman/5.7/en/innodb-deadlock-detection.html)

`innodb_lock_wait_timeout`可以在运行时通过`SET GLOBAL or SET SESSION`语句设置。改变`Global`配置需要有超级管理员权限，并会影响后续新建立的连接。任何客户端设置`Session`级别的`innodb_lock_wait_timeout`的值都不会影响其它客户端。

.......未完待续。

### 参考

[https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_wait_timeout](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_wait_timeout)

[http://www.cnblogs.com/ivictor/p/5979731.html](http://www.cnblogs.com/ivictor/p/5979731.html)

[https://stackoverflow.com/questions/9936699/mysql-rollback-on-transaction-with-lost-disconnected-connection](https://stackoverflow.com/questions/9936699/mysql-rollback-on-transaction-with-lost-disconnected-connection)

