---
layout: post
comments: true
title: java agent技术学习总结
date: 2019-08-25 21:18:49
tags:
- java agent
categories:
- java
---

### 名词解释

JVMDI：jvm debug interface jvm 调试接口。
JVMPI：jvm profile interface jvm 性能分析接口。
JVMTI：jvm tool interface jvm 工具接口，jdk1.5引入，用来替换JVMDI和JVMPI。


### JPDA

JPDA： [Java Platform Debugger Architecture](http://java.sun.com/products/jpda/) SUN 公司提供的让你在各种场景加调试运行中的Java程序的技术。

#### 名词解释

1. JPDA 是一组帮助调试Java程序的API集合。
2. JPDA 不是一个应用程序或调试工具。
3. JPDA 是一组精心设计和实现的接口和协议。
4. debugger 调试程序
5. debuggee 被调试程序

<!-- more -->


### JPDA 构成

JPDA 由三大部分组成：

1. JVMTI（Java Virtual Machine Tools Interface） JVMTI 定义了一个VM必须提供的调试服务。
2. JDWP（The Java Debug Wire Protocol）JDWP 定义了调试程序和被调试程序间的交互协议。 
3. JDI (Java Debug Interface) JDI在用户代码级别定义调试信息和调试请求。

JPAD 架构

{% asset_img jpda.gif %}

debugger 和 debuggee 之间的通讯通道由两部分组成：

1. 连接器 connector。 一个 connector 是一个 JDI 对象，表示debugger 和 debuggee之间建立的连接。JPDA 定义了debugger 和 debuggee 之间有三种连接方式：
    - listening: debugger 监听由debuggee发送的连接请求。
    - attaching: debugger 连接到运行中debuggee上，然后进行调试数据的传输。这是使用最多的方式。
    - launching: The front-end actually launches the Java process that will run the debuggee code and the back-end.
2. 传输通道 transport。 一个 transport 定义了debugger和debuggee底层之间交换数据的方式。
该数据传输机制规范没有规定，可选的机制有:socket, 串口，共享内存或其他方式。 但是传输的数据格式是由JDWP规定好的。

### JPDA 使用

使用JPDA，需要在JVM命令行添加如下格式的参数：

```java
-agentlib:jdwp=<name1>[=<value1>],<name2>[=<value2>]

或 

-Xrunjdwp:<name1>[=<value1>],<name2>[=<value2>]
```

一个例子：

```java
-agentlib:jdwp=transport=dt_socket,server=y,address=8000
```

#### 参数详解

- help: 输出详细的使用信息
- transport: 一般都是用 dt_socket。
- server: 参数值是`y`或`n`。如果设置为`y`，则等待debugger程序来连接。否则，连接到特定地址的debugger程序。
- address: 连接的传输地址。如果`server`参数设置为`n`，则该地址为debugger的地址。如果`server`参数设置为`y`，则在改地址等待dubugger连接。
- timeout: 连接超时的时间，单位为毫秒。
- suspend: 如何设置为`y`，则JVM挂起执行流程，知道dubugger连接到该debuggee JVM。

### 参考

[debug-your-java-code-with-ease-using-jpda](https://www.techrepublic.com/article/debug-your-java-code-with-ease-using-jpda/)
[jpda](https://docs.oracle.com/javase/8/docs/technotes/guides/jpda/)
[谈谈Java Intrumentation和相关应用](http://www.fanyilun.me/2017/07/18/%E8%B0%88%E8%B0%88Java%20Intrumentation%E5%92%8C%E7%9B%B8%E5%85%B3%E5%BA%94%E7%94%A8/)


