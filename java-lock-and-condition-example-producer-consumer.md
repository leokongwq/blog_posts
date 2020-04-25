---
layout: post
comments: true
title: java中使用锁和条件对象实现生产者-消费者模式
date: 2017-02-24 23:22:00
tags:
- 面试
categories:
- java
---

### 背景

你也可以通过使用新的`Lock`接口和条件变量，而不是使用`synchronized`关键字和`wait`和`notify`方法解决生产者消费者问题。 Lock提供了一种在Java中实现互斥和同步的替代方法。锁相对synchronized关键字的优点是众所周知的，显式锁定比synchronized关键字更加细粒度和强大，例如，锁的范围可以从一个方法到另一个方法，但是synchronized关键字的范围不能超过一个方法。条件变量是`java.util.concurrent.locks.Condition`类的实例，它提供类似于wait，notify和notifyAll的线程间通信方法：await()，signal()和signalAll()。因此，如果一个线程通过调用condition.await()来等待条件，那么一旦条件改变，第二个线程可以调用condition.signal()或condition.signalAll()方法来通知等待线程该醒醒了，条件已改变。如果你习惯于使用synchronized关键字进行锁定，使用Lock将使你感到痛苦，因为现在获取和释放锁变成开发人员的责任。无论如何，你可以按照这里显示的代码惯例使用锁，以避免任何并发问题。在本文中，您将学习如何通过解决传统的Producer消费者问题在Java中使用Lock和Condition变量。为了深刻理解这些新的并发概念，我还建议看看[Java7并发编程手册]()，它是Java并发编程中最好的书其中的一本，有一些非常好的例子。

<!-- more -->

### 在Java中如何使用Lock和Condition

当你在Java中使用Lock类时，你需要小心谨慎。 不像synchronized关键字会自动获取和释放锁，这里你需要调用`lock()`方法获取锁和`unlock()`方法释放锁，失败会导致死锁，活锁或任何其他多线程问题 。 为了使开发人员的生活更轻松，Java设计人员建议以下习惯使用锁：

```java
Lock theLock = ...; 
theLock.lock(); 
try { 
    // access the resource protected by this lock 
} finally { 
    theLock.unlock(); 
}
```

虽然这个使用范式看起来很容易，但Java开发人员通常会在编码时犯错误。 他们忘记在finally块内释放锁。因此必须记住的时：不论try块中是否抛出异常，一定要在`finally`块中释放锁。

### Java中如何创建Lock和Condition？

由于Lock是一个接口，你不能创建Lock类的对象，但不要担心，Java提供了两个实现Lock接口`ReentrantLock`和`ReentrantReadWriteLock`的类。 在Java中你可以创建任何此类的对象以使用Lock。然而，这两种锁的使用方式是不同的，因为`ReentrantReadWriteLock`包含两个锁，读锁和写锁。 在本例中，您将学习如何在Java中使用ReentrantLock类。 一旦你有了一个锁对象，你可以调用`lock()`方法获取锁和`unlock()`方法释放锁。 确保你遵循上面显示的范式：在finally子句中释放锁。

为了等待显式锁，你需要创建一个条件变量，一个`java.util.concurrent.locks.Condition`类的实例。 你可以通过调用`lock.newCondtion()`方法创建条件变量。 这个类提供了等待条件并通知等待线程的方法，类似于Object类的`wait`和`notify`方法。 这里不使用`wait()`来等待条件，而是调用`await()`方法。 类似地，为了在一个条件上通知等待线程，不是调用`notify()`和`notifyAll()`，你应该使用`signal()`和`signalAll()`方法。 它更好的做法是使用`signalAll()`来通知所有正在等待某些条件的线程，类似于使用`notifyAll()`而不是`notify()`。

就像大多数的等待方法一样，你会看到三个版本的`await()`方法，首先是`await()`，它使当前线程等待，直到收到信号或中断，`awaitNanos(long timeout)`，等待通知或超时和`awaitUnInterruptibly()` 使线程等待直到收到通知信号。 如果使用这种等待方法，你不能中断等待的线程。 这里是使用Java Lock接口的示例代码：

