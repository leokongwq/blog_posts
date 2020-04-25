---
layout: post
comments: true
title: java中线程join方法的用法
date: 2017-02-26 18:54:02
tags:
- 面试
categories:
- java
---

### 前言

java 中Thread类的join方法用法有时会出现在面试中，但是在日常的开发中很少看到这样的用法。下面就介绍下这个方法的作用。

<!-- more -->

### Thread.join

join方法有3个重载的方法

*public final void join()*

Thread的这个方法使当前线程等待，直到被调用`join`方法的线程死亡。如果当前线程被中断，则会抛出异常`InterruptedException`。

*public final synchronized void join(long millis)*

Thread的这个join方法使当前线程等待，直到被调用`join`方法的线程死亡或当前线程等待的时间到了。由于线程的执行依赖操作系统的实现， 所以不能保证当前线程会精确的等待参数给定的时间。

*public final synchronized void join(long millis, int nanos)*

该方法的作用和上面的方法的作用相同。


下面的例子演示了Thread类的join方法的具体用法。程序的目的是使主线程最后结束，第三个线程只有在第一个线程死亡后才启动。

```java
public class ThreadJoinExample {

    public static void main(String[] args) {
        Thread t1 = new Thread(new MyRunnable(), "t1");
        Thread t2 = new Thread(new MyRunnable(), "t2");
        Thread t3 = new Thread(new MyRunnable(), "t3");
        
        t1.start();
        
        //start second thread after waiting for 2 seconds or if it's dead
        try {
            t1.join(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        
        t2.start();
        
        //start third thread only when first thread is dead
        try {
            t1.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        
        t3.start();
        
        //let all threads finish execution before finishing main thread
        try {
            t1.join();
            t2.join();
            t3.join();
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
        
        System.out.println("All threads are dead, exiting main thread");
    }

}

class MyRunnable implements Runnable{

    @Override
    public void run() {
        System.out.println("Thread started:::"+Thread.currentThread().getName());
        try {
            Thread.sleep(4000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Thread ended:::"+Thread.currentThread().getName());
    }   
}
```

输出：

```
Thread started:::t1
Thread ended:::t1
Thread started:::t2
Thread started:::t3
Thread ended:::t2
Thread ended:::t3
All threads are dead, exiting main thread
```



