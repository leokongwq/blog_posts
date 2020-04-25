---
layout: post
comments: true
title: java程序避免NullPointException最佳实践
date: 2016-10-12 15:58:37
tags:
- java
categories:
- java 
---

### java程序避免NullPointException最佳实践

> Java应用中抛出的空指针异常是解决空指针的最好方式，也是写出能顺利工作的健壮程序的关键。俗话说“预防胜于治疗”，对于这么令人讨厌的空指针异常，这句话也是成立的。值得庆幸的是运用一些防御性的编码技巧，跟踪应用中多个部分之间的联系，你可以将Java中的空指针异常控制在一个很好的水平上。顺便说一句，这是Javarevisited上的第二个空指针异常的帖子。在上个帖子中我们讨论了Java中导致空指针异常的常见原因，而在本教程中我们将会学习一些Java的编程技巧和最佳实践。这些技巧可以帮助你避免Java中的空指针异常。遵从这些技巧同样可以减少Java代码中到处都有的非空检查的数量。作为一个有经验的Java程序员，你可能已经知道其中的一部分技巧并且应用在你的项目中。但对于新手和中级开发人员来说，这将是很值得学习的。顺便说一句，如果你知道其它的避免空指针异常和减少空指针检查的Java技巧，请和我们分享。这些都是简单的技巧，很容易应用，但是对代码质量和健壮性有显著影响。根据我的经验，只有第一个技巧可以显著改善代码质量。如我之前所讲，如果你知道任何避免空指针异常和减少空指针检查的Java技巧，你可以通过评论本文来和分享。

<!-- more -->

 - **从已知的String对象中调用equals()和equalsIgnoreCase()方法，而非未知对象。**

总是从已知的非空String对象中调用equals()方法。因为equals()方法是对称的，调用a.equals(b)和调用b.equals(a)是完全相同的，这也是为什么程序员对于对象a和b这么不上心。如果调用者是空指针，这种调用可能导致一个空指针异常
    
{% codeblock lang:java %}    
    Object unknownObject = null;
     
    //错误方式 – 可能导致 NullPointerException
    if(unknownObject.equals("knownObject")){
       System.err.println("This may result in NullPointerException if unknownObject is null");
    }
     
    //正确方式 - 即便 unknownObject是null也能避免NullPointerException
    if("knownObject".equals(unknownObject)){
        System.err.println("better coding avoided NullPointerException");
    }
{% endcodeblock %}

这是避免空指针异常最简单的Java技巧，但能够导致巨大的改进，因为equals()是一个常见方法。

 - **当valueOf()和toString()返回相同的结果时，宁愿使用前者**。

 因为调用null对象的toString()会抛出空指针异常，如果我们能够使用valueOf()获得相同的值，那宁愿使用valueOf()，传递一个null给valueOf()将会返回“null”，尤其是在那些包装类，像Integer、Float、Double和BigDecimal。
 
{% codeblock lang:java %} 
    BigDecimal bd = getPrice();
    System.out.println(String.valueOf(bd)); //不会抛出空指针异常
    //抛出 "Exception in thread "main" java.lang.NullPointerException"
    System.out.println(bd.toString());  
{% endcodeblock %}

 - **使用null安全的方法和库**
 
有很多开源库已经为您做了繁重的空指针检查工作。其中最常用的一个的是Apache commons中的StringUtils。你可以使用StringUtils.isBlank()，isNumeric()，isWhiteSpace()以及其他的工具方法而不用担心空指针异常。

{% codeblock lang:java %} 
    //StringUtils方法是空指针安全的，他们不会抛出空指针异常
    System.out.println(StringUtils.isEmpty(null));
    System.out.println(StringUtils.isBlank(null));
    System.out.println(StringUtils.isNumeric(null));
    System.out.println(StringUtils.isAllUpperCase(null));
     
    Output:
    true
    true
    false
    false
{% endcodeblock %}
    
但是在做出结论之前，不要忘记阅读空指针方法的类的文档。这是另一个不需要下大功夫就能得到很大改进的Java最佳实践。

 - **避免从方法中返回空指针，而是返回空collection或者空数组**

