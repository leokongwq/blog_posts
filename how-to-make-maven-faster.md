---
layout: post
comments: true
title: maven加速
date: 2018-11-03 12:42:36
tags:
- maven
categories:
---

### 前言

由于网络原因，国内访问maven中央仓库速度很慢。编译大型Maven项目时速度很慢。此时可以通过公用的或私有的镜像站来进行加速。

<!-- more -->

### 国内Maven镜像站点

#### aliyun

```java
<mirror>  
      <id>alimaven</id>  
      <name>aliyun maven</name>  
      <url>http://maven.aliyun.com/nexus/content/groups/public/</url>  
      <mirrorOf>central</mirrorOf>    
</mirror>
```

#### oschina

```java
<mirror>    
    <id>CN</id>  
    <name>OSChina Central</name>         
    <url>http://maven.oschina.net/content/groups/public/</url>  
    <mirrorOf>central</mirrorOf>    
</mirror>
```

### setting.xml 配置

```java
<mirrors>
    <!-- 作为中央仓库的镜像 -->
    <mirror>
      <id>nexus-aliyun</id>
      <name>Nexus aliyun</name>
      <mirrorOf>central</mirrorOf>
      <url>http://maven.aliyun.com/nexus/content/groups/public</url>
    </mirror>
    <!-- 私有， 公司内部使用 -->
    <mirror>
      <id>nexus-mine</id>
      <name>Nexus mine</name>
      <mirrorOf>*</mirrorOf>
      <url>http://xx.xx.xx.xx/nexus/content/groups/public</url>
    </mirror>
  </mirrors>
```

### mirrorOf 配置

`mirrorOf` 用来指定该镜像针对的仓库。用法如下：

- `*` 匹配所有仓库
- `external:*` 匹配除了本机和基于文件的所有外部构建地址。
- `repo,repo1` 匹配仓库`repo`和`repo1`
- `*,!repo1` 除了仓库`repo1`匹配所有

### 参考

[guide-mirror-settings](http://maven.apache.org/guides/mini/guide-mirror-settings.html)


