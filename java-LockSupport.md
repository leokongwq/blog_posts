---
layout: post
comments: true
title: java-LockSupport详解
date: 2017-01-13 13:42:30
tags:
- java
categories:
- 并发编程
---

### JDK定义

> Basic thread blocking primitives for creating locks and other
synchronization classes.

翻译过来及时：*用于创建锁和其他同步类的基本线程阻塞原语。*

<!-- more -->

### 源码

```java
/** 
 * Basic thread blocking primitives for creating locks and other 
 * synchronization classes. 
 */  
public class LockSupport {  
    private LockSupport() {} // Cannot be instantiated.  
  
    // Hotspot implementation via intrinsics API  
    private static final Unsafe unsafe = Unsafe.getUnsafe();  
    private static final long parkBlockerOffset;  
  
    static {  
        try {  
            parkBlockerOffset = unsafe.objectFieldOffset  
                (java.lang.Thread.class.getDeclaredField("parkBlocker"));  
        } catch (Exception ex) { throw new Error(ex); }  
    }  
  
    private static void setBlocker(Thread t, Object arg) {  
        // Even though volatile, hotspot doesn't need a write barrier here.  
        unsafe.putObject(t, parkBlockerOffset, arg);  
    }  
    public static void unpark(Thread thread) {  
        if (thread != null)  
            unsafe.unpark(thread);  
    }  
    public static void park(Object blocker) {  
        Thread t = Thread.currentThread();  
        setBlocker(t, blocker);  
        unsafe.park(false, 0L);  
        setBlocker(t, null);  
    }  
    public static void parkNanos(Object blocker, long nanos) {  
        if (nanos > 0) {  
            Thread t = Thread.currentThread();  
            setBlocker(t, blocker);  
            unsafe.park(false, nanos);  
            setBlocker(t, null);  
        }  
    }  
    public static void parkUntil(Object blocker, long deadline) {  
        Thread t = Thread.currentThread();  
        setBlocker(t, blocker);  
        unsafe.park(true, deadline);  
        setBlocker(t, null);  
    }  
    public static Object getBlocker(Thread t) {  
        if (t == null)  
            throw new NullPointerException();  
        return unsafe.getObjectVolatile(t, parkBlockerOffset);  
    }  
    public static void park() {  
        unsafe.park(false, 0L);  
    }  
    public static void parkNanos(long nanos) {  
        if (nanos > 0)  
            unsafe.park(false, nanos);  
    }  
    public static void parkUntil(long deadline) {  
        unsafe.park(true, deadline);  
    }  
}  
```

解读源码：

- LockSupport不可被实例化（因为构造函数是私有的）
- LockSupport中有两个私有变量unsafe和parkBlockerOffset;
- LockSupport的方法都是静态方法

### 成员变量分析

unsafe:全名`sun.misc.Unsafe`, 关于它的用法可以参考[unsafe详解](/2016/12/31/java-magic-unsafe.html)
parkBlockerOffset：字面理解为parkBlocker的偏移量，但是parkBlocker又是干嘛的？偏移量又是做什么的呢？

1. 关于parkBlocker
在java.lang.Thread源码中有如下代码：

```java
/** 
 * The argument supplied to the current call to 
 * java.util.concurrent.locks.LockSupport.park. 
 * Set by (private) java.util.concurrent.locks.LockSupport.setBlocker 
 * Accessed using java.util.concurrent.locks.LockSupport.getBlocker 
 */  
volatile Object parkBlocker; 
```

从注释上看，这个对象被LockSupport的setBlocker和getBlocker调用。查看JAVADOC会发现这么一段解释：

{% asset_img 4cabf5b0-cbb6-35ec-b36b-3e83e9ac5fc9.png %}

大致意思是，这个对象是用来记录线程被阻塞时被谁阻塞的，用于线程监控和分析工具来定位原因的。
原来parkBlocker是用于记录线程是被谁阻塞的。可以通过LockSupport的getBlocker获取到阻塞的对象。主要用于监控和分析线程用的。

2. 关于parkBlockerOffset

```java
static {  
    try {  
        parkBlockerOffset = unsafe.objectFieldOffset  
            (java.lang.Thread.class.getDeclaredField("parkBlocker"));  
    } catch (Exception ex) { throw new Error(ex); }  
}  
```

