---
layout: post
comments: true
title: java中的枚举是否可以有抽象方法
date: 2017-04-08 00:19:09
tags:
- 面试
categories:
- java
---

### 背景

偶然在代码中看到枚举中定义了抽象方法，觉得很奇怪。以前从来没有想过还可以这么用，真是涨姿势。

### 例子

```java
public enum Animal {
    CAT {
        public String makeNoise() { return "MEOW!"; }
    },
    DOG {
        public String makeNoise() { return "WOOF!"; }
    };

    public abstract String makeNoise();
}
```

参考自：[http://stackoverflow.com/questions/7413872/can-an-enum-have-abstract-methods](http://stackoverflow.com/questions/7413872/can-an-enum-have-abstract-methods)

<!-- more -->

更好的用法是枚举类实现一个接口：

```java
public interface Operator {
    int apply (int a, int b);
}

public enum SimpleOperators implements Operator {
    PLUS { 
        int apply(int a, int b) { return a + b; }
    },
    MINUS { 
        int apply(int a, int b) { return a - b; }
    };
}

public enum ComplexOperators implements Operator {
    // can't think of an example right now :-/
}
```

参考自：[http://stackoverflow.com/questions/2709593/why-would-an-enum-implement-an-interface](http://stackoverflow.com/questions/2709593/why-would-an-enum-implement-an-interface)

### 思考

枚举类Animal相当于是一个父类， Animal里面的每个元素都是它的子类，父类定义的抽象方法子类当然可以实现，甚至覆盖父类的方法。

```java
public enum Animal {
    CAT {
        public String makeNoise() { return "MEOW!"; }
        @Override
        public void sayHi(){
            System.out.println("Cat say HI ");
        }
    },
    DOG {
        public String makeNoise() { return "WOOF!"; }

        @Override
        public void sayHi(){
            System.out.println("Dog say HI ");
        }
    };

    public abstract String makeNoise();

    public void sayHi(){
        System.out.println("Animal say hi");
    }

    public static void main(String[] args) {
        Animal.CAT.sayHi();
        Animal.DOG.sayHi();
    }
}
```





