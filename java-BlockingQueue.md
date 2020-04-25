---
layout: post
comments: true
title: Java的BlockingQueue详解
date: 2016-10-16 09:08:16
tags:
- java
categories:
- java
---

### Java的BlockingQueue详解

### 基础概念

> Java的BlockingQueue本质上还是一个Queue（队列），它其实是在队列的基础功能上添加了如下功能：
1.队列为空时获取数据导致当前线程等待
2.队列没有多余空间时，导致添加元素到队列的线程等待
BlockingQueue提供了4中不同的方式供用户获取队列元素或添加元素到队列，这4种类型的方法导致4种不同的行为：1：抛异常；2：返回特殊值，false或null；3：阻塞当前线程；4：超时阻塞

<!-- more -->

### 主要方法解析

|方法名 | 作用 | 是否阻塞 | 行为 |
|---   | ----|---|---|
| add  | 添加元素到队列 | 否 | 没有多余空间会抛异常。成功添加返回false |
| put  |添加元素到队列 | 是| 一直等待，直到有可以的空间 |
| offer|添加元素到队列 | 否 | 如果当前有空间就添加成功，否则返回false |
| offer(超时) | 添加元素到队列 | 否 | 如果当前没有可用空间，则等待指定的时间。超时返回false |
|take | 获取并删除队列队首元素 | 是 | 如果当前队列为空，则一直等待，直到队列非空 |
| poll | 获取并删除队列队首元素 | 是| 如果当前队列为空，则等待指定的时间，超时返回null|

### 使用场景

 1. 生产者&消费者
 2. 线程间通信
 3. 线程池

### 主要子类
ArrayBlockingQueue, LinkedBlockingQueue 和 SynchronousQueue。LinkedBlockingQueue和ArrayBlockingQueue不同点在于，ArrayBlockingQueue是基于数组的，LinkedBlockingQueue是基于链表的。

SynchronousQueue是一个非常特殊的BlockingQueue。它的内部没有保存元素的缓存区，一个元素也不保存。你不能通过`peek`方法查看同步队列中的元素，因为只有在你删除一个元素时该元素才会出现。你也不能插入一个元素到队列，除非有另一个线程正在等待删除该元素。你也不能迭代该队列，因为队列本身不保存任何元素。队列头元素是第一个排队要插入数据的线程，而不是要交换的数据。数据是在配对的生产者和消费者线程之间直接传递的，并不会将数据缓冲数据到队列中。该队列中包含一个对象在线程间的安全发布过程。

详细信息参考：[synchronousqueue](http://ifeve.com/java-synchronousqueue/) 和JDK文档。

### 注意事项

 - 队列长度
    使用阻塞队列时，一定需要指定队列的长度（可以使用固定长度的ArrayBlockQueue）；否则当消费线程速度慢时，会导致队列快速膨胀，最终内存溢出。
 - 超时机制
    请尽量使用带有超时功能的阻塞方法； 如果没有超时机制，则所有添加元素的线程都会阻塞（消费者线程死掉或速度太慢）或者消费者线程全部阻塞等待（生产者线程太慢或死掉）；                        
                    
                    
                    


