---
layout: post
comments: true
title: java中原子变量之AtomicLong
date: 2016-12-31 13:59:26
tags:
- 并发
categories:
- java
---

### 前言

在阅读[java并发编程实践](https://item.jd.com/10922250.html)时，其中有一个章节讲到了原子变量。这是原子变量类型是在jdk1.5由大神`Doug Lea`编写的，能够对高并发操作提供良好的支持并且`java.util.concurrent`包中的需要并发工具也是基于这些原子变量实现。看过的东西过一段时间就不够清晰了，今天重新学习一下这些原子变量的内容。只以AtomicLong举例，其它的都是类似的。

<!-- more -->

### AtomicLong介绍

jdk的文档说AtomicLong是一个代表能别原子化更新的long值。AtomicLong可以在应用中作为一个序列号生成器来使用。就以这个应用场景将AtomicLong是如何实现原子化更新的。

### AtomicLong生成序列号

    AtomicLong seqNum = new AtomicLong(1);
    long id = seqNum.getAndIncrement();

这两行代码分别创建了一个初始值为1的AtomicLong的变量并调用了它的getAndIncrement获取它当前的值并自增1。

AtomicLong的getAndIncrement方法是不会有线程安全问题的。解释如下：

```java
/**
 * Atomically increments by one the current value.
 * 原子的将当前的值加一
 * @return the previous value 
 * 返回当前的值
 */
public final long getAndIncrement() {
    return unsafe.getAndAddLong(this, valueOffset, 1L);
}
```

上面代码的关键是`unsafe`变量， 这个变量是干嘛的呢？

### Unsafe

```java
// setup to use Unsafe.compareAndSwapLong for updates
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;
```    

简单来说Unsafe就是一个不安全的类，它包含了许多可以执行底层操作的不安全方法，即使说该类和它的方法都是public的。使用该类是受限的，只有受信任的代码才可以使用它。想下面这样使用就会报异常：

```java
Unsafe unsafe = Unsafe.getUnsafe();
try {
    final long valueOffset = unsafe.objectFieldOffset(AtomicLong.class.getDeclaredField("value"));
} catch (NoSuchFieldException e) {
    e.printStackTrace();
}
```

解决的办法有2个：

1. 让虚拟机信任我们的代码
    
        java -Xbootclasspath:/usr/jdk1.7.0/jre/lib/rt.jar:. com.mishadoff.magic.UnsafeClient
        
2. 使用反射

        Field f = Unsafe.class.getDeclaredField("theUnsafe");
        f.setAccessible(true);
        Unsafe unsafe = (Unsafe) f.get(null);         

Unsafe类的详细文档说明可以参考[http://www.docjar.com/docs/api/sun/misc/Unsafe.html](http://www.docjar.com/docs/api/sun/misc/Unsafe.html)

### unsafe.getAndAddLong

在AtomicLong的方法中用到了`unsafe.getAndAddLong`, 该方法最终会调用

```
 /**
* Atomically update Java variable to <tt>x</tt> if it is currently
* holding <tt>expected</tt>.
* @return <tt>true</tt> if successful
*/
public final native boolean compareAndSwapLong(Object o, long offset,
                                              long expected,
                                              long x);
```

可以看到这是一个native方法，更底层是通过CPU支持的类似的指令来实现的。所以比我们一般使用的锁效率要高很多。


### 参考

[java秘密-Unsafe](http://mishadoff.com/blog/java-magic-part-4-sun-dot-misc-dot-unsafe/)

[理解Java中unsafe](http://www.javaworld.com/article/2952869/java-platform/understanding-sun-misc-unsafe.html)



                                              











