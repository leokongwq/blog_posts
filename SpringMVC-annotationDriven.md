---
layout: post
title: mvc:annotation-driver
comments: true
categories:
- spring
tags:
- spring
- spring-mvc
---

### `<mvc:annotation-driver />`背后的故事

** spring-mvc 版本3.0.5**

我们在用spring-mvc开发时，通常会采用少量xml配置加注解的方式进行开发，通常会再xxxx-servelet.xml配置文件中添加如下的配置：

	<mvc:annotation-driven />
	
根据我们对spring工作方式的理解，这样的标签会有对应的java类来进行解析，而所有的标签解析器都是
**BeanDefinitionParser** 接口的子类.	在它的子类中我们找到了**AnnotationDrivenBeanDefinitionParser.class** 这个类，源码文档有这么句简单的解释：

	{@link BeanDefinitionParser} that parses the {@code annotation-driven} element to configure a Spring MVC web application.	
	
由此我们可以判定该类就是上述标签的解析类。
