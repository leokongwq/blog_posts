---
layout: post
comments: true
title: 过去5年Java面试题精选
date: 2017-02-25 10:34:59
tags:
- 面试
categories:
- java
---

### 前言

Time is changing and so is Java interviews. Gone are the days, when knowing the difference between String and StringBuffer can help you to go through the second round of interview, questions are becoming more advanced and interviewers are asking more deep questions. When I started my career, questions like [Vector vs Array](http://javarevisited.blogspot.sg/2011/09/difference-vector-vs-arraylist-in-java.html) and [HashMap vs Hashtable](http://javarevisited.blogspot.sg/2010/10/difference-between-hashmap-and.html) were the most popular ones and just memorizing them gives you a good chance to do well in interviews, but not anymore. Nowadays, you will get questions from the areas where not many Java programmer looks e.g. NIO, patterns, sophisticated unit testing or those which are hard to master e.g. concurrency, algorithms, data structures and coding.

<!-- more -->

Since I like to explore interview questions, I have got this huge list of questions with me, which contains lots and lots of questions from different topics. I have been preparing this MEGA list from quite some time and now It's ready to share with you guys. It contains interview questions not only from classic topics like threads, collections, equals and hashcode, sockets but also from NIO, array, string, java 8 and much more.

It has questions for both entry level Java programmers and senior developers with years of experience. No matter whether you are a Java developer of 1, 2, 3, 4, 5, 6, 8, 9 or even 10 years of experience, you will find something interesting in this list. It contains questions which are super easy to answer, and also, a question which is tricky enough for even seasoned Java programmers.

Given the list is long and we have questions from everywhere, it's imperative that answers must be short, concise and crisp, no fluffing at all. So apart from this paragraph, you will only hear from me is the questions and answers, no more context, no more feedback and no more evaluation. For that, I have already written blog posts, where you can find my views on a particular question, e.g. why I like that question, what makes them challenging and what kind of answer you should expect from candidates.

This list is a little bit different and I encourage you to share questions and answers in a similar way so that it should be easy to revise. I hope this list can be a great use for both interviewer and candidates, the interviewer can, of course, put a little variety on questions to bring novelty and surprise element, which is important for a good interview. While a candidate, can expand and test their knowledge about key areas of Java programming language and platform. In 2015 and in coming years the focus will be more on advanced concurrency concept, JVM internals, 32-bit vs 64-bit JVM, unit testing, and clean code. I am sure, once you read through this MEGA list of Java interview question, you should be able to do well on both [telephonic](http://java67.blogspot.sg/2015/03/top-40-core-java-interview-questions-answers-telephonic-round.html) and face to face programming interviews.

### Important Topics for Java Interviews

Apart from quantity, as you can see with a huge number of questions, I have worked hard to maintain quality as well. I have not only shared questions from all important topics but also ensured to include so-called advanced topics which many programmers do not prefer to prepare or just left out because they have not worked on that. Java NIO and JVM internals questions are best examples of that. You can keep design patterns also on the same list but growing number of an experienced programmer are now well aware of GOF design patterns and when to use them. I have also worked hard to keep this list up-to-date to include what interviewers are asking in 2015 and what will be their core focus on coming years. To give you an idea, this list of Java interview questions includes following topics:

1. Multithreading, concurrency and thread basics
2. Date type conversion and fundamentals
3. Garbage Collection
4. Java Collections Framework
5. Array
6. String
7. GOF Design Patterns
8. SOLID design principles
9. Abstract class and interface
10. Java basics e.g. equals() and hashcode
11. Generics and Enum
12. Java IO and NIO
13. Common Networking protocols
14. Data structure and algorithm in Java
15. Regular expressions
16. JVM internals
17. Java Best Practices
18. JDBC
19. Date, Time, and Calendar
20. XML Processing in Java
21. JUnit
22. Programming

You guys are also lucky that nowadays there are some good books available to prepare for Java interviews, one of them   which I particularly find useful and interesting to read is [Java Programming Interview Exposed by Markham](http://www.amazon.com/Java-Programming-Interviews-Exposed-Markham/dp/1118722868?tag=javamysqlanta-20). It will take you to some of the most important topics for Java and JEE interviews, worth reading even if you are not preparing for Java interview.

### Top 120 Java Interview Questions Answers

So now the time has come to introduce you to this MEGA list of 120 Java questions collected from various interviews of last 5 years. I am sure you have seen many of these questions personally on your interviews and many of you would have answered them correctly as well.


#### Multithreading, Concurrency and Thread basics Questions


1) Can we make array volatile in Java?

This is one of the tricky Java multi-threading questions you will see in senior Java developer Interview. Yes, you can make an array volatile in Java but only the reference which is pointing to an array, not the whole array. What I mean, if one thread changes the reference variable to points to another array, that will provide a volatile guarantee, but if multiple threads are changing individual array elements they won't be having happens before guarantee provided by the volatile modifier.


2) Can volatile make a non-atomic operation to atomic?

This another good question I love to ask on volatile, mostly as a follow-up of the previous question. This question is also not easy to answer because volatile is not about atomicity, but there are cases where you can use a volatile variable to make the operation atomic.

One example I have seen is having a long field in your class. If you know that a long field is accessed by more than one thread e.g. a counter, a price field or anything, you better make it volatile. Why? because reading to a long variable is not atomic in Java and done in two steps, If one thread is writing or updating long value, it's possible for another thread to see half value (fist 32-bit). While reading/writing a volatile long or double (64 bit) is atomic.

3) What are practical uses of volatile modifier? 

One of the practical use of the volatile variable is to make reading double and long atomic. Both double and long are 64-bit wide and they are read in two parts, first 32-bit first time and next 32-bit second time, which is non-atomic but volatile double and long read is atomic in Java. Another use of the volatile variable is to provide a memory barrier, just like it is used in Disrupter framework. Basically, Java Memory model inserts a write barrier after you write to a volatile variable and a read barrier before you read it. Which means, if you write to volatile field then it's guaranteed that any thread accessing that variable will see the value you wrote and anything you did before doing that right into the thread is guaranteed to have happened and any updated data values will also be visible to all threads, because the memory barrier flushed all other writes to the cache.

4) What guarantee volatile variable provides? (answer)

