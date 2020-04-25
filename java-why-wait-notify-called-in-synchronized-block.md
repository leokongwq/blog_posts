---
layout: post
comments: true
title: 为什么wait和notify必须在同步方法或同步块中调用
date: 2017-02-24 21:46:53
tags:
- 面试
categories:
- java
---

本文翻译自：[http://javarevisited.blogspot.jp/2011/05/wait-notify-and-notifyall-in-java.html](http://javarevisited.blogspot.jp/2011/05/wait-notify-and-notifyall-in-java.html)

### 背景

大多数Java开发者都知道`Object`类中的`wait`,`notify`和`notifyAll`方法必须在同步方法或同步块中调用，但我们又有谁思考过为什么呢？最近我一个朋友在面试中被问道了该问题。他思考了一会说：如果我们不在同步上下文中调用这些方法我们就会收到`IllegalMonitorStateException`异常。他的回答在Java语言层面是没有问题的，但是面试官对这个回答并不完全满意，并希望他能对此做出更多解释。在面试结束后，他和我讨论这个问题。我想他应该告诉面试官Java中关于wait和notify之间的`竞态条件`，如果我们不在同步方法或同步块中调用这些方法`竞态条件`也可以出现。让我们看看它如何在Java程序中产生。

<!-- more -->

它也是流行的[线程面试题](http://javarevisited.blogspot.com/2014/07/top-50-java-multithreading-interview-questions-answers.html)之一，经常在电话和面对面的Java开发人员面试中被问到。 所以，如果你准备Java面试，你应该准备这样的问题和一本书，真的可以帮助你是[Java编程面试指南](http://www.amazon.com/Java-Programming-Interviews-Exposed-Markham/dp/1118722868?tag=javamysqlanta-20)。

这是一本罕见的书，涵盖几乎所有重要的Java面试主题，例如。 核心Java，多线程，IO和NIO以及Spring和Hibernate等框架。


### Why wait(), notify() and notifyAll() must be called from synchronized block or method in Java

在Java中，我们使用`wait()`和`nofify()`或`notifyAll()`来实现线程间通信。一个线程在测试条件不满足后进入等待状态。在经典的生产者-消费者问题中，生产者线程因缓存区满而等待，消费者线程在消费了缓存区的一个元素后通知生产者线程。

调用`notify()`和`notifyAll()`方法来通知一个或多个线程一个条件已经改变了。一旦通知线程退出同步方法或同步块，所有等待的线程会争抢它们等待对象上的对象锁。获取锁的线程会从等待状态返回并继续执行。

让我们将整个操作分成几步，以查看Java中wait()和notify()方法之间的竞态条件出现的可能性，我们将使用[Produce-Consumer线程示例](http://javarevisited.blogspot.com/2015/06/java-lock-and-condition-example-producer-consumer.html)来更好地理解场景：

1. 生产者线程测试条件（缓冲已满或没有满）并确保在缓冲区满时进入等待状态
2. 消费者线程在消费了缓冲区的一个元素后设置等待条件。
3. 消费者线程调用`notify()`方法； 如果生产者线程没有处于等待状态则不会收到该通知。
4. 生产者线程调用`wait()`方法进入等待状态。

因此，由于[竞态条件](http://javarevisited.blogspot.com/2012/02/what-is-race-condition-in.html)，这里我们潜在的失去一个通知，并且如果我们使用了缓冲区或只有一个生产者线程，则该线程将永远等待，你的程序将挂起。

{% asset_img wait-notify.jpg %}

现在让我们想想这个潜在的竞态条件如何解决？ 此竞争条件通过使用Java提供的[synchronized](http://javarevisited.blogspot.com/2011/04/synchronization-in-java-synchronized.html)关键字和锁定来解决。 为了在Java中调用wait（），notify（）或notifyAll（）方法，我们必须获取对于我们调用该方法的对象的锁。

由于Java中的`wait()`方法在等待之前释放锁，并在从`wait()`方法返回之前重新获取锁，因此我们必须使用该锁来确保条件检查（缓冲区已满或未满），和条件重置（从缓冲区获取元素）是原子的，这可以通过在Java中使用synchronized方法或块来实现。

我不确定这是否是面试官实际期望的答案，但这我认为这至少是有意义的，如果我错了请更正我。如果关于该问题有任何其他令人信服的理由，请让我们知道，

总结一下，在Java中，我们为什么必须在synchronized方法或同步块中调用wait（），notify（）或notifyAll方法的原因：

1) 避免IllegalMonitorStateException 

2) 避免任何在wait和notify之间潜在的竞态条件


### 更多参考

[竞态条件与临界区](http://ifeve.com/race-conditions-and-critical-sections/)
[竞态条件](http://javarevisited.blogspot.com/2012/02/what-is-race-condition-in.html)
[Produce Consumer thread example](http://javarevisited.blogspot.com/2015/06/java-lock-and-condition-example-producer-consumer.html)


