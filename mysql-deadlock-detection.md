---
layout: post
comments: true
title: mysql检测和处理
date: 2017-07-06 10:30:05
tags:
- mysql
categories:
- mysql
- 数据库
---

### 前言

相信很多同学在多年复杂系统的开发中，应该会碰到过MySQL的死锁问题。那MySQL是如何检测死锁？发送死锁后MySQL会怎么处理呢？
<!-- more -->

### innodb_deadlock_detect

`innodb_deadlock_detect`是MySQL的一个系统变量。

版本：	5.7.15
命令行格式：--innodb-deadlock-detect
影响范围：Global
参数类型：boolean
默认值：ON
作用：该选项使用了禁用MySQL的死锁检测功能的。在高并发系统上，当许多线程等待同一个锁时，死锁检测可能导致速度减慢。 有时，当发生死锁时，如果禁用了死锁检测则可能会更有效，这样可以依赖`innodb_lock_wait_timeout`的设置进行事务回滚。

### MySQL死锁检测

MySQL默认情况下是开启了死锁检测的，InnoDB自动检测发送死锁的事务，并回滚其中的一个事务或所有导致死锁的事务。InnoDB会在导致死锁的十五中选择一个权重比较小的事务来回滚，这个权重值可能是由该事务insert, updated, deleted的行数决定的。

如果`innodb_table_locks = 1(默认值)`并且`autocommit = 0`，则InnoDB能感知到表锁的存在，并且上层的MySQL层知道行级锁。 否则，InnoDB无法检测到由`MySQL LOCK TABLES`语句设置的表锁或由除InnoDB之外的存储引擎设置的锁定的死锁。 通过设置`innodb_lock_wait_timeout`系统变量的值来解决这些情况。

当InnoDB执行事务的完全回滚时，将释放由事务设置的所有锁。 但是，如果单个SQL语句由于错误而回滚，则语句设置的某些锁可能会被保留。 这是因为InnoDB以一种格式存储行锁，以致之后不能知道哪个锁由哪个语句设置。

如果SELECT调用事务中存储的函数，并且函数中的语句失败，则该语句将回滚。 此外，如果在此之后执行ROLLBACK，整个事务将回滚。

如果InnoDB监控器输出的`最近死锁检测`部分包含一条消息，指出`TOO DEEP OR LONG SEARCH IN THE LOCK TABLE WAITS-FOR GRAPH, WE WILL ROLL BACK FOLLOWING TRANSACTION`，这表示处于等待的事务列表长度已达到限制200。超过200个事务的等待列表被视为死锁，并且将回滚尝试检查等待列表的事务。 如果锁定线程必须查看等待列表上的事务拥有的超过1,000,000个锁，则也可能发生相同的错误。

#### 禁用死锁检测

可以通过选项:`innodb_deadlock_detect`来关闭死锁检测。