volatile variables provide the guarantee about ordering and visibility e.g. volatile assignment cannot be re-ordered with other statements but in the absence of any synchronization instruction compiler, JVM or JIT are free to reorder statements for better performance. volatile also provides the happens-before guarantee which ensures changes made in one thread is visible to others. In some cases volatile also provide atomicity e.g. reading 64-bit data types like long and double are not atomic but read of volatile double or long is atomic.

5) Which one would be easy to write? synchronization code for 10 threads or 2 threads?

In terms of writing code, both will be of same complexity because synchronization code is independent of a number of threads. Choice of synchronization though depends upon a number of threads because the number of thread present more contention, so you go for advanced synchronization technique e.g. lock stripping, which requires more complex code and expertise.

6) How do you call wait() method? using if block or loop? Why? (answer)

wait() method should always be called in loop because it's possible that until thread gets CPU to start running again the condition might not hold, so it's always better to check condition in loop before proceeding. Here is the standard idiom of using wait and notify method in Java:

```java
// The standard idiom for using the wait method
synchronized (obj) {
   while (condition does not hold)
      obj.wait(); // (Releases lock, and reacquires on wakeup)
      ... // Perform action appropriate to condition
}
```

See Effective Java Item 69 to learn more about why wait method should call in the loop.

7)  What is false sharing in the context of multi-threading? 

false sharing is one of the well-known performance issues on multi-core systems, where each process has its local cache. false sharing occurs when threads on different processor modify variables that reside on same cache line as shown in the following image:

{% asset_img %}

Java Interview questions for experienced programmers

False sharing is very hard to detect because the thread may be accessing completely different global variables that happen to be relatively close together in memory. Like many concurrency issues, the primary way to avoid false sharing is careful code review and aligning your data structure with the size of a cache line.

8) What is busy spin? Why should you use it?

Busy spin is one of the technique to wait for events without releasing CPU. It's often done to avoid losing data in CPU cached which is lost if the thread is paused and resumed in some other core. So, if you are working on low latency system where your order processing thread currently doesn't have any order, instead of sleeping or calling wait(), you can just loop and then again check the queue for new messages. It's only beneficial if you need to wait for a very small amount of time e.g. in micro seconds or nano seconds. LMAX Disrupter framework, a high-performance inter-thread messaging library has a BusySpinWaitStrategy which is based on this concept and uses a busy spin loop for EventProcessors waiting on the barrier.

9) How do you take thread dump in Java?

You can take a thread dump of Java application in Linux by using kill -3 PID, where PID is the process id of Java process. In Windows, you can press Ctrl + Break. This will instruct JVM to print thread dump in standard out or err and it could go to console or log file depending upon your application configuration. If you have used Tomcat then when

10) is Swing thread-safe? (answer)

No, Swing is not thread-safe. You cannot update Swing components e.g. JTable, JList or JPanel from any thread, in fact, they must be updated from GUI or AWT thread. That's why swings provide invokeAndWait() and invokeLater() method to request GUI update from any other threads. This methods put update request in AWT threads queue and can wait till update or return immediately for an asynchronous update. You can also check the detailed answer to learn more

11) What is a thread local variable in Java? (answer)

Thread-local variables are variables confined to a thread, its like thread's own copy which is not shared between multiple threads. Java provides a ThreadLocal class to support thread-local variables. It's one of the many ways to achieve thread-safety. Though be careful while using thread local variable in manged environment e.g. with web servers where worker thread out lives any application variable. Any thread local variable which is not removed once its work is done can potentially cause a memory leak in Java application.

12) Write wait-notify code for producer-consumer problem? (answer)

Please see the answer for a code example. Just remember to call wait() and notify() method from synchronized block and test waiting for condition on the loop instead of if block.

13) Write code for thread-safe Singleton in Java? (answer)

Please see the answer for a code example and step by step guide to creating thread-safe singleton class in Java. When we say thread-safe, which means Singleton should remain singleton even if initialization occurs in the case of multiple threads. Using Java enum as Singleton class is one of the easiest ways to create a thread-safe singleton in Java.

14) The difference between sleep and wait in Java? (answer)

Though both are used to pause currently running thread, sleep() is actually meant for short pause because it doesn't release lock, while wait() is meant for conditional wait and that's why it release lock which can then be acquired by another thread to change the condition on which it is waiting.

15) What is an immutable object? How do you create an Immutable object in Java? (answer)

Immutable objects are those whose state cannot be changed once created. Any modification will result in a new object e.g. String, Integer, and other wrapper class. Please see the answer for step by step guide to creating Immutable class in Java.

16) Can we create an Immutable object, which contains a mutable object?

Yes, its possible to create an Immutable object which may contain a mutable object, you just need to be a little bit careful not to share the reference of the mutable component, instead, you should return a copy of it if you have to. Most common example is an Object which contain the reference of java.util.Date object.


#### 数据类型和基本的Java面试题

17) Java中使用哪个数据类型来表示价格？

BigDecimal。如果内存不是问题，性能不是关键，否则用预定义精度双精度。

18) 如何将bytes转为String

可以使用String类的构造函数，它接收一个字节数组。需要注意的是字符串的编码，如果你没有指定编码方式，则使用系统默认的值。

19) Java中如何将bytes转为long？

参考[http://blog.csdn.net/defonds/article/details/8782785](http://blog.csdn.net/defonds/article/details/8782785)

20) 是否可以将一个int强制转为byte？如果int的值大于byte所能表达的最大值会发生什么？

可以, 我们可以转换，但在Java中int为32位长，而byte在Java中为8位长，因此当你将一个int转换为字节时，高24位会丢失，一个字节只能保存-128到128的值。

21) 有两个类，B继承自A, C继承自B, 是否可以将B转为C, 如：
    
    C = (C) B;

不可以，只能向上类型转换。

22) 哪一个类包含`clone`方法？ Cloneable 或 Object？

`java.lang.Cloneable` 是一个标志接口，没有包含任何方法。`clone`方法定义在Object类中（是一个native方法）。

23) Java中`++`操作是否是现场安全的？

它不是一个线程安全的操作符，因为它涉及多个指令，如读取一个值，累加，并将其存储回内存。

24) 操作 `a = a + b` 和 `a += b`的区别？

