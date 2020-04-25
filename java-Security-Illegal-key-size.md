---
layout: post
comments: true
title: java-Security-Illegal-key-size
date: 2017-06-19 14:53:51
tags:
categories:
- java
---

### 背景

项目中使用了Java的加密库，使用的过程中出现异常`Caused by: java.security.InvalidKeyException: Illegal key size or default parameters`。 Google后发现需要替换对应的jar包。

### 解决办法

参考: [https://stackoverflow.com/questions/6481627/java-security-illegal-key-size-or-default-parameters](https://stackoverflow.com/questions/6481627/java-security-illegal-key-size-or-default-parameters)

1. 下载JDK对应版本的JCE[http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html](http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html)
2. 解压下载文件，将`JAVA_HOME`的子目录`jre/lib/security`中的文件`US_export_policy.jar, local_policy.jar`替换为JCE下载的jar。

### 原因

Illegal key size or default parameters是指密钥长度是受限制的，java运行时环境读到的是受限的policy文件。这种限制是因为美国对软件出口的控制。


### 扩展阅读

[Java安全技术探索之路系列](http://blog.csdn.net/allenwells/article/details/46505849)

