---
layout: post
title: java Reference 详解
categories:
- java
- jdk
tags:
- java
---

### 概述

> Java中一共有四种引用类型, 强引用(StrongReference), 软引用(SoftReference), 弱引用(WeakReference), 虚引用(PhantomReference); 每个引用对象内部都有一个对应的引用对象`Referent`和一个`ReferenceQueue`. 这四个引用类类型的父类是`Reference`. Reference 是一个抽象类,它定义了所有引用对象共同的操作方法,因为引用对象和垃圾回收操作关系密切,因此一般不需要直接实现该类.

<!-- more -->

### 名词解释

`Referent`: 被引用对象

`RefernceQueue`: 当引用的Referent被回收后该引用会被enqueue到这个ReferenceQueue中

一个对象可以同时拥有多种引用, 可以通过`Reference.get()`方法获取Referent

### StrongReference (强引用)

强引用时Java中使用得最多的引用, 我们使用的普通引用就是Java强引用例如:

    Object obj = new Object();

`obj`就是一个强引用, 强引用不会被JVM GC(前提是该对象是GC ROOTS 节点可达的), 即使内存不够抛出`OutOfMemoryError`异常也不会被回收.

### SoftReference (软引用)

软引用是Java中一个类, 它的`Referent`只有在内存不够的时候在抛出`OutOfMemoryError`前会被 JVM GC, 软引用一般用来实现内存敏感缓存(memory-sensitive caches)

软引用可以和一个ReferenceQueue一起使用, 当SoftReference的Referent被回收以后,这个SoftReference会被自动enqueue到这个queue中,如下: 

```java
import java.lang.ref.ReferenceQueue;
import java.lang.ref.SoftReference;

/**
 * -Xms20M -Xmx20M -Xmn5M
 * Created with IntelliJ IDEA.
 * User: jiexiu
 * Date: 16/11/15
 * Time: 下午6:49
 */
public class TestReference {

    private static int K = 1024;

    private static int M = 1024 * K;

    public static void main(String[] args) {
        byte[] a = new byte[10 * M];
        byte[] b = new byte[2 * M];
        ReferenceQueue queue = new ReferenceQueue();
        SoftReference softReference = new SoftReference(b, queue);
        System.out.println(softReference);
        b = null;
        try {
            byte[] c = new byte[5 * M];
        }catch (Error error){
            System.out.println(softReference.get());
            System.out.println(queue.poll());
        }
    }
}
```   

输出:

    java.lang.ref.SoftReference@610455d6
    null
    java.lang.ref.SoftReference@610455d6
 
代码`byte[] c = new byte[5 * M];` 会导致内存溢出, 但是在溢出前会进行GC, 所以`softReference.get()`会返回`null`(被软引用对象引用的`b`被回收了), 并且应用对象`softReference`会被加入引用队列.
 
### WeakReference (弱引用)

弱引用相较于软引用生命周期更弱, 当一个对象仅有一个弱引用引用它时，被该WeakReference引用Referent会在JVM GC执行以后被回收;同样, 弱引用也可以和一个ReferenceQueue共同使用

```java
private static void testWeakReference(){
    byte[] a = new byte[10 * M];
    ReferenceQueue queue = new ReferenceQueue();
    WeakReference weakReference = new WeakReference(a, queue);
    a = null;
    System.out.println(weakReference.get());
    System.gc();
    System.out.println(weakReference.get());
}
```

**输出**

    [B@610455d6
    null
    java.lang.ref.WeakReference@511d50c0
   
根据输出可以看出, 被弱引用对象引用的对象只有发生GC, 则被引用的对象一定会被回收掉并且若引用对象会被加入引用队列.
 
### PhantomReference (虚引用)

虚引用不会影响对象的生命周期, 它仅仅是一个对象生命周期的一个标记, 他必须与ReferenceQueue一起使用, 构造方法必须传入ReferenceQueue, 因为它的作用就是在对象被JVM决定需要GC后, 将自己enqueue到ReferenceQueue中. 它通常用来在一个对象被GC前作为一个GC的标志,以此来做一些finalize操作,另外,PhantomReference.get()方法永远返回null;

PhantomReference  有两个好处:

1. 它可以让我们准确地知道对象何时被从内存中删除， 这个特性可以被用于一些特殊的需求中(例如 Distributed GC，  XWork 和 google-guice 中也使用 PhantomReference 做了一些清理性工作).  

2. 它可以避免 finalization 带来的一些根本性问题, 上文提到 PhantomReference 的唯一作用就是跟踪 referent 何时GC,  但是 WeakReference 也有对应的功能, 两者的区别到底在哪呢 ? 这就要说到 Object 的 finalize 方法, 此方法将在 gc 执行前被调用, 如果某个对象重载了 finalize 方法并故意在方法内创建本身的强引用,  这将导致这一轮的 GC 无法回收这个对象并有可能引起任意次 GC， 最后的结果就是明明 JVM 内有很多 Garbage 却 OutOfMemory， 使用 PhantomReference 就可以避免这个问题， 因为 PhantomReference 是在 finalize 方法执行后回收的，也就意味着此时已经不可能拿到原来的引用,  也就不会出现上述问题,  当然这是一个很极端的例子, 一般不会出现.  

### ReferenceQueue (引用队列)

如果Reference在构造方法加入ReferenceQueue参数, Reference在它的Referent被GC的时,会将这个Reference加入ReferenceQueue

### WeakHashMap

WeakHashMap是HashMap的WeakReference实现, 他使用WeakReference封装了Entry的Key,  如果这个WeakHashMap的key仅有这个Map持有弱引用,则当JVM GC执行时,它的key和value会被GC. 如果这个key还有别的引用则不会被GC.

```java
WeakHashMap<Object, String> map = new WeakHashMap<Object, String>();
map.put(new Object(), "test");
System.out.println(map);
System.gc();
System.out.println(map);
```

**输出**

    {java.lang.Object@1a758cb=test}
    {}