操作符`+=`会隐式的相加的结果转为保存相加结果变量的类型。当你将类型为`byte`,`shot`或`int`的两个变量相加，第一步会将变量的类型提升为int,然后再相加。如果相加的结果大于变量a的类型最大值，则`a+b`会给出编译错误，但是`a+=b`则不会。

```java
byte a = 127;
byte b = 127;
b = a + b; // error : cannot convert from int to byte
b += a; // ok
```

25) 是否可以不进行类型转换而将一个double值存入long变量？

不，你不能将一个double值存储到一个long变量而不进行转换，因为double的范围更长，你需要类型转换。 回答这个问题并不难，但许多开发者还是错了，因为混淆在Java中double类型和long类型纠结谁大。

26) 表达式 `3*0.1 == 0.3`的返回值是true还是false?

这真的一个诡异的Java问题。在100个Java开发者中只有5个人能正确回答该问题并给出合理的解释。简短的答案是：一些浮点数不能被精确的表示。

27) 哪一个会占用更多的内存空间, int 还是 Integer ?

这个毋庸置疑了，必须是Integer

28) Java中的String为什么是不可变的？

One of my favorite Java interview question. The String is Immutable in java because java designer thought that string will be heavily used and making it immutable allow some optimization easy sharing same String object between multiple clients. See the link for the more detailed answer. This is a great question for Java programmers with less experience as it gives them food for thought, to think about how things works in Java, what Jave designers might have thought when they created String class etc.

29) String可以用在switch case语句中吗？

从Java 7开始，我们可以在switch语句中使用String，但它只是语法糖。 原因是使用了字符串内部的哈希码用于switch。

30) 什么是Java中的构造器链？

当你在一个构造函数中调用另一个构造函数时，这种情况被称为构造器链。这种情况是因为你重载了多个构造函数。

#### JVM Internals and Garbage Collection Interview Questions

在2015年，我在各种Java访谈中看到关于JVM内部原理和垃圾收集调整，监控Java应用程序和处理Java性能问题的关注在持续上升。 这实际上成为面试任何经验丰富的Java开发人员的主要议题，如技术主管，副总裁或团队领导。 如果你觉得你缺乏这方面的经验和知识，那么你应该阅读我的Java性能书籍列表中提到的至少一本书[Java Performance books](http://javarevisited.blogspot.com/2014/07/top-5-java-performance-tuning-books.html)。 

31) int类型在64-bit JVM上的大小？

int变量的大小在Java中是不变的，它总是32位，与平台无关。 这意味着在32位和64位Java虚拟机中基本int的大小是相同的。

32) 串行收集器和并行收集器的区别？

即使串行和并行收集器在垃圾收集期间导致`Stop The World`。 它们之间的主要区别是串行收集器是默认的复制收集器，它只使用一个GC线程进行垃圾收集，而并行收集器使用多个GC线程进行垃圾收集。

33) 32位和64位JVM中的int变量的大小是多少？ 

int的大小在32位和64位JVM中是相同的，它总是32位或4字节。

34) Java中弱引用和软应用的区别是？

虽然WeakReference和SoftReference都有助于垃圾收集器和内存的高效使用，但是WeakReference只要最后一个强引用丢失就可以进行垃圾收集，而SoftReference只有在内存空间不足的情况下才会进行回收。

35) WeakHashMap 是如何工作的？

WeakHashMap的工作方式类似于正常的HashMap，但对键使用WeakReference，这意味着如果键对象没有任何引用，那么键/值映射都将有资格进行垃圾回收。

36) 虚拟机参数-XX:+UseCompressedOops的作用是什么？为什么使用它？

当您将Java应用程序从32位迁移到64位JVM时，堆容量突然增加，几乎翻倍，原因是普通对象指针从32位增加到64位。 这也影响你可以在CPU缓存中保留多少数据(CPU缓存比内存小得多)。 因为移动到64位JVM的主要动机是使用更大的堆，所以可以使用压缩的OOP来节省一些内存。 通过使用`-XX:+UseCompressedOops`，JVM使用32位OOP而不是64位OOP。

37) 你如何在代码中检查JVM是32位的还是64位的？

你可以通过一些系统变量来检查，例如：`sun.arch.data.model` 或 `os.arch`

38) 32位和64位JVM的堆最大值是多少？

理论上，可以分配给32位JVM的最大堆内存是2 ^ 32，就是4GB，但实际上这个限制小得多。 在操作系统之间这个值也是变化的。 在Windows中大概是`1.5GB`，在Solaris中几乎达到`3GB`。 64位JVM允许你指定更大的堆大小，理论上2 ^ 64这是相当大，但实际上你可以指定堆空间高达`100GBs`。 甚至有JVM。 Azul这里也有1000个gig的堆空间。

39) JRE，JDK，JVM和JIT有什么区别？ 

JRE代表Java运行时，它需要运行Java应用程序。 JDK代表Java开发工具包，并提供了开发Java程序的工具。 Java编译器。 它还包含JRE。 JVM代表Java虚拟机，它是负责运行Java应用程序的进程。 JIT代表即时编译，当跨越的某些阈值（即主要是热代码被转换为本地代码）时，通过将Java字节代码转换为本地代码来帮助提高Java应用程序的性能。

{% asset_img JVM-JRE.jpg %}

#### 3年经验的开发者Java面试题

40) 解释Java堆空间和垃圾收集? ([answer](http://javarevisited.blogspot.sg/2011/05/java-heap-space-memory-size-jvm.html))

Garbage collection is the process inside JVM which reclaims memory from dead objects for future allocation.

当使用java命令启动一个Java进程时，操作系统会为该进程分配内存。 此内存的一部分用于创建堆空间，用于在程序中创建对象在时为其分配内存。 垃圾收集是JVM中回收已经死亡对象的内存空间供新对象创建使用的的处理过程。

{% asset_img java_heaps_memory.jpg %}

41) 你能保证垃圾收集过程吗？

不，你不能保证垃圾收集，虽然你可以使用System.gc（）或Runtime.gc（）方法请求。

42) 如何从Java程序中查找内存使用情况？ 有多少百分比的堆被使用？

你可以使用`java.lang.Runtime`类中的内存相关方法来获取Java中的可用内存，总内存和最大堆内存。 通过使用这些方法，您可以了解使用堆的百分比和剩余的堆空间。 `Runtime.freeMemory()`返回以字节为单位的可用内存量，`Runtime.totalMemory()`返回总内存（以字节为单位），`Runtime.maxMemory()`返回最大内存（以字节为单位）。

