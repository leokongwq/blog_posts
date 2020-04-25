---
layout: post
comments: true
title: maven jar包依赖范围总结
date: 2017-03-29 18:39:06
tags:
- maven
categories:
- java
---

### 背景

在Java项目开发中，我们通常都是使用maven来管理整个项目的。而其中依赖jar的管理是非常重要的，如果管理不善，很容易jar包冲突导致应用突然启动不起来。这个一方面是由于jar包引入不规范导致，另一方面也是由于依赖jar本身也会依赖其它的jar，但是这些第三方依赖jar依赖的jar的maven scope可能使用不当导致同一个功能不同版本jar的引入。

<!-- more -->

### maven 依赖jar包scope范围解析

maven针对jar包依赖总共提供了6个scope，分别如下：

#### compile

这个是默认的scope。这个范围的jar包依赖会被打包的部署的功能文件中，并且在maven生命周期的compile，test，package 等阶段都存在于classpath中。并且该范围的依赖会传递到子pom依赖中

#### provided

这个scope和compile有点相似，不同点是这个范围的jar依赖不会被打包到部署的工程文件中，默认由部署的应用环境来提供。非常典型的是servlet-api。该范围的jar依赖需要在maven生命周期的：compile， test 阶段存在。 该范围的jar依赖不具有传递性。

#### runtime

这个范围的依赖不需要在compile阶段使用，但必须在test和runtime运行时存在。也会存在于打包后的部署文件中。

#### test

这个范围的依赖需要在test阶段的compile和runtime存在， 不会传递依赖，不会被打入部署文件中。

#### system

这个范围的依赖类似provided， 不同的地方在于你必须显示指定jar的位置，并且该jar的artifact必须存在，也不会从maven仓库里查找。

#### import (在 Maven 2.0.9 或更高版本使用)

这个范围的依赖只支持pom类型的依赖， 而且必须在存在于模块pom.xml的`dependencyManagement`配置区内。这种依赖用来指示被依赖的pom类型的依赖中存在于`dependencyManagement`中的依赖是用来被替换的。这个功能是用来解决pom.xml的多继承问题的， 让你利用组合而不是继承关系来引入依赖。最典型的springboot。

```xml
<dependencyManagement>  
    <dependencies>  
        <dependency>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-dependencies</artifactId>  
            <version>1.3.3.RELEASE</version>  
            <type>pom</type>  
            <scope>import</scope>  
        </dependency>  
    </dependencies>  
</dependencyManagement>  
  
<dependencies>  
    <dependency>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-web</artifactId>  
    </dependency>  
</dependencies>  
```







