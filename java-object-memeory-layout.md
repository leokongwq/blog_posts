---
layout: post
comments: true
title: Java 对象内存布局
date: 2019-12-14 09:22:49
tags:
 - Java
categories:
---

### 前言

所有试验代码都是在Mac os JDK8 64位JVM上测试（默认开启了指针压缩）。

### 背景

一道面试题。 问题是：A和B两个类，A类中有一个private的字段age，B类继承自A类。创建一个B类的对象b，对象b的内存中是否包含父类A中的字段age的内存空间。

类似代码如下：

```java
/**
 * @author jiexiu
 * created 2019/12/14 - 09:26
 */
public class Animal {

    private int age;

    public int getAge() {
        return age;
    }
}
/**
 * @author jiexiu
 * created 2019/12/14 - 09:26
 */
public class Dog extends Animal {

    private double weight;

}
```

<!-- more -->

这个问题刚开始你一听可能觉得很懵，父类的private字段子类也不能**直接**访问，那么在子类对象的内存空间中还有分配的必要吗？但是你静下心来推敲一下，答案是不言自明的。可以通过反正法来证明。

1. 通过Java反射API，子类中也是可以访问到父类中声明的private字段的。如果子类对象的内存中没有这部分内存内容，那么JVM该从哪里去找呢？不能无中生有的。
2. 父类的字段的是private的，但是父类中提供了public方法来访问声明的private字段。

问题回答完了。但是该问题的引深问题是Java对象占用内存是如何分配的。

### Java对象内存布局

一个Java对象在内存中有三部分组成：1，对象头。2，实例数据。3，内存填充。

看下面一张图：

{% asset_img java-object-memory-layout.jpg %}


#### 对象头

- Mark Word：包含一系列的标记位，比如轻量级锁的标记位，偏向锁标记位等等。在32位系统占4字节，在64位系统中占8字节；
- Class Pointer：用来指向对象对应的Class对象（其对应的元数据对象）的内存地址。在32位系统占4字节，在64位系统中占8字节(如果在64位JVM上开启了压缩指针，那么占用4个字节)；
- Length：如果是数组对象，还有一个保存数组长度的空间，占4个字节；

32位的对象头

{% asset_img java-object-header-32bit.jpg %}

64位的对象头

{% asset_img java-object-header-64bit.jpg %}

小结：

1. 32 位JVM上，非数组对象头占用8个字节，数组对象头占用12个字节。
2. 64 为JVM上，非数组对象头占用16个字节，数组对象占用20个字节。如果开启压缩指针的话，非数组对象头占用12个字节（8 + 4），数组对象头占用16个字节（8 + 4 + 4）。

#### 对象大小

上面了解了对象头的组成和大小信息。下面就了解下对象的大小和内存布局。

先看一个例子：

```java
/**
 * @author jiexiu
 * created 2019/12/14 - 10:15
 */
public class EmptyObject {
}

EmptyObject emptyObject = new EmptyObject();
```

`emptyObject` 大小是多大呢？

答案和分析：没有任何字段，父类是Object类，也没有任何字段。那么它的大小在64JVM开启指针压缩的情况下是 8 + 4 = 12 字节。 理论上没有问题，但是实际上JVM为了内存对齐，还有4字节的填充，所以总的大小是16字节。

测试代码：

```java
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.9</version>
</dependency>

System.out.println(ClassLayout.parseClass(EmptyObject.class).toPrintable());
```

输出结果：

```
com.leokongwq.java.jvm.EmptyObject object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0    12        (object header)                           N/A
     12     4        (loss due to the next object alignment) 内存对齐
Instance size: 16 bytes // 总大小
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

使用文章开头定义的`Animal`类再次试验，输出如下：

```
com.leokongwq.java.jvm.Animal object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0    12        (object header)                           N/A
     12     4    int Animal.age                                N/A
Instance size: 16 bytes
```

因为 age是int类型，占用4个字节。和对象头加起来刚好是8字节的整数倍，不需要填充。

如果我们把age的类型定义为long，输出结果如下：

```
com.leokongwq.java.jvm.Animal object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0    12        (object header)                           N/A
     12     4        (alignment/padding gap)                  
     16     8   long Animal.age                                N/A
