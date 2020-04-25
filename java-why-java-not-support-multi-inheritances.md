---
layout: post
comments: true
title: 为何java不支持多继承
date: 2017-02-24 19:20:33
tags:
- 面试
categories:
- java
---

本文翻译自：[http://javarevisited.blogspot.jp/2011/07/why-multiple-inheritances-are-not.html](http://javarevisited.blogspot.jp/2011/07/why-multiple-inheritances-are-not.html)

### 背景

最近我一个朋友参加了一次面试。几个简单的问题过后，他被问到“为什么java不支持多继承”。随后他说，java可以通过接口来实现多继承。但是面试官继续问为什么呢？也许他只是读过相关的blog，并没有深入了解过这个问题。面试结束后，朋友向我描述了该问题并询问我的答案。这是非常经典的问题，像[Why String is immutable in Java](http://javarevisited.blogspot.com/2010/10/why-string-is-immutable-in-java.html)。 这两个问题之间的相似性主要是由java的创建者或设计者的设计决策驱动的。 为什么Java不支持多继承，下面两个原因是我认为有意义的答案：

<!-- more -->

1) 第一个原因是关于`钻石问题`的，考虑有个类`A`有一个方法`foo()`，然后类 `B` 和 `C` 继承自`A`并拥有自己的`foo()`方法实现，现在类`D` 继承自`B`和`C`，现在我们引用方法`foo()`时编译器不知道该调用哪个`foo()`方法。这也称为钻石问题，因为这种继承情况下的结构类似于4边缘菱形，见下文

```
           A foo()
           / \
          /   \
   foo() B     C foo()
          \   /
           \ /
            D
           foo()
```

在我看来，即使我们删除顶端的类`A`，并且允许多重继承，我们还是会遇到这个问题。 

有些时候，如果你将这个原因给面试官，他就会问为什么C++可以支持多重继承而Java为什么不支持。在这种情况下，我会尝试给出他下面第二个原因：Java不支持多重继承不是因为技术难度，考虑更多的是代码的可维护性和更清晰的设计，但这只能是我们的推测，最终也只能由java的设计者们确认。 [维基百科](http://en.wikipedia.org/wiki/Diamond_problem)有一些很好的解释关于不同的语言在使用多重继承时导致的钻石问题。

第二个更令人信服的理由是，多重继承确实使设计复杂化，并在类型转换，构造函数调用链等过程中产生问题，并且由于没有很多场景需要多重继承，为了简单起见忽略它是明智决定。 另外，java通过支持接口的多继承来避免这种歧义。 因为接口只有方法声明并且不提供任何实现，所以特定的方法只有一个实现，因此不会有任何歧义。


### 更多参考

[http://www.importnew.com/4604.html](http://www.importnew.com/4604.html)


