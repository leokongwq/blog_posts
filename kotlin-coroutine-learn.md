---
layout: post
comments: true
title: kotlin之coroutine学习笔记
date: 2018-01-10 19:17:15
tags:
categories:
- kotlin
---

### runBlocking

#### 声明

```java
@Throws(InterruptedException::class)
public fun <T> runBlocking(context: CoroutineContext = EmptyCoroutineContext, block: suspend CoroutineScope.() -> T): T {
    val currentThread = Thread.currentThread()
    val eventLoop = if (context[ContinuationInterceptor] == null) BlockingEventLoop(currentThread) else null
    val newContext = newCoroutineContext(context + (eventLoop ?: EmptyCoroutineContext))
    val coroutine = BlockingCoroutine<T>(newContext, currentThread, privateEventLoop = eventLoop != null)
    coroutine.initParentJob(context[Job])
    block.startCoroutine(coroutine, coroutine)
    return coroutine.joinBlocking()
}
```

<!-- more -->

#### 功能

创建一个`coroutine`并且阻塞当前线程直到区块执行完毕，这个一般是用于桥接一般的阻塞式编程方式到`coroutine`编程方式的，不应该在已经是`coroutine`的地方使用。

### run

#### 声明

```java
public suspend fun <T> run(
    context: CoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend () -> T
): T = suspendCoroutineOrReturn sc@ { cont ->
    val oldContext = cont.context
    // fast path #1 if there is no change in the actual context:
    if (context === oldContext || context is CoroutineContext.Element && oldContext[context.key] === context)
        return@sc block.startCoroutineUninterceptedOrReturn(cont)
    // compute new context
    val newContext = oldContext + context
    // fast path #2 if the result is actually the same
    if (newContext === oldContext)
        return@sc block.startCoroutineUninterceptedOrReturn(cont)
    // fast path #3 if the new dispatcher is the same as the old one.
    // `equals` is used by design (see equals implementation is wrapper context like ExecutorCoroutineDispatcher)
    if (newContext[ContinuationInterceptor] == oldContext[ContinuationInterceptor]) {
        val newContinuation = RunContinuationDirect(newContext, cont)
        return@sc block.startCoroutineUninterceptedOrReturn(newContinuation)
    }
    // slowest path otherwise -- use new interceptor, sync to its result via a full-blown instance of RunCompletion
    require(!start.isLazy) { "$start start is not supported" }
    val completion = RunCompletion(
        context = newContext,
        delegate = cont,
        resumeMode = if (start == CoroutineStart.ATOMIC) MODE_ATOMIC_DEFAULT else MODE_CANCELLABLE)
    completion.initParentJob(newContext[Job]) // attach to job
    start(block, completion)
    completion.getResult()
}
```

#### 功能

在指定的`CoroutineContext`中调用指定的`suspend`区块。 该方法会导致当前`CoroutineContext`中的`coroutine`挂起并等待该代码块执行完毕。

### launch

#### 声明

```java
public fun launch(
    context: CoroutineContext = DefaultDispatcher,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.initParentJob(context[Job])
    start(block, coroutine, coroutine)
    return coroutine
}
```

#### 功能

创建运行在`CoroutineContext`中的`coroutine`，返回的`Job`支持取消、启动等操作，不会挂起父`coroutine`上下文；可以在非`coroutine`中调用。

参数`context`可以显示指定。默认为`DefaultDispatcher`。
参数`start`表示`coroutine`的启动选项。默认为`CoroutineStart.DEFAULT`
参数`block`表示`coroutine`里面执行的代码。

默认请情况下，coroutine是会被立即调度执行的。如果参数`start`的值为`CoroutineStart.LAZY`，那么会返回一个新建状态的`coroutine`。 可以通过显示调用`Job.start()`方法来启动该`coroutine`。

如果在该`coroutine`里面有未捕获的异常，那么会导致该`coroutine`的父`coroutine`取消。

### async

#### 声明

```java
public fun <T> async(
    context: CoroutineContext = DefaultDispatcher,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> T
): Deferred<T> {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyDeferredCoroutine(newContext, block) else
        DeferredCoroutine<T>(newContext, active = true)
    coroutine.initParentJob(context[Job])
    start(block, coroutine, coroutine)
    return coroutine
}
```

#### 功能

创建运行在CoroutineContext中的coroutine，并且带回返回值(返回的是Deferred，我们可以通过await等方式获得返回值)


### 参考

[Kotlin Coroutines(协程)](https://blog.dreamtobe.cn/kotlin-coroutines/)



