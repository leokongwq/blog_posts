---
layout: post
comments: true
title: golang程序GDB调试
date: 2016-10-14 16:41:01
tags:
    - go
categories:
    golang
---

 **说明：**作为一门静态语言，似乎支持调试是必须的，而且，Go初学者喜欢问的问题也是：大家都用什么IDE？怎么调试？

<!-- more -->

其实，Go是为多核和并发而生，真正的项目，你用单步调试，原本没问题的，可能会调出有问题。更好的调试方式是跟PHP这种语言一样，用打印的方式（日志或print）。

当然，简单的小程序，如果单步调试，可以看到一些内部的运行机理，对于学习还是挺有好处的。下面介绍一下用GDB调试Go程序：（目前IDE支持调试Go程序，用的也是GDB。要求GDB 7.1以上）

以下内容来自雨痕的《Go语言学习笔记》（下载Go资源）：

默认情况下，编译过的二进制文件已经包含了 DWARFv3 调试信息，只要 GDB7.1 以上版本都可以进行调试。 在OSX下，如无法执行调试指令，可尝试用sudo方式执行gdb。

删除调试符号：`go build -ldflags “-s -w”`

-s: 去掉符号信息。
-w: 去掉DWARF调试信息。

关闭内联优化：go build -gcflags “-N -l”

调试相关函数：

runtime.Breakpoint()：触发调试器断点。
runtime/debug.PrintStack()：显示调试堆栈。
log：适合替代 print显示调试信息。
GDB 调试支持：

参数载入：gdb -d $GCROOT 。
手工载入：source pkg/runtime/runtime-gdb.py。                        
                    
                    