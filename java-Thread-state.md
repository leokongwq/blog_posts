---
layout: post
comments: true
title: java线程生命周期和状态
date: 2017-02-26 19:14:52
tags:
- 面试
categories:
- java
---

本文翻译自[http://www.journaldev.com/1044/thread-life-cycle-in-java-thread-states-in-java](http://www.journaldev.com/1044/thread-life-cycle-in-java-thread-states-in-java)

### 前言

如果你在工作中会用到线程或者你需要编写多线程程序，那么理解Java线程的生命周期和线程状态是非常重要的。

<!-- more -->

### Java中线程的生命周期

下图展示了线程生命周期中的不同状态。我们可以创建一个线程并启动它，但是它如何从`Runnable`变为`Running`，从`Running`变为`Blocked` 这取决于操作系统的线程调度器的实现，Java本身没有足够的控制力。

{% asset_img Thread-Lifecycle-States.png %}

另一个图：

{% asset_img XtQm4.png %}
### New

当我们通过`new`关键字创建了一个线程对象后，该线程的状态就是`New`状态。这个状态是Java自己的内部状态。

### Runnable

当我们调用了线程的`start()`方法后，线程的状态就是`runnable`状态。线程的控制权就交由操作系统的线程调度器来管理。

### Running

当一个线程正在使用CPU，则它的状态就是`running`。线程调度器从处于`runnable`状态的线程池中选择一个线程并将它的状态改为`running`。然后CPU就开始执行该线程。一个线程由`running`状态变为 Runnable, Dead 或 Blocked 是由于时间片，线程执行完毕或等待某种资源。

### Blocked/Waiting

一个线程可能因为等待其它线程制执行完毕而等待，或因为等待某种资源可用而等待。例如生产者-消费者例子，或线程等待-通知机制，或因为等待IO完成。一旦线程结束等待则又变为`runnable`状态。

### Dead

一旦线程完成执行，它的状态就变为`dead`了。




