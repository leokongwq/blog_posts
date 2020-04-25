---
layout: post
comments: true
title: JDK8环境下使用struts2
date: 2018-11-07 22:07:54
tags:
- struts2
- JDK8
categories:
---

### 背景

最近将组内项目的部署环境进行了一次升级。将JDK1.7S升级为1.8，Resin替换为Tomcat。在升级替换的过程中遇到了一些问题。特记录再次，希望能帮助有同样需求的朋友。

<!-- more -->

### Struts2 和 JDK8

项目中使用的`Struts2`版本是`2.3.35`。

```xml
<dependency>
    <groupId>org.apache.struts</groupId>
    <artifactId>struts2-core</artifactId>
    <version>2.3.35</version>
</dependency>
```

Struts2里面依赖`xwork-core`

```xml
<dependency>
    <groupId>org.apache.struts.xwork</groupId>
    <artifactId>xwork-core</artifactId>
    <version>2.3.35</version>
</dependency>
```

`xwork-core`依赖`asm-*`

问题来了!

低版本的`ASM`不能在JDK1.8环境中使用。如果强行使用，会导致一些奇怪的问题。

例如：

1. 只有一部分`Action`类可以正常被Struts2加载并处理http请求。某些在JDK1.7环境下可以正常工作的`Action`不能在JDK1.8下使用。原来可以访问的接口，现在是`404`。

具体问题出在：

```java  DefaultClassFinder
 private void readClassDef(String className) {
   if (!className.endsWith(".class")) {
       className = className.replace('.', '/') + ".class";
   }
   try {
       URL resource = classLoaderInterface.getResource(className);
       if (resource != null) {
           InputStream in = resource.openStream();
           try {
               ClassReader classReader = new ClassReader(in);
               classReader.accept(new InfoBuildingVisitor(this), ClassReader.SKIP_DEBUG);
           } finally {
               in.close();
           }
       } else {
           throw new XWorkException("Could not load " + className);
       }
   } catch (IOException e) {
       throw new XWorkException("Could not load " + className, e);
   }
}
```

这部分代码就因为使用了低版本的`ASM`导致类解析失败(`IndexOutOfBoundsException`)。

#### 解决办法一

最简单方便的解决版本就是升级Struts2的版本到`2.5.x`。新版本将`xwork`依赖直接合并到`struts2-core`中了。而且使用了`ASM 5.X`版本，支持JDK8。

#### 解决办法二

使用Struts2官方提供的一个插件。具体用法如下：

-------

在项目中加入依赖：

```xml
<dependency>
    <groupId>org.apache.struts</groupId>
    <artifactId>struts2-java8-support-plugin</artifactId>
    <version>2.3.35</version>
</dependency>
```

排除ASM依赖

```xml
<dependency>
    <groupId>org.apache.struts.xwork</groupId>
    <artifactId>xwork-core</artifactId>
    <exclusions>
        <exclusion>
            <groupId>asm</groupId>
            <artifactId>asm</artifactId>
        </exclusion>
        <exclusion>
            <groupId>asm</groupId>
            <artifactId>asm-commons</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

### Struts2 版本升级问题

#### 标签库不兼容

众所周知，Struts2框架的安全问题很多，建议升级到最新版本`2.5.x`。

但是2.5.x版本的Struts2提供的**标签库**和低版本的不兼容。这就会导致原有的**JSP页面不能正常渲染**。

当然了，如果你的项目里面没有使用Struts2替换的标签，这个问题可以忽略了。

#### 核心类拦截器变化

```xml
<filter>
    <filter-name>struts2</filter-name>
        <filter-class>org.apache.struts2.dispatcher.filter.StrutsPrepareAndExecuteFilter</filter-class>
<!-- org.apache.struts2.dispatcher.ng.filter.StrutsPrepareAndExecuteFilter 
之前的核心过滤器全类名会有个ng  ,struts2.5核心过滤器没有这个
-->
</filter>
<filter-mapping>
    <filter-name>struts2</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

### aspectjweaver

我们项目使用的版本是：

```xml
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.6.9</version>
</dependency>
```

升级JDK1.8以后，需要同时升级该jar的版本到`1.8.13`。


### 参考

[Struts2.5配置](https://blog.csdn.net/gh670011677/article/details/75019003)

[Java 8 Support Plugin](https://struts.apache.org/plugins/java-8-support/)

[Struts+2.3+to+2.5+migration](https://cwiki.apache.org/confluence/display/WW/Struts+2.3+to+2.5+migration)

[what-is-the-difference-between-struts-2-3-x-and-struts-2-5-x](https://stackoverflow.com/questions/41307863/what-is-the-difference-between-struts-2-3-x-and-struts-2-5-x)

[ASM-VERSIONS](https://asm.ow2.io/versions.html)

[Struts2最新RCE漏洞S2-057](https://nosec.org/home/detail/1755.html)


