---
layout: post
comments: true
title: java线程池-RejectedExecutionHandler
date: 2017-02-26 14:57:00
tags:
- 面试
categories:
- java
---

### 前言

Java线程池的工作原理在面试时有很大的概率被问到，大部分人也能回答正确。但有一个小细节可能会忽略，那就是线程池的拒绝执行策略。今天翻看JDK的源码详细整理一下这个问题的答案。

<!-- more -->

### RejectedExecutionHandler 简介

JDK的说明是：

> A handler for tasks that cannot be executed by a ThreadPoolExecutor

意思就是当线程池不能执行该任务是就交由拒绝执行处理器来处理该任务。

JDK 代码：

```java
public interface RejectedExecutionHandler {
    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
```

### RejectedExecutionHandler 在线程池中的使用

我们可以通过下面代码所示的线程池的构造函数来指定线程池的拒绝执行处理器：

```java
 public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              RejectedExecutionHandler handler) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
    Executors.defaultThreadFactory(), handler);
}

public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
}    
```

### 线程池默认的拒绝执行策略

```java
/**
* The default rejected execution handler
*/
private static final RejectedExecutionHandler defaultHandler = new AbortPolicy();
```

从线程池`ThreadPoolExecutor`的源码发现它使用的默认策略是：AbortPolicy。该拒绝执行处理器的逻辑如下：

```java
/**
* A handler for rejected tasks that throws a
* {@code RejectedExecutionException}.
*/
public static class AbortPolicy implements RejectedExecutionHandler {
   /**
    * Creates an {@code AbortPolicy}.
    */
   public AbortPolicy() { }

   /**
    * Always throws RejectedExecutionException.
    *
    * @param r the runnable task requested to be executed
    * @param e the executor attempting to execute this task
    * @throws RejectedExecutionException always
    */
   public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
       throw new RejectedExecutionException("Task " + r.toString() +
                                            " rejected from " +
                                            e.toString());
   }
}
```

我们可以看出：该拒绝执行处理器会直接抛一个异常出来。并没有做其它复杂的处理。

### 线程池拒绝执行策略分类

处理上面说的`AbortPolicy`，线程池还有其它的几个分类。如下所示：

#### CallerRunsPolicy

该拒绝执行处理器的逻辑是：用调用线程本身来执行此次提交的任务。

JDK代码如下：

```java
/* Predefined RejectedExecutionHandlers */

/**
* A handler for rejected tasks that runs the rejected task
* directly in the calling thread of the {@code execute} method,
* unless the executor has been shut down, in which case the task
* is discarded.
*/
public static class CallerRunsPolicy implements RejectedExecutionHandler {
   /**
    * Creates a {@code CallerRunsPolicy}.
    */
   public CallerRunsPolicy() { }

   /**
    * Executes task r in the caller's thread, unless the executor
    * has been shut down, in which case the task is discarded.
    *
    * @param r the runnable task requested to be executed
    * @param e the executor attempting to execute this task
    */
   public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
       if (!e.isShutdown()) {
           r.run();
       }
   }
}
```

#### DiscardPolicy

该拒绝执行处理器的逻辑是：静默的丢掉拒绝执行的任务。

JDK 代码如下：

```java
/**
* A handler for rejected tasks that silently discards the
* rejected task.
*/
public static class DiscardPolicy implements RejectedExecutionHandler {
   /**
    * Creates a {@code DiscardPolicy}.
    */
   public DiscardPolicy() { }

   /**
    * Does nothing, which has the effect of discarding task r.
    *
    * @param r the runnable task requested to be executed
    * @param e the executor attempting to execute this task
    */
   public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
   }
}
```

#### DiscardOldestPolicy

该拒绝执行处理器的逻辑是：丢掉最早提交的任务（马上要被执行的任务），然后重新提交当前要处理的任务。

JDK 代码如下：

```java
/**
* A handler for rejected tasks that discards the oldest unhandled
* request and then retries {@code execute}, unless the executor
* is shut down, in which case the task is discarded.
*/
public static class DiscardOldestPolicy implements RejectedExecutionHandler {
   /**
    * Creates a {@code DiscardOldestPolicy} for the given executor.
    */
   public DiscardOldestPolicy() { }

   /**
    * Obtains and ignores the next task that the executor
    * would otherwise execute, if one is immediately available,
    * and then retries execution of task r, unless the executor
    * is shut down, in which case task r is instead discarded.
    *
    * @param r the runnable task requested to be executed
    * @param e the executor attempting to execute this task
    */
   public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
       if (!e.isShutdown()) {
           e.getQueue().poll();
           e.execute(r);
       }
   }
}
```

### 总结

以上就是线程池对处理不过来的任务的拒绝执行策略。如果默认提供的策略不符合你的业务逻辑那就在构造线程池时指定你自己的拒绝执行处理器。




