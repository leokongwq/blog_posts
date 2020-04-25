---
layout: post
comments: true
title: java线程必备知识
date: 2016-10-16 09:21:46
tags:
- java
categories:
- java
---

> 在Java编程中,多线程是非常重要知识点. 关于Java线程有些不可不知的知识点需要牢记,下面就介绍了这些知识点.

<!-- more -->

### 1.优先级
每个Java线程都有对应的优先级，高优先级的线程比低优先级的线下有更多的执行机会。新创建的Java线程默认的优先级是5, 最低是1，最高是10；

```java
/**
 * The minimum priority that a thread can have.
 */
public final static int MIN_PRIORITY = 1;
/**
 * The default priority that is assigned to a thread.
 */
public final static int NORM_PRIORITY = 5;
/**
 * The maximum priority that a thread can have.
 */
public final static int MAX_PRIORITY = 10;
```

**Note**：如果在一个线程中创建一个新的线程，新线程的默认优先级和所在线程的优先级保持一致。

### 2. daemon 或 非daemon

一个Java线程可以是daemon的也可以是非daemon的，JVM在启动的时候启动了main线程，该线程是非daemon；JVM在下面2中情况下会退回：

1. 调用`Runtime.exit()`方法
2. 所有非daemon的线程都结束了。

### 3.创建线程的方式

有2种创建线程的方式，一种是继承`Thread`类；一种是直接new一个Thread对象，并在构造函数中传入一个Runnable对象。

**继承**
```golang
class PrimeThread extends Thread {
    long minPrime;
    PrimeThread(long minPrime) {
        this.minPrime = minPrime;
    }

    public void run() {
        // compute primes larger than minPrime
    }
}
PrimeThread p = new PrimeThread(143);
p.start();

//new
class PrimeRun implements Runnable {
    long minPrime;
    PrimeRun(long minPrime) {
          this.minPrime = minPrime;
    }

    public void run() {
          // compute primes larger than minPrime
          &nbsp;.&nbsp;.&nbsp;.
    }
}
PrimeRun p = new PrimeRun(143);
new Thread(p).start();
```

### 4.线程的状态

JDK中关于线程状态的定义：

- new : 一个新创建的，没有调用start方法的线程处于该状态；
- RUNNABLE ： 可执行的，在JVM中是处于执行中，但是在等待其它的操作系统资源，如cpu资源；
- BLOCKED ： 阻塞的，一个线程在进入（或再次进入）一个同步方法或同步块时等待监视器锁时处于该状态。
- WAITING ： 等待，一个线程在调用了如下的方法时会处于等待状态：
1.Object.wait 
2.Thread.join //正在主线程中调用一个一个线程的join方法，在主线程处于等待状态
3.LockSupport#park() 
- TIMED_WAITING 等待的，不过该等待状态是有时间限制的。一个线程调用下面的方法会进入该状态：
1.Thread.sleep
2.Object.wait
3.Thread.join
4.LockSupport.parkNanos
5.LockSupport.parkUntil
- TERMINATED 终止，当一个线程执行完成后处于该状态

### 5.线程的调度

理想的情况下，每个程序的线程都拥有一个专属于自己的处理器；在计算机还不能拥有几千，甚至几百万CPU处理器的情况下，多个线下需要共享仅有的cpu资源，如果分配线程执行所需的cpu资源就需要线程调度器的调度了。在操作系统层面有自己的线程调度器，在JVM中也存在Java线程调度器。

在Java线程调度中有2点比较重要：

1. Java规范并没有强制要求每个JVM按照特定的调度规则调用线程，或者必须包含一个线程调度器。线程调度的实现完全是依赖平台的。
2. 在编写Java多线程代码时，我们唯一需要考虑的是不要让一个线程大量的占用cpu时间(eg.死循环)。

大多数平台上JVM的线程调度是依赖操作系统本身的线程调度器的，每个线下有不同的优先级，在基于时间片的规则下，高优先级的线程拥有更多的CPU执行机会；相同优先级的线程可以按照FIFO调度。


       
                
                
                