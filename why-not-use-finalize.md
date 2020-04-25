---
layout: post
title: 为什么不使用finalize
date: 2015-06-10
categories:
 - java
tags:
- jdk
---

### 为什么不使用finalize

我的上一篇博客：System.gc()和-XX:+DisableExplicitGC启动参数，以及DirectByteBuffer的内存释放  在讨论如何回收堆外内存的时候，提到“NIO中direct memory的释放并不是通过finalize()，因为finalize不安全而且影响能”。Effective Java一书中也提到：Avoid Finalizers。人都有潜在的叛逆意识，别人给的结论或者制定的规范，除非有足够的理由说服你，除非懂得这么做背后的原因，否则只能是死记硬背，没有形象深入的理解，不能学到真正的东西。本文通过自己的理解和一些实际的例子，和大家一起更形象的理解finalize。还是那句经典的话“talking is cheap,show me the code”。

<!-- more -->

我们先看下TestObjectHasFinalize这个类提供了finalize方法

```java
package finalize;  
  
public class TestObjectHasFinalize  
{  
    public static void main(String[] args)  
    {  
        while (true)  
        {  
            TestObjectHasFinalize heap = new TestObjectHasFinalize();  
            System.out.println("memory address=" + heap);  <
        }  
    }  
  
    @Override  
    protected void finalize() throws Throwable  
    {  
        super.finalize();  
        System.out.println("finalize.");  
    }  
}  
```

