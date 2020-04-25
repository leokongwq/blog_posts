---
layout: post
comments: true
title: mongodb开启查询日志
date: 2017-05-19 09:29:07
tags:
- mongodb
categories:
- 数据库
---

### 背景

公司的mongodb集群最近有好多慢查询报警，具体原因就是好多查询没有命中查询缓存，导致mongodb的缓存刷新线程来不及删除缓存中的数据；同样也导致大量查询走磁盘。为了分析究竟是哪些查询导致mongodb的缓存利用率不高，就需要统计分析mongodb的查询日志。

<!-- more -->

### 开启查询日志输出

默认mongodb不会输出所有的操作语句到日志中，google出的答案如下：

```
mongod --profile=1 --slowms=1 &
```

或

```
$ mongo
MongoDB shell version: 2.4.9
connecting to: test
> use myDb
switched to db myDb
> db.getProfilingLevel()
0
> db.setProfilingLevel(2)
{ "was" : 0, "slowms" : 1, "ok" : 1 }
> db.getProfilingLevel()
2
> db.system.profile.find().pretty()
```

`db.setProfilingLevel(2)`意味着mongodb会记录所有的操作到日志中。

在上述操作完成后就可以在log文件中查看操作日志记录了：

```shell
tail -f /var/log/mongodb/mongodb.log
//输出
Mon Mar  4 15:02:55 [conn1] query dendro.quads query: { graph: "u:http://example.org/people" } ntoreturn:0 ntoskip:0 nscanned:6 keyUpdates:0 locks(micros) r:73163 nreturned:6 reslen:9884 88ms
```

### db.setProfilingLevel详解

`db.setProfilingLevel`其实是设了mongodb内在工具`profiler`的日志记录级别的。参考:[https://docs.mongodb.com/manual/reference/glossary/#term-database-profiler](https://docs.mongodb.com/manual/reference/glossary/#term-database-profiler)

level有如下的取值：

0：关闭。不做分析
1：打开。只记录慢查询
2：打开。记录所有的操作

### 参考

[http://stackoverflow.com/questions/15204341/mongodb-logging-all-queries](http://stackoverflow.com/questions/15204341/mongodb-logging-all-queries)

[https://docs.mongodb.com/manual/reference/method/db.setProfilingLevel/](https://docs.mongodb.com/manual/reference/method/db.setProfilingLevel/)


