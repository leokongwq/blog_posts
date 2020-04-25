---
layout: post
comments: true
title: direct, non-direct, mapped bytebuffer区别
date: 2017-02-25 16:46:21
tags:
- 面试
categories:
- java
---

本文翻译自：[http://javarevisited.blogspot.jp/2015/08/difference-between-direct-non-direct-mapped-bytebuffer-nio-java.html](http://javarevisited.blogspot.jp/2015/08/difference-between-direct-non-direct-mapped-bytebuffer-nio-java.html)

### 前言

ByteBuffer 是Java NIO API 中一个非常重要的类。它是在JDK1.4中首次添加的，位于包`java.nio`中。它不仅允许你操作堆内的字节数组，同时也允许你操作JVM堆外的直接内存。 ByteBuffer有三个主要的类型：DirectByteBuffer,  HeapByteBuffer 和 MappedByteBuffer。你可以通过`java.nio.ByteBuffer`类创建直接或非直接字节缓存。 `MappedByteBuffer`是`ByteBuffer`的子类，可以通过`FileChannel.map()`方法来创建，可以用来操作内存映射文件。 直接字节缓存和非直接内存缓存的主要区别在于内存的位置。 非直接内存缓存只是字节数组的包装类，且在Java Heap上。直接字节缓存是位于JVM堆之外的内存区域。

<!-- more -->

你也可以通过他们的名字记住这个事实，Direct表示使用直接内存。 由于这个原因，直接字节缓冲区也不受垃圾收集的影响。 MappedByteBuffer也是一种直接字节缓冲区，表示文件的内存映射区域。

在Java NIO教程中，你将看到直接，非直接和映射字节缓冲区之间的更多差异，这将有助于你更好地理解其概念及用法。

如果你喜欢像我写的这样的书，并想要学习高级的概念，例如。 高性能和低延迟应用程序开发，性能调优和JVM内部实现，我建议看看[Definitive guide to Java Performance](http://www.amazon.com/Java-Performance-The-Definitive-Guide/dp/1449358454?tag=javamysqlanta-20)，Java程序员必读的一本书之一。

### Direct vs Non-direct vs MappedByteBuffer in Java

正如我所说的`ByteBuffer`是高性能应用程序中非常重要的类之一。 它被广泛应用于高频交易应用中，其努力实现非常低的延迟，大多在亚微秒级别。 当我第一次提到Java中的内存映射文件时，我已经概述了使用这些文件的一些好处，ByteBuffer类是操作它们的关键。 大多数直接和非直接ByteBuffer之间的差异来自一个事实，一个是[在堆内存](http://javarevisited.blogspot.com/2013/01/difference-between-stack-and-heap-java.html)而其他是外堆。

1)非直接和直接字节缓冲区之间的第一个区别来自于你如何创建它们这个事实。 你可以通过为缓冲区的内容分配空间或通过将现有字节数组包装到缓冲区中来创建非直接字节缓冲区。 而直接字节缓冲区可以通过调用工厂方法`allocateDirect()`或通过将文件的一个区域直接映射到内存中来创建，称为`MappedByteBuffer`。

2) 针对直接字节缓存区, JVM 执行底层 IO 操作时直接将数据写入或读取缓存区的内容，而不会将数据复制到一个中间缓存区中。这使得对它们执行高速IO操作非常有吸引力，但是这个功能的使用应该很小心。 如果内存映射文件在多个进程之间共享，那么你需要确保它不会被损坏，即内存映射文件的某些区域不会变得不可用。

3) 直接和非直接字节缓冲区之间的另一个区别是，前者的内存占用可能不明显，因为它们在Java堆外部分配，而非直接缓冲区使用堆空间并受到垃圾回收影响。

4) 你可以使用`java.nio.ByteBuffer`类的`isDirect()`方法来检查一个字节缓存区对象是否是直接或非直接的。

{% asset_img bytebuffer-mappedbytebuffer.jpg %}

这些就是Java中的直接，非直接和映射字节缓冲区之间的一些区别。 如果在大容量低延迟系统中工作，大多数情况下，您将使用直接或映射字节缓冲区。 由于ByteBuffer索引是基于整数的，这有效地限制了它们的可寻址空间最高达2GB，你也可能想要从Java 1.7 NIO包中检查BigByteBuffer类，它提供长索引，或者也可以使用偏移来映射不同区域到内存映射文件。

这就是Java中直接，非直接和映射字节缓冲区之间的区别。 只要记住，直接缓冲区是在堆外分配的，它们不受垃圾收集控制，而非直接缓冲区只是位于堆内的字节数组的包装器。 可以通过使用MappedByteBuffer来访问内存映射文件，MappedByteBuffer也是直接缓冲区。 还要记住的一点是，ByteBuffer中的字节默认顺序是BIG_ENDIAN，这意味着多字节值的字节从最高有效到最低有效排序。

### Recommended Book for further reading

- Java Performance The Definitive Guide By Scott Oaks [see here](http://www.amazon.com/Java-Performance-The-Definitive-Guide/dp/1449358454?tag=javamysqlanta-20)
- Pro Java 7 NIO.2 by Anghel Leonard [see here](http://www.amazon.com/Pro-Java-NIO-2-Experts-Voice/dp/1430240113?tag=javamysqlanta-20)



