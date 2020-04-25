---
layout: post
comments: true
title: vert.x入门
date: 2017-11-17 16:10:53
tags:
- vert.x
categories:
---

### vert.x 是什么？

[Vert.x](http://vertx.io/)是一个基于JVM、轻量级、高性能的应用平台。基于它可开发各种移动，Web和企业应用程序。一个主要特点是可使用多种语言编写应用，如Java, JavaScript, CoffeeScript, Ruby, Python 或 Groovy等等，它的简单actor-like机制能帮助脱离直接基于多线程编程。它是基于Netty和Java 7的NIO2的编写的。

### vert.x hello world

maven依赖：
```xml
<dependency>
  <groupId>io.vertx</groupId>
  <artifactId>vertx-core</artifactId>
  <version>${vertx.version}</version>
</dependency>
```

<!-- more -->

java代码：

```java
public class MyFirstVerticle extends AbstractVerticle {
    @Override
    public void start() throws Exception {
        vertx.createHttpServer().requestHandler(req -> {
           req.response().putHeader("content-type", "text/plain;charset=UTF-8").end("hello vert.x");
        }).listen(9090);
    }
}

public class Main {
    public static void main(String[] args) {
        Vertx vertx = Vertx.vertx();
        vertx.deployVerticle(MyFirstVerticle.class.getName());
    }
}
```

上面的代码创建了一个监听在在端口9090的http服务器，每个请求到来时返回`hello vert.x`。

入门学习时，上面的代码有一个疑问：为什么`vertx.deployVerticle()`方法传入的参数是一个字符串，而不是一个`Verticle`实例？

答案就在于：vert.x会使用`VerticleFactory.createVerticle`方法根据名称来创建`Verticle`的实例。

### 理解Vert.x机制？

答：Vert.x其实就是建立了一个Verticle内部的线程安全机制，让用户可以排除多线程并发冲突的干扰，专注于业务逻辑上的实现，用了Vert.x，您就不用操心多线程和并发的问题了。Verticle内部代码，除非声明Verticle是Worker Verticle，否则Verticle内部环境全部都是线程安全的，不会出现多个线程同时访问同一个Verticle内部代码的情况。

### Verticle对象和处理器（Handler）是什么关系？Vert.x如何保证Verticle内部线程安全?

Verticle对象往往包含有一个或者多个处理器（Handler）。Vert.x保证同一个普通 Verticle（也就是EventLoop Verticle，非Worker Verticle）内部的所有处理器（Handler）都只会由同一个EventLoop线程调用，由此保证Verticle内部的线程安全。

Vert.x的Handler内部是atomic/原子操作，Verticle内部是thread safe/线程安全的，Verticle之间传递的数据是immutable/不可改变的。

一个vert.x实例/进程内有多个Eventloop和Worker线程，每个线程会部署多个Verticle对象并对应执行Verticle内的Handler，每个Verticle内有多个Handler，普通Verticle会跟Eventloop绑定，而Worker Verticle对象则会被Worker线程所共享，会依次顺序访问，但不会并发同时访问，如果声明为Multiple Threaded Worker Verticle则没有此限制，需要开发者手工处理并发冲突，我们并不推荐这类操作。

### 参考

[Vert.x入门教程](http://www.jdon.com/concurrent/vertx.html)
[https://www.techempower.com/benchmarks/#section=data-r12&hw=peak&test=json](https://www.techempower.com/benchmarks/#section=data-r12&hw=peak&test=json)
[https://vertxchina.github.io](https://vertxchina.github.io)

