---
layout: post
comments: true
title: vert.x web开发之router
date: 2017-11-21 09:12:33
tags:
- vert.x
categories:
---

### 简介

vert.x web开发中router相当于一个核心的控制器，管理多个具体负责处理请求URL的Route对象，它接受来自`HttpServer`的请求`HttpServerRequest`，并将请求路由到第一个匹配的Route上去。

### 主要方法

#### static Router router(Vertx vertx)

该方法用来创建一个Router对象。

#### void accept(HttpServerRequest request)

该方法用来给一个请求提供一个Router。通常的使用方式为：将该对象作为参数传递给`HttpServer.requestHandler(Handler)`方法中。

```java
vertx.createHttpServer(serverOptions).requestHandler(router::accept)
```
<!-- more -->

#### Route route();

该方法用来添加一个没有任何匹配条件的Route。也就是说返回的Route可以处理任何请求。

#### Route route(HttpMethod method, String path)

该方法添加一个处理指定HTTP方法和请求URL的Route。

#### Route route(String path);

该方法添加一个处理指定URL的Route。

#### Route routeWithRegex(HttpMethod method, String regex);

该方法添加一个处理指定HTTP方法，正则匹配URL的Route

#### Route routeWithRegex(String regex);

该方法添加一个正则匹配URL的Route

#### Route get();

添加一个处理所有GET请求的Route。

#### Route get(String path);

添加一个处理指定URL的GET请求的Route。

#### Route getWithRegex(String regex);

添加一个正则匹配URL的GET请求的Route。

> 其它和HTTP方法处理规则和GET方法类似。

#### List<Route> getRoutes();

该方法返回所有该Router包含的Route列表

#### Router clear();

清空该Router包含的Route

#### Router mountSubRouter(String mountPoint, Router subRouter);

挂载一个子Router到该Router上

#### Router exceptionHandler(@Nullable Handler<Throwable> exceptionHandler);

给该Router添加一个异常处理器，来处理Handler抛出的异常。该Handler不会影响正常的失败路由逻辑。

#### void handleContext(RoutingContext context);

Used to route a context to the router. Used for sub-routers. You wouldn't normally call this method directly.

#### void handleFailure(RoutingContext context);

Used to route a failure to the router. Used for sub-routers. You wouldn't normally call this method directly.