我们把这段代码拆解一下：

```java
Field field = java.lang.Thread.class.getDeclaredField("parkBlocker");
parkBlockerOffset = unsafe.objectFieldOffset(field);
```

从这个静态语句块可以看的出来，先是通过反射机制获取Thread类的`parkBlocker`字段对象。然后通过`sun.misc.Unsafe`对象的`objectFieldOffset`方法获取到`parkBlocker`在内存里的偏移量，`parkBlockerOffset`的值就是这么来的.
JVM的实现可以自由选择如何实现Java对象的“布局”，也就是在内存里Java对象的各个部分放在哪里，包括对象的实例字段和一些元数据之类。`sun.misc.Unsafe`里关于对象字段访问的方法把对象布局抽象出来，它提供了`objectFieldOffset()`方法用于获取某个字段相对 Java对象的“起始地址”的偏移量，也提供了`getInt`、`getLong`、`getObject`之类的方法可以使用前面获取的偏移量来访问某个Java 对象的某个字段。

3. 为什么要用偏移量来获取对象？干吗不要直接写个get，set方法。多简单？
仔细想想就能明白，这个`parkBlocker`就是在线程处于阻塞的情况下才会被赋值。线程都已经阻塞了，如果不通过这种内存的方法，而是直接调用线程内的方法，线程是不会回应调用的。

### 深入方法分析

查看源码，你会发现LockSupport中有且只有一个私有方法

```java
private static void setBlocker(Thread t, Object arg) {  
     // Even though volatile, hotspot doesn't need a write barrier here.  
     unsafe.putObject(t, parkBlockerOffset, arg);  
}
```

解读：对给定线程t的parkBlocker赋值。为了防止这个parkBlocker被误用，该方法是不对外公开的。

```java
public static Object getBlocker(Thread t) {  
    if (t == null)  
        throw new NullPointerException();  
    return unsafe.getObjectVolatile(t, parkBlockerOffset);  
}  
```

解读：从线程t中获取它的parkBlocker对象，即返回的是阻塞线程t的Blocker对象。

接下来的方法里可以分两类，一类是以park开头的方法，用于阻塞线程:

```java
publicstaticvoid park(Object blocker) {  
  Thread t = Thread.currentThread();  
  setBlocker(t, blocker);  
  unsafe.park(false, 0L);  
  setBlocker(t, null);  
}  
publicstaticvoid parkNanos(Object blocker, long nanos) {  
  if (nanos > 0) {  
      Thread t = Thread.currentThread();  
      setBlocker(t, blocker);  
      unsafe.park(false, nanos);  
      setBlocker(t, null);  
  }  
}  
publicstaticvoid parkUntil(Object blocker, long deadline) {  
  Thread t = Thread.currentThread();  
  setBlocker(t, blocker);  
  unsafe.park(true, deadline);  
  setBlocker(t, null);  
}  
publicstaticvoid park() {  
  unsafe.park(false, 0L);  
}  
publicstaticvoid parkNanos(long nanos) {  
  if (nanos > 0)  
      unsafe.park(false, nanos);  
}  
publicstaticvoid parkUntil(long deadline) {  
  unsafe.park(true, deadline);  
}  
```
   
一类是以unpark开头的方法，用于解除阻塞：

```java   
public static void unpark(Thread thread) {  
    if (thread != null)  
        unsafe.unpark(thread);  
    }
} 
```

### 举个例子

```java
class FIFOMutex {
	private final AtomicBoolean locked = new AtomicBoolean(false);
	private final Queue<Thread> waiters = new ConcurrentLinkedQueue<Thread>();

	public void lock() {
	  boolean wasInterrupted = false;
	  Thread current = Thread.currentThread();
	  waiters.add(current);

	  // Block while not first in queue or cannot acquire lock
	  while (waiters.peek() != current ||
	         !locked.compareAndSet(false, true)) {
	    LockSupport.park(this);
	    if (Thread.interrupted()) // ignore interrupts while waiting
	      wasInterrupted = true;
	  }

	  waiters.remove();
	  if (wasInterrupted)  // reassert interrupt status on     exit
	    current.interrupt();
	}

	public void unlock() {
	  locked.set(false);
	  LockSupport.unpark(waiters.peek());
	}
}
```

