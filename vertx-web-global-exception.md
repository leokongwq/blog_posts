---
layout: post
comments: true
title: vertx中web全局异常处理
date: 2017-12-02 09:46:41
tags:
- vert.x
categories:
---

### 前言

在以前使用springMVC时，全局异常的处理的方式有很多。现在使用vertx-web开发web应用，同样有这样的需要，而且实现全局异常的处理更简单。


### 404 已经 500 异常处理

我们先看一段简单的代码：

<!-- more -->

```java
public class MyFirstVerticle extends AbstractVerticle {

    @Override
    public void start() throws Exception {
        final Router router = Router.router(vertx);
        router.get("/hello").handler(context -> {
            Integer.parseInt(context.request().getParam("age"));
            context.response().end("hello vert.x");
        }).failureHandler(context -> {
            context.response().end("Route internal error process");
        });
        router.get("/world").handler(context -> {
            Integer.parseInt(context.request().getParam("age"));
            context.response().end("hello world");
        });
        //最后一个Route
        router.route().last().handler(context -> {
            context.response().end("404");
        }).failureHandler(context -> {
           context.response().end("global error process");
        });
        vertx.createHttpServer().requestHandler(router::accept).listen(9090);
    }
}
```

如果我们访问地址[http://127.0.0.1:9090/hello](http://127.0.0.1:9090/hello)，会返回

```
hello vert.x
```

如果我们访问地址[http://127.0.0.1:9090/hello?age=abc](http://127.0.0.1:9090/hello?age=abc)，会返回

```
Route internal error process
```

如果我们访问的请求URL没有匹配的Route, 则返回：

```
404
```

如果在其它的Route处理过程中发生了错误，并且请求匹配的Route没有通过方法`failureHandler`设置自己专属的错误处理器(e.g. `/world`)，那么就会返回：

```
global error process
```









