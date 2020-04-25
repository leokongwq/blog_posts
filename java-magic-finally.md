---
layout: post
comments: true
title: java魔法之finally
date: 2016-12-31 23:52:10
tags:
categories:
- java
---

本文翻译自[http://mishadoff.com/blog/java-magic-part-3-finally/](http://mishadoff.com/blog/java-magic-part-3-finally/)

### 前言

每个有经验的Java程序员都应该知道`finally`代码块总是会执行的。但这是真的吗？

这依赖我们的对程序执行的定义。但是通常都是对的。

<!-- more -->

### 正常的执行逻辑

看下面的代码，某人可能要被打脸了。

```java
try {
  System.exit(1);
} finally {
  System.out.println("I'm here, man");
}
```

你不是说`finally`代码块总是会执行吗？

是的，在上面的例子总确实不会执行，单我们说的是通常的代码执行逻辑。上面的例子不是正常的逻辑。

来自 [官方入门手册](http://docs.oracle.com/javase/tutorial/essential/exceptions/finally.html)

> **注意**：如果JVM在执行try或catch代码时退出，则finally块可能无法执行

Your counter question might be: If second line of that code always exectued?

你的counter问题可能是：该代码的第二行是否总是会执行？

```java
System.out.println("Line 1");
System.out.println("Line 2");
System.out.println("Line 3");
```

当然了, 这是因为这些代码都是顺序执行的，没有任何事情可以阻止 ... 梆的一声，断电了。程序停止了。

对着这种情况该怎么解释呢? 这也是程序执行的异常情况，我们不能保证任何东西100％正确。事实上，这与`System.exit(1)`或计算机上的重置按钮或任何停止程序的原因相同。

这就是为什么我们谈论的是正常的程序执行流程，只有正常才会符合我们日常的认知。

### 死循环

思考下面的代码

```java
try {
  while (true) {
    System.out.println("I print here some useful information");
  }
} finally {
  System.out.println("Let me run");
}
```

"Let me run"会输出吗？可能会, 如果上面的的代码输出到标准输出时发生错误。 但大部分情况下是不会的。

在这种情况下，简单语句和finally块之间没有区别。 没有一个会被执行，忘掉这个例子。

### 线程

对于线程来说又会出现什么情况呢? 我们知道执行流程是被线程控制的，并且线程是可以被中断的。

假设我们有一个在执行任务的线程，其它的线程在该工作线程将要执行到finally代码块前中断或杀掉该线程。finally块就不会执行。

假设我们有两个处于死锁状态的线程，死锁恰好发生在finally代码块前。答案参见[tutorial](http://docs.oracle.com/javase/tutorial/essential/exceptions/finally.html)

> 如果线程在执行try或者catch代码块时被中断或者kill，则finally代码块不会被执行。即使应用程序还在继续运行。

因此我们可以将一个线程当做一个单独的程序并对此指定一个有效的规则：

**规则1**. Finally块总是会被执行，除非控制控制执行流程的程序或线程退出了。

### Finally we return

好的，现在我们知道了什么时候finally块不会被执行。但是我们还不知道finally块什么时候执行。

思考下面的代码片段：

```java
int someFunc() {
  try {
    return 0;
  } finally {
    return 1;
  }
}
```

结果明显是1。这是因为finally块肯定会被执行。

继续，下一个例子：

```java
int someFunc() {
  try {
    throw new RuntimeException();
  } finally {
    return 1;
  }
}
```

结果还是1。这是一个问题。我们丢失了异常。 这种问题称为吞咽异常。 这是非常危险的，因为客户端的代码期望异常或某些值，但它总是只获得值。

一个更常见的例子：

```java
String deposit(int amount) throws DAOException {
  try {
    return dao.deposit(amount);
  } finally {
    return "OK";
  }
}
```

finally块的逻辑是返回认值，我们的存款方法抛出DAOException，编译器强制客户端代码处理它。 不幸的是，编译器强制你处理的这个DAOException它从来不会发生，并总是返回字符串“OK”。

**规则2**. 永远不要再finally块中使用return语句。

### 不是结论

很多程序员都知道这个常见的错误。 但有些人就不知道。 也许这两个简单的规则可以避免你犯错。


