---
layout: post
comments: true
title: lombook工具包简介
date: 2016-10-15 19:56:23
tags:
- java
categories:
- java    
---

 ### 介绍

> lombok 提供了简单的注解的形式来帮助我们简化消除一些必须有但显得很臃肿的 java 代码。特别是相对于 POJO。官方网站[lombook](https://projectlombok.org/)。它由2部分构成。一部分是idea插件，主要实现编译增强。另一部分是它的API。

<!-- more -->

### idea 安装lombook插件

安装文档：[lombook-idea-install](https://github.com/mplushnikov/lombok-intellij-plugin)  

### maven依赖
    
     <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.16.8</version>
        <scope>provided</scope>
    </dependency>

### 示例代码

    @Data
    public class Person {
        private Long id;
        private String name;
        private String email;
    }               

### 总结

lombook的原理就是通过编译时字节码修改技术，通过查看类的注解信息。生成对应的方法或属性。这样就减少了我们手动编写或生成啰嗦但必要的代码。能减轻工作量，并且源代码开启了清晰一些。                    
                    