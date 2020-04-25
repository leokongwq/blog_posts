---
layout: post
comments: true
title: vertx-访问文件系统
date: 2017-11-18 14:10:27
tags:
- vert.x
categories:
---

Vert.x 中的[FileSystem](http://vertx.io/docs/apidocs/io/vertx/core/file/FileSystem.html)对象提供了许多操作文件系统的方法。

每个Vert.x 实例有一个文件系统对象，您可以使用`fileSystem`方法获取它。

每个操作都提供了阻塞和非阻塞版本，其中非阻塞版本接受一个处理器 Handler，当操作完成或发生错误时调用该处理器。

以下是文件异步拷贝的示例：

<!-- more -->

```java
FileSystem fs = vertx.fileSystem();

// Copy file from foo.txt to bar.txt
// 从foo.txt拷贝到bar.txt
fs.copy("foo.txt", "bar.txt", res -> {
    if (res.succeeded()) {
        // Copied ok!
        // 拷贝完成
    } else {
        // Something went wrong
        // 出了些问题
    }
});
```

阻塞版本的方法名为 xxxBlocking，它要么返回结果或直接抛出异常。很多情况下，一些潜在的阻塞操作可以快速返回（这取决于操作系统和文件系统），这就是我们为什么提供它。但是强烈建议您在 Event Loop 中使用它之前测试使用它们究竟需要耗费多长时间，以避免打破黄金法则。

以下是使用阻塞 API的拷贝示例：

```java
FileSystem fs = vertx.fileSystem();

// Copy file from foo.txt to bar.txt synchronously
// 同步拷贝从foo.txt到bar.txt
fs.copyBlocking("foo.txt", "bar.txt");
```

Vert.x 文件系统支持诸如 copy、move、truncate、chmod 和许多其他文件操作。我们不会在这里列出所有内容，请参考 [API 文档](http://vertx.io/docs/apidocs/io/vertx/core/file/FileSystem.html) 获取完整列表。

让我们看看使用异步方法的几个例子：

```java
Vertx vertx = Vertx.vertx();

// Read a file
// 读取文件
vertx.fileSystem().readFile("target/classes/readme.txt", result -> {
    if (result.succeeded()) {
        System.out.println(result.result());
    } else {
        System.err.println("Oh oh ..." + result.cause());
    }
});

// Copy a file
// 拷贝文件
vertx.fileSystem().copy("target/classes/readme.txt", "target/classes/readme2.txt", result -> {
    if (result.succeeded()) {
        System.out.println("File copied");
    } else {
        System.err.println("Oh oh ..." + result.cause());
    }
});

// Write a file
// 写文件
vertx.fileSystem().writeFile("target/classes/hello.txt", Buffer.buffer("Hello"), result -> {
    if (result.succeeded()) {
        System.out.println("File written");
    } else {
        System.err.println("Oh oh ..." + result.cause());
    }
});

// Check existence and delete
// 检测存在以及删除
vertx.fileSystem().exists("target/classes/junk.txt", result -> {
    if (result.succeeded() && result.result()) {
        vertx.fileSystem().delete("target/classes/junk.txt", r -> {
            System.out.println("File deleted");
        });
    } else {
        System.err.println("Oh oh ... - cannot delete the file: " + result.cause());
    }
});
```

### 异步文件访问

Vert.x提供了异步文件访问的抽象，允许您操作文件系统上的文件。

您可以像下边代码打开一个 [AsyncFile](http://vertx.io/docs/apidocs/io/vertx/core/file/AsyncFile.html) ：

```java
OpenOptions options = new OpenOptions();
fileSystem.open("myfile.txt", options, res -> {
    if (res.succeeded()) {
        AsyncFile file = res.result();
    } else {
        // Something went wrong!
        // 出了些问题
    }
});
```

`AsyncFile`实现了`ReadStream` 和`WriteStream` 接口，因此您可以将文件和其他流对象配合 泵 工作，如`NetSocket`、HTTP 请求和响应和`WebSocket` 等。

它们还允许您直接读写。

### 随机访问写

要使用`AsyncFile`进行随机访问写，请使用 `write` 方法。

这个方法的参数有：

- buffer：要写入的缓冲
- position：一个整数指定在文件中写入缓冲的位置，若位置大于或等于文件大小，文件将被扩展以适应偏移的位置
- handler：结果处理器

这是随机访问写的示例：

```java
Vertx vertx = Vertx.vertx();
vertx.fileSystem().open("target/classes/hello.txt", new OpenOptions(), result -> {
    if (result.succeeded()) {
        AsyncFile file = result.result();
        Buffer buff = Buffer.buffer("foo");
        for (int i = 0; i < 5; i++) {
            file.write(buff, buff.length() * i, ar -> {
                if (ar.succeeded()) {
                    System.out.println("Written ok!");
                    // etc
                    // 等
                } else {
                    System.err.println("Failed to write: " + ar.cause());
                }
            });
        }
    } else {
        System.err.println("Cannot open file " + result.cause());
    }
});
```

### 随机访问读

要使用`AsyncFile`进行随机访问读，请使用 `read` 方法。

该方法的参数有：

- buffer：读取数据的 Buffer
- offset：读取数据将被放到 Buffer 中的偏移量
- position：从文件中读取数据的位置
- length：要读取的数据的字节数
- handler：结果处理器

一下是随机访问读的示例：

```java
Vertx vertx = Vertx.vertx();
vertx.fileSystem().open("target/classes/les_miserables.txt", new OpenOptions(), result -> {
    if (result.succeeded()) {
        AsyncFile file = result.result();
        Buffer buff = Buffer.buffer(1000);
        for (int i = 0; i < 10; i++) {
            file.read(buff, i * 100, i * 100, 100, ar -> {
                if (ar.succeeded()) {
                    System.out.println("Read ok!");
                } else {
                    System.err.println("Failed to write: " + ar.cause());
                }
            });
        }
    } else {
        System.err.println("Cannot open file " + result.cause());
    }
});
```

### 打开选项

打开 `AsyncFile` 时，您可以传递一个 OpenOptions 实例，这些选项描述了访问文件的行为。例如：您可使用 `setRead`、`setWrite`和`setPerm` 方法配置文件访问权限。

若打开的文件已经存在，则可以使用 `setCreateNew` 和 `setTruncateExisting` 配置对应行为。

您可以使用 `setDeleteOnClose` 标记在关闭时或JVM停止时要删除的文件。

### 将数据刷新到底层存储

在 `OpenOptions` 中，您可以使用 `setDsync` 方法在每次写入时启用/禁用内容的自动同步。这种情况下，您可以使用 `flush` 方法手动刷新OS缓存中的数据写入。

该方法也可使用一个处理器来调用，这个处理器在 `flush` 完成时被调用。

### 将 AsyncFile 作为 ReadStream 和 WriteStream

`AsyncFile`实现了 `ReadStream` 和 `WriteStream` 接口。您可以使用泵将数据与其他读取和写入流进行数据泵送。例如，这会将内容复制到另外一个`AsyncFile`：

```java
Vertx vertx = Vertx.vertx();
final AsyncFile output = vertx.fileSystem().openBlocking("target/classes/plagiary.txt", new OpenOptions());

vertx.fileSystem().open("target/classes/les_miserables.txt", new OpenOptions(), result -> {
    if (result.succeeded()) {
        AsyncFile file = result.result();
        Pump.pump(file, output).start();
        file.endHandler((r) -> {
            System.out.println("Copy done");
        });
    } else {
        System.err.println("Cannot open file " + result.cause());
    }
});
```

您还可以使用泵将文件内容写入到HTTP 响应中，或者写入任意 WriteStream。

### 从 Classpath 访问文件

当Vert.x找不到文件系统上的文件时，它尝试从类路径中解析该文件。请注意，类路径的资源路径不以 / 开头。

由于Java不提供对类路径资源的异步方法，所以当类路径资源第一次被访问时，该文件将复制到工作线程中的文件系统。当第二次访问相同资源时，访问的文件直接从（工作线程的）文件系统提供。即使类路径资源发生变化（例如开发系统中），也会提供原始内容。

您可以将系统属性`vertx.disableFileCaching`设置为true，禁用此（文件）缓存行为。文件缓存的路径默认为.vertx，它可以通过设置系统属性`vertx.cacheDirBase`进行自定义。

您还可以通过系统属性`vertx.disableFileCPResolving`设置为true来禁用整个类路径解析功能。

> 请注意：当加载`io.vertx.core.impl.FileResolver`类时，这些系统属性将被评估一次，因此，在加载此类之前应该设置这些属性，或者在启动它时作为JVM系统属性来设置。

### 关闭 AsyncFile

您可调用 `close` 方法来关闭`AsyncFile`。关闭是异步的，如果希望在关闭过后收到通知，您可指定一个处理器作为函数（close）参数传入。