从名字上我们能猜到这其实是一个先进先出的锁。先申请锁的线程最先拿到锁。我们来简单分析一下：

1. lock方法将请求锁的当前线程放入队列
2. 如果等待队列的队首元素就是当前线程，则当前线程修改变量`locked`的值为true表示已经上锁了。 然后删除等待锁队列中的队首元素也就是当前的线程。当前线程继续正常执行。这里需要注意一点，如果当前线程再次掉用`lock`方法则当前线程会被阻塞。这样就可能发生死锁，也就是说这个锁是不可重入的。
3. 如果等待队列的队首元素就不是当前线程（第一个获取锁的线程还没有执行remove）或上锁失败（第一个线程还没有释放锁），则直接通过调用`LockSupport.park(this)`来阻塞当前线程的执行。
4. 当获取锁的线程执行完后，调用unlock方法将锁变量修改为false,并解除队首线程的阻塞状态。位于队首的线程判断自己是不是队首元素，如果是了就修改原子变量的值来上锁。

从上面的代码来看实现一个FIFO锁对象竟然这么简单。单我们的关注点还是要看`LockSupport.park(this)` 是如何阻塞当前线程的。

### LockSupport.park

```
public static void park(Object blocker) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(false, 0L);
    setBlocker(t, null);
}
```

功能解释：

1. 该方法使得当前线程不能参与线程调度，除非当前线程获取许可。
2. 如果获取了许可并消费，则该方法调用就立刻返回了。
3. 有三种情况可以是该方法调用返回。第一种情况是调用`LockSupport.unpark`方法并将该线程作为参数传入。第二种是其它线程中断了该线程。第三种情况是该方法(park)虚假返回。

那`park`方法纠结是怎样阻塞当前线程的呢？继续深入

1. `setBlocker`方法将给当前线程的字段`parkBlocker`的值修改为方法参数指定的值
2. 调用`UNSAFE.park`方法
3. `setBlocker`方法将给当前线程的字段`parkBlocker`的值修改为null

我们知道`setBlocker`并没有导致当前线程阻塞，那只能是`UNSAFE.park`方法导致的。
`Unsafe`类的具体作用可以参考我以前的文章。当然了，也可以参考这篇:[Understanding sun.misc.Unsafe](https://dzone.com/articles/understanding-sunmiscunsafe)和[http://hg.openjdk.java.net/jdk7/jdk7/jdk/file/9b8c96f96a0f/src/share/classes/sun/misc/Unsafe.java](http://hg.openjdk.java.net/jdk7/jdk7/jdk/file/9b8c96f96a0f/src/share/classes/sun/misc/Unsafe.java)

直接点：

```java
/**
* Block current thread, returning when a balancing
* <tt>unpark</tt> occurs, or a balancing <tt>unpark</tt> has
* already occurred, or the thread is interrupted, or, if not
* absolute and time is not zero, the given time nanoseconds have
* elapsed, or if absolute, the given deadline in milliseconds
* since Epoch has passed, or spuriously (i.e., returning for no
* "reason"). Note: This operation is in the Unsafe class only
* because <tt>unpark</tt> is, so it would be strange to place it
* elsewhere.
*/
public native void park(boolean isAbsolute, long time);

/**
* Unblock the given thread blocked on <tt>park</tt>, or, if it is
* not blocked, cause the subsequent call to <tt>park</tt> not to
* block.  Note: this operation is "unsafe" solely because the
* caller must somehow ensure that the thread has not been
* destroyed. Nothing special is usually required to ensure this
* when called from Java (in which there will ordinarily be a live
* reference to the thread) but this is not nearly-automatically
* so when calling from native code.
* @param thread the thread to unpark.
*
*/
public native void unpark(Object thread);
```

相信看了源代码的注释你已经明白了一切。单需要注意的是如果park的时间参数是0,则表示一直阻塞。


### 更多参考

[java线程阻塞中断和LockSupport的常见问题](http://agapple.iteye.com/blog/970055)







  
 