43) java中堆和栈的区别是？

栈和堆是JVM中不同的内存区域，它们用于不同的目的。 栈用于保存方法调用栈帧和局部变量，而对象总是从堆中分配内存。 堆栈通常比堆内存小得多，并且没有在多个线程之间共享，但是堆在JVM中的所有线程之间共享。

{% asset_img Difference-between-stack-heap.jpg %}

#### Java基本概念面试题

44) "a == b" 和 "a.equals(b)"之间的区别？

如果a和b都是一个对象，并且只有当两个引用都指向堆空间中的同一个对象时，a = b才会进行对象引用匹配，另一方面，a.equals（b）用于逻辑映射， 它期望一个对象重写这个方法来提供逻辑等同。 例如，String类覆盖此equals()方法，以便可以比较两个字符串，这两个字符串是不同的对象，但包含相同的字母。

45) `a.hashCode()`的作用是？它和`a.equals(b)`有什么关联？

`hashCode()`方法返回对应于对象的int哈希值。 它用于基于散列的集合类，例如Hashtable，HashMap，LinkedHashMap等。 它与`equals()`方法紧密相关。 根据Java规范，使用`equals()`方法彼此相等的两个对象必须具有相同的哈希码。

46) final, finalize 和 finally之间的区别?

The final is a modifier which you can apply to variable, methods and classes. If you make a variable final it means its value cannot be changed once initialized. finalize is a method, which is called just before an object is a garbage collected, giving it last chance to resurrect itself, but the call to finalize is not guaranteed. finally is a keyword which is used in exception handling along with try and catch. the finally block is always executed irrespective of whether an exception is thrown from try block or not.

final是一个修饰符，你可以应用于变量，方法和类。 如果你一个变量final的，这意味着它的值一旦初始化就不能改变。 finalize是一个方法，它在一个对象被垃圾收集之前被调用，给它最后一次机会来复活自己，但是对finalize的调用不能保证。 finally是一个用于异常处理以及try和catch的关键字。 无论是否从try块抛出异常，始终执行finally块

47) 什么是Java中的编译时常量？ 使用它的风险是什么？

`public static final`变量也称为编译时常量，public是可选的。 它们在编译时被替换为实际值，因为编译器预先知道它们的值，并且知道它在运行时不能被改变。 其中一个问题是，如果你碰巧使用一个内部或第三方库中的`public static final`变量，并且它们的值更改后，客户端仍将使用旧值，即使部署新版本的JAR 。 为避免这种情况，请确保在升级依赖项JAR文件时编译程序。

#### Java集合框架面试题

它还包含Java中的数据结构和算法访问问题，数组，链表，HashMap，ArrayList，Hashtable，Stack，Queue，PriorityQueue，LinkedHashMap和ConcurrentHashMap问题。

48) Java中List, Set, Map, 和 Queue 之间的区别是？

List是有序的集合并允许元素重复。它还提供了常数时间的查询（根据下标）。但这不是由List接口包装的。 Set是一个无序的集合，不允许重复元素。

49) poll() 和 remove() 的区别？

`poll()` 和 `remove()`都是从队列里面获取对象，如果队列为空则poll返回null, remove则会抛出异常。

50) LinkedHashMap 和 PriorityQueue 的区别？

PriorityQueue保证最低或最高优先级元素始终保留在队列的头部，但LinkedHashMap维护元素插入的顺序。 当你遍历一个PriorityQueue，迭代器不保证任何顺序，但LinkedHashMap的迭代器确保插入元素的顺序。

51) Java ArrayList 和 LinkedList的不同？

它们之间的明显区别是ArrrayList由数组数据结构支持，支持随机访问，LinkedList由链表数据结构支持，不支持随机访问。 使用索引访问ArrayList中的元素时间复杂度是O（1），但是在LinkedList中是O（n）。 

52) 有哪几种方法可以排序一个集合？

你可以使用本身具有排序功能的集合,例如：TreeSet, TreeMap。 当然也可以通过`Collections.sort()`对集合进行排序。

53) Java中如何打印数组？

你可以使用`Arrays.toString()`和`Arrays.deepToString()`方法打印数组。 由于数组本身不实现`toString()`，将数组传递给`System.out.println()`将不会打印其内容，而`Arrays.toString()`将打印数组中的每个元素。

54) Java中的 LinkedList 是单链表还是双链表?

答案是：双链表

55) Java中使用哪种类型的树来实现TreeMap?

java中使用红黑树来实现TreeMap。

56) Hashtable 和 HashMap的区别？

这两个类之间有很多区别，其中一些如下：
a）Hashtable是一个遗留的类，从JDK1开始就存在了，HashMap是后来添加。
b）Hashtable线程安全的，速度更慢，HashMap线程安全的，速度更快
c）Hashtable不允许空键值对，但HashMap允许一个空键值对。

57) Java中的HashSet时如何工作的？

HashSet在内部使用HashMap实现。 由于Map需要键和值，因此所有键都使用一个默认值。 与HashMap类似，HashSet不允许重复的键，并允许一个null键，我的意思是你只能在HashSet中存储一个null对象。

58) 编写代码在迭代ArrayList的过程中删除元素？

这里的关键是检查候选者是否使用ArrayList的remove（）还是 Iterator的remove（）。 这里是示例代码，它使用正确的方式在循环中从ArrayList中删除元素并避免ConcurrentModificationException异常。

```java
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

/**
  * Java program to demonstrate how to remove object form List and differnece
  * between Iterator's remove() method and Colection's remove() method in Java
  *
  * @author http://javarevisited.blogspot.com
 */
public class ObjectRemovalTest {
  
    public static void main(String args[]) {
      
       List markets = new ArrayList();
     
       StockExchange TSE = new StockExchange(){
         
            @Override
            public boolean isClosed() {
                return true;
            }         
       };
     
       StockExchange HKSE = new StockExchange(){

            @Override
            public boolean isClosed() {
                return true;
            }         
       };
      
       StockExchange NYSE = new StockExchange(){

            @Override
            public boolean isClosed() {
                return false;
            }         
       };
      
       markets.add(TSE);
       markets.add(HKSE);
       markets.add(NYSE);
     
     
       Iterator itr = markets.iterator();
     
       while(itr.hasNext()){
           StockExchange exchange = itr.next();
           if(exchange.isClosed()){
               markets.remove(exchange); //Use itr.remove() method
           }
       }
           
    }    
  
}

interface StockExchange{
    public boolean isClosed();
}

// 输出

Exception in thread "main" java.util.ConcurrentModificationException
        at java.util.AbstractList$Itr.checkForComodification(AbstractList.java:372)
        at java.util.AbstractList$Itr.next(AbstractList.java:343)
        at ObjectRemovalTest.main(StringReplace.java:63)
Java Result: 1
```

