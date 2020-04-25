---
layout: post
comments: true
title: vertx-context
date: 2017-11-17 20:45:24
tags:
- vert.x
categories:
---

### Context 对象

当 Vert.x 传递一个事件给处理器或者调用 Verticle 的 start 或 stop 方法时，它会关联一个`Context`对象来执行。通常来说这个 Context 会是一个 `Event Loop Context`，它绑定到了一个特定的 `Event Loop` 线程上。所以在该 `Context` 上执行的操作总是在同一个 `Event Loop` 线程中。对于运行内联的阻塞代码的 `Worker Verticle` 来说，会关联一个 `Worker Context`，并且所有的操作运都会运行在 Worker 线程池的线程上。

> 译者注：每个 Verticle 在部署的时候都会被分配一个 Context（根据配置不同，可以是Event Loop Context 或者 Worker Context），之后此 Verticle 上所有的普通代码都会在此 Context 上执行（即对应的 Event Loop 或Worker 线程）。一个 Context 对应一个 Event Loop 线程（或 Worker 线程），但一个 Event Loop 可能对应多个 Context。

<!-- more -->

您可以通过`getOrCreateContext`方法获取`Context`实例：

```java
Context context = vertx.getOrCreateContext();
```

若已经有一个 Context 和当前线程关联，那么它直接重用这个 Context 对象，如果没有则创建一个新的。您可以检查获取的 Context 的类型：

```java
Context context = vertx.getOrCreateContext();
if (context.isEventLoopContext()) {
  System.out.println("Context attached to Event Loop");
} else if (context.isWorkerContext()) {
  System.out.println("Context attached to Worker Thread");
} else if (context.isMultiThreadedWorkerContext()) {
  System.out.println("Context attached to Worker Thread - multi threaded worker");
} else if (! Context.isOnVertxThread()) {
  System.out.println("Context not attached to a thread managed by vert.x");
}
```

当您获取了这个 Context 对象，您就可以在 Context 中异步执行代码了。换句话说，您提交的任务将会在同一个 Context 中运行：

```java
vertx.getOrCreateContext().runOnContext(v -> {
  System.out.println("This will be executed asynchronously in the same context");
});
```

当在同一个 Context 中运行了多个处理函数时，可能需要在它们之间共享数据。 Context 对象提供了存储和读取共享数据的方法。举例来说，它允许您将数据传递到 runOnContext 方法运行的某些操作中：

```java
final Context context = vertx.getOrCreateContext();
context.put("data", "hello");
context.runOnContext((v) -> {
  String hello = context.get("data");
});
```

您还可以通过 config 方法访问 Verticle 的配置信息。查看 向 Verticle 传入配置 章节了解更多配置信息




