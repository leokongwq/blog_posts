---
layout: post
comments: true
title: java8中CompletableFuture解析
date: 2017-11-30 13:19:35
tags:
categories:
- java
---

### 背景

在JDK8以前我们都知道Future是用来代表一个异步计算的结果的。在最新的JDK8中引入了一个新的类`CompletableFuture`。它和以前的`Future`有哪些不同？它解决`Future`的哪些不足？这些个问题就是我需要搞清楚的问题。

### CompletionStage

在学习`CompletableFuture`前需要先搞清楚`CompletionStage`的作用。从字面意思来看，该类表示任务的完成阶段。JDK的说明文档如下：

> 一个可能执行的异步计算的某个阶段，在另一个`CompletionStage`完成时执行一个操作或计算一个值。一个阶段完成后，其计算结束。但是，该计算阶段可能会触发下一个计算阶段。

该接口的主要方法如下：

<!-- more -->

#### public <U> CompletionStage<U> thenApply(Function<? super T,? extends U> fn);

该方法的作用是在该计算阶段正常完成后，将该计算阶段的结果作为参数传递给参数`fn`值的函数`Function`，并会返回一个新的`CompletionStage`，

#### public <U> CompletionStage<U> thenApplyAsync(Function<? super T,? extends U> fn);

该方法和上面的方法`thenApply`功能类似，不同的是对该计算阶段的结果进行计算的函数`fn`的执行时异步的。

#### public <U> CompletionStage<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor);

该方法和上面的方法`thenApplyAsync`功能类似，不同的是对该计算阶段的结果进行计算的函数`fn`的执行时异步的， 并且是在调用者提供的线程池中执行的。

#### public CompletionStage<Void> thenAccept(Consumer<? super T> action);

该方法的作用是在该计算阶段完成后，将该阶段的计算结果交给一个`Consumer`去消费，并返回一个新的CompletionStage

#### public CompletionStage<Void> thenAcceptAsync(Consumer<? super T> action);

该方法的作用是在该计算阶段完成后，将该阶段的计算结果交给一个`Consumer`去消费，消费者的执行是异步的。最终返回一个新的CompletionStage

#### public CompletionStage<Void> thenAcceptAsync(Consumer<? super T> action, Executor executor);

该方法的作用是在该计算阶段完成后，将该阶段的计算结果交给一个`Consumer`去消费。消费者的执行是异步的，并且是在调用者提供的线程池中进行的。最终返回一个新的CompletionStage

#### public CompletionStage<Void> thenRun(Runnable action);

该方法的作用是在该计算阶段完成后，执行参数`action`指定的动作。

#### public CompletionStage<Void> thenRunAsync(Runnable action);

该方法的作用是在该计算阶段完成后，执行参数`action`指定的动作。不同的是，该动作的执行是异步的。最终返回一个新的CompletionStage

#### public CompletionStage<Void> thenRunAsync(Runnable action, Executor executor);

该方法的作用是在该计算阶段完成后，执行参数`action`指定的动作。不同的是，该动作的执行是异步的，并且是在调用者提供的线程池中异步执行的。最终返回一个新的CompletionStage

#### public <U,V> CompletionStage<V> thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn);

该方法的作用是在该计算阶段和参数`other`指定的被组合的计算阶段都正常完成后，将这两个计算阶段的计算结果作为参数，调用参数`fn`值的函数。 最终返回一个新的CompletionStage。

#### public <U,V> CompletionStage<V> thenCombineAsync(CompletionStage<? extends U> other,
BiFunction<? super T,? super U,? extends V> fn);

该方法的作用和上面方法`thenCombine`类似，不同点在于参数`fn`指定函数执行时异步的。

#### public <U,V> CompletionStage<V> thenCombineAsync(CompletionStage<? extends U> other,
BiFunction<? super T,? super U,? extends V> fn, Executor executor);

该方法的作用和上面方法`thenCombine`类似，不同点在于参数`fn`指定函数执行时异步的，并且是在调用者提供的线程池中进行执行的。

#### public <U> CompletionStage<Void> thenAcceptBoth(CompletionStage<? extends U> other,
BiConsumer<? super T, ? super U> action);

该方法的作用在该计算阶段和被组合的计算阶段都执行完成后，将两个阶段的结果作为参数调用参数`action`指定的函数。

#### public <U> CompletionStage<Void> thenAcceptBothAsync(CompletionStage<? extends U> other,
BiConsumer<? super T, ? super U> action);

该方法的作用在该计算阶段和被组合的计算阶段都执行完成后，将两个阶段的结果作为参数调用参数`action`指定的函数。但是函数的执行时异步的。

#### public <U> CompletionStage<Void> thenAcceptBothAsync(CompletionStage<? extends U> other,
BiConsumer<? super T, ? super U> action, Executor executor);

该方法的作用在该计算阶段和被组合的计算阶段都执行完成后，将两个阶段的结果作为参数调用参数`action`指定的函数。但是函数的执行时异步的，并且是在调用者指定的线程池中执行。

#### public CompletionStage<Void> runAfterBoth(CompletionStage<?> other, Runnable action);

