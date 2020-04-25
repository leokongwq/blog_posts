---
layout: post
comments: true
title: 分布式锁的几种实现方式
date: 2018-01-19 14:02:59
tags:
- 分布式
categories:
---

### 背景

进程内的对共享资源的互斥访问可以通过各个语言提供的Lock来实现。在分布式环境下对共享资源的访问就需要分布式锁的支持。分布式是锁的本质是都是找一个共同可以访问的资源，可以是分布式文件系统中的文件，某个固定结点的一块内存，等等。共享资源所在机器的进程来协调分布式的加锁和解锁请求。分布式的锁比进程内的锁需要考虑的问题还要多。下面就总结一下常用的分布式锁解决方案。

但是：能不用分布式锁就不要用了。性能，可扩展性，容错都很难处理。

<!-- more -->

### 分布式锁需要实现的功能

- 互斥性 统一时刻只能被集群中某个应用的单个线程访问

- 可重入

- 阻塞式

- 高可用

- 高性能

### 数据库实现

#### 基于table的实现

要实现分布式锁，最简单的方式可能就是直接创建一张锁记录表，然后通过操作该表中的数据来实现了。

当我们要锁住某个方法或资源时，我们就在该表中增加一条带有唯一索引的记录，想要释放锁的时候就删除这条记录。

创建这样一张数据库表：

```sql
CREATE TABLE `distLock` (
  `id` int NOT NULL AUTO_INCREMENT COMMENT '主键',
  `lock_name` varchar(64) NOT NULL DEFAULT '' COMMENT '锁名称',
  `desc` varchar(1024) NOT NULL DEFAULT '备注信息',
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '保存数据时间，自动生成',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uniq_lock_name` (`lock_name`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='分布式锁';
```

加锁：insert一条记录
解锁：删除该记录

存在的问题：

1. 严重依赖数据库的可用性，存在单点风险
2. 没有失效时间，一旦解锁操作失败，就会导致锁记录一直在数据库中，其他线程无法再获得到锁。
3. 需要客户端实现自旋，阻塞，超时控制等。
4. 锁是非重入的，同一个线程在没有释放锁之前无法再次获得该锁。因为数据中锁
5. 性能依赖数据的写入性能力，扩展性差。记录已经存在了。

优化方式：

1. 数据库双写。消除单点
2. 记录锁获取时间和失效时间。定时任务删除释放失败的锁。
3. 客户端封装自旋逻辑，锁超时和进程内的线程互斥逻辑。
4. 记录获取锁的客户端唯一标志。insert前进行查询判断。
5. 搭建多个高可用集群，锁的名称进行哈希处理

#### 基于数据库排他锁

除了可以通过增删操作数据表中的记录以外，其实还可以借助数据中自带的锁来实现分布式的锁。

我们还用刚刚创建的那张数据库表。可以通过数据库的排他锁来实现分布式锁。 基于MySql的InnoDB引擎，可以使用以下方法来实现加锁操作：

```java
public boolean lock(){
    connection.setAutoCommit(false)
    while(true){
        try{
            result = select * from methodLock where method_name=xxx for update;
            if(result==null){
                return true;
            }
        }catch(Exception e){

        }
        sleep(100);
    }
    return false;
}
```

在查询语句后面增加for update，数据库会在查询过程中给数据库表增加排他锁（这里再多提一句，InnoDB引擎在加锁的时候，只有通过索引进行检索的时候才会使用行级锁，否则会使用表级锁。这里我们希望使用行级锁，就要给`lock_name`添加索引，值得注意的是，这个索引一定要创建成`唯一索引`。

我们可以认为获得排它锁的线程即可获得分布式锁，当获取到锁之后，可以执行方法的业务逻辑，执行完方法之后，再通过以下方法解锁：

```java
public void unlock(){
    connection.commit();
}
```

这种方法可以有效的解决上面提到的无法释放锁(事务有自己的超时时间)和阻塞锁（MySQL帮你阻塞）的问题。

阻塞锁？ `for update`语句会在执行成功后立即返回，在执行失败时一直处于阻塞状态，直到成功。
锁定之后服务宕机，无法释放？使用这种方式，服务宕机之后数据库会自己把锁释放掉。
但是还是无法直接解决数据库单点和可重入问题。

这里还可能存在另外一个问题，虽然我们对`lock_name`使用了唯一索引，并且显示使用`for update`来使用行级锁。但是，MySql会对查询进行优化，即便在条件中使用了索引字段，但是否使用索引来检索数据是由 MySQL 通过判断不同执行计划的代价来决定的，如果 MySQL 认为全表扫效率更高，比如对一些很小的表，它就不会使用索引，这种情况下 InnoDB 将使用表锁，而不是行锁。如果发生这种情况就悲剧了。。。

还有一个问题，就是我们要使用排他锁来进行分布式锁的lock，那么一个排他锁长时间不提交，就会占用数据库连接。一旦类似的连接变得多了，就可能把数据库连接池撑爆。

#### 总结

总结一下使用数据库来实现分布式锁的方式，这两种方式都是依赖数据库的一张表，一种是通过表中的记录的存在情况确定当前是否有锁存在，另外一种是通过数据库的排他锁来实现分布式锁。

数据库实现分布式锁的优点：

直接借助数据库，容易理解。

数据库实现分布式锁的缺点

会有各种各样的问题，在解决问题的过程中会使整个方案变得越来越复杂。

操作数据库需要一定的开销，性能问题需要考虑。

使用数据库的行级锁并不一定靠谱，尤其是当我们的锁表并不大的时候。

### 基于缓存实现分布式锁

redis 的实现可以参考下面的文章:

[基于redis分布式锁实现“秒杀”](http://blog.csdn.net/u010359884/article/details/50310387)
[jedisLock—redis分布式锁实现](https://www.cnblogs.com/0201zcr/p/5942748.html)
[《Redis官方文档》用Redis构建分布式锁](http://ifeve.com/redis-lock/)

### 基于zooKeeper实现分布式锁

[ZooKeeper的用法： 分布式锁](http://ifeve.com/zookeeper-lock/)
[zookeeper入门之curator框架--几种锁的操作](http://blog.csdn.net/sqh201030412/article/details/51456143)

### 参考

[分布式锁实现](http://www.hollischuang.com/archives/1716)