59) 如何编写我们自己的容器类，并可以在for-each循环中使用？

是的，你可以编写自己的容器类。 如果你想使用Java中的高级for循环你需要实现Iterable接口。 如果你实现Collection接口，那么默认你就会获得该能力。

60) ArrayList 和 HashMap 的默认大小是？

从Java7开始，ArrayList的默认大小是10，HashMap的默认容量是16，并且不是2的N次幂。

```java
// from ArrayList.java JDK 1.7
private static final int DEFAULT_CAPACITY = 10;  

//from HashMap.java JDK 7
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
```

61) 两个不等的对象可能具有相同的哈希码吗?

当然了

62) 两个相等的对象有不同的哈希码吗？

不可以，根据hash code的契约这是不可能的。

63) 是否可以在hashCode方法中返回随机的值？

不可以，同一个对象的哈希码应该总是相同的。

64) Comparator 和 Comparable的区别？

Comparable接口用于定义对象的自然顺序，而Comparator用于自定义顺序。 Comparable可以总是一个，但我们可以有多个比较器来定义对象的比较顺序。

65) 为什么在复写equals方法时需要同时复写hashCode方法？

因为equals有代码约定，重写equals和hashcode需在一起。因为许多容器类像HashMap或HashSet取决于hashcode和equals契约。

### Java IO 和 NIO 面试问题

从Java面试的角度来看，IO非常重要。 您应该对旧的Java IO，NIO和NIO2非常了解， 并了解一些操作系统和磁盘IO基础知识。 这里有一些Java IO的常见问题。

66) 在我的程序中，我有三个socket, 那我应该使用多少个线程来处理它们呢？

67) 在Java中如何创建 ByteBuffer?

答案是：wrap, allocate, allocateDirect

68) 在Java中如何读取或写入ByteBuffer?

69) Java使用的时大端还是小端字节序？

答案是：大端。 ByteOrder.nativeOrder()可以查看主机的字节序 ； 网络字节序也是大端。

70) ByteBuffer 的字节序是什么样的?

答案是：大端

71) Java中`direct buffer` 和 `non-direct buffer` 的区别是？

http://javarevisited.blogspot.jp/2015/08/difference-between-direct-non-direct-mapped-bytebuffer-nio-java.html

72) 什么是Java中的内存映射缓存？

http://javarevisited.blogspot.sg/2012/01/memorymapped-file-and-io-in-java.html

73) socket的选项`TCP NO DELAY`是什么意思?

[ java socket参数详解:TcpNoDelay](http://blog.csdn.net/huang_xw/article/details/7340241)

74) What is the difference between TCP and UDP protocol?

http://javarevisited.blogspot.jp/2014/07/9-difference-between-tcp-and-udp-protocol.html

75) The difference between ByteBuffer and StringBuffer in Java?

答案：者两个可以比较吗？


### java 最佳实践面试题

包含来自Java编程不同部分的最佳实践。 集合，字符串，IO，多线程，错误和异常处理，设计模式等。本节主要面向负责产品的Java开发人员，技术主管，VP，团队领导和编程人员。 如果你想创造优质的产品，你必须知道并遵循最佳实践。

76) 编写多线程代码时你所遵循的最佳实践是什么[answer](http://javarevisited.blogspot.com/2015/05/top-10-java-multithreading-and.html)？

下面是我在编写多线程代码是遵循的最佳实践：

a) 总是给线程进行命名，这样在调试时非常有帮助。
b) 最小化同步范围，不要同步整个方法，而是应该只同步临界区代码块。
c) 如果可以的话优先使用 volatile 而不是 synchronized
e) 使用高级别并发工具类来实现线程间的通信。例如BlockingQueue, CountDownLatch and Semeaphore.
e) 优先使用并发集合而不是同步集合。它们提供了更好的可扩展性。

77) 告诉我一些你使用Java集合框架的最佳实践？

下面是我使用Java集合框架的最佳实践：

a) 总是使用合适的集合类。例如：如果你需要非同步的list，你应该使用ArrayList而不是Vector。
b) 优先使用并发集合而不是同步结合类。并发集合有更好的扩展性。
c) 总是使用接口类型来引用类的实例。例如：List , Map等
d) 使用迭代器遍历集合元素。
e) 总是使用泛型集合。

78) 告诉我在Java中你使用线程的5条最佳实践？

a) 命名你的线程。
b) 保持你的任务和线程的分离，使用 Runnable 或 Callable 配和executor一起。
c) 使用线程池。
d) 使用 volatile 来指示编译器处理重拍序，内存可见性和原子性问题。
e) 编码使用ThreadLocal,因为不正确的使用方式会导致内存泄露。

79) 说出 5 条 IO 相关的最佳实践？

IO对于Java应用程序的性能非常重要。 理想情况下，应在应用程序的关键路径避免IO。 以下是你可以遵循的Java IO最佳做法：

a) 使用带有缓存区的IO类，而不是每次处理一个字节或字符的类。
b) 使用NIO或NIO2里面的类
c) 总是在finally块中关闭流，或者使用 try-with-resource 语句。
d) 使用内存映射文件来提高IO速度。

如果Java候选人不知道IO和NIO，尤其是如果他有至少2到4年的经验，那他需要阅读一些这方面的资料。

80) 说出5个你使用JDBC时遵循的最佳实践？

另一个对有7到8年的经验的Java开发人员来说的最佳实践。 为什么它很重要？ 因为它们是在代码中设定趋势并教育初级开发人员的。 有很多最佳做法，你可以根据你的confort和convinners命名。 这里有一些更常见的：

