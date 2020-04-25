---
layout: post
comments: true
title: 异步调用转为同步
date: 2020-02-05 13:41:21
tags:
- Java
categories:
---

### 背景

在日常的开发工作中，我们经常会有一些任务需要异步执行。那么如何获取异步任务的结果呢？常见的解决方案有两种：1，任务异步执行，调用方同步获取结果（本篇文章需要讨论的）；2，通过Callback 或 Listener进行处理。

针对需要同步获取异步计算结果的需求，下面分场景进行讨论并给出解决方案和原理分析。

### Future + 线程池

这个是最常见模式：

```java
ExecutorService executorService = Executors.newFixedThreadPool(1);
Future<String> result = executorService.submit(new Callable<String>() {
    @Override
    public String call() throws Exception {
        Thread.sleep(1000);
        return "Hello";
    }
});
System.out.println(result.get());
```

<!-- more -->

线程池用来执行异步任务，主线程等待任务执行完成，获取结果。很容易想到底层实现必然涉及到线程的等待-通知。下面就深入进行下原理分析：

上面代码`executorService.submit`返回值的真实类型是`FutureTask`，它实现了`Future`接口和`Runnable`接口

#### FutureTask

```java
// 异步任务的执行状态
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;
    
// 异步任务的结算结果
private Object outcome; // non-volatile, protected by state reads/writes

 /**
 * @throws CancellationException {@inheritDoc}
 */
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    // 任务是新创建 或 执行中，则将当前线程添加到等待队列中。
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}

/**
 * @throws CancellationException {@inheritDoc}
 */
public V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException {
    if (unit == null)
        throw new NullPointerException();
    int s = state;
    if (s <= COMPLETING &&
        (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
        throw new TimeoutException();
    return report(s);
}

private V report(int s) throws ExecutionException {
    Object x = outcome;
    if (s == NORMAL)
        return (V)x;
    if (s >= CANCELLED)
        throw new CancellationException();
    throw new ExecutionException((Throwable)x);
}

/**
 * Awaits completion or aborts on interrupt or timeout.
 *
 * @param timed true if use timed waits
 * @param nanos time to wait, if timed
 * @return state upon completion
 */
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    // 自旋 或 等待执行完成。
    for (;;) {
        // 调用线程被中断，抛异常
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }
        
        int s = state;
        // 异步任务 正常结束，被中断，被取消，执行异常，直接返回。
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        // 让出CPU，提高异步任务的执行机会
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        else if (q == null)
            // 新创建的任务，第一次循环，创建节点
            q = new WaitNode();
        else if (!queued)
            // 添加到等待队列， 第二次循环
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        else if (timed) {
            nanos = deadline - System.nanoTime();
            // 超时了
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            // 使当前线程等待指定的时间 或 被unpark
            LockSupport.parkNanos(this, nanos);
        }
        else
            // 使当前线程一直等待，直到被unpark
            LockSupport.park(this);
    }
}
// 实现了 Runnable 接口的run方法，被线程池调用，完成异步任务的执行。
public void run() {
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            // 正常执行完成，设置执行结果，
            if (ran)
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}  
 
// 设置异步任务执行的结果
protected void set(V v) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        finishCompletion();
    }
}

/**
 * Removes and signals all waiting threads, invokes done(), and
 * nulls out callable.
 */
private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    // 唤醒等待的线程。
                    LockSupport.unpark(t);
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }
    // 预留的扩展钩子方法，任务执行完成后获得通知。
    done();

    callable = null;        // to reduce footprint
}
```

#### 小结

1. 异步任务由线程池里面的线程进行异步执行。
2. Future的作用是当任务未完成时，使得调用线程排队等待。
3. 异步任务完成后，设置Future中表示结果字段的值，并唤醒等待的线程。

### 自定义Future实现

在RPC调用中，一般有三种调用方式：
- oneWay 不需要获取返回结果
- sync 同步调用
- async 异步调用

#### RPC 同步调用实现分析

大多数RPC框架目前都是基于一些NIO框架(Netty, Mina等)实现的，远程调用的过程就是将请求信息进行序列化并写入到Socket的过程。问题在于完成请求发送完成后，代码执行流程会继续执行，如何获得远程调用的结果呢？

如果是全异步编程环境或者异步处理返货结果，很简单，只需要注册一个Callback，请求结果返回后，由RPC框架进行调用。

如果需要同步获取结果，那么还是需要一种机制使当前调用线程进行等待，调用结果返回后通知唤醒等待的调用线程。

下面给出一个RPC框架中实现：

```java
// 该Listener 在请求响应返回调用
public interface ResponseListener<T> {
    void responseReceived(T obj);
}    
```

基于Lock实现的 Future 

```java 
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @author leokongwq
 */
public class LockBasedResponseListenerFuture<T> implements Future<T>, ResponseListener<T> {

    private final Lock lock = new ReentrantLock();

    private final Condition condition = lock.newCondition();
    // 是否完成的标识
    private volatile boolean done = false;
    // 计算结果
    private volatile T response;

    @Override
    public boolean cancel(boolean mayInterruptIfRunning) {
        return false;
    }

    @Override
    public boolean isCancelled() {
        return false;
    }

    @Override
    public boolean isDone() {
        return done;
    }

    @Override
    public T get() throws InterruptedException {
        if (!isDone()) {
            this.lock.lock();
            try {
                // 没有完成，则使得当前调用线程进行等待
                if (!isDone()) {
                    condition.await();
                }
            } finally {
                this.lock.unlock();
            }
        }
        return response;
    }

    @Override
    public T get(long timeout, TimeUnit unit) throws InterruptedException, TimeoutException {
        if (!isDone()) {
            this.lock.lock();
            try {
                // 没有完成，则使得当前调用线程进行等待
                if (!isDone()) {
                    boolean done = condition.await(timeout, unit);
                    if (!done) {
                        throw new TimeoutException("waiting response timeout!");
                    }
                }
            } finally {
                this.lock.unlock();
            }
        }
        return response;
    }
    
    @Override
    public void responseReceived(T response) {
        this.lock.lock();
        try {
            // 计算完成，设置结果，唤醒等待的线程。
            this.response = response;
            this.done = true;
            this.condition.signalAll();
        } finally {
            this.lock.unlock();
        }
    }
}
```

这里面的Lock也也可以替换为`CountDownLatch`，本质都是线程间协调。

### CompletableFuture

`CompletableFuture`是JDK8提供的一个类，更好的支持异步编程和函数式编程风格。
`CompletableFuture`实现了接口`Future`，但是功能更强，可以提供了异步任务完成方法。
关于`CompletableFuture`的详细介绍建议参考文章[Java-CompletableFuture](https://colobu.com/2016/02/29/Java-CompletableFuture/)

那如何使用`CompletableFuture`来完成异步转同步呢？如下示例:

```java
CompletableFuture<String> completableFuture = new CompletableFuture<>();
executorService.execute(() -> {
    try {
        Thread.sleep(500);
    } catch (InterruptedException e) {
    }
    //异步计算任务线程，完成异步计算。
    completableFuture.complete("hello");
});
// 阻塞获取计算结果
String str = completableFuture.get(1000, TimeUnit.SECONDS);
System.out.println(str);
```

翻看最新的Dubbo代码，发现同步和异步调用都使用了`CompletableFuture`，这里需要赞一下。

需要注意的是，最底层的原理还是线程的协调，和`FutureTask`一样底层都是使用了`LockSupport`工具类。

