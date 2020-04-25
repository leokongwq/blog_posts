---
layout: post
comments: true
title: java中生成线程堆栈
date: 2017-02-26 20:12:16
tags:
- 面试
categories:
- java
---

本文翻译自:[http://www.journaldev.com/1053/java-thread-dump-visualvm-jstack-kill-3-jcmd](http://www.journaldev.com/1053/java-thread-dump-visualvm-jstack-kill-3-jcmd)

### Java 线程堆栈

Java线程堆栈用来分析应用的瓶颈和线程死锁问题是非常有用的。

下面介绍一些生成Java线程堆栈的方法。这些方式都是基于*nix操作系统，在windows上可能有所不同。

<!-- more -->

### VisualVM Profiler: 

如果你在分析应用变慢的原因，那你必须使用一个profiler。我们可以使用`VisualVM`profiler非常容易的生成线程堆栈信息。你只需要右键点击运行中的进程，并选择`Thread Dump`选项进行单击即可。

{% asset_img visualvm-java-thread-dump.png %}

### jstack

发行版自带的工具`jstack`可以用来生成线程堆栈信息。该操作分为两步：

1. 使用`ps -eaf | grep java`来找出进程ID 
2. 使用`jstack PID`来生成堆栈信息。默认堆栈信息会输出到标准输出，你也可以将输出重定向到指定的文件`jstack PID >> mydumps.tdump`

### kill -3 

我们可以使用`kill -3 PID`命令来生成线程堆栈信息。这种方式和其它的方法有一点不同，当kill命令被处理后，线程堆栈信息会输出到应用指定的系统输出中。所以如果一个java程序将控制台作为系统输出，则线程堆栈将输出到控制台上。 如果java程序是一个Tomcat服务程序，则系统输出为catalina.out，那么线程堆栈信息将在文件中生成。

### jcmd 

在Java8中提供了`jcmd`工具。你可以使用这个工具来代替`jstack`，在Java8或更高的JDK版本中生成线程堆栈信息。使用方式是

    jcmd PID Thread.print.
    
以上是在java中生成线程转储的四种不同方法。 通常我喜欢jstack或jcmd命令来生成线程转储和分析。 注意，无论你选择什么方式，线程转储将总是相同的。

### Java Thread Dump 例子

死锁例子

```java
public class ThreadDeadlock {

    public static void main(String[] args) throws InterruptedException {
        Object obj1 = new Object();
        Object obj2 = new Object();
        Object obj3 = new Object();
    
        Thread t1 = new Thread(new SyncThread(obj1, obj2), "t1");
        Thread t2 = new Thread(new SyncThread(obj2, obj3), "t2");
        Thread t3 = new Thread(new SyncThread(obj3, obj1), "t3");
        
        t1.start();
        Thread.sleep(5000);
        t2.start();
        Thread.sleep(5000);
        t3.start();
        
    }

}

class SyncThread implements Runnable{
    private Object obj1;
    private Object obj2;

    public SyncThread(Object o1, Object o2){
        this.obj1=o1;
        this.obj2=o2;
    }
    @Override
    public void run() {
        String name = Thread.currentThread().getName();
        System.out.println(name + " acquiring lock on "+obj1);
        synchronized (obj1) {
         System.out.println(name + " acquired lock on "+obj1);
         work();
         System.out.println(name + " acquiring lock on "+obj2);
         synchronized (obj2) {
            System.out.println(name + " acquired lock on "+obj2);
            work();
        }
         System.out.println(name + " released lock on "+obj2);
        }
        System.out.println(name + " released lock on "+obj1);
        System.out.println(name + " finished execution.");
    }
    private void work() {
        try {
            Thread.sleep(30000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

线程堆栈信息

```
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.51-b03 mixed mode):

"DestroyJavaVM" #13 prio=5 os_prio=31 tid=0x00007fa385076800 nid=0x1303 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"t3" #12 prio=5 os_prio=31 tid=0x00007fa3850c8800 nid=0x2f07 waiting for monitor entry [0x000000012715c000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at com.leokongwq.jdk.SyncThread.run(ThreadDeadlock.java:41)
	- waiting to lock <0x0000000795805df0> (a java.lang.Object)
	- locked <0x0000000795805e10> (a java.lang.Object)
	at java.lang.Thread.run(Thread.java:745)

"t2" #11 prio=5 os_prio=31 tid=0x00007fa38401d000 nid=0x4f03 waiting for monitor entry [0x0000000129407000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at com.leokongwq.jdk.SyncThread.run(ThreadDeadlock.java:41)
	- waiting to lock <0x0000000795805e10> (a java.lang.Object)
	- locked <0x0000000795805e00> (a java.lang.Object)
	at java.lang.Thread.run(Thread.java:745)

"t1" #10 prio=5 os_prio=31 tid=0x00007fa384837000 nid=0x4d03 waiting for monitor entry [0x0000000129304000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at com.leokongwq.jdk.SyncThread.run(ThreadDeadlock.java:41)
	- waiting to lock <0x0000000795805e00> (a java.lang.Object)
	- locked <0x0000000795805df0> (a java.lang.Object)
	at java.lang.Thread.run(Thread.java:745)