```java
    Lock lock = new ReentrantLock();
    
    public void accessProtectedResource(){
        lock.lock();
        try {
            //do something                             
        } finally {
            lock.unLock();
        } 
    }
```

### 使用Lock和Condition解决生产者-消费者问题

这里是针对经典的Producer和Consumer问题在Java中的解决方案，这次我们使用Lock和Condition变量来解决这个问题。 如果你还记得以前，我使用`wait`，`notify`和新的并发队列类`BlockingQueue`来解决生产者消费者问题。 就困难程度而言，第一个使用`wait`和`notify`是最难得正确解决该问题的。相对而言使用`BlockingQueue`就很容易了。 利用Java 5 Lock接口和条件变量的这个解决方案困难程度在二者之间。

#### 解决方案

在这个程序中，我们有四个类，ProducerConsumerSolutionUsingLock只是一个包装类来测试我们的解决方案。 此类创建ProducerConsumerImpl的对象和生产者和消费者线程，这是其他两个类扩展到Thread在此解决方案中作为生产者和消费者。

```java
import java.util.LinkedList;
import java.util.Queue;
import java.util.Random;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Java Program to demonstrate how to use Lock and Condition variable in Java by
 * solving Producer consumer problem. Locks are more flexible way to provide
 * mutual exclusion and synchronization in Java, a powerful alternative of
 * synchronized keyword.
 * 
 * @author Javin Paul
 */
public class ProducerConsumerSolutionUsingLock {

    public static void main(String[] args) {

        // Object on which producer and consumer thread will operate
        ProducerConsumerImpl sharedObject = new ProducerConsumerImpl();

        // creating producer and consumer threads
        Producer p = new Producer(sharedObject);
        Consumer c = new Consumer(sharedObject);

        // starting producer and consumer threads
        p.start();
        c.start();
    }

}

class ProducerConsumerImpl {
    // producer consumer problem data
    private static final int CAPACITY = 10;
    private final Queue queue = new LinkedList<>();
    private final Random theRandom = new Random();

    // lock and condition variables
    private final Lock aLock = new ReentrantLock();
    private final Condition bufferNotFull = aLock.newCondition();
    private final Condition bufferNotEmpty = aLock.newCondition();

    public void put() throws InterruptedException {
        aLock.lock();
        try {
            while (queue.size() == CAPACITY) {
                System.out.println(Thread.currentThread().getName()
                        + " : Buffer is full, waiting");
                bufferNotEmpty.await();
            }

            int number = theRandom.nextInt();
            boolean isAdded = queue.offer(number);
            if (isAdded) {
                System.out.printf("%s added %d into queue %n", Thread
                        .currentThread().getName(), number);

                // signal consumer thread that, buffer has element now
                System.out.println(Thread.currentThread().getName()
                        + " : Signalling that buffer is no more empty now");
                bufferNotFull.signalAll();
            }
        } finally {
            aLock.unlock();
        }
    }

    public void get() throws InterruptedException {
        aLock.lock();
        try {
            while (queue.size() == 0) {
                System.out.println(Thread.currentThread().getName()
                        + " : Buffer is empty, waiting");
                bufferNotFull.await();
            }

            Integer value = queue.poll();
            if (value != null) {
                System.out.printf("%s consumed %d from queue %n", Thread
                        .currentThread().getName(), value);

                // signal producer thread that, buffer may be empty now
                System.out.println(Thread.currentThread().getName()
                        + " : Signalling that buffer may be empty now");
                bufferNotEmpty.signalAll();
            }

        } finally {
            aLock.unlock();
        }
    }
}

class Producer extends Thread {
    ProducerConsumerImpl pc;

    public Producer(ProducerConsumerImpl sharedObject) {
        super("PRODUCER");
        this.pc = sharedObject;
    }

    @Override
    public void run() {
        try {
            pc.put();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

class Consumer extends Thread {
    ProducerConsumerImpl pc;

    public Consumer(ProducerConsumerImpl sharedObject) {
        super("CONSUMER");
        this.pc = sharedObject;
    }

    @Override
    public void run() {
        try {
            pc.get();
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
    }
}

输出：
CONSUMER : Buffer is empty, waiting
PRODUCER added 279133501 into queue 
PRODUCER : Signalling that buffer is no more empty now
CONSUMER consumed 279133501 from queue 
CONSUMER : Signalling that buffer may be empty now
```