a) 使用批量语句来insert或update数据
b) 使用预编译SQL来避免SQL异常和提高效率
c) 使用数据库连接池
d) 使用列名来访问结果集而不是下标索引。
e) 不要使用用户的输入数据来拼接SQL，避免SQL注入。


81) 说出Java中方法重载的最佳实践？

以下是在Java中重载方法时可以遵循的一些最佳做法，以避免与自动装箱混淆：

a) 不要重载一个参数类型是`int`,另一个参数类型是`Integer`的方法。
b) 不要重载方法，其中参数数量相同，只有参数的顺序不同。
c) 在重载方法时，参数数量超过5个请使用可变参数。


### 日期，时间，日历相关Java面试题

82) SimpleDateFormat 是多线程安全的吗？

不幸的是，DateFormat及其所有的实现，包括SimpleDateFormat不是线程安全的，因此不应该在多线程程序中使用，直到应用外部线程安全措施。 将SimpleDateFormat对象绑定到ThreadLocal变量中。 如果你不这样做，你将得到一个不正确的结果，在解析或格式化日期在Java。 虽然，对于所有实际日期时间的目的，我强烈建议joda时间图书馆。

83) 在Java中你如何格式化一个日期对象？

在Java中你可以使用`SimpleDateForma` 和 `joda-time`库。DateFormat 类允许你已各种流行的方式格式化日期。

84) 如何在Java中的格式化日期中显示时区? ([answer](http://java67.blogspot.sg/2013/01/how-to-format-date-in-java-simpledateformat-example.html))

85) Java中`java.util.Date` 和  `java.sql.Date` 有什么不同？[answer](http://java67.blogspot.sg/2014/02/how-to-convert-javautildate-to-javasqldate-example.html)

86) Java中如何计算两个日期的差异? ([program](http://javarevisited.blogspot.sg/2015/07/how-to-find-number-of-days-between-two-dates-in-java.html))

87) Java中如何将字符串(YYYYMMDD) 转为Date类型？[code](http://java67.blogspot.sg/2014/12/string-to-date-example-in-java-multithreading.html)

答案：DateFormat.parse


### JUnit 单元测试面试题

89) 你如何测试静态方法? (answer)

在Java中可以使用PowerMock库来测试静态方法。

90) How to do you test a method for an exception using JUnit? (answer)

91) 你曾经使用哪个单元测试库来测试Java程序? ([answer]())

92) @Before 和 @BeforeClass 注解的区别是? ([answer](http://javarevisited.blogspot.sg/2013/04/JUnit-tutorial-example-test-exception-thrown-by-java-method.html))

### 编程代码面试 Questions

93) 如何测试一个字符串只包含数字字符？([solution](http://java67.blogspot.com/2014/01/java-regular-expression-to-check-numbers-in-String.html))

94) How to write LRU cache in Java using Generics? (answer)

95) Write a Java program to convert bytes to long? (answer)

96) How to reverse a String in Java without using StringBuffer? (solution)

97) How to find the word with the highest frequency from a file in Java? (solution)

98) How do you check if two given String are anagrams? (solution)

99) How to print all permutation of a String in Java? (solution)

100) How do you print duplicate elements from an array in Java? (solution)

101) How to convert String to int in Java? (solution)

102) How to swap two integers without using temp variable? (solution)

###  OOP 和 Design Patterns

It contains Java Interview questions from SOLID design principles, OOP fundamentals e.g. class, object, interface, Inheritance, Polymorphism, Encapsulation, and Abstraction as well as more advanced concepts like Composition, Aggregation, and Association. It also contains questions from GOF design patterns.

103) 什么是接口? 如果你不能编写任何的具体逻辑时你为什么使用接口？

接口是用来定义API的。 它告诉你如何使用它的契约。它还支持抽象，因为客户端可以使用接口方法来引用多个实现。 通过使用List接口，您可以利用ArrayList的随机访问以及LinkedList的灵活插入和删除。 该接口不允许你编写代码保持事物抽象，但从Java 8，你可以声明静态和默认方法内部接口是具体的。

104) Java中抽象类和接口有哪些不同?[answer](http://javarevisited.blogspot.sg/2013/05/difference-between-abstract-class-vs-interface-java-when-prefer-over-design-oops.html)

在Java中的抽象类和接口之间有多个区别，但最重要的是Java对允许一个类只扩展一个类但允许它实现多个接口的限制。 抽象类很好地定义类的默认行为，但是接口很好定义类型，稍后用于实现多态性。 

105) Which design pattern have you used in your production code? apart from Singleton?

This is something you can answer from your experience. You can generally say about dependency injection, factory pattern, decorator pattern or observer pattern, whichever you have used. Though be prepared to answer follow-up question based upon the pattern you choose.

106) Can you explain Liskov Substitution principle? (answer)

This is one of the toughest questions I have asked in Java interviews. Out of 50 candidates, I have almost asked only 5 have managed to answer it. I am not posting an answer to this question as I like you to do some research, practice and spend some time to understand this confusing principle well.

107) What is Law of Demeter violation? Why it matters? (answer)

Believe it or not, Java is all about application programming and structuring code. If  you have good knowledge of common coding best practices, patterns and what not to do than only you can write quality code.  Law of Demeter suggests you "talk to friends and not stranger", hence used to reduce coupling between classes.

108) What is Adapter pattern? When to use it?

Another frequently asked Java design pattern questions. It provides interface conversion. If your client is using some interface but you have something else, you can write an Adapter to bridge them together. This is good for Java software engineer having 2 to 3 years experience because the question is neither difficult nor tricky but requires knowledge of OOP design patterns.

109) What is "dependency injection" and "inversion of control"? Why would someone use it? (answer)

110) What is an abstract class? How is it different from an interface? Why would you use it? (answer)

One more classic question from Programming Job interviews, it is as old as chuck Norris. An abstract class is a class which can have state, code and implementation, but an interface is a contract which is totally abstract. Since I have answered it many times, I am only giving you the gist here but you should read the article linked to answer to learn this useful concept in much more detail.

111) Which one is better constructor injection or setter dependency injection? (answer)

Each has their own advantage and disadvantage. Constructor injection guaranteed that class will be initialized with all its dependency, but setter injection provides flexibility to set an optional dependency. Setter injection is also more readable if you are using an XML file to describe dependency. Rule of thumb is to use constructor injection for mandatory dependency and use setter injection for optional dependency.