"Monitor Ctrl-Break" #9 daemon prio=5 os_prio=31 tid=0x00007fa385065000 nid=0x4b03 runnable [0x00000001291a2000]
   java.lang.Thread.State: RUNNABLE
	at java.net.PlainSocketImpl.socketAccept(Native Method)
	at java.net.AbstractPlainSocketImpl.accept(AbstractPlainSocketImpl.java:404)
	at java.net.ServerSocket.implAccept(ServerSocket.java:545)
	at java.net.ServerSocket.accept(ServerSocket.java:513)
	at com.intellij.rt.execution.application.AppMain$1.run(AppMain.java:90)
	at java.lang.Thread.run(Thread.java:745)

"Service Thread" #8 daemon prio=9 os_prio=31 tid=0x00007fa384037000 nid=0x4703 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C1 CompilerThread2" #7 daemon prio=9 os_prio=31 tid=0x00007fa3850b9000 nid=0x4503 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread1" #6 daemon prio=9 os_prio=31 tid=0x00007fa38401c000 nid=0x4303 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread0" #5 daemon prio=9 os_prio=31 tid=0x00007fa385076000 nid=0x4103 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Signal Dispatcher" #4 daemon prio=9 os_prio=31 tid=0x00007fa38502c800 nid=0x3413 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Finalizer" #3 daemon prio=8 os_prio=31 tid=0x00007fa383806000 nid=0x2d03 in Object.wait() [0x0000000127013000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x0000000795586f58> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:143)
	- locked <0x0000000795586f58> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:164)
	at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)

"Reference Handler" #2 daemon prio=10 os_prio=31 tid=0x00007fa383805800 nid=0x2b03 in Object.wait() [0x0000000126f10000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x0000000795586998> (a java.lang.ref.Reference$Lock)
	at java.lang.Object.wait(Object.java:502)
	at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:157)
	- locked <0x0000000795586998> (a java.lang.ref.Reference$Lock)

"VM Thread" os_prio=31 tid=0x00007fa38480b000 nid=0x2903 runnable 

"GC task thread#0 (ParallelGC)" os_prio=31 tid=0x00007fa384012800 nid=0x2103 runnable 

"GC task thread#1 (ParallelGC)" os_prio=31 tid=0x00007fa384013800 nid=0x2303 runnable 

"GC task thread#2 (ParallelGC)" os_prio=31 tid=0x00007fa384014000 nid=0x2503 runnable 

"GC task thread#3 (ParallelGC)" os_prio=31 tid=0x00007fa384014800 nid=0x2703 runnable 

"VM Periodic Task Thread" os_prio=31 tid=0x00007fa38480b800 nid=0x4903 waiting on condition 

JNI global references: 21


Found one Java-level deadlock:
=============================
"t3":
  waiting to lock monitor 0x00007fa385057b58 (object 0x0000000795805df0, a java.lang.Object),
  which is held by "t1"
"t1":
  waiting to lock monitor 0x00007fa385055008 (object 0x0000000795805e00, a java.lang.Object),
  which is held by "t2"
"t2":
  waiting to lock monitor 0x00007fa385057aa8 (object 0x0000000795805e10, a java.lang.Object),
  which is held by "t3"

Java stack information for the threads listed above:
===================================================
"t3":
	at com.leokongwq.jdk.SyncThread.run(ThreadDeadlock.java:41)
	- waiting to lock <0x0000000795805df0> (a java.lang.Object)
	- locked <0x0000000795805e10> (a java.lang.Object)
	at java.lang.Thread.run(Thread.java:745)
"t1":
	at com.leokongwq.jdk.SyncThread.run(ThreadDeadlock.java:41)
	- waiting to lock <0x0000000795805e00> (a java.lang.Object)
	- locked <0x0000000795805df0> (a java.lang.Object)
	at java.lang.Thread.run(Thread.java:745)
"t2":
	at com.leokongwq.jdk.SyncThread.run(ThreadDeadlock.java:41)
	- waiting to lock <0x0000000795805e10> (a java.lang.Object)
	- locked <0x0000000795805e00> (a java.lang.Object)
	at java.lang.Thread.run(Thread.java:745)

Found 1 deadlock.

Heap
 PSYoungGen      total 38400K, used 6656K [0x0000000795580000, 0x0000000798000000, 0x00000007c0000000)
  eden space 33280K, 20% used [0x0000000795580000,0x0000000795c00340,0x0000000797600000)
  from space 5120K, 0% used [0x0000000797b00000,0x0000000797b00000,0x0000000798000000)
  to   space 5120K, 0% used [0x0000000797600000,0x0000000797600000,0x0000000797b00000)
 ParOldGen       total 87552K, used 0K [0x0000000740000000, 0x0000000745580000, 0x0000000795580000)
  object space 87552K, 0% used [0x0000000740000000,0x0000000740000000,0x0000000745580000)
 Metaspace       used 3129K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 345K, capacity 388K, committed 512K, reserved 1048576K
```


