---
layout: post
comments: true
title: java中HashMap的最大容量解析
date: 2016-11-22 17:08:05
tags:
- java
categories:
- java
---

> 这是一个面试题引起的。面试官问：Java中HashMap最多能存储多少个键值对？记得当时的回答是Integer.Max_VALUE.这是由于记得`size()`方法返回的值是int。那HashMap中究竟能存储多少个键值对呢？我们来一探究竟。

** 以下代码基于JDK7**

<!-- more -->

### MAXIMUM_CAPACITY

       /**
        * The maximum capacity, used if a higher value is implicitly specified
        * by either of the constructors with arguments.
        * MUST be a power of two <= 1<<30.
        */
       static final int MAXIMUM_CAPACITY = 1 << 30;

上面的代码是HashMap中的代码片段，说的很明确， HashMap的最大容量是`MAXIMUM_CAPACITY`.
这个容量指的是桶数组的大小。

### reSize()

    void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

        Entry[] newTable = new Entry[newCapacity];
        transfer(newTable, initHashSeedAsNeeded(newCapacity));
        table = newTable;
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }

从上面的代码可以看出如果桶数组的大小已经等于`MAXIMUM_CAPACITY`（`oldCapacity == MAXIMUM_CAPACITY`），则HashMap不会再进行扩容了。

### 结论
现在想想这个面试题问的不严谨，也可以说问题和答案没有关系。就算我们的内存足够大，桶数组的大小已经达到了`MAXIMUM_CAPACITY`，此时HashMap只是不再进行扩容了，新的Key-Value对还是可以添加到HashMap中，毕竟它是通过链表来解决Hash冲突的。