112) What is difference between dependency injection and factory design pattern? (answer)

Though both patterns help to take out object creation part from application logic, use of dependency injection results in cleaner code than factory pattern. By using dependency injection, your classes are nothing but POJO which only knows about dependency but doesn't care how they are acquired. In the case of factory pattern, the class also needs to know about factory to acquire dependency. hence, DI results in more testable classes than factory pattern. Please see the answer for a more detailed discussion on this topic.

113) Difference between Adapter and Decorator pattern? (answer)

Though the structure of Adapter and Decorator pattern is similar, the difference comes on the intent of each pattern. The adapter pattern is used to bridge the gap between two interfaces, but Decorator pattern is used to add new functionality into the class without the modifying existing code.

114) Difference between Adapter and Proxy Pattern? (answer)

Similar to the previous question, the difference between Adapter and Proxy patterns is in their intent. Since both Adapter and Proxy pattern encapsulate the class which actually does the job, hence result in the same structure, but Adapter pattern is used for interface conversion while the Proxy pattern is used to add an extra level of indirection to support distribute, controlled or intelligent access.

115) What is Template method pattern? (answer)

Template pattern provides an outline of an algorithm and lets you configure or customize its steps. For examples, you can view a sorting algorithm as a template to sort object. It defines steps for sorting but let you configure how to compare them using Comparable or something similar in another language. The method which outlines the algorithms is also known as template method.

116) When do you use Visitor design pattern? (answer)

The visitor pattern is a solution of problem where you need to add operation on a class hierarchy but without touching them. This pattern uses double dispatch to add another level of indirection.

117) When do you use Composite design pattern? (answer)

Composite design pattern arranges objects into tree structures to represent part-whole hierarchies. It allows clients treat individual objects and container of objects uniformly. Use Composite pattern when you want to represent part-whole hierarchies of objects.

118) The difference between Inheritance and Composition? (answer)

Though both allows code reuse, Composition is more flexible than Inheritance because it allows you to switch to another implementation at run-time. Code written using Composition is also easier to test than code involving inheritance hierarchies.

119) Describe overloading and overriding in Java? (answer)

Both overloading and overriding allow you to write two methods of different functionality but with the same name, but overloading is compile time activity while overriding is run-time activity. Though you can overload a method in the same class, but you can only override a method in child classes. Inheritance is necessary for overriding.

120) The difference between nested public static class and a top level class in Java? (answer)

You can have more than one nested public static class inside one class, but you can only have one top-level public class in a Java source file and its name must be same as the name of Java source file.

121) Difference between Composition, Aggregation and Association in OOP? (answer)

If two objects are related to each other, they are said to be associated with each other. Composition and Aggregation are two forms of association in object-oriented programming. The composition is stronger association than Aggregation. In Composition, one object is OWNER of another object while in Aggregation one object is just USER of another object. If an object A is composed of object B then B doesn't exist if A ceased to exists, but if object A is just an aggregation of object B then B can exists even if A ceased to exist.

122) Give me an example of design pattern which is based upon open closed principle? (answer)

This is one of the practical questions I ask experienced Java programmer. I expect them to know about OOP design principles as well as patterns. Open closed design principle asserts that your code should be open for extension but closed for modification. Which means if you want to add new functionality, you can add it easily using the new code but without touching already tried and tested code.  There are several design patterns which are based upon open closed design principle e.g. Strategy pattern if you need a new strategy, just implement the interface and configure, no need to modify core logic. One working example is Collections.sort() method which is based on Strategy pattern and follows the open-closed principle, you don't modify sort() method to sort a new object, what you do is just implement Comparator in your own way.

123) Difference between Abstract factory and Prototype design pattern? (answer)

This is the practice question for you, If you are feeling bored just reading and itching to write something, why not write the answer to this question. I would love to see an example the, which should answer where you should use the Abstract factory pattern and where is the Prototype pattern is more suitable.

124) When do you use Flyweight pattern? (answer)

This is another popular question from the design pattern. Many Java developers with 4 to 6 years of experience know the definition but failed to give any concrete example. Since many of you might not have used this pattern, it's better to look examples from JDK. You are more likely have used them before and they are easy to remember as well. Now let's see the answer.
Flyweight pattern allows you to share object to support large numbers without actually creating too many objects. In order to use Flyweight pattern, you need to make your object Immutable so that they can be safely shared. String pool and pool of Integer and Long object in JDK are good examples of Flyweight pattern.

### Miscellaneous Java Interview Questions

It contains XML Processing in Java Interview question, JDBC Interview question, Regular expressions Interview questions, Java Error and Exception Interview Questions, Serialization,

125) The difference between nested static class and top level class? (answer)

One of the fundamental questions from Java basics. I ask this question only to junior Java developers of 1 to 2 years of experience as it's too easy for an experience Java programmers. The answer is simple, a public top level class must have the same name as the name of the source file, there is no such requirement for nested static class. A nested class is always inside a top level class and you need to use the name of the top-level class to refer nested static class e.g. HashMap.Entry is a nested static class, where HashMap is a top level class and Entry is nested static class.

126) Can you write a regular expression to check if String is a number? (solution)

If you are taking Java interviews then you should ask at least one question on the regular expression. This clearly differentiates an average programmer with a good programmer. Since one of the traits of a good developer is to know tools, regex is the best tool for searching something in the log file, preparing reports etc. Anyway, answer to this question is, a numeric String can only contain digits i.e. 0 to 9 and + and - sign that too at start of the String, by using this information you can write following regular expression to check if given String is number or not

127) The difference between checked and unchecked Exception in Java? (answer)

checked exception is checked by the compiler at compile time. It's mandatory for a method to either handle the checked exception or declare them in their throws clause. These are the ones which are a sub class of Exception but doesn't descend from RuntimeException. The unchecked exception is the descendant of RuntimeException and not checked by the compiler at compile time. This question is now becoming less popular and you would only find this with interviews with small companies, both investment banks and startups are moved on from this question.

128) The difference between throw and throws in Java? (answer)

the throw is used to actually throw an instance of java.lang.Throwable class, which means you can throw both Error and Exception using throw keyword e.g.

throw new IllegalArgumentException("size must be multiple of 2")