运行这段程序，使用jmap命令查看对象占用的内存情况。

    C:\Documents and Settings\Administrator>jps  
    4232 Jps  
    3236 TestObjectHasFinalize  
    5272  
  
    C:\Documents and Settings\Administrator>jmap -histo:live 3236  
      
     num     #instances         #bytes  class name  
    ----------------------------------------------  
       1:        106983        3423456  java.lang.ref.Finalizer  
       2:        106977         855816  finalize.TestObjectHasFinalize  
       3:           642         841384  [I  
       4:          5204         521984  <constMethodKlass>  
       5:          8678         460712  <symbolKlass>  
       6:          5204         460672  <methodKlass>  
       7:          1694         206832  [C  
       8:           351         206024  <constantPoolKlass>  
       9:           351         142864  <instanceKlassKlass>  
      10:           325         140040  <constantPoolCacheKlass>  
      11:           421          79648  [B  
      12:          1701          40824  java.lang.String  
      13:           420          40320  java.lang.Class  
      14:           519          33720  [S  
      15:           547          32800  [[I  
      16:           758          24256  java.util.TreeMap$Entry  
      17:            94          18024  <methodDataKlass>  
      18:            40          13120  <objArrayKlassKlass>  
      19:           312          12776  [Ljava.lang.Object;  
      20:            76           6080  java.lang.reflect.Method  
      21:           181           5840  [Ljava.lang.String;  
      22:            37           2664  java.lang.reflect.Field  
      23:             8           2624  <typeArrayKlassKlass>  

可以发现占用内存较多的是java.lang.ref.Finalizer对象和TestObjectHasFinalize，为什么会有这么多个Finalizer对象呢？为什么要这么多的TestObjectHasFinalize对象呢？类似的，我们看下没有finalize()方法的情况

```java
public class TestObjectNoFinalize  
{  
    public static void main(String[] args)  
    {  
        while (true)  
        {  
            TestObjectNoFinalize heap = new TestObjectNoFinalize();  
            System.out.println("object no finalize method." + heap);  
        }  
    }  
}  
```


    C:\Documents and Settings\Administrator>jps  
    4436 TestObjectNoFinalize  
    6012 Jps  
    5272  
      
    C:\Documents and Settings\Administrator>jmap -histo:live 4436  
      
     num     #instances         #bytes  class name  
    ----------------------------------------------  
       1:          5203         521896  <constMethodKlass>  
       2:          8677         460696  <symbolKlass>  
       3:          5203         460584  <methodKlass>  
       4:          1689         206544  [C  
       5:           351         205992  <constantPoolKlass>  
       6:           351         142864  <instanceKlassKlass>  
       7:           325         140024  <constantPoolCacheKlass>  
       8:           421          79640  [B  
       9:          1696          40704  java.lang.String  
      10:           420          40320  java.lang.Class  
      11:           519          33720  [S  
      12:           547          32800  [[I  
      13:           758          24256  java.util.TreeMap$Entry  
      14:           407          22088  [I  
      15:            74          14720  <methodDataKlass>  
      16:            40          13120  <objArrayKlassKlass>  
      17:           312          12776  [Ljava.lang.Object;  
      18:            76           6080  java.lang.reflect.Method  
      19:           181           5840  [Ljava.lang.String;  
      20:            37           2664  java.lang.reflect.Field  
      21:             8           2624  <typeArrayKlassKlass>  
      22:           100           1976  [Ljava.lang.Class;  
      23:            20           1664  [Ljava.util.HashMap$Entry;  
      24:            12           1440  <klassKlass>  
      25:            59           1416  java.util.Hashtable$Entry  
      26:            13            728  java.net.URL  
      27:            18            720  java.util.HashMap  
      28:             7            680  [Ljava.util.Hashtable$Entry;  
      29:             6            672  java.lang.Thread  
      30:            10            640  java.lang.reflect.Constructor  
  
可以发现，java.lang.ref.Finalizer和TestObjectHasFinalize没有占用大量的堆内存。没有提供finalize()方法的类，占用的堆内存更少，垃圾回收速度更快，而且JVM也不会创建那么多java.lang.ref.Finalizer对象。

1. 使用finalize会导致严重的内存消耗和性能损失
《Effective Java》中提到“使用finalizer会导致严重的性能损失。在我的机器上，创建和销毁一个简单对象的实践大约是5.6ns，增加finalizer后时间增加到2400ns。换言之，创建和销毁带有finalizer的对象会慢430倍”。额外的内存消耗这个很容看出，因为使用了finalize的时候，堆内存中会多出很多java.lang.ref.Finalizer对象。性能损失这个可以通过分析得出结论，因为使用了finalize的时候，堆内存会驻留大量的无用TestObjectHasFinalize对象。为什么有这么多TestObjectHasFinalize对象呢？很简单，垃圾回收的速度变慢了，对象的销毁速度小于对象的创建速度。为什么有这么多的java.lang.ref.Finalizer对象对象呢？这是JVM内部的机制，用来保证finalize只被调用一次。

2. JVM不确保finalize一定会被执行，而且执行finalize的时间也不确定。

从一个对象变为不可达，到其finalizer被执行，可能会经过任意长时间。这意味着你不能在finalizer中执行任何注重时间的任务。依靠finalizer来关闭文件就是一个严重错误，因为打开文件的描述符是一个有限资源。JVM会延迟执行finalizer，所以大量文件会被保持在打开状态，当一个程序不再能打开文件的时候，就会运行失败。

```java
import sun.misc.Unsafe;  
  
public class RevisedObjectInHeap  
{  
    private long address = 0;  
  
    private Unsafe unsafe = GetUsafeInstance.getUnsafeInstance();  
  
    public RevisedObjectInHeap()  
    {  
        address = unsafe.allocateMemory(2 * 1024 * 1024);  
    }  
  
    @Override  
    protected void finalize() throws Throwable  
    {  
        super.finalize();  
        unsafe.freeMemory(address);  
    }  
  
    public static void main(String[] args)  
    {  
        while (true)  
        {  
            RevisedObjectInHeap heap = new RevisedObjectInHeap();  
            System.out.println("memory address=" + heap.address);  
        }  
    }  
  
}  
```

运行这段代码，很快就会出现堆外内存溢出。为什么呢？就是因为RevisedObjectInHeap.finalize方法不能及时执行，不能及时释放堆外内存。可以参考我的另一篇博客：java中使用堆外内存，关于内存回收需要注意的事和没有解决的遗留问题

当然使用finalize还有其他问题，具体的可以参考《Effective Java》。接下来介绍下JVM执行finalize方法的一些理论知识。实现了finalize()的对象，创建和回收的过程都更耗时。创建时，会新建一个额外的Finalizer 对象指向新创建的对象。 而回收时，至少需要经过两次GC。

先来看下新建对象的时候发生的事，测试步骤如下：

1、在TestObjectNoFinalize和TestObjectHasFinalize这2个类的while循环中打上断点，在java.lang.ref.Finalizer的Finalizer()和register()打上断点

2、分别debug运行TestObjectNoFinalize和TestObjectHasFinalize，观察进入Finalizer断点的次数

测试结果如下：

1、TestObjectNoFinalize没有finalize()方法

      进入Finalizer断点的次数很少（3次，指向的是JarFile、Inflater、FileInputStream），之后进入while循环中的断点，不会再进入Finalizer中的断点。也就是说：创建TestObjectNoFinalize对象的时候，不会创建相应的Finalizer对象。

2、TestObjectHasFinalize提供了finalize()方法

每次创建TestObjectHasFinalize对象的时候，都会创建相应的Finalizer对象指向它。

再看下回收对象的时候发生的事：加上-XX:+PrintGCDetails参数，观察下垃圾回收的过程。

可以看到TestObjectNoFinalize中都是在新生代中发生时的垃圾回收，很快就回收掉了内存。

TestObjectHasFinalize进行了几次新生代内存回收之后，频繁的进行[Full GC。这是以为回收内存的速度太慢，导致新生代内存不能及时释放，所以必须进行Full GC以期望获取空闲的内存空间。这个实验虽然不能直接证明至少需要进行2次GC，但是可以清楚的看到：含有finalize()的对象垃圾回收速度会很慢。


finalize机制的一些总结：

1. 如果一个类A实现了finalize()方法，那么每次创建A类对象的时候，都会多创建一个Finalizer对象(指向刚刚新建的对象)；如果类没有实现finalize()方法，那么不会创建额外的Finalizer对象

2. Finalizer内部维护了一个unfinalized链表，每次创建的Finalizer对象都会插入到该链表中。源码如下

```java
// 存储Finalizer对象的链表头指针  
static private Finalizer unfinalized = null;  
static private Object lock = new Object();  
  
private Finalizer next = null, prev = null;  
  
private void add()   
{  
    synchronized (lock)   
    {  
        if (unfinalized != null) {  
        this.next = unfinalized;  
        unfinalized.prev = this;  
        }  
        unfinalized = this;  
    }  
}  
```

3. 如果类没有实现finalize方法，那么进行垃圾回收的时候，可以直接从堆内存中释放该对象。这是速度最快，效率最高的方式

4. 如果类实现了finalize方法，进行GC的时候，如果发现某个对象只被java.lang.ref.Finalizer对象引用，那么会将该Finalizer对象加入到Finalizer类的引用队列（F-Queue）中，并从unfinalized链表中删除该结点。这个过程是JVM在GC的时候自动完成的。

```java
 /* Invoked by VM */      
private void remove()  
{  
    synchronized (lock)   
    {  
        if (unfinalized == this) {  
        if (this.next != null) {  
            unfinalized = this.next;  
        } else {  
            unfinalized = this.prev;  
        }  
        }  
        if (this.next != null) {  
        this.next.prev = this.prev;  
        }  
        if (this.prev != null) {  
        this.prev.next = this.next;  
        }  
        this.next = this;   /* Indicates that this has been finalized */  
        this.prev = this;  
    }  
}  
```

// 这就是F-Queue队列,存放的是Finalizer对象    

    static private ReferenceQueue queue = new ReferenceQueue();  

5. 含有finalize()的对象从内存中释放，至少需要两次GC。

第一次GC, 检测到对象只有被Finalizer引用，将这个对象放入 java.lang.ref.Finalizer.ReferenceQueue 此时，因为Finalizer的引用，对象还无法被GC。java.lang.ref.Finalizer$FinalizerThread 会不停的清理Queue的对象，remove掉当前元素，并执行对象的finalize方法。清理后对象没有任何引用，在下一次GC被回收。

6. Finalizer是JVM内部的守护线程，优先级很低。Finalizer线程是个单一职责的线程。这个线程会不停的循环等待java.lang.ref.Finalizer.ReferenceQueue中的新增对象。一旦Finalizer线程发现队列中出现了新的对象，它会弹出该对象，调用它的finalize()方法，将该引用从Finalizer类中移除，因此下次GC再执行的时候，这个Finalizer实例以及它引用的那个对象就可以回垃圾回收掉了。

```java
private static class FinalizerThread extends Thread   
{  
	FinalizerThread(ThreadGroup g) {  
    	super(g, "Finalizer");  
	}  
	public void run() {  
    	for (;;) {  
	    	try {  
    	    	Finalizer f = (Finalizer)queue.remove();  
	        	f.runFinalizer();  
		    } catch (InterruptedException x) {  
    		    continue;  
		    }  
    	}  
	}  
}  
```
7. 使用finalize容易导致OOM，因为如果创建对象的速度很快，那么Finalizer线程的回收速度赶不上创建速度，就会导致内存垃圾越来越多