Instance size: 24 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total
```

结果表明还是有4字节的填充。

在`age`字段后面再定义一个4字节的字段，再次测试。

```java
public class Animal {

    private long age;

    private float weight;

    public long getAge() {
        return age;
    }
}
```

输出结果如下：

```
com.leokongwq.java.jvm.Animal object internals:
 OFFSET  SIZE    TYPE DESCRIPTION                               VALUE
      0    12         (object header)                           N/A
     12     4   float Animal.weight                             N/A
     16     8    long Animal.age                                N/A
Instance size: 24 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

从这次结果可以看出来，JVM会对字段进行排序，尽可能的利用内存，减少padding。

#### 一个复杂的内存布局

```java
/**
 * @author jiexiu
 * created 2019/12/14 - 09:26
 */
public class Animal {

    private int height;
    private int age;

    public long getAge() {
        return age;
    }
}
/**
 * @author jiexiu
 * created 2019/12/14 - 09:26
 */
public class Dog extends Animal {

    private int f1;

    private char f2;

    private double weight;

    private boolean f3;

    private Object object;

    private byte f4;
}
```

运行测试代码，结果如下：

```
com.leokongwq.java.jvm.Dog object internals:
 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
      0    12                    (object header)                           N/A
     12     4                int Animal.height                             N/A
     16     4                int Animal.age                                N/A
     20     4                int Dog.f1                                    N/A
     24     8             double Dog.weight                                N/A
     32     2               char Dog.f2                                    N/A
     34     1            boolean Dog.f3                                    N/A
     35     1               byte Dog.f4                                    N/A
     36     4   java.lang.Object Dog.object                                N/A
Instance size: 40 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

观察数据总结结论：

1. 任何对象的大小都是8个字节的整数倍，不够的进行padding。目的是为了提高CPU的工作效率。
2. 实例域按照如下优先级进行排列：长整型和双精度类型；整型和浮点型；字符和短整型；字节类型和布尔类型，最后是引用类型。这些实例域都按照各自的单位对齐。目的是为了减少padding。
3. 不同类继承关系中的实例域不能混合排列。首先按照规则2处理父类中的实例域，接着才是子类的实例域
4. 当父类中最后一个成员和子类第一个成员的间隔如果不够4个字节的话，就必须扩展到4个字节的基本单位。
5. 如果子类第一个实例域是一个双精度或者长整型，并且父类并没有用完8个字节，JVM会破坏规则2，按照整形（int），短整型（short），字节型（byte），引用类型（reference）的顺序，向未填满的空间填充。

#### 回到面试题

测试Dog类的内存布局，结果如下：

```java
com.leokongwq.java.jvm.Dog object internals:
 OFFSET  SIZE     TYPE DESCRIPTION                               VALUE
      0    12          (object header)                           N/A
     12     4      int Animal.age                                N/A
     16     8   double Dog.weight                                N/A
Instance size: 24 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

结果一目了然，父类Animal的private字段age也分配了内存。

### 题外话-Java对象全部分配在堆上吗？

答案1：全部分配在堆上。没问题，可以这么设计，但是需要考虑分配效率问题和GC压力。
答案2：也可能分配在栈上。当开启逃逸分析`-XX:+DoEscapeAnalysis`时，方法内部分配的对象完全可以在栈上分配，只要对象引用不会逸出到方法外面。方法调用结束，内存释放，GC压力也没有了。

做个试验：

```java
-Xmx10M -XX:+PrintGCDetails -XX:+DoEscapeAnalysis

/**
 * @author jiexiu
 * created 2019/12/14 - 09:26
 */
public class Dog extends Animal {

//    private byte[] _M = new byte[1024 * 1024];

    private int f1;

    private char f2;

    private double weight;

    private boolean f3;

    private Object object;

    private byte f4;
}

public static void main(String[] args) {
    System.out.println(ClassLayout.parseClass(Dog.class).toPrintable());
    System.out.println("********************");
    long startTime = System.currentTimeMillis();
    for (int i = 0; i < 100000000; i++) {
        new Dog();
    }
    System.out.println("take time:" + (System.currentTimeMillis() - startTime) + "ms");
}
```

