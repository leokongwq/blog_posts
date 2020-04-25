---
layout: post
comments: true
title: mysql只读事务解析
date: 2017-10-21 21:40:42
tags:
- mysql
categories:
- database
---

### 背景

在项目中发现有大量的查询方法设置了`@Transactional(readOnly=true)`，印象中也了解到数据库会对只读事务做一些优化，但并没有升入的了解具体是如何实现的，应用中该如何使用只读事务。这篇文章就对只读事务做一个分析总结。

### autocommit

在分析只读事务前很有必要提一下`autocommit`。MySQL默认对每一个新建立的连接都启用了`autocommit`模式。在该模式下，每一个发送到MySQL服务器的`sql`语句都会在一个单独的事务中进行处理，执行结束后会自动提交事务，并开启一个新的事务。具体可以参考[innodb-autocommit-commit-rollback](https://dev.mysql.com/doc/refman/5.7/en/innodb-autocommit-commit-rollback.html)。

### 只读事务

A type of transaction that can be optimized for InnoDB tables by eliminating some of the bookkeeping involved with creating a read view for each transaction. Can only perform non-locking read queries. It can be started explicitly with the syntax START TRANSACTION READ ONLY, or automatically under certain conditions. See Section 8.5.3, “Optimizing InnoDB Read-Only Transactions” for details.

只读事务是事务的一种，可以用来优化`InnoDB`表上查询事务创建的效率，并可以提供非锁定查询的性能。

InnoDB引擎有一下两种方式来检测一个事务是否是只读事务。

1. 通过`START TRANSACTION READ ONLY`语句来检测。
在该事务中如果有对数据库的修改操作(InnoDB, MyISAM, 或其它类型的数据库表)，那么将会产生一个错误，并且事务会继续以只读事务进行。
但是在只读事务中，你可以对`session`级别的临时表进行操作，又或者执行一个锁定查询。这是因为这些修改操作和加锁操作对其它事务是不可见的。
2. 打开`autocommit`设置，此时事务确保只有一个SQL语句被执行，并且事务中的语句是一个非锁定查询的语句。非锁定查询的语句指的是不使用`FOR UPDATE` 或 `LOCK IN SHARED MODE`限制的查询语句。

这样，针对报表生成这类查询比较集中的应用，你可以将一些查询语句组合起来放在一个`START TRANSACTION READ ONLY` 和 `COMMIT`语句中间，或者在执行查询语句前打开`autocommit`设置，又或者避免在事务中的多个语句中包含导致数据变化的语句。

### 只读事务的优化

InnoDB can avoid the overhead associated with setting up the transaction ID (TRX_ID field) for transactions that are known to be read-only. A transaction ID is only needed for a transaction that might perform write operations or locking reads such as SELECT ... FOR UPDATE. Eliminating unnecessary transaction IDs reduces the size of internal data structures that are consulted each time a query or data change statement constructs a read view.

### 参考

[glos_read_only_transaction](https://dev.mysql.com/doc/refman/5.6/en/glossary.html#glos_read_only_transaction)






