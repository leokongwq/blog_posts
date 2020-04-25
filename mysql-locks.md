---
layout: post
comments: true
title: mysql中的锁
date: 2017-06-11 10:07:41
tags:
- mysql
categories:
- mysql
- 数据库
---

### autocommit, Commit, and Rollback

在innodb中，用户的每个操作内部都会对应一个事务，如果开启了`autocommit`模式，则每一个SQL语句都会在它自己的事务中执行，默认情况下，MySQL对每个新建立连接上的会话都开启了`autocommit`模式，因此如果每个SQL语句的执行没有返回错误，则MySQL自动会该SQL执行后执行`commit`操作。如果SQL执行失败，则MySQL会根据错误的类型来选择执行`commit`还是`rollback`。详情参考[https://dev.mysql.com/doc/refman/5.7/en/innodb-error-handling.html](https://dev.mysql.com/doc/refman/5.7/en/innodb-error-handling.html)

<!-- more -->

启用了`autocommit`选项的会话可以显式的通过`start transaction`或`begin`来开启一个多语句事务，并通过`commit`或`rollback`来结束该事务。详情参考[https://dev.mysql.com/doc/refman/5.7/en/commit.html](https://dev.mysql.com/doc/refman/5.7/en/commit.html)

如果在一个会话中，通过`set autocommit = 0`关闭了`autocommit`， 则该会话总是处于一个事务中，在`commit`或`rollback`执行后会结束当前的事务，并开启一个新的事务。

如果在一个禁用了`autocommit`的会话中没有显式的提交最后的事务，则MySQL会回滚最后的事务。

一些语句会隐式的结束事务，就好像在执行语句之前完成了一个COMMIT。详情参考[https://dev.mysql.com/doc/refman/5.7/en/implicit-commit.html](https://dev.mysql.com/doc/refman/5.7/en/implicit-commit.html)

`COMMIT`意味着当前事务中所做的更改是永久性的，并且对其它的会话变得可见。 另一方面，`ROLLBACK`语句取消了当前事务所做的所有修改。`COMMIT`和`ROLLBACK`都释放了在当前事务中设置的所有InnoDB锁。

MySQL默认开启`autocommit`，可以通过下面的命令来查看`autocommit`的状态：

```sql
show variables like 'autocommit'
```

### Grouping DML Operations with Transactions

By default, connection to the MySQL server begins with autocommit mode enabled, which automatically commits every SQL statement as you execute it. This mode of operation might be unfamiliar if you have experience with other database systems, where it is standard practice to issue a sequence of DML statements and commit them or roll them back all together.

To use multiple-statement transactions, switch autocommit off with the SQL statement SET autocommit = 0 and end each transaction with COMMIT or ROLLBACK as appropriate. To leave autocommit on, begin each transaction with START TRANSACTION and end it with COMMIT or ROLLBACK. The following example shows two transactions. The first is committed; the second is rolled back.

```sql
shell> mysql test

mysql> CREATE TABLE customer (a INT, b CHAR (20), INDEX (a));
Query OK, 0 rows affected (0.00 sec)
mysql> -- Do a transaction with autocommit turned on.
mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)
mysql> INSERT INTO customer VALUES (10, 'Heikki');
Query OK, 1 row affected (0.00 sec)
mysql> COMMIT;
Query OK, 0 rows affected (0.00 sec)
mysql> -- Do another transaction with autocommit turned off.
mysql> SET autocommit=0;
Query OK, 0 rows affected (0.00 sec)
mysql> INSERT INTO customer VALUES (15, 'John');
Query OK, 1 row affected (0.00 sec)
mysql> INSERT INTO customer VALUES (20, 'Paul');
Query OK, 1 row affected (0.00 sec)
mysql> DELETE FROM customer WHERE b = 'Heikki';
Query OK, 1 row affected (0.00 sec)
mysql> -- Now we undo those last 2 inserts and the delete.
mysql> ROLLBACK;
Query OK, 0 rows affected (0.00 sec)
mysql> SELECT * FROM customer;
+------+--------+
| a    | b      |
+------+--------+
|   10 | Heikki |
+------+--------+
1 row in set (0.00 sec)
mysql>
```

### 客户端语言中的事务

在诸如PHP，Perl DBI，JDBC，ODBC或MySQL的标准C调用接口之类的API中，您可以像SQL或SQL语句一样将事务控制语句（如COMMIT）作为字符串发送到MySQL服务器。 一些API还提供单独的特殊事务提交和回滚函数或方法。

### Locking Reads

如果你查询数据，然后在同一事务中插入或更新相关数据，则常规SELECT语句不能提供足够的保护。 其他交易可以更新或删除刚才查询的相同行。 InnoDB支持两种类型的锁定读取，提供额外的安全性：

#### select **** lock in share mode

在读取的任何行上设置共享模式锁定。 其它会话可以读取行，但在你的事务提交之前不能修改它们。 如果任何这些行由尚未提交的另一个事务更改，则你的查询将等待直到该事务结束，然后使用最新值。

需要注意的是：该语句必须在事务中执行，否则因为`autocommit`的原因，就不会得到期望的结果。

#### select **** for update

对于索引记录，搜索遇到，锁定行和任何关联的索引条目，就像您为这些行发出UPDATE语句一样。 阻止其他事务更新这些行，从执行SELECT ... LOCK IN SHARE MODE或从某些事务隔离级别读取数据。 一致的读取忽略在读取视图中存在的记录上设置的任何锁定。 （记录的旧版本无法锁定;通过在记录的内存中复制应用撤销日志来重构记录。）

处理树形结构或图形结构的数据时，这些子句在单个表中或跨多个表分割时主要有用。 您可以从一个地方到另一个地方穿过边缘或树枝，同时保留返回权限并更改任何这些“指针”值。

当事务提交或回滚时，所有由`LOCK IN SHARE MODE`和`FOR UPDATE`查询设置的锁将被释放。

> 注意
使用`SELECT FOR UPDATE`锁定更新的行仅适用于禁用自动提交时（通过以`START TRANSACTION`开始事务或将autocommit设置为0）。如果启用了自动提交功能，则与规范匹配的行不会被锁定。

#### Locking Read Examples

假设要将一个新行插入子表，并确保子行在父表中有父行。 你的应用程序代码可以确保整个操作顺序的引用完整性。

首先，使用一致的读取来查询表PARENT并验证父行是否存在。 你可以安全地将子行插入表CHILD吗？ 不，因为一些其它会话可能会在`select`和`insert`之间删除父行，而你并没有感知。

为了避免潜在的问题, 可以通过使用`LOCK IN SHARE MODE`来执行查询：

```sql
SELECT * FROM parent WHERE NAME = 'Jones' LOCK IN SHARE MODE;
```

在`LOCK IN SHARE MODE`查询返回父“Jones”行后，你可以安全地将子记录添加到CHILD表并提交事务。 任何尝试获取PARENT表中满足条件的行中的排它锁的任何事务将等待，直到你的事务完成，也就是说，直到所有表中的数据处于一致状态。

另一个例子，考虑一个表CHILD_CODES中的整数计数器字段，用于为添加到表CHILD的每条记录分配唯一的标识符。 如果不使用一致性读取或共享模式读取来读取计数器的当前值，则数据库的两个用户可以看到计数器的值相同，如果两个事务尝试添加行，则会出现重复键错误 与CHILD表相同的标识符。

在这里，`LOCK IN SHARE MODE`不是一个好的解决方案，因为如果两个用户同时读取计数字段的值，则它们之中至少有一个在尝试更新计数器时会发生死锁。 原因是：A 和 B 事务都可以成功执行`LOCK IN SHARE MODE`， 然而，当其中的一个需要更新锁定的数据时，会发现其它事务也锁定了数据就会发生死锁。

要实现读取和递增计数器，首先使用`FOR UPDATE`执行计数器的锁定读取，然后增加计数器。 例如：

```sql
SELECT counter_field FROM child_codes FOR UPDATE;
UPDATE child_codes SET counter_field = counter_field + 1;
```

`SELECT ... FOR UPDATE`读取最新的可用数据，在其读取的每一行上设置*排他锁*。

上面的描述只是用来说明`SELECT ... FOR UPDATE`工作原理的一个例子。 在MySQL中，生成唯一标识符的具体任务实际上可以使用对表的单一访问来实现：

```sql
UPDATE child_codes SET counter_field = LAST_INSERT_ID(counter_field + 1);
SELECT LAST_INSERT_ID();
```

SELECT语句仅检索标识符信息（特定于当前连接）。 它没有访问任何表。原理是：

> 两个事务并发更新同一条记录， 其中只有一个事务能给需要更新的记录加排它锁，另一个事务必须等到前一个事务完成计数字段的更新后，才能获取锁完成更新。

扩展：

可以由专门的一个计数表来生成每个业务表主键ID， 每个业务表一行记录：

```sql
mysql> select * from sequence;
+----+---------+----------+
| id | tblName | sequence |
+----+---------+----------+
|  1 | users   |        0 |
|  2 | blogs   |        0 |
+----+---------+----------+
2 rows in set (0.00 sec)
```

每次获取指定表新增记录的主键时，就可以通过执行下面的语句来获取：

```sql
UPDATE sequence SET sequence = LAST_INSERT_ID(sequence + 1) where tblName = '表名';
SELECT LAST_INSERT_ID();
```

搭配MySQL主从， 就可以实现高可用。 这样的设计能满足中小型应用的主键需求了。 

更多关于`LAST_INSERT_ID()`的问题可以参考[对mysql中last_insert_id()的新理解](http://sucre.blog.51cto.com/1084905/723808) 这篇文章。

### innodb锁分类

#### 共享锁和排它锁

InnoDB 实现了标准的行级锁， 锁的类型分为两种：[共享锁(S)](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_shared_lock), [排它锁(X)](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_exclusive_lock)。

- 共享锁允许事务持有锁来读取行。
- 排它锁允许事务持有锁来执行行的更新和删除。

如果事务`T1`拥有行`r`上的共享锁，其它的事务可以同时请求行`r`上的共享锁， 如果其它事务请求行`r`上的排它锁，则其它事务必须等待事务`T1`释放了共享锁之后才能获取行`r`上的排它锁。

如果事务`T1`拥有行`r`上的排它锁， 则其它任何事务都不能再获取行`r`上的任何类型的锁， 必须等事务`T1`释放了`r`上的锁之后才能继续执行。

#### 意向锁

InnoDB支持多种锁定粒度，允许行级锁和表锁共存。 为了实现多粒度级别的锁定，InnoDB使用了一种称为`意向锁`的锁。 意向锁是InnoDB中的表级锁，表示该事务在随后需要在该表某行上加锁的类型（共享或排他）。 在InnoDB中使用了两种类型的意向锁（假设事务T已经在表t上请求了指示类型的锁）：

- 意向共享锁(IS): 事务T意图在表t中的各个行上设置S锁。
- 意向排它锁(IX): 事务T意图在这些行上设置X锁。

例如: `SELECT ... LOCK IN SHARE MODE` 设置意向共享锁， `SELECT ... FOR UPDATE`设置意向排它锁

意向锁协议具体说明如下:

- 在事务可以获取表`t`中一行的`S`锁之前，必须首先在`t`上获取`IS`或更强的锁。
- 在事务可以获取表`t`中一行上的X锁之前，必须首先在t上获取IX锁。

这些规则可以通过以下锁类型兼容性矩阵方便地进行总结。


| 锁类型 | X | IX | S | IS |
| --- | --- | --- | --- | --- |
| x | Conflict | Conflict | Conflict | Conflict |
| IX | Conflict | Compatible | Conflict | Compatible |
| S | Conflict | Conflict | Compatible | Compatible |
| IS | Conflict | Compatible | Compatible | Compatible |

一个事务能获取它所需要的锁的前提是，新申请的锁和已有的锁是兼容的。 一个事务会因为锁冲突而一直等待，直到发生冲突的锁释放为止。如果一个加锁请求和已经存在的锁发送冲突，则该加锁请求不会成功，因为它可能导致死锁或错误的发生。

因此，意图锁不会阻止除了全表请求之外的任何东西（例如，LOCK TABLES ... WRITE）。 `IX`和`IS`锁的主要目的是显示某人正在锁定一行，或者将要表中锁定一行。

意图锁的事务数据类似于[SHOW ENGINE INNODB STATUS](https://dev.mysql.com/doc/refman/5.7/en/show-engine.html)和[InnoDB监视器](https://dev.mysql.com/doc/refman/5.7/en/innodb-standard-monitor.html)输出中的以下内容：

```sql
TABLE LOCK table `test`.`t` trx id 10080 lock mode IX
```

#### 记录锁（行锁）

记录锁是索引记录上的锁。 例如，`SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;` 阻止任何其它事务插入，更新或删除`t.c1`的值为10的行。

记录锁总是锁定索引记录，即使一个表没有定义任何索引。 对于没有定义任何所有这种情况，InnoDB创建一个隐藏的聚簇索引，并使用此索引进行记录锁定。 请参见[Clustered and Secondary Indexes](https://dev.mysql.com/doc/refman/5.7/en/innodb-index-types.html)

在[SHOW ENGINE INNODB STATUS](https://dev.mysql.com/doc/refman/5.7/en/show-engine.html)和[InnoDB监视器](https://dev.mysql.com/doc/refman/5.7/en/innodb-standard-monitor.html)输出中，记录锁的事务数据类似于以下内容：

```sql
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t` 
trx id 10078 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```

#### 间隙锁(Gap 锁)

间隙锁是在索引记录之间的间隙上加锁，或在第一个索引记录之前或在最后一个索引之后的间隙上加锁的锁。例如，`SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE;`阻止其他事务将值15插入到列`t.c1`中，无论列中是否有任何此类值，因为该范围中所有现有值之间的空白被锁定。

间隙可能跨越单个索引值，多个索引值，甚至为空。

间隙锁是性能和并发性之间权衡的一部分，并且只会在某些事务隔离级别中使用（RR），而不是其他事务隔离级别。

使用唯一索引来搜索并锁定唯一行的语句不需要间隙锁定。（但这不包括搜索条件仅包含多列唯一索引的某些列的情况; 在这种情况下，会发生间隙锁定）。例如，如果`id`列具有唯一的索引，则以下语句仅使用`ID`为100的行的索引记录锁，它并不会在意其它的会话是否在前面的间隙中插入行。

```sql
SELECT * FROM child WHERE id = 100;
```

如果id没有被索引或者有一个非唯一的索引，则该语句确实锁定了前面的间隙。

在这里也值得注意的是，不同的事务可以在同一个间隙加有冲突的锁。 例如，事务A可以在间隙上保持共享间隙锁定（gap X-lock），而事务B在同一间隙上保持专用间隙锁定（gap X-lock）。 允许间隙锁定冲突的原因是，如果从索引中清除记录，则必须合并由不同事务记录在记录上的间隙锁定。

InnoDB中的间隔锁是“纯粹的抑制”，这意味着它们只阻止其它事务插入记录到间隙。 它们不会阻止不同的事务在同一间隙上加间隙锁。 因此，间隙X锁具有与间隙S锁相同的效果。

间隙锁定可以被明确禁用。 如果将事务隔离级别更改为READ COMMITTED或启用`innodb_locks_unsafe_for_binlog`系统变量（现在已被弃用），则会发生这种情况。 在这种情况下，针对搜索和索引扫描禁用间隙锁定，仅用于外键约束检查和重复键检查。

使用`READ COMMITTED`隔离级别或启用`innodb_locks_unsafe_for_binlog`会有其它的效果。 在MySQL已经计算了WHERE条件之后，非匹配行的记录锁将被释放（InnoDB层会返回不匹配的加锁行，MySQL Server层面进行过滤和锁释放）。 对于UPDATE语句，InnoDB执行“半一致”读取，以便将最新的提交版本返回给MySQL，以便MySQL可以确定该行是否与UPDATE的WHERE条件匹配。

#### Next-Key Locks

`next-key`锁是索引记录上的记录锁和索引记录之前的间隙上的间隙锁的组合。

InnoDB以这样的方式执行行级锁定，即当它搜索或扫描表索引时，它会在遇到的索引记录上设置共享或排他锁。 因此，行级锁实际上是索引记录锁。 索引记录上的`next-key`锁也会影响索引记录之前的“间隙”。 也就是说，`next-key`是索引记录锁加上索引记录之前的间隙上的间隙锁。 如果一个会话在索引中的记录R上具有共享或排他锁定，则另一个会话不能在索引顺序中紧跟在R之前的间隙中插入新的索引记录。

假设索引包含值10,11,13和20。该索引可能的`next-key`锁定会覆盖以下间隔，其中圆括号表示排除间隔端点，而方括号表示包含端点：

```sql
(negative infinity, 10]
(10, 11]
(11, 13]
(13, 20]
(20, positive infinity)
```

对于最后一个时间间隔，`next-key`锁定索引中最大值之上的间隙，并且“supremum”伪记录的值高于索引中实际的任何值。 上限不是一个真正的索引记录，所以实际上，这个下一个键锁只锁定最大索引值之后的间隙。

InnoDB默认的事务隔离级别是`REPEATABLE READ` 。 在这种情况下， InnoDB 在搜索和索引遍历中会使用`next-key`锁来避免幻象读。

Transaction data for a next-key lock appears similar to the following in SHOW ENGINE INNODB STATUS and InnoDB monitor output:

```sql
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t` 
trx id 10080 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
 ```
 
#### 插入意向锁

插入意图锁是在行插入之前由INSERT操作设置的一种间隙锁定。 该锁以这样的方式发出插入意图，使得插入到相同的索引间隙中的多个事务不需要彼此等待，如果它们不插入在间隙内的相同位置。 假设存在值为4和7的索引记录。尝试分别插入值为5和6的单独事务分别在插入意图锁之前锁定4到7之间的间隙，然后才能获得插入行上的排他锁， 但不要阻止彼此，因为这些行是不冲突的。

以下示例演示了在获取插入的记录上的排他锁之前采取插入意图锁定的事务。 这个例子涉及两个客户A和B.

客户端A创建一个包含两个索引记录（90和102）的表，然后启动一个事务，该事务在ID大于100的索引记录上放置独占锁。排它锁包括记录102之前的间隙锁：

```sql
mysql> CREATE TABLE child (id int(11) NOT NULL, PRIMARY KEY(id)) ENGINE=InnoDB;
mysql> INSERT INTO child (id) values (90),(102);

mysql> START TRANSACTION;
mysql> SELECT * FROM child WHERE id > 100 FOR UPDATE;
+-----+
| id  |
+-----+
| 102 |
+-----+
```

客户端B开启一个事务，插入一条记录到该间隙中。该事务在等待获取排它锁的时候就会使用`插入意向锁`。

```sql
mysql> START TRANSACTION;
mysql> INSERT INTO child (id) VALUES (101);
```

Transaction data for an insert intention lock appears similar to the following in SHOW ENGINE INNODB STATUS and InnoDB monitor output:

```sql
RECORD LOCKS space id 31 page no 3 n bits 72 index `PRIMARY` of table `test`.`child`
trx id 8731 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 80000066; asc    f;;
 1: len 6; hex 000000002215; asc     " ;;
 2: len 7; hex 9000000172011c; asc     r  ;;...
```

#### AUTO-INC Locks

AUTO-INC锁是通过使用AUTO_INCREMENT列插入到表中的事务采取的特殊表级锁。 在最简单的情况下，如果一个事务将值插入到表中，任何其他事务必须等待前一个事务的记录插入到表中，以便第一个事务插入的行接收连续的主键值。

`innodb_autoinc_lock_mode`配置选项控制用于自动增量锁定的算法。 它允许您选择如何在插入操作的自动增量值的可预测序列和最大并发性之间进行折衷。

详情参考:[innodb-auto-increment-handling](https://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html)

#### 空间索引的谓词锁定(Predicate Locks for Spatial Indexes)

InnoDB支持对包含空间列的列进行SPATIAL索引（请参见第11.5.3.5节“优化空间分析”）。

要处理涉及SPATIAL索引的操作的锁定，下一键锁定不能很好地支持REPEATABLE READ或SERIALIZABLE事务隔离级别。 在多维数据中没有绝对排序概念，所以不清楚“下一个”键是哪个。

为了支持具有SPATIAL索引的表的隔离级别，InnoDB使用谓词锁。 SPATIAL索引包含最小边界矩形（MBR）值，因此InnoDB通过在查询的MBR值上设置谓词锁来强制执行索引上的一致性读取。 其他事务无法插入或修改与查询条件匹配的行。

### 参考文章

[https://dev.mysql.com/doc/refman/5.7/en/innodb-locking-reads.html](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking-reads.html)

[https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html)

[深入理解SELECT ... LOCK IN SHARE MODE和SELECT ... FOR UPDATE](http://blog.csdn.net/cug_jiang126com/article/details/50544728)





















