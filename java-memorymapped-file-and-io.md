---
layout: post
comments: true
title: java中内存映射文件和IO
date: 2017-02-25 18:58:29
tags:
- 面试
categories:
- java
---

### 前言

对大多数Java开发人员来说，Java中的内存映射文件都是一个新的概念，即使它早在JDK1.4时已经被添加到包`java.nio`中了。在引入NIO和内存映射文件后，Java中拥有了非常快的IO操作能力，这也是为什么高性能Java应用程序使用内存映射文件来持久化数据的主要原因。它已经在高频交易系统中非常流行了，其中电子交易系统需要超快速，并且单向交换的延迟必须在亚微秒级别上。 IO一直是性能敏感的应用程序中需要关注的，内存映射文件允许你通过使用直接和非直接字节缓冲区直接从内存读取和写入内存。

<!-- more -->

> 内存映射文件的主要优点是操作系统负责读写，即使你的程序在写入内存后崩溃。 操作系统将负责将内容写入文件。

另一个显着的优点是共享内存，内存映射文件可以由多个进程访问，并且可以充当具有极低延迟的共享内存。

早些时候我们已经看到了[如何在Java中读取xml文件](http://javarevisited.blogspot.com/2011/12/parse-xml-file-in-java-example-tutorial.html)和[如何在Java读取文本文件](http://javarevisited.blogspot.com/2011/12/read-and-write-text-file-java.html)在这个Java IO教程我们将料及到什么是内存映射文件，如何从内存映射文件读取和写入内存映射文件以及与内存映射文件相关的重点。

###  What is Memory Mapped File and IO in Java

内存映射文件是Java中的特殊文件，允许Java程序直接访问内存中的内容，这是通过将整个文件或文件的一部分映射到内存中实现的，操作系统负责处理页请求和写入文件，而应用程序只处理结果所在内存区域，这导致非常快的IO操作。 用于加载内存映射文件的内存超出Java堆空间的大小。 Java编程语言通过`java.nio`包来支持内存映射文件，并且有MappedByteBuffer类用来读取内存和写入内存。

#### Advantage and Disadvantage of Memory Mapped file

内存映射IO的主要优点是性能，这对构建高频电子交易系统很重要。内存映射文件比通过标准IO访问标准文件的方式快。内存映射IO的另一个大的优点是，它允许你加载潜在的更大的文件，否则不能访问。实验表明，内存映射IO对大文件执行得更好。虽然它在增加页错误的数量方面具有缺点。因为操作系统只将一部分文件加载到内存中，如果请求的页面不存在于内存中而不会导致页面错误大多数主要操作系统，如Windows平台，UNIX，Solaris和其他类UNIX操作系统支持内存映射IO和使用64位架构，您可以将几乎任何文件映射到内存中，并使用Java编程语言直接访问它。另一个优点是可以共享文件，在进程之间提供共享内存，并且比使用基于Socket和回环地址的同学可以降低10倍以上的延迟。

#### MappedByteBuffer Read Write Example in Java

下面的例子将告诉你如何对Java中的内存映射文件进行读写。 我们使用RandomAccesFile打开一个文件，然后使用`FileChannel`的`map()`方法将其映射到内存，map方法采用三个参数, 模式，开始和要映射的区域的长度。 它返回MapppedByteBuffer，它是一个用于处理内存映射文件的ByteBuffer。

```java
import java.io.RandomAccessFile;
import java.nio.MappedByteBuffer;
import java.nio.channels.FileChannel;

  public class MemoryMappedFileInJava {

    private static int count = 10485760; //10 MB

    public static void main(String[] args) throws Exception {

        RandomAccessFile memoryMappedFile = new RandomAccessFile("largeFile.txt", "rw");

        //Mapping a file into memory

        MappedByteBuffer out = memoryMappedFile.getChannel().map(FileChannel.MapMode.READ_WRITE, 0, count);

        //Writing into Memory Mapped File
        for (int i = 0; i < count; i++) {
            out.put((byte) 'A');
        }

        System.out.println("Writing to Memory Mapped File is completed");

     

        //reading from memory file in Java
        for (int i = 0; i < 10 ; i++) {
            System.out.print((char) out.get(i));
        }

        System.out.println("Reading from Memory Mapped File is completed");

    }

}
```

### Summary

快速总结本文关于内存映射文件和Java IO的知识点如下：

1) Java使用`java.nio`包来支持内存映射文件IO
2) 内存映射文件用在堆性能非常敏感的应用程序中，例如高频电子交易系统。
3) 通过使用内存映射的IO，你可以加载大文件的一部分到内存中。
4) 如果请求的页面不在内存中，内存映射文件可能导致页面错误.
5) 将文件的一部分映射到的内存的大小取决于内存的可寻址大小。 在32位机器中，你不能访问超过4GB或2 ^ 32。
6) 内存映射的IO比Java中的Stream IO快得多.
7) 用于加载文件的内存在Java堆之外，驻留在共享内存上，允许两个不同的进程访问文件。 顺便说一下，这取决于你是否使用直接或非直接字节缓冲区。
8) 在内存映射文件上的读写是由操作系统完成的，所以即使你的Java程序崩溃后，只要操作系统没有奔溃，则操作系统会将其中的内如写入磁盘。
9) 优先使用直接字节缓冲区而不是非直接缓冲区，以此获得更高的性能。
10) 不要经常调用`MappedByteBuffer.force()`方法，该方法会强制操作系统将内存的内容写入磁盘，所以如果每次写入内存映射文件后调用`force()`，你不会看到使用映射字节缓冲区的真正好处，而是类似于磁盘IO.
11) 在电源故障或主机故障的情况下，内存映射文件的内容不会写入磁盘的，这意味着您可能会丢失关键数据.
12) 除非缓冲区被垃圾收集，否则`MappedByteBuffer`和文件映射一直保持有效。`sun.misc.Cleaner`可能是清除内存映射文件的唯一方法。

这是Java中关于内存映射文件和内存映射IO的所有内容。如果你需要开发高频交易系统，则应该了解更多这方面的内容。

### 扩展

[http://blog.csdn.net/fcbayernmunchen/article/details/8635427](http://blog.csdn.net/fcbayernmunchen/article/details/8635427)




