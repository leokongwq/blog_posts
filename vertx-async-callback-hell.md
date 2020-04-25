---
layout: post
comments: true
title: vert.x异步回调地狱处理方式总结
date: 2017-12-16 13:31:07
tags:
- vert.x
categories:
---

### 前言

什么是callback hell呢？看下图

{% asset_img call-back-hello.jpg %}

有过[Node.js](https://nodejs.org/)开发经验的人，一定程度上都被各种回调嵌套`折磨`过。过去和现在有很多针对Node.js异步回调嵌套处理的解决办法，这里就不讨论了。感兴趣的可以Google。

Vert.x号称JVM上的Node.js 肯定也会有遇到各种回调嵌套的问题。总结这些回调嵌套处理方式对我们使用Vert.x是非常有帮助的。

<!-- more -->

### 通过Future来解决

前面我们知道`Future`接口中有一个方法是`setHandler`方法，这个方法设置Future完成时被调用的处理器；另外一个与之相对应的是`completer()`方法，这个方法返回的就是`setHandler`设置的handler函数。

有两个这个基础，下面来看一个例子：

```java
String filePath = "/data/abc.txt";
fileSystem.createFile(filePath, as -> {
    fileSystem.writeFile(filePath, Buffer.buffer("hello".getBytes()), wr -> {
        System.out.println("write success");
    });
});
```

上面的代码创建了文件，并在文件中写入一个字符串。下面我们用Future进行一下改写:

```java
Future<Void> future = Future.future();
future.setHandler(as -> {
  System.out.println("write success");
  async.complete();
});
String filePath = "/data/abc.txt";
FileSystem fileSystem = vertx.fileSystem();
fileSystem.createFile(filePath, as -> {
    fileSystem.writeFile(filePath, Buffer.buffer("hello".getBytes()), future);
});
```

这样就消除了一个嵌套的回调函数。

再来看一个例子：

```java
@Test
public void testCreateFile2(TestContext context) throws Exception {
   Async async = context.async();
   String filePath = "/data/abc.txt";
   FileSystem fileSystem = vertx.fileSystem();

   Future<Void> createFileFuture = Future.future();
   Future<Void> writeFileFuture = Future.future();
   writeFileFuture.setHandler(as -> {
       System.out.println("success");
       async.complete();
   });
   createFileFuture.compose(as -> {
       fileSystem.writeFile(filePath, Buffer.buffer("Hello".getBytes()), writeFileFuture);
   }, writeFileFuture);

   fileSystem.createFile(filePath, createFileFuture);
}
```

通过Future的compose方法我们可以对Future进行组合，以次来消除callback hell。





