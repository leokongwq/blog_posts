---
layout: post
comments: true
title: vertx-启动器
date: 2017-11-18 16:05:31
tags:
- vert.x
categories:
---

### 简介

Vert.x `Launcher` 在 fat-jar 中作为主类，由`vertx`命令行实用程序调用。它可执行一组命令，如`run`、`bare`和`start`等

### 扩展 Vert.x 启动器

您可以通过实现自己的 `Command` 类来扩展命令集（仅限于Java）：

<!-- more -->

```java
@Name("my-command")
@Summary("A simple hello command.")
public class MyCommand extends DefaultCommand {

  private String name;

  @Option(longName = "name", required = true)
  public void setName(String n) {
    this.name = n;
  }

  @Override
  public void run() throws CLIException {
    System.out.println("Hello " + name);
  }
}
```

您还需要实现一个 CommandFactory：

```java
public class HelloCommandFactory extends DefaultCommandFactory<HelloCommand> {
  public HelloCommandFactory() {
   super(HelloCommand.class);
  }
}
```

然后创建 `src/main/resources/META-INF/services/io.vertx.core.spi.launcher.CommandFactory` 并且添加一行表示工厂类的完全限定名称：

```
io.vertx.core.launcher.example.HelloCommandFactory
```

构建包含命令的jar。确保包含了SPI文件（`META-INF/services/io.vertx.core.spi.launcher.CommandFactory`）。

然后，将包含该命令的jar放入fat-jar（或包含在其中）的类路径中，或放在Vert.x发行版的lib目录中，您将可以执行：

```java
vertx hello vert.x
java -jar my-fat-jar.jar hello vert.x
```

执行上面的命令就会输出：vert.x

### 命令行相关注解

#### @Name

@Name 该注解用来指定命令的名称，在上面的例子中`hello`就是由该注解指定的

#### @Summary

@Summary 注解用来指定该命令的概要介绍信息，如果需要详细的描述该命令可以使用下面的注解。

#### @Description

@Description 注解用来编写命令和命令选项的文档信息。

#### @Option

@Option 注解用来编写命令选项的配置信息，它有如下的属性

- longName 选项的长名称（`--`前缀）
- shortName 选项的短名称（`-`前缀），如果没有指定，则该命令选项没有短名称。
- argName 该参数的名称，默认值是`value`。 用在doc文档中
- required boolean 该选项是否是必须的
- acceptValue boolean， 默认是true, 表示该选项是否接受一个值
- acceptMultipleValues boolean 表示该选项是否接受多个值, 多个值以`,`或空格分割
- flag 表示该选手是否作为一个标志值（开关值），也就是该选项没有值。
- help boolean 表示该选项是否是帮助信息选项。
- choices String[] 该选项能接受的值集合。

### 在 fat-jar 中使用启动器

要在 `fat-jar` 中使用 `Launcher` 类，只需要将 `MANIFEST` 的 `Main-Class` 设置为 `io.vertx.core.Launcher`。另外，将 `MANIFEST` 中 `Main-Verticle` 条目设置为您的`Main Verticle`的名称。

默认情况下，它会执行 `run` 命令。但是，您可以通过设置 `MANIFEST` 的`Main-Command`条目来配置默认命令。若在没有命令的情况下启动 fat-jar，则使用默认命令。

### 启动器子类

您还可以创建 `Launcher` 的子类来启动您的应用程序。这个类被设计成易于扩展的。

一个 `Launcher` 子类可以：

- 在 beforeStartingVertx 中自定义 Vert.x 配置
- 通过覆盖 afterStartingVertx 来读取由“run”或“bare”命令创建的Vert.x实例
- 使用 getMainVerticle 和 getDefaultCommand 方法配置默认的Verticle和命令
- 使用 register 和 unregister 方法添加/删除命令

### 启动器和退出代码

当您使用 `Launcher` 类作为主类时，它使用以下退出代码：

- 若进程顺利结束，或抛出未捕获的错误：0
- 用于通用错误：1
- 若Vert.x无法初始化：11
- 若生成的进程无法启动、发现或停止：12，该错误代码一般由start和stop命令使用
- 若系统配置不符合系统要求（如找不到 java 命令）：14
- 若主Verticle不能被部署：15






