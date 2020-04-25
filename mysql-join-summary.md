---
layout: post
comments: true
title: MySQL中join操作总结
date: 2019-12-13 23:17:50
tags:
- MySQL
categories:
- 数据库
---

### 前言

在数据库中Join操作被称为连接。目的是从多个表中获取数据作为结果集返回给客户端。join 操作分为如下几种：

外连接：`left join`，`right join`
内连接：`inner join`
全连接：`full join`
笛卡尔积： `cross join` 

### 准备工作

新建2个表t1, t2，结构相同。

```sql
CREATE TABLE `t1` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `t2` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`)
) ENGINE=InnoDB;

INSERT INTO t1 VALUES (1,1,1), (2,2,2), (3, 3, 3);
INSERT INTO t2 VALUES (1,1,1), (2,2,2), (4, 4, 4);
```

<!-- more -->

### left join

`left join` 只需要记住左表的数据全部保留，右表满足连接条件的记录展示，不满足的条件的记录全是null。

```sql
mysql> SELECT * FROM t1 LEFT  JOIN t2 ON t1.id = t2.id;
+----+------+------+------+------+------+
| id | a    | b    | id   | a    | b    |
+----+------+------+------+------+------+
|  1 |    1 |    1 |    1 |    1 |    1 |
|  2 |    2 |    2 |    2 |    2 |    2 |
|  3 |    3 |    3 | NULL | NULL | NULL |
+----+------+------+------+------+------+
3 rows in set (0.00 sec)		
```

### right join

`right join` 只需要记住右表的数据全部保留，左表满足连接条件的记录展示，不满足的条件的记录全是null。

```sql
mysql> SELECT * FROM t1 right JOIN t2 ON t1.id = t2.id;
+------+------+------+----+------+------+
| id   | a    | b    | id | a    | b    |
+------+------+------+----+------+------+
|    1 |    1 |    1 |  1 |    1 |    1 |
|    2 |    2 |    2 |  2 |    2 |    2 |
| NULL | NULL | NULL |  4 |    4 |    4 |
+------+------+------+----+------+------+
3 rows in set (0.00 sec)
```

### inner join 

`inner join` 只保留左右两个表中满足连接条件的数据：

```sql
mysql> SELECT * FROM t1 INNER JOIN t2 ON t1.id = t2.id;
+----+------+------+----+------+------+
| id | a    | b    | id | a    | b    |
+----+------+------+----+------+------+
|  1 |    1 |    1 |  1 |    1 |    1 |
|  2 |    2 |    2 |  2 |    2 |    2 |
+----+------+------+----+------+------+
2 rows in set (0.00 sec)
```

### full join 

mysql 不支持`full join`，不过可以通过`union`来实现：

```sql
mysql> SELECT * FROM t1 LEFT  JOIN t2 ON t1.id = t2.id union SELECT * FROM t1 right JOIN t2 ON t1.id = t2.id;
+------+------+------+------+------+------+
| id   | a    | b    | id   | a    | b    |
+------+------+------+------+------+------+
|    1 |    1 |    1 |    1 |    1 |    1 |
|    2 |    2 |    2 |    2 |    2 |    2 |
|    3 |    3 |    3 | NULL | NULL | NULL |
| NULL | NULL | NULL |    4 |    4 |    4 |
+------+------+------+------+------+------+
4 rows in set (0.00 sec)
```

从结果可以看出来，左右表不满足连接条件的记录也保留了下来。

### cross join 

> 笛卡尔乘积是指在数学中，两个集合X和Y的笛卡尓积（Cartesian product），又称直积，表示为X × > Y，第一个对象是X的成员而第二个对象是Y的所有可能有序对的其中一个成员。

`cross join` 结果如下：

```sql
mysql> SELECT * FROM t1 CROSS JOIN t2;
+----+------+------+----+------+------+
| id | a    | b    | id | a    | b    |
+----+------+------+----+------+------+
|  1 |    1 |    1 |  1 |    1 |    1 |
|  2 |    2 |    2 |  1 |    1 |    1 |
|  3 |    3 |    3 |  1 |    1 |    1 |
|  1 |    1 |    1 |  2 |    2 |    2 |
|  2 |    2 |    2 |  2 |    2 |    2 |
|  3 |    3 |    3 |  2 |    2 |    2 |
|  1 |    1 |    1 |  4 |    4 |    4 |
|  2 |    2 |    2 |  4 |    4 |    4 |
|  3 |    3 |    3 |  4 |    4 |    4 |
+----+------+------+----+------+------+
9 rows in set (0.00 sec)
```

那我们再看一下不加连接条件的`inner join`的执行结果：

```sql
mysql> SELECT * FROM t1 INNER JOIN t2;
+----+------+------+----+------+------+
| id | a    | b    | id | a    | b    |
+----+------+------+----+------+------+
|  1 |    1 |    1 |  1 |    1 |    1 |
|  2 |    2 |    2 |  1 |    1 |    1 |
|  3 |    3 |    3 |  1 |    1 |    1 |
|  1 |    1 |    1 |  2 |    2 |    2 |
|  2 |    2 |    2 |  2 |    2 |    2 |
|  3 |    3 |    3 |  2 |    2 |    2 |
|  1 |    1 |    1 |  4 |    4 |    4 |
|  2 |    2 |    2 |  4 |    4 |    4 |
|  3 |    3 |    3 |  4 |    4 |    4 |
+----+------+------+----+------+------+
9 rows in set (0.00 sec)
```

可以看出不加连接条件的`inner join` 和 `cross join` 的结果相同。

在MySQL中 `join`，`cross join`, `inner join`


### Join语句是怎么执行的？

MySQL中join语句的执行有3中方式，分别如下所示：

#### Index Nested-Loop Join

```sql
select * from t1 straight_join t2 on (t1.a = t2.a)
```

执行流程：

1. 从表 t1 中读入一行数据R；
2. 从数据行 R 中，取出 a 字段到表 t2 里去查找；
3. 取出表 t2 中满足条件的行，和 R 组成一行，作为结果集的一部分；
4. 重复执行步骤 1 和 3 ，知道 t1 的末尾循环结束。

在形式上，这个过程就跟我们写成程序时的嵌套查询类似，并且**可以用上被驱动表的索引**，所以我们成为`Index Nested-Loop Join`，简称`NLJ`。

如果不使用join:

1. 执行select * from t1，查出t1的所有数据，100行
2. 循环遍历这100行：
    - 从每一行R取出字段a的值$R.a;
    - 执行select * from t2 where a=$R.a;
    - 把返回结果和R构成结果集的一行。

可以看到，在这个查询过程，也是扫描了200行，但是总共执行了101条语句，比直接join多了100次交互。除此之外，客户端还要自己拼接SQL语句。

显然，这么做还不如直接join好。

假设被驱动表的行数是M。每次查找被驱动表中一行数据时，要先搜索索引a，再搜索主键索引树。每次搜索一棵树近似复杂度是以 2 为底的M的对数，记为log2M，所以再被驱动表上查一行的时间复杂度是 2*log2M。

结论：
1. 使用join语句，性能比强行拆成多个单表执行sql语句的性能要好。
2. 如果使用join语句的话，需要让小表做驱动表

但是，需要注意，这个结论的前提是**可以使用被驱动表的索引**

#### Simple Nested-loop join

```sql
select * from t1 straight_join t2 on (t1.a = t2.b)
```

t1是驱动表，t2是被驱动表。

由于表t2的字段b上没有索引，因此再用上面提到的执行流程时，每次到t2去匹配的时候，就要做一次**全表扫描**

这时，每次t2去匹配的时候，就要做一次全表扫描。这个sql请求就要扫描表t2多达100次，总共扫描100*1000=10万行。

当然MYSQL没有使用这个方法，而是下面的 **Block Nested-Loop Join**

#### Block Nested-Loop Join

这时候，被驱动表上没有可用的索引，算法的流程是这样的：

1. 把表t1的数据读入线程内存`join_buffer`中，由于我们这个语句中写的是`select *`，因此是把整个t1放入了内存；
2. 扫描表t2，把表t2的每一行取出来，跟`join_buffer`中的数据做对比，满足join条件的(也就是on指定的条件)，作为结果集的一部分返回。

可以看到，在这个过程中，对表t1和t2都做了一次全表扫描，因此总的扫描行数是1100.由于join_buffer是以无序数组的方式组织的，因此对表t2中的每一行，都要做100次判断，总共需要在内存中做的判断次数是：100*1000=10万行。

前面我们说过，如果使用`Simple Nested-Loop Join`算法进行查询，扫描行数也是10万行。因此，从时间复杂度上来说，这两个算法是一样的。但是Block Nested-Loop Join算法的这10万次判断是内存操作，速度上会快很多，性能也更好

接下来，我们来看一下，在这种情况下，应该选择哪个表做驱动表。

假设小表的行数据是N，大表的行数是M，那么在这个算法里：

两个表都做一次全表扫描，所以总的扫描行数是M+N；
内存中的判断次数是M*N
可以看到，调换这两个算式中的M和N没有区别，因此这时候选择大表还是小表做驱动表，执行耗时是一样的。

然后，你可能马上就会问了，这个例子里t1才100行，要是表t1是一个大表，`join_buffer`放不下怎么办呢？

`join_buffer` 的大小是由`join_buffer_size`设定的，默认值是256k。如果放不下t1的所有数据的话，策略很简单，就是分段放。把`join_buffer_size`改成1200，再执行：

**过程就变成了：**

1. 扫描表t1，顺序读取数据行放入`join_buffer`中，放完第88行join_buffer满了，继续第二步
2. 扫描表t2，把t2中的每一行取出来，跟`join_buffer`中的数据做对比，满足join条件的，作为结果集的一部分返回；
清空`join_buffer`
继续扫描表t1，顺序读取最后的12行数据放入`join_buffer`中，继续执行第2步
这个流程才体现出了这个算法名字中`Block`的由来，表示`分块去join`

可以看到，这时候由于表t1被分成两次放入join_buffer中，导致表t2会被扫描两次，但是判断等值条件的次数还是不变的，（88+12）*1000=10万次。

假设，驱动表的数据行数是N，需要分K段才能完成算法流程，被驱动表的数据行数是M。

注意，这里的 K 不是常数，N 越大 K 就会越大，因此把 K 表示为λ*N，显然λ的取值范围是 (0,1)。

所以，在这个算法的执行过程中：

1. 扫描行数是 N+λ*N*M；
2. 内存判断 N*M 次。

显然，内存判断次数是不受选择哪个表作为驱动表影响的。而考虑到扫描行数，在 M 和 N 大小确定的情况下，N 小一些，整个算式的结果会更小。

当然，你会发现，在 N+λ*N*M 这个式子里，λ才是影响扫描行数的关键因素，这个值越小越好。

刚刚我们说了 N 越大，分段数 K 越大。那么，N 固定的时候，什么参数会影响 K 的大小呢？（也就是λ的大小）答案是 join_buffer_size。join_buffer_size越大，一次可以放入的行越多，分成的段数也就越少，对被驱动表的全表扫描次数就越少。

这就是为什么，你可能会看到一些建议告诉你，如果你的 join 语句很慢，就把 join_buffer_size 改大。

### on 字句和 where 的顺序问题

直接说结论：从上面的分析可以知道，on字句的条件用来过滤被驱动表中的数据。where字句指定的条件用来过滤最终join的结果集。

### 参考资料

[https://dev.mysql.com/worklog/task/?id=1604](https://dev.mysql.com/worklog/task/?id=1604)

[MySQL45讲 - 我订阅的课程](https://time.geekbang.org/column/article/83183)

[https://dev.mysql.com/doc/refman/5.7/en/join.html](https://dev.mysql.com/doc/refman/5.7/en/join.html)