输出如下：

```
com.leokongwq.java.jvm.Dog object internals:
 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
      0    12                    (object header)                           N/A
     12     4                int Animal.height                             N/A
     16     4                int Animal.age                                N/A
     20     4                int Dog.f1                                    N/A
     24     8             double Dog.weight                                N/A
     32     2               char Dog.f2                                    N/A
     34     1            boolean Dog.f3                                    N/A
     35     1               byte Dog.f4                                    N/A
     36     4   java.lang.Object Dog.object                                N/A
Instance size: 40 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

********************
[GC (Allocation Failure) [PSYoungGen: 1631K->512K(2048K)] 3217K->2297K(9216K), 0.0006654 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 1535K->224K(2048K)] 3321K->2209K(9216K), 0.0012281 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 1247K->0K(2048K)] 3233K->2193K(9216K), 0.0004912 secs] [Times: user=0.00 sys=0.01, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 1023K->96K(2048K)] 3217K->2289K(9216K), 0.0003646 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 1119K->0K(2048K)] 3313K->2217K(9216K), 0.0009835 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 1024K->0K(2048K)] 3241K->2217K(9216K), 0.0004940 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 1024K->0K(2048K)] 3241K->2217K(9216K), 0.0003785 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 1024K->0K(2048K)] 3241K->2217K(9216K), 0.0003809 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 1024K->0K(2048K)] 3241K->2217K(9216K), 0.0003755 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 1024K->0K(2048K)] 3241K->2217K(9216K), 0.0003896 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[GC (Allocation Failure) [PSYoungGen: 1024K->0K(2048K)] 3241K->2217K(9216K), 0.0003721 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 1024K->0K(2048K)] 3241K->2217K(9216K), 0.0003373 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 1024K->0K(2048K)] 3241K->2217K(9216K), 0.0003653 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
take time:13ms
Heap
 PSYoungGen      total 2048K, used 350K [0x00000007bfd00000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 1024K, 34% used [0x00000007bfd00000,0x00000007bfd57930,0x00000007bfe00000)
  from space 1024K, 0% used [0x00000007bff00000,0x00000007bff00000,0x00000007c0000000)
  to   space 1024K, 0% used [0x00000007bfe00000,0x00000007bfe00000,0x00000007bff00000)
 ParOldGen       total 7168K, used 2217K [0x00000007bf600000, 0x00000007bfd00000, 0x00000007bfd00000)
  object space 7168K, 30% used [0x00000007bf600000,0x00000007bf82a460,0x00000007bfd00000)
 Metaspace       used 8039K, capacity 8280K, committed 8576K, reserved 1056768K
  class space    used 950K, capacity 1032K, committed 1152K, reserved 1048576K
```

耗时： 13毫秒，分配了 40byte * 100000000 ≈ 4G。 很明显如果是在10M的堆上进行分配，不能够这么快，GC日志输出也不是如此。

结论：

1. 不会溢出的对象完全没有再堆上分配的必要，可以减少内存分配的竞争，减轻GC的压力。
2. 不是所有的对象都在栈上分配。因为栈的大小是有限的，例如1M的栈大小怎么能分配2M大小的对象呢？
3. 如果把Dog类中的1M大小的字节数组字段打开注释，重新测试，内存分配就没有这么快了。

### 作业题

1. 访问一个对象的字段，JVM是如何定位对象字段的内存地址呢？

### 参考资料

[聊聊java对象内存布局](https://zhuanlan.zhihu.com/p/50984945)
[jol](http://openjdk.java.net/projects/code-tools/jol/)
[https://www.zhihu.com/question/51920553](https://www.zhihu.com/question/51920553)
[一个Java对象到底占多少内存](https://juejin.im/post/5d0fa403f265da1bb67a2335)



