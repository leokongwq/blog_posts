---
layout: post
comments: true
title: java8 lambda表达式详解
date: 2016-12-29 17:50:00
tags:
- java
categories:
- java
---


### Why Lambda Expressions

将lambda表达式引入Java的动机是和一个称为`行为参数化`模式相关的。该模式让你通过编写更灵活的代码来应对需求的变化。在Java8之前该模式的实现非常繁琐。通过lambda表达式你可以非常简单明显的实现该模式。下面的例子就非常说明问题：

<!-- more -->

```java
List<Invoice> findInvoicesGreaterThanAmount(List<Invoice> invoi ces, double amount) {
    List<Invoice> result = new ArrayList<>(); 
    for(Invoice inv: invoices) {
        if(inv.getAmount() > amount) {
            result.add(inv);
        }
    }
    return result; 
}
```

该方法非常简单。然而如果现在需要找出所有小于某个值的列表你就得重新写一个方法或者使用如下的方式来实现：

```java
interface InvoicePredicate { 
    boolean test(invoice inv);
}
List<Invoice> findInvoices(List<Invoice> invoices, InvoicePredi cate p) {
    List<Invoice> result = new ArrayList<>(); 
    for(Invoice inv: invoices) {
        if(p.test(inv)) { 
            result.add(inv);
        }    
    }
    return result; 
}
```

虽然通过上面的代码我们可以应对大部分的数据过滤需求(只需在使用的时候传入不同的条件对象)，但是代码的可读性变差了。如果想要编写既功能灵活有可读性良好的代码，那就可以使用lambda表达式来实现。通过使用lambda表示我们可以将上面的代码改造为如下格式：

```java
List<Invoice> expensiveInvoicesFromOracle = findInvoices(invoices, inv -> inv.getAmount() > 10_000 && inv.getCustomer() == Customer.ORACLE);
```

### Lambda Expressions De ned

知道了为什么使用lambda表达式, 现在是时候学习lambda表达式的明确定义了。简而言之,lambda表达式是一个匿名函数,可以传递。让我们更详细地看看这个定义:**Anonymous**lambda表达式是一个匿名的方法，这是因为它不像一个正常方法一样拥有一个确切的名称。它还有一点像一个没有名字的匿名类。**Function**

lambda表达式像一个方法，它拥有参数列表，方法体，返回值和可能抛出的异常列表。然而它不像一个通常的方法那样是因为它没有申明为一个类的一部分。
**Passed around**
一个 lambda 表达式可以作为方法的参数进行传递，也可以作为一个方法的返回值。 

### Lambda Expression Syntax

在编写lambda表达式前你需要知道它的语法格式：

    Runnable r = () -> System.out.println("Hi");    FileFilter isXml = (File f) -> f.getName().endsWith(".xml");
    
上面的两个lambda表达式由三部分组成：
• 一个参数列表, 如 (File f)• 一个有两个字符组成的箭头 `-` 和 `>`
• 一个方法体, 如：f.getName().endsWith(".xml")

lambda表达式有两种格式：

    (parameters) -> expression
    
如果方法体只是一个简单的表达式或一个语句则使用该格式。

    (parameters) -> { statements;}    
    
如果方法体是由多个语句组成，则使用该格式。

需要注意的是如果有需要你也可以申明参数的类型

    （int age） -> { return age > 30; }
    
### Where to Use Lambda Expressions    

现在你知道如何写lambda表达式,下一个问题是在哪里以及如何使用它们。简而言之,您可以在一个需要函数式接口对象的上下文中使用lambda。一个函数式接口是有且只有一个抽象方法的接口类型。举个例子:

```java
Runnable r = () -> System.out.println("Hi");FileFilter isXml = (File f) -> f.getName().endsWith(".xml");
```
    
Runnable是一个函数式接口，因为它只定义了一个方法。FileFilter也是同样的原因：    

```   
@FunctionalInterfacepublic interface Runnable { 
    void run();}@FunctionalInterfacepublic interface FileFilter { 
    boolean accept(File pathname);} 
```

这里的重点是lambda表达式是你可以创建一个函数式接口类型的实例。lambda表达式的方法体实现了函数式接口中定义的抽象方法。 下面的例子实现的功能是相同的：   

```java
Runnable r1 = new Runnable() { 
    public void run() {
            System.out.println("Hi!");
    }
}; 
r1.run();
Runnable r2 = () -> System.out.println("Hi!");
r2.run();
```

**注意：**

我们经常会看到有的接口上会有这样的注解`@FunctionalInterface`,这其实就是表示该接口是作为函数式接口使用的。编译器也会对使用了该注解但是不符合函数式接口规范的接口进行编译检查，如果不符合就会报错。

