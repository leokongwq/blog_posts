---
layout: post
comments: true
title: 为什么用枚举实现的单例模式可以防止反序列化？
date: 2017-08-21 15:41:32
tags:
categories:
- java
---

### 背景

在阅读Java中实现单例的N种方式相关文章中，讲到用枚举来实现单例可以防止被反序列化。当时只是记住了这个答案，并没有探究背后的原理，今天通过查找相关的问题才算找到了答案。

原文地址:[https://www.quora.com/Why-are-enum-singleton-serialization-safe](https://www.quora.com/Why-are-enum-singleton-serialization-safe)

<!-- more -->

### 译文

Java的序列化机制针对枚举类型是特殊处理的。简单来讲，在序列化枚举类型时，只会存储枚举类的引用和枚举常量的名称。随后的反序列化的过程中，这些信息被用来在运行时环境中查找存在的枚举类型对象。

这样你就可以在同一个运行时环境中反序列化枚举常量，并且你会得到同一个实例对象。

然而，在不同的JVM中对枚举类型进行反序列化，可能会得到不同的`hashcode`。但是，对单例对象来说，拥有相同的`hashcode`并不是一个必要的条件。重点是该类永远不能有多余一个的实例(同一个JVM)，枚举类型的序列化机制保证只会查找已经存在的枚举类型实例，而不是创建新的实例。

### 验证代码：

```java
public class EnumSerriliaztionTest {

    static enum Color {
        RED,
        WHITE,
        ;
    }

    private static void  writeEnum(Color color, String filePath) throws Exception {
        ObjectOutputStream outputStream = new ObjectOutputStream(new FileOutputStream(new File(filePath)));
        outputStream.writeObject(color);
        outputStream.close();
    }

    private static Color readEnum(String filePath) throws Exception {
        ObjectInputStream inputStream = new ObjectInputStream(new FileInputStream(new File(filePath)));
        return (Color) inputStream.readObject();
    }

    public static void main(String[] args) throws Exception {
        String filePath = "/data/logs/enum.dat";
        writeEnum(Color.RED, filePath);
        System.out.println(Color.RED == readEnum(filePath));
    }

}
```

### 另一个问题-当单例类被多个类加载器加载，如何还能保持单例？

如果了解Java类加载器原理的同学，应该能找到答案。

答案就是：用多个类加载器的父类来加载单例类。

### 单例类防止反序列化

```java
public class Singleton implements java.io.Serializable {
   public static Singleton INSTANCE = new Singleton();
   protected Singleton() {
      // Exists only to thwart instantiation.
   }
      private Object readResolve() {
            return INSTANCE;
      }
}
```

### 单例类如何防止反射？

```java
public class Singleton {  
  
    private static boolean flag = false;  
  
    private Singleton(){  
        synchronized(Singleton.class){  
            if(flag == false){  
                flag = !flag;  
            } else {  
                throw new RuntimeException("单例模式被侵犯！");  
            }  
        }  
    }  
  
    private  static class SingletonHolder {  
        private static final Singleton INSTANCE = new Singleton();  
    }  
  
    public static Singleton getInstance(){  
        return SingletonHolder.INSTANCE;  
    }  
}  
```

### 其它资料

[http://docs.oracle.com/javase/1.5.0/docs/guide/serialization/spec/serial-arch.html#enum](http://docs.oracle.com/javase/1.5.0/docs/guide/serialization/spec/serial-arch.html#enum)
[https://www.javaworld.com/article/2073352/core-java/simply-singleton.html](https://www.javaworld.com/article/2073352/core-java/simply-singleton.html)



