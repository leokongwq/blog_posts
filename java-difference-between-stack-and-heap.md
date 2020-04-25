---
layout: post
comments: true
title: java中栈和堆的区别
date: 2017-02-25 11:55:57
tags:
categories:
- java
---

本文转载自：[http://javarevisited.blogspot.jp/2013/01/difference-between-stack-and-heap-java.html](http://javarevisited.blogspot.jp/2013/01/difference-between-stack-and-heap-java.html)


### 前言

在开始学习Java和其他的编程语言时，有个经常被问到的问题就是关于`栈`和`堆`的区别。栈和堆是程序员开始编程时最早听到的两个词，但是关于这两者并没有清晰和明确的解释。在Java中缺少对什么是堆，什么是栈的认识会导致关于两者之间错误的认知。Java提供栈这个数据结构，它以LIFO（先进先出）顺序存储元素，在Java API中以`java.util.Stack`提供，这更增加了堆和栈之间迷惑性。 一般来说，栈和堆都是内存的一部分，一个程序中分配的并用于不同的目的。 Java程序在JVM上运行，它通过“java”命令进程启动。 Java也使用栈和堆内存来满足不同的需求。 

<!-- more -->

### Difference between Stack vs Heap in Java

这里有一些对Java中堆和栈的区别的说明：

1. 堆和栈的最大区别是：Java中栈是用来保存[局部变量](http://javarevisited.blogspot.com/2012/02/difference-between-instance-class-and.html) 和 方法调用信息的，然而堆是用来保存对象信息的。不论对象是在哪里创建的，作为一个成员变量，局部变量或类变量，它们都会被创建在堆上。
2. 每个[Java中的线程](http://javarevisited.blogspot.com/2011/02/how-to-implement-thread-in-java.html) 拥有它自己的栈空间，可以通过虚拟机参数`-Xss`来指定，同样的你可以通过`-Xms`和`-Xmx`来指定堆的最小值和最大值。更多JVM的选项可以参考: [10 JVM option Java programmer should know](http://javarevisited.blogspot.com/2011/11/hotspot-jvm-options-java-examples.html).
{% asset_img Difference-between-stack-heap.jpg %}    
如果栈空间没有剩余的空间来保存局部变量和方法调用的信息，则JVM会抛出`java.lang.StackOverFlowError`异常；如果没有更多的堆空间来保存创建的对象，JVM会抛出`java.lang.OutOfMemoryError`异常。
4. 如果你使用递归，则你可以非常快的占满栈空间。 栈和堆的另一个区别是：栈空间的大小，栈空间的大小远小于堆的大小。
5. 保存在栈空间的变量只有拥有该栈空间的线程能看到，然而保存在堆空间的对象是所有线程都可见的。换句话说，栈内存空间是Java线程的另一种类型的私有空间，但是堆空间是所有线程共享的。
    
这就是Java中Stack和Heap内存的区别。 正如我所说，重要的是要了解在Java中什么是堆和什么是栈，哪种类型的变量保存在哪里。