在这里你可以看到CONSUMER线程已经在PRODUCER线程之前启动，发现缓冲区是空的，所以它在条件“bufferNotFull”上等待。 后来当PRODUCER线程启动时，它将一个元素添加到共享队列中，并指示所有线程（这里只有一个CONSUMER线程）等待条件bufferNotFull，此条件现在已经改变，醒来并执行你的工作。 在调用signalAll()后，我们的CONSUMER线程唤醒并再次检查条件，发现shred队列确实不再是空的，所以它可以继续执行并从队列中消耗了这个元素。

因为我们在程序中没有使用任何无限循环，所以在这个操作后，消费者线程退出run()方法，线程结束。 PRODUCER线程已经完成，所以我们的程序在这里结束。

这就是如何使用Java中的锁和条件变量来解决生产者消费者问题。 这是一个很好的例子，学习如何使用这些相对较少使用但强大的工具。 让我知道如果你有任何关于锁接口或条件变量的问题，我很高兴地回答。 如果你喜欢这个Java并发教程，并想了解其他并发实用程序，你也可以看看下面的教程。

### Java Concurrency Tutorials for Beginners

- How to use Future and FutureTask class in Java? ([solution](http://javarevisited.blogspot.com/2015/01/how-to-use-future-and-futuretask-in-Java.html))
- How to use CyclicBarrier class in Java? ([example](http://javarevisited.blogspot.com/2012/07/cyclicbarrier-example-java-5-concurrency-tutorial.html))
- How to use Callable and Future class in Java? ([example](http://javarevisited.blogspot.com/2015/06/how-to-use-callable-and-future-in-java.html))
- How to use CountDownLatch utility in Java? ([example](http://javarevisited.blogspot.com/2012/07/countdownlatch-example-in-java.html))
- How to use Semaphore class in Java? ([code sample](http://javarevisited.blogspot.com/2012/05/counting-semaphore-example-in-java-5.html))
- What is difference between CyclicBarrier and CountDownLatch in Java? ([answer](http://java67.blogspot.sg/2012/08/difference-between-countdownlatch-and-cyclicbarrier-java.html))
- How to use Lock interface in multi-threaded programming? ([code sample](http://javarevisited.blogspot.com/2014/10/how-to-use-locks-in-multi-threaded-java-program-example.html))
- How to use Thread pool Executor in Java? ([example](http://javarevisited.blogspot.com/2013/07/how-to-create-thread-pools-in-java-executors-framework-example-tutorial.html))
- 10 Multi-threading and Concurrency Best Practices for Java Programmers ([best practices](http://javarevisited.blogspot.com/2015/05/top-10-java-multithreading-and.html))
- 50 Java Thread Questions for Senior and Experienced Programmers ([questions](http://javarevisited.blogspot.com/2014/07/top-50-java-multithreading-interview-questions-answers.html))
- Top 5 Concurrent Collection classes from Java 5 and Java 6 ([read here](http://javarevisited.blogspot.com/2013/02/concurrent-collections-from-jdk-56-java-example-tutorial.html))
- how to do inter thread communication using wait and notify? ([solution](http://javarevisited.blogspot.com/2013/12/inter-thread-communication-in-java-wait-notify-example.html))
- How to use ThreadLocal variables in Java? ([example](http://javarevisited.blogspot.com/2012/05/how-to-use-threadlocal-in-java-benefits.html))





