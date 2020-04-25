---
layout: post
comments: true
title: 诡异的java面试题TOP10
date: 2017-02-22 22:23:15
tags:
- 面试
categories:
- java
---

本文翻译自：

### Question: What does the following Java program print?

```java
public class Test { 
    public static void main(String[] args){ 
        System.out.println(Math.min(Double.MIN_VALUE, 0.0d)); 
    } 
}
```

答案: 这问题的迷点在于它不像`Integer`一样最小值是负数， `Double`的`MAX_VALUE `和 `MIN_VALUE`都是正数。`Double.MIN_VALUE` 是 `2^(-1074)`。

<!-- more -->

### Question: What do the expression 1.0 / 0.0 will return? will it throw Exception? any compile time error?

答案: 这是另一个和`Double`相关的诡异问题。虽然Java开发人员知道`double`基本类型和`Double`包装类，但是在做浮点算术时，他们没有特别注意`Double.INFINITY`，`NaN`和`-0.0`以及其它涉及它们的算术计算的规则。 这个问题的简单答案是，它不会抛出`ArithmeticExcpetion`并返回Double.INFINITY。

同样需要注意比较操作`x == Double.NaN`, 该表达式总是返回false,即使x是`NaN`。如果要判断x是否是`NaN`则需要使用`Double.isNaN(x)`。 如果你了解SQL,这和SQL中的NULL非常类似。

### Does Java support multiple inheritances?

