---
layout: post
comments: true
title: java队列Queue接口详解
date: 2016-10-15 21:12:40
tags:
categories:
- java
---

### Queue接口简介

java中的队列接口Queue是一个被设计用来按一定的优先级处理元素的集合，除了拥有基本的集合操作和数据结构队列的特性外它还添加了额外的元素插入，获取，检查操作。所有的这些额外添加的操作都有一个共性，如果操作失败，一种情况是：抛出异常；一种情况是：返回特殊的值(根据情况可能是null或false)。返回特殊值这种方式是针对有容量队列的插入操作。在大多数队列的实现中插入操作是不能失败的。

<!-- more -->

### 接口声明

```java
public interface Queue<E> extends Collection<E> {
    //....
}
```

从声明来看`Queue`首先是一个集合，也就是说具有基本的集合操作功能。单同时它还是一个队列。下面就看下它具体都有哪些方法定义。


### 方法解析

#### boolean add(E e);
    
add 方法将指定的元素添加入队列，如果队列是有界的且没有空闲空间则抛出异常。

#### boolean offer(E e);

offer 方法将指定的元素添加入队列，如果队列是有界的且没有空闲空间则返回false。在使用有界队列时推荐使用该方法来替代add方法。

#### E remove();

remove删除并返回队首的元素。如果队列为空则会抛异常。

####  E poll();

poll删除并返回队首的元素。如果队列为空返回null。

#### E element();

element返回队首元素但是不删除。如果队列为空会抛出异常。

#### E peek();
    
element返回队首元素但是不删除。如果队列为空则返回null。

### 总结

在JDK里面没有一个队列的实现是仅仅实现Queue接口定义的功能。可能是因为具有基本功能的队列实现比较简单而且实际的用途有点少。队列的实现基本可以分为：1.并发队列(ConcurrentLinkedQueue); 2.阻塞队列（ArrayBlockingQueue， LinkedBlockingQueue）； 3.双端队列（Deque, ArrayDeque, LinkedList, ConcurrentLinkedDeque）; 4:优先级队列(PriorityQueue, PriorityBlockingQueue) 每种类型的队列都是针对不同的应用场景的，所以还是需要仔细区分来选择合适的队列实现。



