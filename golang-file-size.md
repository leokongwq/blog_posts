---
layout: post
comments: true
title: golang编译后文件编写的技巧
date: 2016-10-15 19:56:15
tags:
- go
categories:
- golang
---

> 总有人说Go程序“好大”，一个Hello World都1M多。其实，随着程序源码越来越大，编译后的文件并非那么快速的增长，这点大小真心没必要那么在乎，又不是软盘时代。但总有一些人非得想要小点。

首先我们看一下为什么会比其他语言大些：

    Go 编译的可执行文件都包含了一个运行时(runtime)，和我们习惯的Java/.NET VM有些类似。运行时负责内存分配（Stack Handing、    GC Heap）、垃圾回收（Garbage Collection）、Goroutine调度（Schedule）、引用类型（slice、map、channel）管理，以及反射（Reflection）等工作。Go程序进程启动后会自动创建两个goroutine，分别用于执行main入口函数和GC Heap管理。也正是因为编译文件中嵌入了运行时，使得其可执行文件相较其他语言更大一些。但Go的二进制可执行文件都是静态编译的，无需其他任何链接库等文件，更利于分发。                        


我们可以通过下面的方法将其变小点：

采用：go build -ldflags "-s -w" 这种方式编译。

解释一下参数的意思：

-ldflags： 表示将后面的参数传给连接器（5/6/8l）
-s：去掉符号信息
-w：去掉DWARF调试信息
注意：

-s 去掉符号表（这样panic时，stack trace就没有任何文件名/行号信息了，这等价于普通C/C+=程序被strip的效果）

-w 去掉DWARF调试信息，得到的程序就不能用gdb调试了

两个可以分开使用

实际项目中不建议做这些处理，没啥必要。                    

**原文转自** [http://studygolang.com/topics/98](http://studygolang.com/topics/98)
                    
                    
                    