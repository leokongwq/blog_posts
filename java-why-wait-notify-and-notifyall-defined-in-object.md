---
layout: post
comments: true
title: java-为什么wait，notify和notifyall定义在Object中
date: 2017-02-22 15:58:47
tags:
- 面试
categories:
- java
---

本文翻译自：[http://javarevisited.blogspot.jp/2012/02/why-wait-notify-and-notifyall-is.html](http://javarevisited.blogspot.jp/2012/02/why-wait-notify-and-notifyall-is.html)

### 背景

为什么`wait`,`notify`,`notifyAll`声明在类`Object`中而不是在`Thread`中这个问题是一个非常有名的java核心面试题，在各种级别的Java开发者面试中都会出现（2年，4年，甚至资深的工程师）。

<!-- more -->

这个问题的魅力在于，它反映了面试者是否知道等待通知机制，他如何看待整个等待和通知功能，以及他的理解是否不浅于这个主题。 像[为什么在Java中不支持多重继承](http://javarevisited.blogspot.com/2011/07/why-multiple-inheritances-are-not.html)和[为什么Java中String是final类型的](http://javarevisited.blogspot.com/2010/10/why-string-is-immutable-in-java.html)问题一样，这些问题都有多个答案并且每一个都有可以证明的理由。

在我的所有面试经验中，我发现`wait`和`notify`仍然是大多数Java程序员特别是2到3年最困惑的，如果要求他们使用`wait`和`notify`编写代码，他们经常会挣扎者难以实现。 所以，如果你要去参加任何Java面试，确保你对`wait`和`notify`机制有清晰认识，并且你能很容易的使用`wait`和`notify`机制编程代码来实现像生产消费者问题或实现阻塞队列等。本文是继续 的我早先的文章有关等待和通知例如 为什么Wait和notify需要从同步块或方法中调用，Java中的等待，睡眠和yield方法之间的区别，如果你还没有阅读你可能会发现有趣。

顺便说一下这篇文章是我的早期文章的延续，有关等待和通知。 [为什么Wait和notify需要从同步块或方法中调用](http://javarevisited.blogspot.com/2011/05/wait-notify-and-notifyall-in-java.html)，[Java中的等待，睡眠和yield方法之间的区别](http://javarevisited.blogspot.com/2011/12/difference-between-wait-sleep-yield.html)，如果你还没有阅读你可能会发现有趣。

### 答案

`为什么它们不应该在Thread类`这里有一些想法，对我来说是有意义的。

1. `wait`和`nofity`不是常见的普通java方法或同步工具，在Java中它们更多的是实现两个线程之间的通信机制。 如果不能通过类似`synchronized`这样的Java关键字来实现这种机制，那么`Object`类中就是定义它们最好的地方，以此来使任何Java对象都可以拥有实现线程通信机制的能力。记住`synchronized`和`wait`,`notify`是两个不同的问题域，并且不要混淆它们的相似或相关性。 同步类似竞态条件，是提供线程间互斥和确保Java类的线程安全性的，而`wait`和`notify`是两个线程之间的通信机制。
2. 每个对象都可以作为锁，这是另一个原因`wait`和`notify`在Object类中声明，而不是Thread类。
3. 在Java中，为了进入临界区代码段，线程需要获得锁并且它们等待锁可用，它们不知道哪些线程持有锁而它们只知道锁是由某个线程保持，它们应该等待锁而不是知道哪个线程在同步块内并要求它们释放锁。 这个比喻适合等待和通知在`object`类而不是Java中的线程。

这些只是我的想法`为什么wait和notify方法在Object类中声明`，而不是Java中的Thread，当然你可以有不同的观点。 在现实中，它就是Java不支持操作符重载一样，只是Java设计者做的一个设计决定。 无论如何，如果你有任何其它令人信服的理由请发布出来。

### 补充

"Java is based on Hoare's monitors idea (http://en.wikipedia.org/wiki/Monitor_%28synchronization%29). In Java all object has a monitor. Threads waits on monitors so, to perform a wait, we need 2 parameters:

- a Thread
- a monitor (any object)

In the Java design, the thread can not be specified, it is always the current thread running the code. However, we can specify the monitor (which is the object we call wait on). This is a good design, because if we could make any other thread to wait on a desired monitor, this would lead to an "intrusion", posing difficulties on designing/programming concurrent programs. Remember that in Java all operations that are intrusive in another thread's execution are deprecated (e.g. stop())."




