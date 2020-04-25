---
layout: post
comments: true
title: java中notify和notifyall的区别
date: 2017-02-22 14:10:31
tags:
- 面试
categories:
- java
---

### 背景

曾经被问到Java中`nofify`和`notifyAll`的区别，我印象中是notify只会唤醒所有等待该对象监视器锁中的一个线程，具体唤醒哪一个线程是由具体的虚拟机实现的。notifyAll会唤醒所有的等待线程。听到答案的人表示不太满意。后来就查看JDK的文档说明看看究竟区别在哪里？文档基于oracle官方JDK8版本

<!-- more -->

### notify

JDK官方方法说明

```
Wakes up a single thread that is waiting on this object's monitor. If any threads are waiting on this object, one of them is chosen to be awakened. The choice is arbitrary and occurs at the discretion of the implementation. A thread waits on an object's monitor by calling one of the {@code wait} methods.

The awakened thread will not be able to proceed until the current thread relinquishes the lock on this object. The awakened thread will compete in the usual manner with any other threads that might be actively competing to synchronize on this object; for example, the awakened thread enjoys no reliable privilege or disadvantage in being the next thread to lock this object.

This method should only be called by a thread that is the owner of this object's monitor. A thread becomes the owner of the object's monitor in one of three ways:

1.By executing a synchronized instance method of that object.
2.By executing the body of a {@code synchronized} statement that synchronizes on the object.
3.For objects of type {@code Class,} by executing a synchronized static method of that class.
```

翻一下： 唤醒一个等待在该对象监视器上的线程。如果有多个线程等待在该对象上，则选择其中的一个来唤醒。该选择是随机的并且由具体的JVM实现决定。一个线程通过调用该对象的`wait`方法来在该对象的监视器上进行等待的。

被唤醒的线程不能马上运行，直到当前线程释放了该对象的锁。被唤醒的线程以通常的方式来和其它正在想要获取该对象上锁的线程来竞争锁，并和其它线程处于同等地位，并没有什么特殊的地方。

该方法必须在一个已经获取该对象监视器的线程中调用。一个线程要成为该对象上监视器的拥有者可以通过以下三种方法：

1. 调用该对象上的同步方法
2. 执行使用该对象作为锁对象的同步代码块
3. 调用一个类的静态同步方法

### notifyAll

JDK官方解释

```
Wakes up all threads that are waiting on this object's monitor. A thread waits on an object's monitor by calling one of the {@code wait} methods.

The awakened threads will not be able to proceed until the current thread relinquishes the lock on this object. The awakened threads will compete in the usual manner with any other threads that might
be actively competing to synchronize on this object; for example, the awakened threads enjoy no reliable privilege or disadvantage in being the next thread to lock this object.

This method should only be called by a thread that is the owner of this object's monitor. See the {@code notify} method for a description of the ways in which a thread can become the owner of a monitor.
```

解释以下：

唤醒所有等待该对象监视器的线程。一个线程可以通过调用该对象的`wait`方法来等待该对象的监视器。被唤醒的所有线程不能马上被执行，直到当前的线程释放了该对象的锁。

被唤醒的线程以通常的方式来和其它处于活跃状态的线程共同竞争该对象的锁，且并没有什么特殊性。


### 总结

这个区别可以从两方面来说，

1. 就JDK的说明或JVM规范来说，我的回答也不算有问题。如果说有问题就是没有把重新竞争锁说出来。
2. 从应用场景来说，如果所有等待的线程在等待结束后可以执行一些有意义的动作，例如：一些线程都在等待某个任务的结算。一旦任务结束则所有等待的线程都可以执行自己的业务逻辑了。那么我们就可以使用notifyAll。另一个场景：只有一个等待的线程被唤醒后可以执行一些有意义的动作（例如：获取锁），此时就应该使用`notify`，当然你也可以使用`nofityAll`，但是没有必要。应该其它被唤醒的线程并不能够做什么。

我不知道除了这些区别还有那些神奇的东西在里面。请知道的东西留言指导，多谢。

### 参考资料

