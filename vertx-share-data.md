---
layout: post
comments: true
title: vertx-共享数据
date: 2017-11-18 13:56:03
tags:
- vert.x
categories:
---

共享数据（Shared Data）包含的功能允许您可以安全地在应用程序的不同部分之间、同一 Vert.x 实例中的不同应用程序之间，集群中的不同 Vert.x 实例之间安全地共享数据。

共享数据包括本地共享Map、分布式、集群范围Map、异步集群范围锁和异步集群范围计数器。

> 重要提示：分布式数据结构的行为取决于您使用的集群管理器，网络分区面临的备份（复制）和行为由集群管理器和它的配置来定义。请参阅集群管理器文档以及底层框架手册。

<!-- more -->

### 本地共享Map

本地共享Map [LocalMap](http://vertx.io/docs/apidocs/io/vertx/core/shareddata/LocalMap.html) 允许您在同一个 Vert.x 实例中的不同 Event Loop（如不同的 Verticle 中）之间安全共享数据。

本地共享Map仅允许将某些数据类型作为键值和值，这些类型*必须是不可变的*，或可以像Buffer那样复制某些其他类型。在后一种情况中，键/值将被复制，然后再放到Map中。

这样，我们可以确保在Vert.x应用程序不同线程之间 `没有访问共享可变状态`，因此您不必担心需要通过同步访问来保护该状态。

以下是使用 LocalMap 的示例：

```java
SharedData sd = vertx.sharedData();

LocalMap<String, String> map1 = sd.getLocalMap("mymap1");

// String是不可变的，所以不需要复制
map1.put("foo", "bar"); // Strings are immutable so no need to copy

LocalMap<String, Buffer> map2 = sd.getLocalMap("mymap2");

// Buffer将会在添加到Map之前拷贝
map2.put("eek", Buffer.buffer().appendInt(123)); // This buffer will be copied before adding to map

// Then... in another part of your application:
// 之后，您的应用另外一部分
map1 = sd.getLocalMap("mymap1");

String val = map1.get("foo");

map2 = sd.getLocalMap("mymap2");

Buffer buff = map2.get("eek");
```

### 集群范围异步Map

集群范围异步Map(Cluster-wide asynchronous maps)允许从集群的任何节点将数据放到 Map 中，并从任何其他节点读取。

这使得它们对于托管Vert.x Web应用程序的服务器场中的会话状态存储非常有用。

您可以使用 [getClusterWideMap](http://vertx.io/docs/apidocs/io/vertx/core/shareddata/SharedData.html#getClusterWideMap-java.lang.String-io.vertx.core.Handler-) 方法获取[AsyncMap](http://vertx.io/docs/apidocs/io/vertx/core/shareddata/AsyncMap.html)的实例。

获取Map的过程是异步的，返回结果可以传给您指定的处理器中。以下是一个例子：

```java
SharedData sd = vertx.sharedData();

sd.<String, String>getClusterWideMap("mymap", res -> {
  if (res.succeeded()) {
    AsyncMap<String, String> map = res.result();
  } else {
    // Something went wrong!
    // 出现一些错误
  }
});
```

### 将数据放入Map

您可以使用 put 方法将数据放入Map。put方法是异步的，一旦完成它会通知处理器：

```java
map.put("foo", "bar", resPut -> {
  if (resPut.succeeded()) {
    // Successfully put the value
    // 成功放入值
  } else {
    // Something went wrong!
    // 出了些问题
  }
});
```

### 从Map读取数据

您可以使用 get 方法从Map读取数据。get 方法也是异步的，一段时间过后它会通知处理器并传入结果。

```java
map.get("foo", resGet -> {
  if (resGet.succeeded()) {
    // Successfully got the value
    // 成功读取值
    Object val = resGet.result();
  } else {
    // Something went wrong!
    // 出了些问题
  }
});
```

### 其他Map操作

您还可以从异步Map中删除条目、清除Map、读取它的大小。

有关更多信息，请参阅 [API 文档](http://vertx.io/docs/apidocs/io/vertx/core/shareddata/AsyncMap.html)。

v