该方法的作用在该计算阶段和被组合的计算阶段都执行完成后，执行参数`action`指定的动作

#### public CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other,Runnable action);

该方法的作用在该计算阶段和被组合的计算阶段都执行完成后，执行参数`action`指定的动作。该动作是异步执行的。

#### public CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other, Runnable action,
Executor executor);

该方法的作用在该计算阶段和被组合的计算阶段都执行完成后，执行参数`action`指定的动作。该动作是异步执行的，并且是在调用者指定的线程池中。

#### public <U> CompletionStage<U> applyToEither(CompletionStage<? extends T> other,
Function<? super T, U> fn);

该方法的作用是在该计算阶段或被组合的计算阶段完成后，将对应的计算完成的结果作为参数调用参数`fn`指定的函数。

#### public <U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other,
Function<? super T, U> fn);

该方法的作用和上面方法`applyToEitherAsync`的作用类似，不同的是`fn`的执行时异步的。

#### public <U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other,
Function<? super T, U> fn, Executor executor);

该方法的作用和上面方法`applyToEitherAsync`的作用类似，不同的是`fn`的执行时异步的，并且是在调用者提供的线程池中执行。

#### public CompletionStage<Void> acceptEither(CompletionStage<? extends T> other,
Consumer<? super T> action);

#### public CompletionStage<Void> acceptEitherAsync(CompletionStage<? extends T> other,
Consumer<? super T> action);

#### public CompletionStage<Void> acceptEitherAsync(CompletionStage<? extends T> other,
Consumer<? super T> action, Executor executor);

#### public CompletionStage<Void> runAfterEither(CompletionStage<?> other, Runnable action);

#### public CompletionStage<Void> runAfterEitherAsync(CompletionStage<?> other, Runnable action);

#### public CompletionStage<Void> runAfterEitherAsync(CompletionStage<?> other, Runnable action, Executor executor);

#### public <U> CompletionStage<U> thenCompose(Function<? super T, ? extends CompletionStage<U>> fn);

该方法的作用是在该计算阶段完成后，将该`CompletionStage`作为参数调用参数`fn`指定的函数，并最终返回一个新的`CompletionStage`。

#### public <U> CompletionStage<U> thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn);

该方法的作用是在该计算阶段完成后，将该`CompletionStage`作为参数调用参数`fn`指定的函数，并最终返回一个新的`CompletionStage`。函数`fn`的执行时异步的。

#### public <U> CompletionStage<U> thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn, Executor executor);

该方法的作用是在该计算阶段完成后，将该`CompletionStage`作为参数调用参数`fn`指定的函数，并最终返回一个新的`CompletionStage`。函数`fn`的执行时异步的，并且在调用者指定的线程池中执行。

#### public CompletionStage<T> exceptionally(Function<Throwable, ? extends T> fn);

当前计算阶段异常结束后，执行参数`fn`值的函数，并返回一个新的`CompletionStage`。

#### public CompletionStage<T> whenComplete(BiConsumer<? super T, ? super Throwable> action);

返回一个新的`CompletionStage`，新的`CompletionStage`和当前`CompletionStage`拥有相同的结果或执行异常。执行结果或异常会作为参数，调动`action`指定的动作。

#### public CompletionStage<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action);

异步执行参数指定的动作。

#### public CompletionStage<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action, Executor executor);

异步执行参数指定的动作。

#### public <U> CompletionStage<U> handle(BiFunction<? super T, Throwable, ? extends U> fn);

当该计算阶段正常结束或异常结束后，都会调用参数`fn`指定的函数。最后返回一个新的`CompletionStage`。

#### public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn);

当该计算阶段正常结束或异常结束后，都会调用参数`fn`指定的函数，函数是异步执行的。最后返回一个新的`CompletionStage`。

#### public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn, Executor executor);

当该计算阶段正常结束或异常结束后，都会调用参数`fn`指定的函数，函数是在调用者提供的线程池中执行的。最后返回一个新的`CompletionStage`。

#### public CompletableFuture<T> toCompletableFuture();

### CompletableFuture

```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {
    ....
}
```

`CompletableFuture`可以作为一个显示设置计算结果的Future，并可以作为一个`CompletionStage`，在该计算完成时触发依赖该计算的计算或动作。

### CompletableFuture 创建

有很多种方式来创建一个CompletableFuture，

#### public CompletableFuture() {}

#### public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) 

由参数`supplier`异步创建一个。内部使用ForkJoin框架来执行。

#### public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,Executor executor) 

由参数`supplier`异步创建一个。使用参数`executor`指定的线程池来异步执行。

#### public static CompletableFuture<Void> runAsync(Runnable runnable);

由参数指定的异步任务来创建一个。

#### public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor);

由参数指定的异步任务来创建一个。异步任务在参数指定的线程池执行。

#### public static <U> CompletableFuture<U> completedFuture(U value)；

创建一个已经完成的`CompletableFuture`。

### 完成 CompletableFuture

#### public boolean complete(T value) 

使用参数指定的值完成 CompletableFuture

#### public boolean completeExceptionally(Throwable ex) 

使用参数指定的`异常`值完成 CompletableFuture








