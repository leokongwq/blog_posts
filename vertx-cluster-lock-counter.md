---
layout: post
comments: true
title: vertx-集群范围锁-计数器
date: 2017-11-18 14:06:00
tags:
- vert.x
categories:
---

集群范围锁（Lock）允许您在集群中获取独占锁 —— 当您想要在任何时间只在集群一个节点上执行某些操作或访问资源时，这很有用。

集群范围锁具有异步API，它和大多数等待锁释放的阻塞调用线程的API锁不相同。

可使用`getLock`方法获取锁。

它不会阻塞，但当锁可用时，将 Lock 的实例传入处理器并调用它，表示您现在拥有该锁。

若您拥有的锁没有其他调用者，集群上的任何地方都可以获得该锁。

当您用完锁后，您可以调用 release 方法来释放它，以便另一个调用者可获得它。

<!-- more -->

```java
sd.getLock("mylock", res -> {
  if (res.succeeded()) {
    // Got the lock!
    // 获得锁
    Lock lock = res.result();

    // 5 seconds later we release the lock so someone else can get it
    // 5秒后我们释放该锁其他人可以得到它
    vertx.setTimer(5000, tid -> lock.release());

  } else {
    // Something went wrong
    // 出了些问题
  }
});
```

您可以为锁设置一个超时，若在超时时间期间无法获取锁，将会进入失败状态，处理器会去处理对应的异常：

```java
sd.getLockWithTimeout("mylock", 10000, res -> {
  if (res.succeeded()) {
    // Got the lock!
    // 获得锁
    Lock lock = res.result();

  } else {
    // Failed to get lock
    // 锁获取失败
  }
});
```

### 集群范围计数器

很多时候我们需要在集群范围内维护一个原子计数器。

您可以用 Counter 来做到这一点。

您可以通过 getCounter 方法获取一个实例：

```java
sd.getCounter("mycounter", res -> {
  if (res.succeeded()) {
    Counter counter = res.result();
  } else {
    // Something went wrong!
    // 出了些问题
  }
});
```

一旦您有了一个实例，您可以获取当前的计数，以原子方式递增、递减，并使用各种方法添加一个值。

有更多信息，请参阅 [API 文档](http://vertx.io/docs/apidocs/io/vertx/core/shareddata/Counter.html)。







