---
layout: post
comments: true
title: vertx-Future
date: 2017-12-15 19:57:31
tags:
- vert.x
categories:
---

### 前言

在Java中`Future`表示一个异步计算的结果，提供了许多方法来获取结果，取消计算等。在Vert.x中`Future`表示一个动作的结果，该结果可能出现也可能没有出现。二者有许多相似之处。

Vert.x中`Future`的继承层次如下：

<!-- more -->

```java
public interface Future<T> extends AsyncResult<T>, Handler<AsyncResult<T>> {
    ...
}
```

在分析`Future`前我们先来看下`AsyncResult`和`Handler`的具体功能。

### AsyncResult

`AsyncResult`包装了一个异步操作的结果。接口声明如下：

```java
public interface AsyncResult<T> {
    ...
}
```

主要方法有：

#### T result()

`T result()` 方法用来获取异步操作的结果。

#### Throwable cause()

`Throwable cause()` 方法用来异步操作失败的异常。

#### boolean succeeded()

`boolean succeeded()` 方法用来判断异步操作是否成功。

#### boolean failed();

`boolean failed()` 方法用来判断异步操作是否失败。

#### default <U> AsyncResult<U> map(Function<T, U> mapper)

该方法用来该异步操作的结果通过一个映射`mapper`函数转换为另一个结果。

#### default <V> AsyncResult<V> map(V value) 

该方法用来该异步操作的结果映射为参数`value`指定的值。

#### default <V> AsyncResult<V> mapEmpty() 

该方法用来将异步操作的结果映射为一个空值。

#### default AsyncResult<T> otherwise(Function<Throwable, T> mapper)

该方法用来在该异步操作结果是失败时，通过参数`mapper`指定映射函数将结果映射为其它值。

#### default AsyncResult<T> otherwise(T value)

该方法和上一个方法功能相同。

#### default AsyncResult<T> otherwiseEmpty() 

该方法和上一个方法功能相同，不同点在于返回一个空值。

### Handler 

`Handler` 在Vert.x中非常重要，用来处理Vert.x中所有的异步操作结果。

它只有一个方法：

```java
    void handle(E event);
```

### Future 

`Future`的方法很多，我们只分析非常重要的一些。

#### 创建一个`Future`对象

```java
static <T> Future<T> future()
```

上面的代码创建了一个还没有完成的Future。


```java
static <T> Future<T> future(Handler<Future<T>> handler) 
```

上面的代码创建了一个还没有完成的Future，并且在该Future还没有返回前就传递给参数指定的`Handler`。

```java
static <T> Future<T> succeededFuture() 
```

上面的代码创建了一个已经完成的Future，其结果为null。

```java
static <T> Future<T> succeededFuture(T result) 
```

上面的代码创建了一个已经完成的Future，其结果为参数`result`指定的值。

```java
static <T> Future<T> failedFuture(Throwable t) 
static <T> Future<T> failedFuture(String failureMessage)
```

上面的代码创建了失败的Future。

#### 判断Future的结果

```java
boolean isComplete(); //是否完成
boolean succeeded(); //是否成功
boolean failed(); //是否是否
```

#### 设置Future的结果

```java
void complete(T result); //设置Future的结果为参数指定值
void complete(); //设置Future的结果为null
boolean tryComplete(T result);  //尝试设置Future的结果为参数指定值
boolean tryComplete(); //尝试设置Future的结果为null
void fail(Throwable cause); //设置Future为失败状态
void fail(String failureMessage); //设置Future为失败状态
boolean tryFail(Throwable cause); //尝试设置Future为失败状态
boolean tryFail(String failureMessage); //尝试设置Future为失败状态
```

#### Future的组合

```java
default <U> Future<U> compose(Handler<T> handler, Future<U> next)
default <U> Future<U> compose(Function<T, Future<U>> mapper)
```

#### Future 的转换/映射

```java
default <U> Future<U> map(Function<T, U> mapper)
default <V> Future<V> map(V value)
default <V> Future<V> mapEmpty()
```

#### Future<T> setHandler(Handler<AsyncResult<T>> handler)

```java
Future<T> setHandler(Handler<AsyncResult<T>> handler)
```

该方法用来设置该Future完成时被调用的处理器的。

#### default Handler<AsyncResult<T>> completer()

```java
default Handler<AsyncResult<T>> completer()
```

该方法用来返回该Future完成时被调用的处理器的。

### Future 使用的一个例子

```java
Async async = context.async();
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