Java8提供了一些函数式接口：

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}
@FunctionalInterface
public interface Supplier<T> {

    /**
     * Gets a result.
     *
     * @return a result
     */
    T get();
}
```

### Method References

方法引用是你可以重用已经定义的方法，而且可以像lambda表达式一样进行传递。相比lambda表达式，使用方法引用你可以编写感觉更自然流畅的代码。例如你通过lambda表达式查找隐藏的文件：
    File[] hiddenFiles = mainDirectory.listFiles(f -> f.isHidden());
使用方法引用, 你可以直接使用`File`类的方法`isHidden`
    File[] hiddenFiles = mainDirectory.listFiles(File::isHidden);   

理解方法引用最简单的方式就是讲它看做是lambda表达式的速记符号。总共有四种类型的方法引用：

• 引用一个静态方法

    Function<String, Integer> converter = Integer::parseInt;    Integer number = converter.apply("10");

• 引用一个实例方法。

引用一个实例方法相当于将该对象作为第一参数传递到被引用的对象中 
    Function<Invoice, Integer> invoiceToId = Invoice::getId;
• 引用一个已经存在的类实例的方法
    Consumer<Object> print = System.out::println;       
这种类型的方法引用非常有用，尤其当你想要使用一个私有工具方法并注入到另一个方法中时。    File[] hidden = mainDirectory.listFiles(this::isXML);    private boolean isXML(File f) { 
        return f.getName.endsWith(".xml");    }
• 构造方法引用    Supplier<List<String>> listOfString = List::new;

### Putting It All Together

我们已经学习了lambda表达式和方法引用，现在是时候将它们结合起来改造已有的代码了,看下面的例子：
    Collections.sort(invoices, new Comparator<Invoice>() {        public int compare(Invoice inv1, Invoice inv2) {            return Double.compare(inv2.getAmount(), inv1.getAmount());        } 
    });

很明显接口`Comparator`是一个函数式接口，它的compare方法接受两个参数并返回一个int值表示比较的结果。

    Collections.sort(invoices, (Invoice inv1, Invoice inv2) -> {        return Double.compare(inv2.getAmount(), inv1.getAmount());    });

由于lambda表达式的方法体只是返回语句体的值，因此我们可以用更简单的格式：

    Collections.sort(invoices, (Invoice inv1, Invoice inv2) ->  Double.compare(inv2.getAmount(), inv1.getAmount())    );

在Java8中, List 接口支持了`sort`方法，因此你可以使用该方法替代`Collections.sort`    invoices.sort((Invoice inv1, Invoice inv2) -> Double.compare(inv2.getAmount(),    inv1.getAmount()));   

同时Java8提供了一个静态帮助方法`Comparator.comparing`，该方法接受一个lambda表达式，返回一个Comparator对象。如下所示：
    Comparator<Invoice> byAmount = Comparator.comparing((Invoice inv) ->    inv.getAmount());    invoices.sort(byAmount);

你也应该注意到使用方法引用可以替换该lambda表达式同时可以更清晰的表达意图：

    Comparator<Invoice> byAmount = Comparator.comparing(Invoice::getAmount);    invoices.sort(byAmount);

由于getAmount方法返回类型是double，所以也可以使用Comparator.comparingDouble

```java
Comparator<Invoice> byAmount = Comparator.comparingDouble(Invoice::getAmount);invoices.sort(byAmount);
```

通过静态方法导入可以进一步简化

```java
import static java.util.Comparator.comparingDouble; invoices.sort(comparingDouble(Invoice::getAmount));
```

### Testing with Lambda Expressions你可能关心的lambda表达式会如何影响测试。毕竟,lambda表达式引入行为需要被测试。当决定如何测试包含lambda表达式的代码,考虑以下两个选择:• 如果lambda表达式比较简单，那就可以只测试包裹lambda表达的方法的功能• 如果lambda表达式相对比较复杂, 则可以将该表达式抽取出来放入一个方法，然后使用方法引用注入到需要的地方。

### Summary
关于lambda表达的关键概念如下:- lambda表达式可以被理解为一种匿名函数。- Lambda表达式和行为参数化模式让你编写的代码,既灵活又简洁。
- 函数接口是一个只包含一个抽象方法的接口。- lambda表达式只能用在需要函数式接口的上下文环境中。- 当你想重用已有的方法时，方法引用相比lambda表达来说更自然。- 在测试时，将负责的lambda表达式抽取到一个方法中并通过方法引用来注入到需要的地方







        




