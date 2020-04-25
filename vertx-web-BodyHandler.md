---
layout: post
comments: true
title: vert.x web开发之BodyHandler
date: 2017-12-04 16:36:34
tags:
- vert.x
categories:
---


### 前言

Web 开发肯定需要处理POST请求和文件上传请求。默认Vert.x并不会处理http请求体，除非你显示设置了`BodyHandler`。`BodyHandler`的作用是读取整个请求体，并将解析后请求体数据设置到`RoutingContext`。

`BodyHandler`的默认实现是`BodyHandlerImpl`。

### BodyHandler 核心

```java
@Override
public void handle(RoutingContext context) {
    HttpServerRequest request = context.request();
    // we need to keep state since we can be called again on reroute
    Boolean handled = context.get(BODY_HANDLED);
    if (handled == null || !handled) {
        BHandler handler = new BHandler(context);
        request.handler(handler);
        request.endHandler(v -> handler.end());
        context.put(BODY_HANDLED, true);
    } else {
        // on reroute we need to re-merge the form params if that was desired
        if (mergeFormAttributes && request.isExpectMultipart()) {
            request.params().addAll(request.formAttributes());
        }
        context.next();
    }
}
```

### BodyHandler 处理 POST请求。

<!-- more -->

`BodyHandler`默认会将解析后的POST请求参数，和URL中的请求参数进行merge。这样我们就可以以统一的API访问所有的请求参数:

```java  BodyHandler
/**
* Default value of whether form attributes should be merged into request params
*/
boolean DEFAULT_MERGE_FORM_ATTRIBUTES = true;
```

一个例子：

```java
final Router router = Router.router(vertx);
router.route().handler(BodyHandler.create());
....
context.request().getParam("id")
```

> 需要注意的是，应该将`BodyHandler`放在其它Handler的前面进行设置。

### 文件上传

```java
/**
* 请求体大小限制， 默认为 -1，就是不限制
*/
long DEFAULT_BODY_LIMIT = -1;

/**
* 默认的文件上传路径
*/
String DEFAULT_UPLOADS_DIRECTORY = "file-uploads";
```

文件上传实际上是由：`BHandler`来处理的，当文件上传成功后会调用该处理器来处理：

```java
public BHandler(RoutingContext context) {
   makeUploadDir(context.vertx().fileSystem());
   context.request().setExpectMultipart(true);
   //设置文件上传处理器
   context.request().uploadHandler(upload -> {
     // we actually upload to a file with a generated filename
     uploadCount.incrementAndGet();
     String uploadedFileName = new File(uploadsDir, UUID.randomUUID().toString()).getPath();
     upload.streamToFileSystem(uploadedFileName);
     FileUploadImpl fileUpload = new FileUploadImpl(uploadedFileName, upload);
     fileUploads.add(fileUpload);
     upload.exceptionHandler(context::fail);
     upload.endHandler(v -> uploadEnded());
   });
 }
 context.request().exceptionHandler(context::fail);
}
```







