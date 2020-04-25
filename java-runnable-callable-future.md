---
layout: post
comments: true
title: Java中的Runnable、Callable、Future、FutureTask的区别与示例
date: 2016-10-16 09:04:10
tags:
- java
categories:
- java
---

> Java中存在Runnable、Callable、Future、FutureTask这几个与线程相关的类或者接口，在Java中也是比较重要的几个概念，我们通过下面的简单示例来了解一下它们的作用于区别。 

<!-- more -->

### Runnable

Runnable 应该是这几个类我们使用的最多的一个。JDK的文档说明：如果一个类的实例想要通过一个线程来执行，则该类应该实现Runnable接口。Runnable被设计用来对那些处于active状态时会执行代码的对象提供一个统一的协议。例如Thread类就实现了Runnable接口。

```java
public interface Runnable {
    public abstract void run();
}
```

### Callable 

Callable 表示一个可以携带返回结果的任务。该接口的实现类需要一个没有参数的call方法。Callable 和 Runnable类似，都是被设计用来被另外一个线程执行的任务。但是Runnable不能返回一个结果，且不能抛出一个checked异常。

```java
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

### Future

一个Futrue表示一个异步计算的结果。它提供了一系列方法，用来检测计算是否完成，获取计算结果，等待计算结果等。只能通过`get`方法获取计算结果；如果计算没有完成，在必要条件下，则get方法一直等待，直到任务完成。`cancel`方法用来取消任务。其它的方法都是用来测试计算是否完成，或是否取消。一旦计算完成后，就不可以被取消。如果你想使用Future，但并不需要返回一个结果，则可以使用`Future<?>`并返回null。

一个简单的例子（JDK原文）：

```java
class App {
    ExecutorService executor = ...
    ArchiveSearcher searcher = ...
    void showSearch(final String target) throws InterruptedException {
        Future<String> future = executor.submit(new Callable<String>() {
            public String call() {
                return searcher.search(target);
            }});
        displayOtherThings(); 
        try{
             displayText(future.get()); // use future
        }catch(ExecutionException ex){
            cleanup(); return; 
        }
    }
}
```

### FutureTask

FutureTask 表示一个可以取消的异步计算任务。它实现了Runnable接口和Future接口。FutureTask 可以用来包装Runnable和Callable对象。应该FutureTask实现了Runnable接口。同时可以通过被提交到Executor去执行。

```java
public FutureTask(Callable<V> callable) {  
    if (callable == null)  
    throw new NullPointerException();  
    this.callable = callable;  
    this.state = NEW;       // ensure visibility of callable  
}  

public FutureTask(Runnable runnable, V result) {  
    this.callable = Executors.callable(runnable, result);  
    this.state = NEW;       // ensure visibility of callable  
}  
```

可以看到，Runnable注入会被Executors.callable()函数转换为Callable类型，即FutureTask最终都是执行Callable类型的任务。该适配函数的实现如下 ：

```java
public static <T> Callable<T> callable(Runnable task, T result) {  
    if (task == null)  
        throw new NullPointerException();  
    return new RunnableAdapter<T>(task, result);  
}  
```

**RunnableAdapter适配器**

```java
/** 
 * A callable that runs given task and returns given result 
 */  
static final class RunnableAdapter<T> implements Callable<T> {  
    final Runnable task;  
    final T result;  
    RunnableAdapter(Runnable task, T result) {  
        this.task = task;  
        this.result = result;  
    }  
    public T call() {  
        task.run();  
        return result;  
    }  
} 
```

### 总结

由于FutureTask实现了Runnable，因此它既可以通过Thread包装来直接执行，也可以提交给ExecuteService来执行。
并且还可以直接通过get()函数获取执行结果，该函数会阻塞，直到结果返回。因此FutureTask既是Future、
Runnable，又是包装了Callable(如果是Runnable最终也会被转换为Callable )， 它是这两者的合体。








        




                   
                
                
                