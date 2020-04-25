---
layout: post
comments: true
title: java实现单例模式的几种方式
date: 2016-11-23 12:49:23
tags:
categories:
- java
---

### 前言

> 在日常的开发中实现一个单例类是非常常见的需求.每个人都有自己喜欢的实现方式.每种实现方式都有可以讨论的点.这篇文章收集了Java中能见到的实现单例模式的几乎所有方式作为参考.

<!-- more -->

### 懒汉模式

```java
public class Singleton {  
    
    private static Singleton instance;
        
    private Singleton (){
    }
    
    public static Singleton getInstance() {  
        if (instance == null) {
            instance = new Singleton();
        }        
        return instance;
    }  
}
```  

这种写法可以实现`lazy loading`，但是致命的是在多线程不能正常工作(话说很少有在多线程环境下初始化单例类, 可以在容器启动时就进行初始化).

### 懒汉-线程安全

```java
public class Singleton {  
    
    private static Singleton instance;
        
    private Singleton (){
    }
    
    public static synchronized Singleton getInstance() {  
        if (instance == null) {
            instance = new Singleton();
        }        
        return instance;
    }  
}
``` 
这种写法能实现线程安全, 但是每次方法实例都要同步,这是不可取的.

### 懒汉-线程安全2

```java
public class Singleton {  
    
    private static volatile Singleton instance;
        
    private Singleton (){
    }
    
    public static Singleton getInstance() {  
        if (instance == null) {
            synchronized(Singleton.class){
                if(instance == null){
                    instance = new Singleton();
                }
            }
        }        
        return instance;
    }  
}
```

这种写法能实现线程安全, 并且不会在每次调用`getInstance`方法时都进行同步 

### 饿汉模式

```java
public class Singleton {  
     
     private static Singleton instance = new Singleton();  
     
     private Singleton (){
     }
     
     public static Singleton getInstance() {  
        return instance;  
     }  
}  
```

这种方式基于classloder机制避免了多线程的同步问题，不过instance在类装载时就实例化，虽然导致类装载的原因有很多种，在单例模式中大多数都是调用getInstance方法， 但是也不能确定有其他的方式（或者其他的静态方法）导致类装载，这时候初始化instance显然没有达到`lazy loading`的效果。

### 饿汉模式变种

```java
public class Singleton {
  
    private static Singleton instance;
    
    static {
        instance = new Singleton();
    }  
             
    private Singleton (){
    }
    public static Singleton getInstance() {  
       return instance;  
    }  
}  
``` 

这种实现方式和上面的方式没有本质的不同,都是在类加载的时候进行初始化实例.
  
### 静态内部类

```java
public class Singleton {  
    
    private static class SingletonHolder {
        private static Singleton instance = new Singleton();    
    }
        
    private Singleton (){
    }
    
    public static Singleton getInstance() {  
        return SingletonHolder.instance;
    }  
}
```

这种方式同样利用了classloder的机制来保证初始化instance时只有一个线程，它跟第三种和第四种方式不同的是（很细微的差别）：第三种和第四种方式是只要Singleton类被装载了，那么instance就会被实例化（没有达到lazy loading效果），而这种方式是Singleton类被装载了，instance不一定被初始化。因为SingletonHolder类没有被主动使用，只有显示通过调用getInstance方法时，才会显示装载SingletonHolder类，从而实例化instance。想象一下，如果实例化instance很消耗资源，我想让他延迟加载，另外一方面，我不希望在Singleton类加载时就实例化，因为我不能确保Singleton类还可能在其他的地方被主动使用从而被加载，那么这个时候实例化instance显然是不合适的。这个时候，这种方式相比第三和第四种方式就显得很合理。
  
### 枚举

```java
public enum Singleton {  
   INSTANCE;  
   public void whateverMethod() {  
   }  
}  
```

这种方式是Effective Java作者Josh Bloch 提倡的方式，它不仅能避免多线程同步问题，而且还能防止反序列化重新创建新的对象，可谓是很坚强的壁垒啊，不过，个人认为由于1.5中才加入enum特性，用这种方式写不免让人感觉生疏，在实际工作中，我也很少看见有人这么写过
  
### 总结
  
有两个问题需要注意：

1. 如果单例由不同的类装载器装入，那便有可能存在多个单例类的实例。假定不是远端存取，例如一些servlet容器对每个servlet使用完全不同的类装载器，这样的话如果有两个servlet访问一个单例类，它们就都会有各自的实例。
2. 如果Singleton实现了java.io.Serializable接口，那么这个类的实例就可能被序列化和复原。不管怎样，如果你序列化一个单例类的对象，接下来复原多个那个对象，那你就会有多个单例类的实例。  

对第一个问题修复的办法是：

```java
 private static Class getClass(String classname) throws ClassNotFoundException {     
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();     
        if(classLoader == null) {     
           classLoader = Singleton.class.getClassLoader();
        }
        return (classLoader.loadClass(classname));     
    }     
}  
```

对第二个问题修复的办法是：

```java
public class Singleton implements java.io.Serializable {     
     
     public static Singleton INSTANCE = new Singleton();     
        
     protected Singleton() {     
          
     }
          
     private Object readResolve() {     
              return INSTANCE;     
     }    
}   
```

关于`readResolve`方法可以参考[https://docs.oracle.com/javase/7/docs/platform/serialization/spec/input.html](https://docs.oracle.com/javase/7/docs/platform/serialization/spec/input.html)

### 面试相关

#### 你如何阻止使用clone()方法创建单例实例的另一个实例？
     
该类型问题有时候会通过如何破坏单例或什么时候Java中的单例模式不是单例来被问及。在JAVA里要注意的是，所有的类都默认的继承自Object，所以都有一个clone方法。为保证只有一个实例，要把这个口堵上。有两个方面，一个是单例类一定要是final的，这样用户就不能继承它了。另外，如果单例类是继承于其它类的，还要override它的clone方法，让它抛出异常。如果阻止通过使用反射来创建单例类的另一个实例？开放的问题。在我的理解中，从构造方法中抛出异常可能是一个选项。

#### 通过反射创建单例类的另一个实例：

如果借助AccessibleObject.setAccessible方法，通过反射机制调用私有构造器，反射攻击. 这个问题可以通过设置标志位, 当第二次调用构造方法时抛出异常.
     
### 参考

[http://www.blogjava.net/kenzhh/archive/2011/09/02/357824.html](http://www.blogjava.net/kenzhh/archive/2011/09/02/357824.html)
[http://ycymio.com/blog/2016/10/08/designpattern-singleton/](http://ycymio.com/blog/2016/10/08/designpattern-singleton/)