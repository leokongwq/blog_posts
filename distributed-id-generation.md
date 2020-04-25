---
layout: post
comments: true
title: 5分钟了解分布式ID生成方案
date: 2016-11-02 10:19:54
tags:
- java
categories:
- 架构
---

> 中大型的互联网应用中,分布式ID生成是基础的必须存在的基础应用. 然而想要成功实施还是很困难的.

<!-- more -->

### UUID

UUID是指在一台机器上生成的数字，它保证对在同一时空中的所有机器都是唯一的。通常平台会提供生成的API。按照开放软件基金会(OSF)制定的标准计算，用到了以太网卡地址、纳秒级时间、芯片ID码和许多可能的数字.
 
UUID由以下几部分的组合：
- 当前日期和时间，UUID的第一个部分与时间有关，如果你在生成一个UUID之后，过几秒又生成一个UUID，则第一个部分不同，其余相同。
- 时钟序列。
- 全局唯一的IEEE机器识别号，如果有网卡，从网卡MAC地址获得，没有网卡以其他方式获得。

有 4 种不同的基本 UUID 类型：基于时间的 UUID、DCE 安全 UUID、基于名称的 UUID 和随机生成的 UUID。

Java使用UUID非常方便

    UUID uuid = UUID.randomUUID();
    
格式是`35b3dda6-f481-410a-b393-9adc91703c68`
 
**优点**：使用方便，无状态
**缺点**：太长，32位，不适合做数据库主键索引


### SnowFlake
Twitter的SnowFlake是一个非常优秀的id生成方案，实现也非常简单，8Byte 是一个Long，8Byte等于64bit，核心代码就是毫秒级时间41位+机器ID 10位+毫秒内序列12位。当然可以调整机器位数和毫秒内序列位数比例，实际上从这个项目衍生出来很多实现方式。该项目地址为：https://github.com/twitter/snowflake是用Scala实现的。

优点：

- 比UUID短，一般9-17位左右；
- 性能非常出色，每秒几十万；

如果能用snowFlaker的方式，绝对不用UUID。

### Flickr

{% asset_img Flickr-mysql.jpg %}

采用mysql自增长id实现，实现也非常简单，至少使用两个节点容灾，

设置节点一

    auto_increment_increment=2;
    auto_increment_offset=1;

设置节点二

    auto_increment_increment=2;
    auto_increment_offset=2;

需要注意的是这两个配置最好放在配置文件中，否则mysql服务重启将丢失设置；如果直接通过set来设置，已经建立的连接不会改变变量的值，特别是使用连接池的时候。

```sql 分表在两个库里面建表
CREATE TABLE `tickets1` (
  `id` bigint(20) unsigned NOT NULL auto_increment,
  `stub` char(1) NOT NULL default '',
  PRIMARY KEY  (`id`),
  UNIQUE KEY `stub` (`stub`)
) ENGINE=MyISAM
```

注意建的是MyISAM表，MyISAM是表级锁，能保证所有replace的原子性。

不断的通过replace促使id自增，这样表中只有一条记录。
    
    REPLACE INTO tickets1 (stub) VALUES ('a')REPLACE INTO tickets1 (stub) VALUES ('a')

在同一个连接内，通过last insert id获取自增的id值。

对MyISAM性能影响比较大的参数是key_buffer_size，可以适当调整。

**优点：**

- 对于数据量不是特别大的应用，长度最小，从零开始。
- 如果已经有应用是基于sequence或者mysql的自增id，采用此方案非常容易迁移，兼容性好。
- 这种方案是没有绝对顺序的，只能有一个近似顺序。有可能一个实例跑的更快。
 
### 总结

1. Id首先要保证的是全局唯一，可以不必保证精确的顺序性，保证顺序意义不大，试想，就算这里保证了，由于现在的系统都是分布式的，先拿到的也不一定会先执行完。

2. 只要有大概的顺序对于调试、简单识别而言已经非常不错。

3. 如果在高并发，大数据量的情况下，建议采用SnowFake方案，性能非常突出，而对于需要兼容老业务的应用，特别是拿到id要进行拼装以表示某种意义，并发又不那么高，可以使用flickr的方案，两台物理机也能轻松到达1.8万tps，响应时间在毫秒级。

4. Id的长度也是一个非常重要的指标，如果在并发量比较大的情况下，可以计算一下，传输的数据量非常可观。
 
### SnowFlake Java版实现

```
package com.meililinc.mls.datastruct;

public class SnowFlakeIdWorker {

    private final long workerId;
    private final static long twepoch = 1390838400000L;
    private Long sequence = 0L;
    private final static long workerIdBits = 4L;
    public final static long maxWorkerId = -1L ^ -1L << workerIdBits;
    private final static long sequenceBits = 10L;
    private final static long workerIdShift = sequenceBits;
    private final static long timestampLeftShift = sequenceBits + workerIdBits;
    public final static long sequenceMask = -1L ^ -1L << sequenceBits;
    private long lastTimestamp = -1L;

    public SnowFlakeIdWorker(final long workerId) {
        super();
        if (workerId > this.maxWorkerId || workerId < 0) {
            throw new IllegalArgumentException(String.format(
                    "worker Id can't be greater than %d or less than 0",
                    this.maxWorkerId));
        }
        this.workerId = workerId;
    }

    public synchronized long nextId() {
        long timestamp = this.timeGen();
        if (this.lastTimestamp == timestamp) {
            this.sequence = (this.sequence + 1) & this.sequenceMask;
            if (this.sequence == 0) {
                System.out.println("###########" + sequenceMask);
                timestamp = this.tilNextMillis(this.lastTimestamp);
            }
        } else {
            this.sequence = 0L;
        }
        if (timestamp < this.lastTimestamp) {
            try {
                throw new Exception(
                        String.format(
                                "Clock moved backwards.  Refusing to generate id for %d milliseconds",
                                this.lastTimestamp - timestamp));
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        this.lastTimestamp = timestamp;
        long nextId = ((timestamp - twepoch << timestampLeftShift))
                | (this.workerId << this.workerIdShift) | (this.sequence);
        return nextId;
    }

    private long tilNextMillis(final long lastTimestamp) {
        long timestamp = this.timeGen();
        while (timestamp <= lastTimestamp) {
            timestamp = this.timeGen();
        }
        return timestamp;
    }

    private long timeGen() {
        return System.currentTimeMillis();
    }
}
``` 
 
 
 