这是Java中最棘手的问题，如果C++可以支持直接多继承，那为什么JAVA不支持。这是面试官经常会问的问题。这个问题的答案比它看起来更微妙，因为Java通过允许一个接口扩展其它接口支持多个继承的类型；什么Java不支持是多个实现的继承。 这种区别也变得模糊，因为Java 8的默认方法，现在提供Java类行为的多个继承。 查看[为什么Java中不支持多继承](http://javarevisited.blogspot.sg/2011/07/why-multiple-inheritances-are-not.html)来回答这个棘手的Java问题。

### What will happen if we put a key object in a HashMap which is already there?

这个棘手的问题是另一个常见的问题:**Java中的HashMap是如何工作**的一部分。因此在Java中，HashMap的是一个热门的话题，通常带来一些混乱和棘手的问题。这个问题的回答是，如果你又把相同的key放入HashMap，则它会取代旧的value，因为HashMap中不允许重复键。相同的keys将导致相同的hashCode，并位于桶中相同的位置。

每一个桶都是一个包含`Map.Entry`的链表，每一个`Map.Entry`包含了`key`和对应的`value`。在放入key-value对时，会遍历对应桶中的每一个`Key`对象，然后使用`equals`方法来比较是否和新的`Key`对象相同，如果返回true,则表示相同, 此时旧的`Value`会被替换为新的`Value`。


### Question: What does the following Java program print?

```java
public class Test { 
    public static void main(String[] args) throws Exception { 
        char[] chars = new char[] {'\u0097'}; 
        String str = new String(chars); 
        byte[] bytes = str.getBytes();
        System.out.println(Arrays.toString(bytes)); 
    } 
}
```

Answer: The trickiness of this question lies on character encoding and how String to byte array conversion works. In this program, we are first creating a String from a character array, which just has one character '\u0097', after that we are getting the byte array from that String and printing that byte. Since \u0097 is within the 8-bit range of byte primitive type, it is reasonable to guess that the str.getBytes() call will return a byte array that contains one element with a value of -105 ((byte) 0x97).

However, that's not what the program prints and that's why this question is tricky. As a matter of fact, the output of the program is operating system and locale dependent. On a Windows XP with the US locale, the above program prints [63], if you run this program on Linux or Solaris, you will get different values.

To answer this question correctly, you need to know about how Unicode characters are represented in Java char values and in Java strings, and what role character encoding plays in String.getBytes().

In simple word, to convert a string to a byte array, Java iterate through all the characters that the string represents and turn each one into a number of bytes and finally put the bytes together. The rule that maps each Unicode character into a byte array is called a character encoding. So It's possible that if same character encoding is not used during both encoding and decoding then retrieved value may not be correct. When we call str.getBytes() without specifying a character encoding scheme, the JVM uses the default character encoding of the platform to do the job.

The default encoding scheme is operating system and locale dependent. On Linux, it is UTF-8 and on Windows with a US locale, the default encoding is Cp1252. This explains the output we get from running this program on Windows machines with a US locale. No matter which character encoding scheme is used, Java will always translate Unicode characters not recognized by the encoding to 63, which represents the character U+003F (the question mark, ?) in all encodings.


### 如果父类中的一个方法抛出了**NullPointerException**异常，是否可以在子类中覆写该方法并抛出一个**RuntimeException**?

这是一个关于方法重载和重写的概念问题。答案是在方法重写中，你可以抛出运行时异常的父类类型异常，但如果是受检异常，则不可以这样做。具体可以参考：[Java中方法重写的规则](http://javarevisited.blogspot.sg/2011/12/method-overloading-vs-method-overriding.html) 。

### 在Java中，下面代码所示的实现`compareTo`方法有什么问题？

```
public int compareTo(Object o){ 
    Employee emp = (Employee) o; 
    return this.id - e.id; 
}
```

这里id是Integer类型。

See How to override compareTo method in Java for the complete answer of this Java tricky question for an experienced programmer.

如果你能保证ID一直是正整数，则该方法在Java中的实现没有任何问题。 该问题的诡异之处在于你不能保证ID拥有是正整数，一旦ID的值是负数，则会导致减法操作变为加法。结果有可能导致Integer范围溢出。


### 如何保证N各线下访问N个资源而不会导致死锁的发生？

If you are not well versed in writing multi-threading code then this is a real tricky question for you. This Java question can be tricky even for the experienced and senior programmer, who are not really exposed to deadlock and race conditions. The key point here is ordering, if you acquire resources in a particular order and release resources in the reverse order you can prevent deadlock. See how to avoid deadlock in Java for a sample code example.

### 思考下面的代码，初始化2个非`volatile`变量， 并且线程T1和线程T2以如下的逻辑修改变量，并且T1和T2都没有任何的同步

```java
int x = 0; 
boolean bExit = false; 
Thread 1 (not synchronized) 
x = 1; 
bExit = true;
Thread 2 (not synchronized) 
if (bExit == true) 
System.out.println("x=" + x);
```

告诉我们，线程T2是否会输出`x=0`?

答案: 关于Java多线程的一系列诡异问题是不能确定任何事情的。这是我能得到的最简单的结论。这个问题的答案是`Yes`。线程`T2`可能会输出`x=0`。为什么呢？ 因为没有任何的指令告诉编译器，例如`synchronized` 或 `volatile`。因为在编译器的重排序下，`bExit=true` 可能会被排在`x=1`前。 同时 `x=1`的结果也可能对线程`T2`是不可见的，因此线程`T2`可能会加载`x=0`。现在你该如何修复它？

我问过好多人这个问题，答案是各有不同。 有人猜测说可使用同一个互斥量来同步两个线程，也有人说将两个变量都变为`volatile`。 这个两个答案都是对的，因为它们都能实现：阻止重排序的发生，并且保证变量的可见性。

但最好的答案是：你只需要将`bExit`设为`volatile`， 这样线程`T2`就只能打印`x=1`了。`x`不需要是`volatile`，这是因为当`bExit`是`volatile`时，`x`不能被重排序到`bExit=true`后面。


### 在Java中 CyclicBarrier 和 CountDownLatch 有哪些异同？

相对来说比较新的有趣Java问题，仅在JDK5被引入。它们之间主要的区别是，你可以重用`CyclicBarrier`，即使屏障已经被打破，但是你不能重用`CountDownLatch`。可以参考[CyclicBarrier vs CountDownLatch in Java](http://java67.blogspot.sg/2012/08/difference-between-countdownlatch-and-cyclicbarrier-java.html)来了解它们之间更多的不同之处。

### 在Java中StringBuffer和StringBuilder有哪些不同？

经典的Java问题，该问题一部分人认为很狡猾，而另一部分人认为很简单。`StringBuilder`是在Java5引入的，它们之间唯一的不同是`StringBuffer`中类似`length()`,`capacity()`,`append()`都是`synchronized`的，而相对应的`StringBuiler`中的方法是非`synchronized`的。

### 你可以在一个静态上下文中访问一个非静态变量吗？

这是另一个Java基础中的令人感到棘手的问题。在Java中，你不可以在一个静态上下文中访问一个非静态变量。如果你尝试这么做，则会导致编译错误。其实这是一个Java初学者普遍会遇到的一个的问题，当他们试图在`main`方法中访问实例变量。因为`main`方法在Java中是静态的，而实例变量是非静态的，你不能`main`方法里面访问实例变量。参见[为什么你不能从静态方法访问非静态变量](http://javarevisited.blogspot.sg/2012/02/why-non-static-variable-cannot-be.html)详细了解这个棘手的Java问题。

### 下面的代码会创建多少个String对象？

```java
String s = new String("abc");
1) One
2) Two
3) Three
```

现在是练习时间，这里有一些问题，需要你们解答。这些问题都是这篇文章的阅读者提出了的，非常感谢他们的贡献。

1. 在Java中什么情况下一个单例类不再保持单例？
2. 是否可以通过2个`ClassLoader`加载同一个类？
3. 有没有两个内容相同的Java对象，进行`equals`而返回false？
4. 为什么compareTo()方法需要和equals()方法保持一致？
5. 针对equals和compareTo==0, 什么时候Double和BigDecimal会给出不同的答案？
6. "has before" 针对volatile是如何工作的?
7. 为什么`0.1 * 3`不等于`0.3`
8. 为什么（Integer）1 == (Integer) 1 但是 (Integer) 222 != (Integer) 222，哪一个命令行参数可以改变这个行为？
9. 当一个线程抛出异常会发生什么事情？
10. notify() 和 notifyAll() 调用的区别是什么?
11. System.exit() 和 System.halt() 方法的区别是什么?
12. 在Java中下面的代码是合法的吗？这是一个方法重载还是方法重写的例子？

    ```java
    public String getDescription(Object obj){ 
        return obj.toString; 
    } 
    public String getDescription(String obj){ 
        return obj; 
    } 
    public void getDescription(String obj){ 
        //错误
        return obj; 
    }
    ```

这就是我整理的Java中常见的令人感到棘手的问题。在Java核心或J2EE面试前准备一些这样的问题总不是件坏事情。在Java中总能碰见一两个开放式或令人棘手的面试问题。

#### 答案：

问题1：

- 单例类在多个JVM中
- 被多个不同的类加载器加载的时候
- 单例类被GC后，又重新加载
- 有目的的重新加载单例类
- 复制一个已经序列化和反序列化的类
可以参考：[when-is-a-singleton-not-a-singleton](http://www.javaworld.com/article/2074897/java-web-development/when-is-a-singleton-not-a-singleton-.html?page=2)

问题2：

可以

问题3：

这个当然了，默认的equals()方法是进行引用比较，new处理的2个对象是不同的应用，当然有可能是false。

问题4：

理论上，JDK中拥有compareTo方法的类是有可能与equals()方法拥有不一致的行为的，典型的类就是`BigDecimal`。单下面的例子说明了为什么这两个方法需要保持一致：

```java
public static void main(String[] args) {
    Map<String, String> brokenMap = new TreeMap<String, String> (new Comparator<String>() {

        @Override
        public int compare(String o1, String o2) {
            return 0;
        }
    });

    brokenMap.put("a", "a");
    brokenMap.put("b", "b");
    System.out.println("size: " + brokenMap.size());
    System.out.println("content: " + brokenMap);
}
```
输出：
```
size: 1
content: {a=b}
```

问题5：

这个问题上代码：
```java
 public static void main(String[] args) {
    // 针对equals
    System.out.println(new BigDecimal(0).equals(0.0));
    System.out.println(new Double(0).equals(0.0));
    //输出
    false
    true
    // compareTo
    System.out.println(new BigDecimal(0).compareTo(new BigDecimal(0.00)));
    System.out.println(new Double(0).compareTo(0.0));
    //输出
    0
    0

}
```
问题6：

问题7：

这个是应为`0.1`是浮点数，计算机不能精确的表示浮点数。



### 扩展阅读

如果你正在寻找一些非常有挑战的关于Java的编程问题，你应该阅读一本由`Joshua Bloch` 编写的另一本经典的书籍[Java解惑](http://www.amazon.com/dp/032133678X/?tag=javamysqlanta-20), 我敢保证解决这些问题你一点会觉得会有挑战。

如果你对Java面试题和对应的答案如饥似渴，你可以阅读下面列出的问题和答案。

- [18 Java design pattern question asked in interviews](http://java67.blogspot.sg/2012/09/top-10-java-design-pattern-interview-question-answer.html)
- [10 Java coding interview questions answer for 2  to 4 years experience](http://java67.blogspot.sg/2012/08/10-java-coding-interview-questions-and.html)
- [Top 21 Most Frequently Asked Java Questions and Answers](http://java67.blogspot.sg/2014/07/21-frequently-asked-java-interview-questions-answers.html)
- [133 Core Java Interview Questions from last 5 years](http://javarevisited.blogspot.com/2015/10/133-java-interview-questions-answers-from-last-5-years.html) 
- [Top 50 Multithreading and Concurrency Interview Questions](http://javarevisited.blogspot.com/2014/07/top-50-java-multithreading-interview-questions-answers.html)
- [21 Frequently asked SQL queries from Java Interviews](http://www.java67.com/2013/04/10-frequently-asked-sql-query-interview-questions-answers-database.html)






