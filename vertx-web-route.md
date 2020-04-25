---
layout: post
comments: true
title: vert.x web开发之route
date: 2017-11-21 09:43:38
tags:
- vert.x
categories:
---

### 简介

一个Route包含了一个条件集合，它用这些条件来判断一个http请求或失败是否应该被路由到指定的Handler。

### 主要方法

#### Route method(HttpMethod method);

添加一个支持的HTTP方法到该Route中。默认情况下，一个Route匹配所有的HTTP方法，如果指定了任意的HTTP方法后，则该Route只会匹配指定的HTTP方法。

<!-- more -->

#### Route path(String path);

设置该Route匹配的路径前缀。设置后，该Route只会处理以参数`path`开头的请求URI。

#### Route pathRegex(String path);

设置该Route匹配path。`path`的值是一个正则表达式。

#### Route produces(String contentType);

添加一个该Route产生的Content-Type值，用来处理基于`Content-Type`的路由。
  
#### Route consumes(String contentType);

添加一个该Route支持的Content-Type值，用来处理基于`Content-Type`的路由。

####  Route order(int order);

设置该Route的顺序

#### Route last();

指定该Route是Router中的最后一个Route

#### Route handler(Handler<RoutingContext> requestHandler);

给该Route指定一个处理器。Router会根据各种条件将请求路由到该处理器。一个Route有且只能有一个处理器。
如果多次调用该方法，那么只有最有一次的调用会生效。

#### Route blockingHandler(Handler<RoutingContext> requestHandler);

该方法和下面的方法功能类似，会议参数ordered=true调用下面的方法。

#### Route blockingHandler(Handler<RoutingContext> requestHandler, boolean ordered);

给该Route指定一个阻塞式的处理器。该方法的功能和方法`handler`类似，只是它会在worker线程池中处理请求，如此一来，它就不会阻塞事件循环`eventloop`线程。

默认情况下，在一个 Context（Vert.x Core 的 Context，例如同一个 Verticle 实例） 上执行的所有阻塞式处理器的执行是顺序的，也就意味着只有一个处理器执行完了才会继续执行下一个。 如果你不关心执行的顺序，并且不介意阻塞式处理器以并行的方式执行，则可以在调用`blockingHandler`方法时将`ordered`设置为`false`。

> 注意，如果你需要在一个阻塞处理器中处理一个`multipart`类型的表单数据，你需要首先使用一个非阻塞的处理器来调用`setExpectMultipart(true)`。 下面是一个例子：

```java
router.post("/some/endpoint").handler(ctx -> {
  ctx.request().setExpectMultipart(true);
  ctx.next();
}).blockingHandler(ctx -> {
  // 执行某些阻塞操作
});
```

#### Route failureHandler(Handler<RoutingContext> failureHandler);

给该Route设置一个失败处理器。失败处理器和普通的请求处理器拥有相同的路由匹配规则。一个Route有且只有一个失败处理器。

当一个处理器抛出了异常，或者通过`fail`方法设置了HTTP状态码时，失败的处理器就会被调用。

```java
Route route1 = router.get("/somepath/path1/");

route1.handler(routingContext -> {

  // 这里抛出一个 RuntimeException
  throw new RuntimeException("something happened!");

});

Route route2 = router.get("/somepath/path2");

route2.handler(routingContext -> {
  // 这里故意将请求处理为失败状态
  // 例如 403 - 禁止访问
  routingContext.fail(403);

});

// 定义一个失败处理器，上述的处理器发生错误时会调用这个处理器
Route route3 = router.get("/somepath/*");

route3.failureHandler(failureRoutingContext -> {

  int statusCode = failureRoutingContext.statusCode();

  // 对于 RuntimeException 状态码会是 500，否则是 403
  HttpServerResponse response = failureRoutingContext.response();
  response.setStatusCode(statusCode).end("Sorry! Not today");

});
```






