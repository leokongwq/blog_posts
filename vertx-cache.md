---
layout: post
comments: true
title: 配置 Vert.x 缓存
date: 2017-11-18 16:14:35
tags:
- vert.x
categories:
---

当 Vert.x 需要从类路径中读取文件（嵌入在 fat-jar 中，类路径中jar文件或其他文件）时，它会把文件复制到缓存目录。背后原因很简单：从 jar 或从输入流读取文件是阻塞的。所以为了避免每次都付出代价，Vert.x 会将文件复制到其缓存目录中，并随后读取该文件。这个行为也可配置。

<!-- more -->

首先，默认情况下，Vert.x 使用 `$CWD/.vertx` 作为缓存目录，它在此之间创建一个唯一的目录，以避免冲突。可以使用 `vertx.cacheDirBase` 系统属性配置该位置。如，若当前工作目录不可写（例如在不可变容器上下文环境中），请使用以下命令启动应用程序：

```java
vertx run my.Verticle -Dvertx.cacheDirBase=/tmp/vertx-cache
# 或者
java -jar my-fat.jar vertx.cacheDirBase=/tmp/vertx-cache
```

> 重要提示： 该目录必须是可写的。

当您编辑资源（如HTML、CSS或JavaScript）时，这种缓存机制可能令人讨厌，因为它仅仅提供文件的第一个版本（因此，若您想重新加载页面，则不会看到您的编辑改变）。要避免此行为，请使用 -Dvertx.disableFileCaching=true 启动应用程序。使用此设置，Vert.x 仍然使用缓存，但始终使用原始源刷新存储在缓存中的版本。因此，如果您编辑从类路径提供的文件并刷新浏览器，Vert.x 会从类路径读取它，将其复制到缓存目录并从中提供。不要在生产环境使用这个设置，它很有可能影响性能。

最后，您可以使用-Dvertx.disableFileCPResolving=true完全禁用高速缓存。这个设置不是没有后果的。Vert.x将无法从类路径中读取任何文件（仅从文件系统）。使用此设置时要非常小心。