这个Java最佳实践或技巧由Joshua Bloch在他的书Effective Java中提到。这是另外一个可以更好的使用Java编程的技巧。通过返回一个空collection或者空数组，你可以确保在调用如size(),length()的时候不会因为空指针异常崩溃。Collections类提供了方便的空List，Set和Map: Collections.EMPTY_LIST，Collections.EMPTY_SET，Collections.EMPTY_MAP。这里是实例。

{% codeblock lang:java %} 
    public List getOrders(Customer customer){
        List result = Collections.EMPTY_LIST;
        return result;
    }
{% endcodeblock %} 
    
你同样可以使用Collections.EMPTY_SET和Collections.EMPTY_MAP来代替空指针    
     
 - **使用annotation@NotNull 和 @Nullable**    

在写程序的时候你可以定义是否可为空指针。通过使用像@NotNull和@Nullable之类的annotation来声明一个方法是否是空指针安全的。现代的编译器、IDE或者工具可以读此annotation并帮你添加忘记的空指针检查，或者向你提示出不必要的乱七八糟的空指针检查。IntelliJ和findbugs已经支持了这些annotation。这些annotation同样是JSR 305的一部分，但即便IDE或工具中没有，这个annotation本身可以作为文档。看到@NotNull和@Nullable，程序员自己可以决定是否做空指针检查。顺便说一句，这个技巧对Java程序员来说相对比较新，要采用需要一段时间。

 - **避免你的代码中不必要的自动包装和自动解包。**

且不管其他如创建临时对象的缺点，如果wrapper类对象是null，自动包装同样容易导致空指针异常。例如如果person对象没有电话号码的话会返回null，如下代码会因为空指针异常崩溃

{% codeblock lang:java %} 
    Person ram = new Person("ram");
    int phone = ram.getPhone();
{% endcodeblock %}
    
当使用自动包装和自动解包的时候，不仅仅是等号，`>` ; `<`; 同样会抛出空指针异常。你可以通过这篇文章来学习更多的Java中的自动包装和拆包的陷阱。    

- **遵从Contract并定义合理的默认值。**

在Java中避免空指针异常的一个最好的方法是简单的定义contract并遵从它们。大部分空指针异常的出现是因为使用不完整的信息创建对象或者未提供所有的依赖项。如果你不允许创建不完整的对象并优雅地拒绝这些请求，你可以在接下来的工作者预防大量的空指针异常。类似的，如果对象允许创建，你需要给他们定义一个合理的默认值。例如一个Employee对象不能在创建的时候没有id和name，但是是否有电话号码是可选的。现在如果Employee没有电话号码，你可以返回一个默认值（例如0）来代替返回null。但是必须谨慎选择，哟有时候检查空指针比调用无效号码要方便。同样的，通过定义什么可以是null，什么不能为null，调用者可以作出明智的决定。failing fast或接受null同样是一个你需要进行选择并贯彻的，重要的设计决策

 - **定义数据库中的字段是否可为空**

如果你在使用数据库来保存你的域名对象，如Customers，Orders 等，你需要在数据库本身定义是否为空的约束。因为数据库会从很多代码中获取数据，数据库中有是否为空的检查可以确保你的数据健全。在数据库中维护null约束同样可以帮助你减少Java代码中的空指针检查。当从数据库中加载一个对象是你会明确，哪些字段是可以为null的，而哪些不能，这可以使你代码中不必要的!= null检查最少化。

- **使用空对象模式（Null Object Pattern）**

还有一种方法来避免Java中的空指针异常。如果一个方法返回对象，在调用者中执行一些操作，例如Collection.iterator()方法返回迭代器，其调用者执行遍历。假设如果一个调用者并没有任何迭代器，其可以返回空对象（Null object）而非null。空对象是一个特殊的对象，其在不同的上下文中有不同的意义。例如一个空的迭代器调用hasNext()返回false时，可以是一个空对象。同样的在返回Container和Collection类型方法的例子中，空对象可以被用来代替null作为返回值。我打算另写一篇文章来讲空对象模式，分享几个Java空对象的例子。

转自：[http://www.importnew.com/7268.html](http://www.importnew.com/7268.html)                        
                    
                    