On the other hand, throws is used as part of method declaration and signals which kind of exceptions are thrown by this method so that its caller can handle them. It's mandatory to declare any unhandled checked exception in throws clause in Java. Like the previous question, this is another frequently asked Java interview question from errors and exception topic but too easy to answer.

129) The difference between Serializable and Externalizable in Java? (answer)

This is one of the frequently asked questions from Java Serialization. The interviewer has been asking this question since the day Serialization was introduced in Java, but yet only a few good candidate can answer this question with some confidence and practical knowledge. Serializable interface is used to make Java classes serializable so that they can be transferred over network or their state can be saved on disk, but it leverages default serialization built-in JVM, which is expensive, fragile and not secure. Externalizable allows you to fully control the Serialization process, specify a custom binary format and add more security measure.

130) The difference between DOM and SAX parser in Java? (answer)

Another common Java question but from XML parsing topic. It's rather simple to answer and that's why many interviewers prefers to ask this question on the telephonic round. DOM parser loads the whole XML into memory to create a tree based DOM model which helps it quickly locate nodes and make a change in the structure of XML while SAX parser is an event based parser and doesn't load the whole XML into memory. Due to this reason DOM is faster than SAX but require more memory and not suitable to parse large XML files.

131) Tell me 3 features introduced on JDK 1.7? (answer)

This is one of the good questions I ask to check whether the candidate is aware of recent development in Java technology space or not. Even though JDK 7 was not a big bang release like JDK 5 or JDK 8, it still has a lot of good feature to count on e.g. try-with-resource statements, which free you from closing streams and resources when you are done with that, Java automatically closes that. Fork-Join pool to implement something like the Map-reduce pattern in Java. Allowing String variable and literal into switch statements. Diamond operator for improved type inference, no need to declare generic type on the right-hand side of variable declaration anymore, results in more readable and succinct code. Another worth noting feature introduced was improved exception handling e.g. allowing you to catch multiple exceptions in the same catch block.

132) Tell me 5 features introduced in JDK 1.8? (answer)

This is the follow-up question of the previous one. Java 8 is path breaking release in Java's history, here are the top 5 features from JDK 8 release
Lambda expression, which allows you pass an anonymous function as object.
Stream API, take advantage of multiple cores of modern CPU and allows you to write succinct code.
Date and Time API, finally you have a solid and easy to use date and time library right into JDK
Extension methods, now you can have static and default method into your interface
Repeated annotation, allows you apply the same annotation multiple times on a type

133) What is the difference between Maven and ANT in Java? (answer)

Another great question to check the all round knowledge of Java developers. It's easy to answer questions from core Java but when you ask about setting things up, building Java artifacts, many Java software engineer struggles. Coming back to the answer of this question, Though both are build tool and used to create Java application build, Maven is much more than that. It provides standard structure for Java project based upon "convention over configuration" concept and automatically manage dependencies (JAR files on which your application is dependent) for Java application. Please see the answer for more differences between Maven and ANT tool.

That's all guys, lots of Java Interview questions? isn't it? I am sure if you can answer this list of Java questions you can easily crack any core Java or advanced Java interview. Though I have not included questions from Java EE or J2EE topics e.g. Servlet, JSP, JSF, JPA, JMS, EJB or any other Java EE technology or from major web frameworks like Spring MVC, Struts 2.0, Hibernate or both SOAP and RESTful web services, it's still useful for Java developers preparing for Java web developer position, because every Java interview starts with questions from fundamentals and JDK API. If you think, I have missed any popular Java question here and you think it should be in this list then feel free to suggest me. My goal is to create the best list of Java Interview Questions with latest and greatest question from recent interviews.

### Related Java EE Interview Questions

For my Java EE friends, here are web development specific questions, which you can use to prepare for JEE part:
Top 10 Spring Framework Interview Questions with Answers (see here)
10 Great XML Interview Questions for Java Programmers (read here)
20 Great Java Design Pattern Questions asked on Interviews (see here)
10 popular Struts Interview Questions for Java developers (list)
20 Tibco Rendezvous and EMS Interview Questions (read more)
10 frequently asked Servlet Interview Questions with Answers (see here)
20 jQuery Interview Questions for Java Web Developers (list)
10 Great Oracle Interview Questions for Java developers (see here)
Top 10 JSP Questions  from J2EE Interviews (read here)
12 Good RESTful Web Services Questions from Interviews (read here)
Top 10 EJB Interview Questions and Answers (see here)
Top 10 JMS and MQ Series Interview Questions and Answers (list)
10 Great Hibernate Interview Questions for Java EE developers (see here)
10 Great JDBC Interview Questions for Java Programmers (questions)
15 Java NIO and Networking Interview Questions with Answers (see here)
Top 10 XSLT Interview Questions with Answers (read more)
15 Data Structure and Algorithm Questions from Java Interviews (read here)
Top 10 Trick Java Interview Questions and Answers (see here)
Top 40 Core Java Phone Interview Questions with answers (list)

### Recommended Books for Java Programmers

If you are looking for some goods to prepare for your Java Interviews, You can take a look at following books to cover both theory and coding questions:
9 Must Read Books for Java Developers ([see book](https://javarevisited.blogspot.com/2013/01/top-5-java-programming-books-best-good.html))
5 Java Performance Tuning Books for Experienced Programmers ([see book](https://javarevisited.blogspot.com/2014/07/top-5-java-performance-tuning-books.html))
5 Good Books for Java JEE Interviews ([see book](https://javarevisited.blogspot.com/2015/12/5-good-books-for-java-jee-programming.html))
5 Books to learn Data Structure and Algorithms ([see book](https://javarevisited.blogspot.com/2015/07/5-data-structure-and-algorithm-books-best-must-read.html))
5 Hibernate Books for Java Developers ([see book](https://javarevisited.blogspot.com/2014/01/top-5-hibernate-books-for-java-programmers-learning.html))
5 Spring Books for Java Developers ([see book](https://javarevisited.blogspot.com/2013/03/5-good-books-to-learn-spring-framework-mvc-java-programmer.html))

### References

The Java Virtual Machine Spefication ([see here](https://docs.oracle.com/javase/specs/jvms/se7/html/))
The Java language Specification (Java SE 8) ([see here](https://docs.oracle.com/javase/specs/jls/se8/html/index.html))
Java SE 8 API Specification ([see here](https://docs.oracle.com/javase/8/docs/api/))



