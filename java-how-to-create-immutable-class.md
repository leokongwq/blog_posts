---
layout: post
comments: true
title: java中如何创建不可变对象
date: 2017-02-25 11:50:29
tags:
- 面试
categories:
- java
---

本文翻译自：[http://javarevisited.blogspot.jp/2013/03/how-to-create-immutable-class-object-java-example-tutorial.html](http://javarevisited.blogspot.jp/2013/03/how-to-create-immutable-class-object-java-example-tutorial.html)

### 前言

由于不可变对象在并发和多线程环境中有非常多的优势，因此在Java中编写或创建不可变类正日益流行。不可变对象提供了比常规可变对象更多的优点，特别是在创建并发Java应用程序时。不可变对象不仅保证对象状态的安全发布，而且可以在没有任何外部同步的情况下与其它线程共享。事实上，JDK本身包含了一些不可变的类，如String，Integer和其它的包装类。对于那些不知道什么是不可变的类或对象的人，不可变对象是那些一旦创建其状态就不能改变的对象，例如：`java.lang.String`，一旦创建就不能修改，转为大写或小写。 String中的所有修改都会导致新对象的产生，有关更多详细信息，请参阅[为什么String在Java中是不可变的](http://javarevisited.blogspot.com/2010/10/why-string-is-immutable-in-java.html)。在这个Java编程教程中，我们将学习如何在Java中编写不可变类，或者如何使类不可变。顺便说一下，使类不可变在代码级别上并不难，难点是决定使哪个类可变或不可变。我还建议阅读，`Java Concurrency in Practice`来了解更多关于Immutable对象提供的并发效益。

<!-- more -->

### 什么是Java中的不可变类

如上所述，不可变类就是那些一旦对象创建后就不能对该对象进行任何修改的类。者也就意味着任何对不可变对象的修改都会创建一个新的不可变对象。理解什么是可变和不可变对象的最好例子是`String`和`StringBuffer`。由于String是不可变类，任何已经存在的String对象的改变都会创建新的String对象。然而像可变对象`StringBuffer`，任何对该对象自身的修改不会有新对象的产生。这也是某些情况下String对象会导致安全漏洞和密码应该保存在char数组中的原因。

### Java中如何编写不可变类

尽管不可变类有几个缺点，但它仍然提供了一些优势在多线程编程中，它是在Java代码中实现线程安全的绝佳选择。 这里有几个规则，这有助于使Java中的类不可变

1. 在构造完成后不能修改不可变对象的状态，任何的修改的结果是创造一个新的对象。
2. 所有不可变类的字段都应该是final的。
3. 对象应该被正确合适的构造，如：不能在构造时将对象的引用泄露。
4. 类应该是final类型的，以此防止子类的扩展破坏父类的不可变性。

顺便说一句，你仍然可以通过违反一些规则创建不可变对象，如String类的`hashcode`字段就是非final字段，但它能保证hashcode总是相同的，无论你计算多少次，因为hashcode是从final类型的字段计算得出的。 这需要对Java内存模型的深入了解，如果不能正确解决，可能会产生微妙的竞争条件。 在下一节中，我们将看到在Java中编写不可变类的简单示例。 顺便说一下，如果你的不可变类有很多可选和必填字段，那么你也可以使用Builder的设计模式，在Java中创建一个类不可变。

### Java中的不可变类示例

下面是在Java中编写不可变类的完整代码示例。 我们遵循最简单的方法和使类不可变的所有规则，包括它使类最终避免由于继承和多态性而使不可变性处于风险中。

```java
public final class Contacts {

    private final String name;
    private final String mobile;

    public Contacts(String name, String mobile) {
        this.name = name;
        this.mobile = mobile;
    }
  
    public String getName(){
        return name;
    }
  
    public String getMobile(){
        return mobile;
    }
}
```

这个Java类是不可变的，因为它的状态一旦创建就不能改变。 你可以看到它的所有字段是final。 这是在Java中创建不可变类的最简单的方法之一，其中类的所有字段在上面的情况下仍然是不可变的，如String。 有些时候，你可能需要编写不可变的类，其中包括java.util.Date等可变类，尽管将Date存储到final字段中，但如果将内部日期返回给客户端，则可以在内部进行修改。 为了在这种情况下保持不变性，它建议返回原始对象的副本，这也是Java最佳实践之一。 这里是另一个例子，使类不可变的Java，其中包括可变成员变量。

```java
public final class ImmutableReminder{
    private final Date remindingDate;
  
    public ImmutableReminder (Date remindingDate) {
        if(remindingDate.getTime() < System.currentTimeMillis()){
            throw new IllegalArgumentException("Can not set reminder” +
                        “ for past time: " + remindingDate);
        }
        this.remindingDate = new Date(remindingDate.getTime());
    }
  
    public Date getRemindingDate() {
        return (Date) remindingDate.clone();
    }
}
```

在上面创建不可变类的例子中，`Date`类是一个可变的对象。 如果`getRemindingDate()`返回实际的`Date`对象而不是`remindingDate.clone`生成的副本对象，则客户端可以修改返回的`Date`对象，这样我们的类就是可变的了。我们通过返回`remindingDate`的克隆对象则可以保证我们类的不可变性。

### Java中不可变类的优点

正如我前面所说的不可变类提供了几个好处，下面就提到的几点好处：

1. 不可变对象默认是线程安全的，可以在并发环境中共享而不需要额外的同步。
2. 不可变对象简化了开发，因为它更容易在多个线程之间共享而不需要外部同步。
3. 不可变对象通过减少代码中的同步来提高Java应用程序的性能。
4. 不可变对象的另一个重要优点是可重用性，你可以缓存不可变对象并重用它们，就像String字面量和整数一样。 你可以使用`类似valueOf()`这样的静态工厂方法，它可以从缓存返回一个现有的不可变对象，而不是创建一个新的对象。

除了上述优点之外，不可变对象也具有产生许多垃圾的缺点。 因为不可变对象不能被修改，所以每次的修改都是通常产生新的对象来实现的，这样就会导致大量垃圾对象的产生。String就是一个典型的例子。如果正确使用不可变对象则它带来的优势会大于它的劣势。

这就是如何在Java中编写不可变类的所有内存。 我们已经看到了编写不可变类的规则，不可变对象提供的好处，以及我们如何在Java中创建包含可变字段的不可变类。 不要忘记阅读`Java并发编程实践`这本被Java程序员大力推荐的书中更多关于由不可变对象提供的并发优势的内容。