[http://javarevisited.blogspot.jp/2012/10/difference-between-notify-and-notifyall-java-example.html](http://javarevisited.blogspot.jp/2012/10/difference-between-notify-and-notifyall-java-example.html)

[http://stackoverflow.com/questions/37026/java-notify-vs-notifyall-all-over-again](http://stackoverflow.com/questions/37026/java-notify-vs-notifyall-all-over-again)

### 一个例子

```java
import java.util.logging.Level;
import java.util.logging.Logger;

/**
 * Java program to demonstrate How to use notify and notifyAll method in Java and
 * How to notify and notifyAll method notifies thread, which thread gets woke up etc.
 */
public class NotificationTest {

    private volatile boolean go = false;

    public static void main(String args[]) throws InterruptedException {
        final NotificationTest test = new NotificationTest();
      
        Runnable waitTask = new Runnable(){
      
            @Override
            public void run(){
                try {
                    test.shouldGo();
                } catch (InterruptedException ex) {
                    Logger.getLogger(NotificationTest.class.getName()).
                           log(Level.SEVERE, null, ex);
                }
                System.out.println(Thread.currentThread() + " finished Execution");
            }
        };
      
        Runnable notifyTask = new Runnable(){
      
            @Override
            public void run(){
                test.go();
                System.out.println(Thread.currentThread() + " finished Execution");
            }
        };
      
        Thread t1 = new Thread(waitTask, "WT1"); //will wait
        Thread t2 = new Thread(waitTask, "WT2"); //will wait
        Thread t3 = new Thread(waitTask, "WT3"); //will wait
        Thread t4 = new Thread(notifyTask,"NT1"); //will notify
      
        //starting all waiting thread
        t1.start();
        t2.start();
        t3.start();
      
        //pause to ensure all waiting thread started successfully
        Thread.sleep(200);
      
        //starting notifying thread
        t4.start();
      
    }
    /*
     * wait and notify can only be called from synchronized method or bock
     */
    private synchronized void shouldGo() throws InterruptedException {
        while(go != true){
            System.out.println(Thread.currentThread()
                         + " is going to wait on this object");
            wait(); //release lock and reacquires on wakeup
            System.out.println(Thread.currentThread() + " is woken up");
        }
        go = false; //resetting condition
    }
  
    /*
     * both shouldGo() and go() are locked on current object referenced by "this" keyword
     */
    private synchronized void go() {
        while (go == false){
            System.out.println(Thread.currentThread()
            + " is going to notify all or one thread waiting on this object");

            go = true; //making condition true for waiting thread
            //notify(); // only one out of three waiting thread WT1, WT2,WT3 will woke up
            notifyAll(); // all waiting thread  WT1, WT2,WT3 will woke up
        }
      
    }
  
}
```

输出：

```java
Output in case of using notify
Thread[WT1,5,main] is going to wait on this object
Thread[WT3,5,main] is going to wait on this object
Thread[WT2,5,main] is going to wait on this object
Thread[NT1,5,main] is going to notify all or one thread waiting on this object
Thread[WT1,5,main] is woken up
Thread[NT1,5,main] finished Execution
Thread[WT1,5,main] finished Execution

Output in case of calling notifyAll
Thread[WT1,5,main] is going to wait on this object
Thread[WT3,5,main] is going to wait on this object
Thread[WT2,5,main] is going to wait on this object
Thread[NT1,5,main] is going to notify all or one thread waiting on this object
Thread[WT2,5,main] is woken up
Thread[NT1,5,main] finished Execution
Thread[WT3,5,main] is woken up
Thread[WT3,5,main] is going to wait on this object
Thread[WT2,5,main] finished Execution
Thread[WT1,5,main] is woken up
Thread[WT1,5,main] is going to wait on this object
```

我强烈建议运行该代码和理解它的输出，并尝试理解它的逻辑。理论应该补充实际的例子，如果你看到任何不一致，请让我们知道或试图纠正它。 除了[死锁](http://javarevisited.blogspot.sg/2010/10/what-is-deadlock-in-java-how-to-fix-it.html)，[竞争条件](http://javarevisited.blogspot.sg/2012/02/what-is-race-condition-in.html)和[线程安全](http://javarevisited.blogspot.sg/2012/01/how-to-write-thread-safe-code-in-java.html)之外，线程间通信是Java中并发编程的基础之一。

### 总结

总之，这里是我们在本教程开始时提出关于notify和notify的问题的答案

#### Which thread will be notified if I use notify()?

没有保证,  线程调度器会随机从等待该监视器的池子中挑选一个线程。可以保证的是只有一个线程会被通知。

#### How do I know how many threads are waiting, so that I can use notifyAll() ?

这取决你的应用程序逻辑。在编码时你需要思考“一个代码片段是否可以被多个线程执行”。Java中一个理解线程内通信非常好的例子是生存者-消费者模式。

#### How to call notify()?

`Wait()`和`notify()`方法只能在同步方法或同步块内调用。你应该在其它线程正在等待监视器的对象上调用`notify`方法。

#### What are these thread waiting for being notified etc?

线程等待的时某些条件。在生产者-消费者模式中，如果共享的队列满了的时候生产者会等待，如果队列为空时消费者将等待。由于多个线程在一个共享资源上工作，因此它们通过`wait`和`notify`方法进行通信。




