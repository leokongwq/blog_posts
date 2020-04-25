---
layout: post
comments: true
title: java线程池工具类Executors
date: 2017-02-27 23:28:44
tags:
- 面试
categories:
- java
---

关于Executors这个类的用法可以参考jdk文档或：[https://my.oschina.net/20076678/blog/33392](https://my.oschina.net/20076678/blog/33392)。在本文我只想提出一个问题，就是`newCachedThreadPool()`这个方法。 该方法的逻辑如下：

```java
 public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
}
```

该方法创建的线程池的核心线程数等于零，最大线程数为`Integer.MAX_VALUE`。 这就说明了，如果我提交任务的速度很快，并且任务比较耗时，那么服务器就会因为线程数过多而负载飙升而挂断（会影响其他正常执行的业务线程，并且每个线程都会占用内存）。所以这点需要注意。在使用线程池的时候尽量使用固定数目的线程，和有界队列并配合自定义的拒绝执行处理器来处理任务。如果有可能，不同类型的任务可以使用不同的线程池。这样可以将任务分类和隔离，不至于因一类任务的问题导致整个服务不可用。






