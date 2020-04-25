---
layout: post
comments: true
title: vertx-执行周期性/延迟性操作
date: 2017-11-17 20:48:56
tags:
- vert.x
categories:
---

在 Vert.x 中，想要延迟之后执行或定期执行操作很常见。

在 Standard Verticle 中您不能直接让线程休眠以引入延迟，因为它会阻塞 Event Loop 线程。取而代之是使用 Vert.x 定时器。定时器可以是一次性或周期性的，两者我们都会讨论到。

<!-- more -->

### 一次性计时器

一次性计时器会在一定延迟后调用一个 Event Handler，以毫秒为单位计时。

您可以通过 setTimer 方法传递延迟时间和一个处理器来设置计时器的触发。

```java
long timerID = vertx.setTimer(1000, id -> {
  System.out.println("And one second later this is printed");
});

System.out.println("First this is printed");
```

返回值是一个唯一的计时器id，该id可用于之后取消该计时器，这个计时器id会传入给处理器。

### 周期性计时器

您同样可以使用 setPeriodic 方法设置一个周期性触发的计时器。第一次触发之前同样会有一段设置的延时时间。

setPeriodic 方法的返回值也是一个唯一的计时器id，若之后该计时器需要取消则使用该id。传给处理器的参数也是这个唯一的计时器id。

请记住这个计时器将会定期触发。如果您的定时任务会花费大量的时间，则您的计时器事件可能会连续执行甚至发生更坏的情况：重叠。这种情况，您应考虑使用 setTimer 方法，当任务执行完成时设置下一个计时器。

```java
long timerID = vertx.setPeriodic(1000, id -> {
  System.out.println("And every second this is printed");
});

System.out.println("First this is printed");
```

### 取消计时器

指定一个计时器id并调用 cancelTimer 方法来取消一个周期性计时器。如：

```java
vertx.cancelTimer(timerID);
```

### Verticle 中自动清除定时器

如果您在 Verticle 中创建了计时器，当这个 Verticle 被撤销时这个计时器会被自动关闭。

### Verticle Worker Pool

Verticle 使用 Vert.x 中的 `Worker Pool` 来执行阻塞式行为，例如`executeBlocking` 或 `Worker Verticle`。

可以在部署配置项中指定不同的Worker 线程池：

```java
vertx.deployVerticle("the-verticle", new DeploymentOptions().setWorkerPoolName("the-specific-pool"));
```






