---
layout: post
comments: true
title: java8-optional
date: 2016-12-23 22:42:27
tags:
- java8
categories:
- java
---

### 前言

Optional 最早是我在Google的Guava包中看见的，是用来尽可能的避免程序出现**NullPointerException**异常的工具，现在是Java 8库的一部分。

<!-- more -->

### Optional 的方法

Optional 提供了很多方法，每个方法的使用需要熟记于心。

#### Optional.of

该方法用来生产一个Optional实例，参数是需要Optional保存的值，如果参数的值为null则会报空指针异常。

    Optional<Integer> optional = Optional.of(null);
    
如果参数不确定是否是null，则可以使用下面的提到的方法。

####  Optional.ofNullable

该方法和上面的`of`方法有相同的返回值，但是方法参数可以为null
    Optional<String> optional = Optional.ofNullable(name);      
    
#### Optional.get

直接使用`get`方法获取Optional中保存的值可能会发生异常，如下代码：

    String name = null;
    Optional<String> optional = Optional.ofNullable(name);
    System.out.println(optional.get());

使用该方法时应该和下面提到的方法配合使用， 或者非常肯定 Optional中的值非空。

#### Optional.isPresent

该方法用来判断Optional中的值是否存在

    if (optional.isPresent()){
        System.out.println(optional.get());
    }else {
        System.out.println("optional's values is null");
    }   

#### Optional.ifPresent 

如果Optional非空则调用参数指定的函数

    Optional.of("hello").ifPresent(value -> System.out.println(value));    

#### Optional.empty    

该方法返回一个空的Optional实例。

#### Optional.orElseGet

方法`orElseGet`提供了回退机制，当Optional的值为空时接受一个方法返回默认值。例如

    Optional<String> optional = Optional.ofNullable(name);
    System.out.println(optional.orElseGet(() -> "world"));

当name的值为null时， 输出是：world 否是 name 的值

#### Optional.orElse

`orElse`方法和`orElseGet`都是用来提供缺省情况下的默认值的，这2个方法的参数不同；`orElse`方法的参数类型就是Optional的泛型参数类型， 而`orElseGet`的参数类型是`Supplier`一个函数接口，也就是说可以接受一个lamda表达式作为参数。


#### Optional.map

`map`方法的参数也是一个函数接口`Function`; 如果Optional的值非空则该方法会将Optional的值作为参数传递给该方法的函数接口参数并返回转换后的值对应的Optional实例，否则返回一个空的Optional实例。例如：

    optional = Optional.of("hello");
    optional = optional.map(str -> str.toUpperCase());
    System.out.println(optional.get());
    
#### Optional.flatMap

`flatMap`方法的签名：

public<U> Optional<U> flatMap(Function<? super T, Optional<U>> mapper);

该方法的作用是：如果Optional保存的值存在则调用参数`mapper`指定函数并返回转换后的结果。该方法和上面的`map`很相似，单不同之处在于该方法的转换后的结果已经是一个Optional对象了，而`map`方法中转换后的结果需要包装成一个Optional对象。

    
#### Optional.filter    

如果Optional的value非空并且匹配参数predicate则返回该Optional实例否则返回一个空的Optional实例。


### 总结

Optional常用的方法上面已经提到了，如何在业务开发中使用Optional对象作为方法的返回值还需要做很多代码改造。

        
    
    


