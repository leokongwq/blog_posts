---
layout: post
comments: true
title: Java中数据类型真实的内存占用
date: 2016-10-20 12:08:11
tags:
- jvm
- java
categories:
- java
---

> 由一个面试题引起的，对JVM底层机制的探究。

<!-- more -->

### 数据类型的有效宽度 

JVM规范规定的是抽象的、概念中的JVM所需要支持的行为。其中对于数据类型的规定在这里： 
http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.2 
例如说，这里说明了int的语义是32位补码的整型数，其值域为 -2147483648 到 2147483647 (-2^31 到 2^31 - 1)。 

那么用更大的存储空间来存它行不行呢？当然可以，只要从上层的Java代码只能感知到其中的32位是有效值即可。 char也是同理。JVM规范规定了它的语义是表示Unicode codepoint的16位无符号整型数，值域为 0 到 65535。用更大的空间来存它是没问题的，只要从上层的Java代码只能看到其中16位是有效值即可。 
（例子很重要所以几乎一样的话说了两遍⋯）那么具体到存储空间，JVM规范又做了怎样的规定呢？ 

在JVM栈上有局部变量与操作数栈。其中局部变量的规定是： 
http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.6.1 

引用

> A single local variable can hold a value of type boolean, byte, char, short, int, float, reference, or returnAddress. A pair of local variables can hold a value of type long or double.

它只说了1个局部变量的slot至少要能存下boolean一直到returnAddress的值，但没规定一定要有多宽。少了不行，多了是没问题的。 

操作数栈呢？ 
http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.6.2 
引用
Each entry on the operand stack can hold a value of any Java Virtual Machine type, including a value of type long or type double.

跟局部变量的slot的规定相似。 然后看Java堆上的Java对象里的字段。JVM规范说： 
http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.7 

**引用**
> The Java Virtual Machine does not mandate any particular internal structure for objects.

也就是说实现JVM的时候，你想如何设计Java对象在内存里的布局都没关系，只要该存的值都存着就行。跟上面一样，空间太小了不行，但多了是没问题的。 

### 实际实现使用的存储空间大小 

那么真实的JVM会怎么做呢？ 

以解释器为主的JVM会采用比较贴近JVM规范所说的方式来组织自己的运行时数据结构。特别是入门级、简单的JVM实现更是如此。 

Sun JDK 1.0.2时代的Sun JVM的做法是：它是一个32位JVM，解释器栈用32位为单位的slot来组织。JVM规范里所规定的局部变量、操作数栈的项就跟这里的slot对应。int与比int窄的整型、float、reference、returnAddress都占1个slot，long与double占两个slot。 
楼主问char实际存储占多少空间，在这个JVM里char在栈上占4字节。甚至连有效数据只有1位的boolean在栈上也占4字节。 

浪费JVM栈空间影响大吗？有影响，但还通常不算糟糕。 
毕竟JVM栈空间主要受局部变量的个数以及方法嵌套调用的深度影响；只要嵌套调用不太深，其实用不了多少JVM栈空间。只有在深度递归或者深度嵌套调用的情况下，JVM栈空间才会显得紧缺。 

然后看这个JVM如何组织Java堆来存储Java对象。要分为Java类的实例和数组实例两方面来看。
它很偷懒，Java类的实例的字段同样使用32位为单位的slot来组织，字段在内存里的顺序与Class文件存的field_info顺序一致，没有重排序。这里1个char也是占了4字节，虽然其中只有2字节是有效数据。 
而数组则采用紧凑布局，数组元素的宽度与其有效宽度一致（boolean除外，1个boolean占1字节但有效数据只有1位）。于是这个JVM里char[]里的char就占2字节。 

浪费Java堆影响大吗？这就大了。典型的面向对象应用会产生相当大量的对象，每个小字段都浪费一点就可以浪费很多。特别是当编写Java代码的程序员原本为了省内存而用较窄的字段时，会失望的发现在这个JVM上几乎是无用功。 

楼主有没有感受到原始数据类型“占多少字节”不是那么简单的事情了？ 在哪个JVM实现里、存在什么地方都有关系。 

那么拉到现在我们身边的现实，Oracle/Sun JDK里的HotSpot VM。 HotSpot VM有32位和64位版。楼主问的问题在这俩版本上表现不同。 另外在JVM栈的方面，解释器用的栈桢（interpreted frame）与被JIT编译后的代码用的栈桢（compiled frame）的布局也不一样。 

32位版上，解释器的栈桢布局是以32位为单位的slot来存储局部变量与操作数栈项的。情况跟楼主的想像相同：1个char在局部变量与在操作数栈上都占4字节。 
编译后代码的栈桢布局里，局部变量也是用32位slot来存储，没有操作数栈。 

64位版上，解释器的栈桢的slot是64位的。这里1个char会占用8字节，1个boolean、short、int、float也是8字节；而1个long或double仍然占2个slot，也就是64*2＝128字节。这主要是为了实现方便，代码容易与32位版统一起来。 
编译后代码的栈桢里，局部变量也是用64位slot，但long和double可以只占1个slot；同样没有操作数栈。这里1个char也占8字节。 

在Java堆上，HotSpot VM采用紧凑的对象布局。字段会根据其宽度做重排序，以期尽量有效的使用内存。除了boolean占1字节外，其它数据类型的字段都占与其有效宽度相同的大小。数组也是一样。所以在HotSpot VM里Java对象的char类型字段以及char[]里的char都是占2字节。 

另外还有更有趣的：在Oracle/Sun JDK 6的后期版本里有压缩字符串功能，可以用-XX:+UseCompressedStrings打开；JDK7与JDK8暂时去掉了这个功能。 
本来1个java.lang.String实例会引用着1个char[]来存储实际字符串内容。当这个功能开启的时候，构造String实例时会检查里面的内容是否只有ASCII范围内的字符，如果是就用byte[]来存字符串内容，不是就退回到用char[]来存。String的用户则完全不受影响，仍然“觉得”String里装着的是一串char。 
这样，表面上有6个char的1个String，背后的每个字符可能只占了1字节（因为每个字符都是存在byte[]里的一个元素）。 
要留意这里是说“压缩字符串”，而不是“压缩字符”。这种实现中char仍然是UTF-16的code point，仍然至少要占2个字节的有效空间。 

================================================= 

### 如何“证明” 

开源的JVM的话读源码就足以知道了。HotSpot VM不但开源，而且有许多工具便于调试。例如我以前介绍过的Serviceability Agent用来看内存布局的信息就很方便。 
http://rednaxelafx.iteye.com/blog/1847971 

不开源的JVM有些有零星的文档里有说实现细节。再不济可以自己用native debugger去调试一吧。windbg、gdb之类都是你的伙伴。喜欢IDA或者OllyDBG也行。上面说的JDK 1.0.2的Sun JVM我是自己调试来了解其实现细节的。

