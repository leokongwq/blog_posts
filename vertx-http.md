---
layout: post
comments: true
title: vertx编程http服务端和客户端
date: 2017-11-17 23:16:37
tags:
- vert.x
categories:
---

### 编写 HTTP 服务端和客户端

Vert.x 允许您轻松编写非阻塞的 HTTP 客户端和服务端。

Vert.x 支持 HTTP/1.0、HTTP/1.1 和 HTTP/2 协议。

用于 HTTP 的基本 API 对 HTTP/1.x 和 HTTP/2 是相同的，特定的API功能也可用于处理 HTTP/2 协议。

<!-- more -->

### 创建 HTTP 服务端

使用所有默认选项创建 HTTP 服务端的最简单方法如下：

```java
HttpServer server = vertx.createHttpServer();
```

### 配置 HTTP 服务端

若您不想用默认值，可以在创建服务器时传递一个 `HttpServerOptions` 实例给它：

```java
HttpServerOptions options = new HttpServerOptions().setMaxWebsocketFrameSize(1000000);
HttpServer server = vertx.createHttpServer(options);
```

### 配置 HTTP/2 服务端

Vert.x支持 TLS h2和TCP h2c之上的 HTTP/2 协议。

- h2 表示使用了TLS的应用层协议协商(ALPN)协议来协商的 HTTP/2 协议
- h2c 表示在TCP层上使用明文形式的 HTTP/2 协议，这样的连接是使用 HTTP/1.1升级 请求或者直接建立要处理 h2 请求，你必须调用 setUseAlpn 方法来启用TLS：

```java
HttpServerOptions options = new HttpServerOptions()
    .setUseAlpn(true)
    .setSsl(true)
    .setKeyStoreOptions(new JksOptions().setPath("/path/to/my/keystore"));

HttpServer server = vertx.createHttpServer(options);
```

ALPN是一个TLS的扩展，它在客户端和服务器开始交换数据之前协商协议。

不支持ALPN的客户端仍然可以执行经典的SSL握手。

通常情况，ALPN会对 `h2` 协议达成一致，尽管服务器或客户端决定了仍然使用 HTTP/1.1 协议。

要处理 `h2c` 请求，TLS必须被禁用，服务器将升级到 HTTP/2 以满足任何希望升级到 HTTP/2 的 HTTP/1.1 请求。它还将接受以 `PRI*HTTP/2.0\r\nSM\r\n` 开始的`h2c`直接连接。

> 警告：大多数浏览器不支持 h2c，所以在建站时，您应该使用 h2 而不是 h2c。

当服务器接受 HTTP/2 连接时，它会向客户端发送其初始设置。定义客户端如何使用连接，服务器的默认初始设置为：

- getMaxConcurrentStreams：按照 HTTP/2 RFC建议推荐值为100
- 其他默认的 HTTP/2 的设置

> 请注意：`Worker Verticle` 和 HTTP/2 不兼容。

### 记录服务端网络活动

为了进行调试，可记录网络活动。

```java
HttpServerOptions options = new HttpServerOptions().setLogActivity(true);
HttpServer server = vertx.createHttpServer(options);
```

详细说明请参阅 [记录网络活动](https://vertxchina.github.io/vertx-translation-chinese/core/Core.html#记录网络活动) 章节。

### 开启服务端监听

要告诉服务器监听传入的请求，您可以使用其中一个 `listen` 方法。

在配置项中告诉服务器监听指定的主机和端口：

```java
HttpServer server = vertx.createHttpServer();
server.listen();
```

或在调用 listen 方法时指定主机和端口号，这样就忽略了配置项（中的主机和端口）：

```java
HttpServer server = vertx.createHttpServer();
server.listen(8080, "myhost.com");
```

默认主机名是 `0.0.0.0`，它表示：监听所有可用地址；默认端口号是`80`。

实际的绑定也是异步的，因此服务器也许并没有在调用 `listen` 方法返回时监听，而是在一段时间过后才监听。

若您希望在服务器实际监听时收到通知，您可以向 `listen` 提供一个处理器。例如：

```java
HttpServer server = vertx.createHttpServer();
server.listen(8080, "myhost.com", res -> {
  if (res.succeeded()) {
    System.out.println("Server is now listening!");
  } else {
    System.out.println("Failed to bind!");
  }
});
```

### 收到传入请求的通知

若您需要在收到请求时收到通知，则需要设置一个 requestHandler：

```java
HttpServer server = vertx.createHttpServer();
server.requestHandler(request -> {
  // 在这里处理请求
});
```

### 处理请求

当请求到达时，Vert.x 会像对应的处理函数传入一个 HttpServerRequest 实例并调用请求处理函数，此对象表示服务端 HTTP 请求。当请求的头信息被完全读取时会调用该请求处理器。

如果请求包含请求体，那么该请求体将在请求处理器被调用后的某个时间到达服务器。

服务请求对象允许您检索 uri，path，params 和 headers 等其他信息。

每一个服务请求对象和一个服务响应对象绑定，您可以用 response 方法获取一个 HttpServerResponse 对象的引用。

这是服务器处理请求并回复 “hello world” 的简单示例。

```java
vertx.createHttpServer().requestHandler(request -> {
  request.response().end("Hello world");
}).listen(8080);
```

### 请求版本

在请求中指定的 HTTP 版本可通过 version 方法获取。

### 请求方法

使用 method 方法读取请求中的 HTTP Method（即GET、POST、PUT、DELETE、HEAD、OPTIONS等）。

### 请求URI

使用 uri 方法读取请求中的URI路径。

请注意，这是在HTTP 请求中传递的实际URI，它总是一个相对的URI。

这个URI是在 Section 5.1.2 of the HTTP specification - Request-URI 中定义的。

### 请求路径

使用 path 方法读取URI中的路径部分。

例如，请求的URI为：

```
/a/b/c/page.html?param1=abc&param2=xyz
```

路径部分应该是：

```
/a/b/c/page.html
```

### 请求查询

使用query读取URI中的查询部分。

例如，请求的URI为：

```
a/b/c/page.html?param1=abc&param2=xyz
```

查询部分应该是：

```
param1=abc&param2=xyz
```

### 请求头部

使用 `headers` 方法获取HTTP 请求中的请求头部信息。

这个方法返回一个 `MultiMap` 实例。它像一个普通的Map或Hash，并且它还允许同一个键支持多个值 —— 因为HTTP允许同一个键支持多个请求头的值。

它的键值不区分大小写，这意味着您可以执行以下操作：

```java
MultiMap headers = request.headers();

// Get the User-Agent:
// 读取User-Agent
System.out.println("User agent is " + headers.get("user-agent"));

// You can also do this and get the same result:
// 这样做可以得到和上边相同的结果
System.out.println("User agent is " + headers.get("User-Agent"));
```

### 请求主机

使用 `host` 方法返回 HTTP 请求中的主机名。

对于 HTTP/1.x 请求返回请求头中的 `host` 值，对于 HTTP/1 请求则返回伪头中的:authority的值。

### 请求参数

您可以使用 `params` 方法返回HTTP请求中的参数信息。像 headers 方法一样它也会返回一个 MultiMap 实例，因为可以有多个具有相同名称的参数。

请求参数在请求URI的 path 部分之后，例如URI是：

```
/page.html?param1=abc&param2=xyz
```

那么参数将包含以下内容：

```
param1: 'abc'
param2: 'xyz'
```

请注意，这些请求参数是从请求的 `URI` 中解析读取的，若您已经将表单属性存放在请求体中发送出去，并且该请求为 multi-part/form-data 类型请求，那么它们将不会显示在此处的参数中。

### 远程地址

可以使用 `remoteAddress` 方法读取请求发送者的地址。

### 绝对URI

HTTP 请求中传递的URI通常是相对的，若您想要读取请求中和相对URI对应的绝对URI，可调用 `absoluteURI` 方法。

### 结束处理器

当整个请求（包括任何正文）已经被完全读取时，请求中的 `endHandler` 方法会被调用。

### 请求体中读取数据

HTTP请求通常包含我们需要读取的主体。如前所述，当请求头部达到时，请求处理器会被调用，因此请求对象在此时没有请求体。

这是因为请求体可能非常大（如文件上传），并且我们不会在内容发送给您之前将其全部缓冲存储在内存中，这可能会导致服务器耗尽可用内存。

要接收请求体，您可在请求中调用 handler 方法设置一个处理器，每次请求体的一小块数据收到时，该处理器都会被调用。以下是一个例子：

```java
request.handler(buffer -> {
  System.out.println("I have received a chunk of the body of length " + buffer.length());
});
```

传递给处理器的对象是一个 `Buffer`，当数据从网络到达时，处理器可以多次被调用，这取决于请求体的大小。

在某些情况下（例：若请求体很小），您将需要将这个请求体聚合到内存中，以便您可以按照下边的方式进行聚合：

```java
Buffer totalBuffer = Buffer.buffer();

request.handler(buffer -> {
    System.out.println("I have received a chunk of the body of length " + buffer.length());
    totalBuffer.appendBuffer(buffer);
});

request.endHandler(v -> {
    System.out.println("Full body received, length = " + totalBuffer.length());
});
```

这是一个常见的情况，Vert.x为您提供了一个 `bodyHandler` 方法来执行此操作。当所有请求体被收到时，bodyHandler 绑定的处理器会被调用一次：

```java
request.bodyHandler(totalBuffer -> {
    System.out.println("Full body received, length = " + totalBuffer.length());
});
```

### Pumping 请求

请求对象实现了 `ReadStream` 接口，因此您可以将请求体读取到任何 `WriteStream` 实例中。

详细请参阅 [流和管道](https://vertxchina.github.io/vertx-translation-chinese/core/Core.html#流) 章节。

### 处理 HTML 表单

您可使用 `application/x-www-form-urlencoded` 或 `multipart/form-data` 这两种 `content-type` 来提交 HTML 表单。

对于使用 URL 编码过的表单，表单属性会被编码在URL中，如同普通查询参数一样。

对于 `multipart` 类型的表单，它会被编码在请求体中，而且在整个请求体被完全读取之前它是不可用的。 `Multipart`表单还可以包含文件上传。

若您想要读取 `multipart` 表单的属性，您应该告诉 Vert.x 您会在读取任何正文 之前 调用 `setExpectMultipart(true)` 方法，然后在整个请求体都被读取后，您可以使用 `formAttributes` 方法来读取实际的表单属性。

```java
server.requestHandler(request -> {
  request.setExpectMultipart(true);
  request.endHandler(v -> {
    // 请求体被完全读取，所以直接读取表单属性
    MultiMap formAttributes = request.formAttributes();
  });
});
```

### 处理文件上传

Vert.x 可以处理以 `multipart` 编码形式上传的的文件。

要接收文件，您可以告诉 Vert.x 使用 `multipart` 表单，并对请求设置 `uploadHandler`。

当服务器每次接收到上传请求时，该处理器将被调用一次。

传递给处理器的对象是一个 `HttpServerFileUpload` 实例。

```java
server.requestHandler(request -> {
  request.setExpectMultipart(true);
  request.uploadHandler(upload -> {
    System.out.println("Got a file upload " + upload.name());
  });
});
```

上传的文件可能很大，我们不会在单个缓冲区中包含整个上传的数据，因为这样会导致内存耗尽。相反，上传数据是以块的形式被接收的：

```java
request.uploadHandler(upload -> {
  upload.handler(chunk -> {
    System.out.println("Received a chunk of the upload of length " + chunk.length());
  });
});
```

上传对象实现了 `ReadStream` 接口，因此您可以将请求体读取到任何 `WriteStream` 实例中。详细说明请参阅[流和管道（泵）] (https://vertxchina.github.io/vertx-translation-chinese/core/Core.html#流)章节。

若您只是想将文件上传到服务器的某个磁盘，可以使用 `streamToFileSystem` 方法：

```java
request.uploadHandler(upload -> {
    upload.streamToFileSystem("myuploads_directory/" + upload.filename());
});
```

> 警告：确保您检查了生产系统的文件名，以避免恶意客户将文件上传到文件系统中的任意位置。有关详细信息，参阅[安全说明](http://vertx.io/docs/vertx-core/java/#_security_notes)。

### 处理压缩体

Vert.x 可以处理在客户端通过 deflate 或 gzip 算法压缩过的请求体信息。

若要启用解压缩功能则您要在创建服务器时调用[`setDecompressionSupported`](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerOptions.html#setDecompressionSupported-boolean-)方法设置配置项。默认情况下解压缩是被禁用的。

### 接收自定义 HTTP/2 帧

HTTP/2 是用于 HTTP 请求/响应模型的包含各种帧的一种帧协议，该协议允许发送和接收其他类型的帧。

若要接收自定义帧(frame)，您可以在请求中使用`customFrameHandler`，每次当自定义的帧数据到达时，这个处理器会被调用。这而是一个例子：

```java
request.customFrameHandler(frame -> {
  System.out.println("Received a frame type=" + frame.type() +
      " payload" + frame.payload().toString());
});
```

HTTP/2 帧不受流量控制限制 —— 当接收到自定义帧时，不论请求是否暂停，自定义帧处理器都将立即被调用。

### 非标准的 HTTP 方法

[OTHER](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpMethod.html#OTHER) HTTP 方法可用于非标准方法，在这种情况下，`rawMethod` 方法返回客户端发送的实际 HTTP 方法。

### 发回响应

服务器响应对象是一个`HttpServerResponse`实例，它可以从`request` 对应的 `response` 方法中读取。

您可以使用响应对象回写一个响应到 HTTP客户端。

### 设置状态码和消息

默认的 HTTP 状态响应码为 200，表示 OK。

可使用 `setStatusCode` 方法设置不同状态代码。

您还可用 `setStatusMessage` 方法指定自定义状态消息。

若您不指定状态信息，将会使用默认的状态码响应。

> 注意：对于 HTTP/2 中的状态不会在响应中描述 —— 因为协议不会将消息发送回客户端。

### 向 HTTP 响应写入数据

想要将数据写入 HTTP Response，您可使用任意一个 `write` 方法。

它们可以在响应结束之前被多次调用，它们可以通过以下几种方式调用：

对用单个缓冲区：

```java
HttpServerResponse response = request.response();
response.write(buffer);
```

写入字符串，这种请求字符串将使用 `UTF-8` 进行编码，并将结果写入到报文中。

```java
HttpServerResponse response = request.response();
response.write("hello world!");
```

写入带编码方式的字符串，这种情况字符串将使用指定的编码方式编码，并将结果写入到报文中。

```java
HttpServerResponse response = request.response();
response.write("hello world!", "UTF-16");
```

响应写入是异步的，并且在写操作进入队列之后会立即返回。

若您只需要将单个字符串或 Buffer 写入到HTTP 响应，则可使用 `end` 方法将其直接写入响应中并发回到客户端。

第一次写入操作会触发响应头的写入，因此，若您不使用HTTP 分块，那么必须在写入响应之前设置 Content-Length 头，否则会太迟。若您使用 HTTP 分块则不需要担心这点。

### 完成 HTTP 响应

一旦您完成了 HTTP 响应，可调用 `end` 将其发回客户端。

这可以通过几种方式完成：

没有参数，直接结束响应，发回客户端：

```java
HttpServerResponse response = request.response();
response.write("hello world!");
response.end();
```

您也可以和调用write方法一样传 String 或 Buffer 给 end 方法。这种情况，它和先调用带 String 或 Buffer 参数的 write 方法，之后调用无参 end 方法一样。例如：

```java
HttpServerResponse response = request.response();
response.end("hello world!");
```

### 关闭底层连接

您可以调用 [`close`](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerResponse.html#close--) 方法关闭底层的TCP 连接。

当响应结束时，Vert.x 将自动关闭非 keep-alive 的连接。

默认情况下，Vert.x 不会自动关闭 keep-alive 的连接，若您想要在一段空闲时间之后让 Vert.x 自动关闭 keep-alive 的连接，则使用 [`setIdleTimeout`](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerOptions.html#setIdleTimeout-int-) 方法进行配置。

HTTP/2 连接在关闭响应之前会发送 `GOAWAY` 帧。

### 设置响应头

HTTP 响应头可直接添加到 HTTP 响应中，通常直接操作 headers：

```java
HttpServerResponse response = request.response();
MultiMap headers = response.headers();
headers.set("content-type", "text/html");
headers.set("other-header", "wibble");
```

或您可使用 putHeader 方法：

```java
HttpServerResponse response = request.response();
response.putHeader("content-type", "text/html").putHeader("other-header", "wibble");
```

**响应头必须在写入响应正文消息之前进行设置。**

### 分块 HTTP 响应和附加尾部

Vert.x 支持 分块传输编码([HTTP Chunked Transfer Encoding](http://en.wikipedia.org/wiki/Chunked_transfer_encoding))。

这允许HTTP 响应体以块的形式写入，通常在响应体预先不知道尺寸、需要将很大响应正文以流式传输到客户端时使用。

您可以通过如下方式开启分块模式：

```java
HttpServerResponse response = request.response();
response.setChunked(true);
```

默认是不分块的，当处于分块模式，每次调用任意一个 write 方法将导致新的 HTTP 块被写出。

在分块模式下，您还可以将响应的HTTP 响应附加尾部(trailers)写入响应，这种方式实际上是在写入响应的最后一块。

> 注意：分块响应在 HTTP/2 流中无效

若要向响应添加尾部，则直接添加到 [trailers](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerResponse.html#trailers--) 里。

```java
HttpServerResponse response = request.response();
response.setChunked(true);
MultiMap trailers = response.trailers();
trailers.set("X-wibble", "woobble").set("X-quux", "flooble");
```

或者调用 putTrailer 方法：

```java
HttpServerResponse response = request.response();
response.setChunked(true);
response.putTrailer("X-wibble", "woobble").putTrailer("X-quux", "flooble");
```

### 直接从磁盘或 Classpath 读文件

若您正在编写一个 Web 服务端，一种从磁盘中读取并提供文件的方法是将文件作为 `AsyncFile` 对象打开并其传送到HTTP 响应中。

或您可以使用 `readFile` 方法一次性加载它，并直接将其写入响应。

或者，Vert.x 提供了一种方法，允许您在一个操作中将文件从磁盘或文件系统中读取并提供给HTTP 响应。若底层操作系统支持，这会导致操作系统不通过用户空间复制而直接将文件内容中字节数据从文件传输到Socket。这是使用`sendFile` 方法完成的，对于大文件处理通常更有效，而这个方法对于小文件可能很慢。

这儿是一个非常简单的 Web 服务器，它使用 sendFile 方法从文件系统中读取并提供文件：

```java
vertx.createHttpServer().requestHandler(request -> {
  String file = "";
  if (request.path().equals("/")) {
    file = "index.html";
  } else if (!request.path().contains("..")) {
    file = request.path();
  }
  request.response().sendFile("web/" + file);
}).listen(8080);
```

发送文件是异步的，可能在调用返回一段时间后才能完成。如果要在文件写入时收到通知，可以在`sendFile` 方法中设置一个处理器。

请阅读 [从 Classpath 访问文件](https://vertxchina.github.io/vertx-translation-chinese/core/Core.html#从classpath访问文件) 章节了解类路径的限制或禁用它。

> 注意：若在 HTTPS 协议中使用 sendFile 方法，它将会通过用户空间进行复制，因为若内核将数据直接从磁盘复制到 Socket，则不会给我们任何加密的机会。
警告：若您要直接使用 Vert.x 编写 Web 服务器，请注意，您想提供文件和类路径之外访问的位置 —— 用户是无法直接利用路径访问的。更安全的做法是使用Vert.x Web替代。

当需要提供文件的一部分，从给定的字节开始，您可以像下边这样做：

```java
vertx.createHttpServer().requestHandler(request -> {
  long offset = 0;
  try {
    offset = Long.parseLong(request.getParam("start"));
  } catch (NumberFormatException e) {
    // 异常处理
  }

  long end = Long.MAX_VALUE;
  try {
    end = Long.parseLong(request.getParam("end"));
  } catch (NumberFormatException e) {
    // 异常处理
  }

  request.response().sendFile("web/mybigfile.txt", offset, end);
}).listen(8080);
```

若您想要从偏移量开始发送文件直到尾部，则不需要提供长度信息，这种情况下，您可以执行以下操作：

```java
vertx.createHttpServer().requestHandler(request -> {
  long offset = 0;
  try {
    offset = Long.parseLong(request.getParam("start"));
  } catch (NumberFormatException e) {
    //异常处理
  }

  request.response().sendFile("web/mybigfile.txt", offset);
}).listen(8080);
```

### Pumping 响应

服务端响应`HttpServerResponse` 也是一个`WriteStream`实例，因此您可以从任何 `ReadStream` 向其泵送数据，如`AsyncFile`、`NetSocket`、`WebSocket`或`HttpServerRequest`。

这儿有一个例子，它回应了任何 PUT 方法的响应中的请求体，它为请求体使用了 Pump，所以即使 HTTP 请求体很大并填满了内存，任何时候它依旧会工作：

```java
vertx.createHttpServer().requestHandler(request -> {
  HttpServerResponse response = request.response();
  if (request.method() == HttpMethod.PUT) {
    response.setChunked(true);
    Pump.pump(request, response).start();
    request.endHandler(v -> response.end());
  } else {
    response.setStatusCode(400).end();
  }
}).listen(8080);
```

### 写入 HTTP/2 帧

HTTP/2 是用于 HTTP 请求/响应模型的包含各种帧的一种帧协议，该协议允许发送和接收其他类型的帧。

要发送这样的帧，您可以在响应中使用 writeCustomFrame 方法，以下是一个例子：

```java
int frameType = 40;
int frameStatus = 10;
Buffer payload = Buffer.buffer("some data");
// 向客户端发送一帧
response.writeCustomFrame(frameType, frameStatus, payload);
```

这些帧被立即发送，并且不受流程控制的影响——当这样的帧被发送到那里时，可以在其他的 DATA 帧之前完成。

### 流重置

HTTP/1.x 不允许请求或响应流执行清除重置，如当客户端上传的资源已经存在于服务器上，服务器就需要接受整个响应。

HTTP/2 在请求/响应期间随时支持流重置：

```java
request.response().reset();
```

默认的 `NO_ERROR(0)` 错误代码会发送，您也可以发送另外一个错误代码：

```java
request.response().reset(8);
```

HTTP/2 规范中定义了可用的 [错误码](http://httpwg.org/specs/rfc7540.html#ErrorCodes) 列表：

若使用了 [request handler](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerRequest.html#exceptionHandler-io.vertx.core.Handler-) 和 [response handler](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerResponse.html#exceptionHandler-io.vertx.core.Handler-) 两个处理器过后，在流重置完成时您将会收到通知：

```java
request.response().exceptionHandler(err -> {
  if (err instanceof StreamResetException) {
    StreamResetException reset = (StreamResetException) err;
    System.out.println("Stream reset " + reset.getCode());
  }
});
```

### 服务器推送

服务器推送(Server Push)是 HTTP/2 支持的一个新功能，可以为单个客户端请求并行发送多个响应。

当服务器处理请求时，它可以向客户端推送请求/响应：

```java
HttpServerResponse response = request.response();

// 推送main.js到客户端
response.push(HttpMethod.GET, "/main.js", ar -> {

  if (ar.succeeded()) {

    // 服务器准备推送响应
    HttpServerResponse pushedResponse = ar.result();

    // 发送main.js响应
    pushedResponse.
        putHeader("content-type", "application/json").
        end("alert(\"Push response hello\")");
  } else {
    System.out.println("Could not push client resource " + ar.cause());
  }
});

// 发送请求的资源内容
response.sendFile("<html><head><script src=\"/main.js\"></script></head><body></body></html>");
```

当服务器准备推送响应时，推送响应处理器会被调用，并会发送响应。

推送响应处理器客户能会接收到失败，如：客户端可能取消推送，因为它已经在缓存中包含了 `main.js`，并不在需要它。

您必须在响应结束之前调用 [push](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerResponse.html#push-io.vertx.core.http.HttpMethod-java.lang.String-java.lang.String-io.vertx.core.Handler-) 方法，但是在推送响应过后依然可以写响应。

### HTTP 压缩

Vert.x 支持 HTTP 压缩。

这意味着在响应发送回客户端之前，您可以将响应体自动压缩。

若客户端不支持HTTP 压缩，则它可以发回没有压缩过的请求。

这允许它同时处理支持HTTP 压缩的客户端和不支持的客户端。

要启用压缩，可以使用 [setCompressionSupported](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerOptions.html#setCompressionSupported-boolean-) 方法进行配置。默认情况下，未启用压缩。

当启用HTTP 压缩时，服务器将检查客户端请求头中是否包含了 `Accept-Encoding` 并支持常用的 `deflate` 和 `gzip` 压缩算法。Vert.x 两者都支持。若找到这样的请求头，服务器将使用所支持的压缩算法之一自动压缩响应正文并发送回客户端。

> 注意：压缩可以减少网络流量，但是CPU密集度会更高。

为了解决后边一个问题，Vert.x也允许您调整原始的 `gzip/deflate` 压缩算法的 “压缩级别” 参数

压缩级别允许根据所得数据的压缩比和压缩/解压的计算成本来配置 `gzip/deflate` 算法。

压缩级别是从 1 到 9 的整数值，其中 1 表示更低的压缩比但是最快的算法，9 表示可用的最大压缩比但比较慢的算法。

使用高于 1-2 的压缩级别通常允许仅仅保存一些字节大小 —— 它的增益不是线性的，并取决于要压缩的特定数据 —— 但它可以满足服务器所要求的CPU周期的不可控的成本（注意现在Vert.x不支持任何缓存形式的响应数据，如静态文件，因此压缩是在每个请求体生成时进行的）,它可生成压缩过的响应数据、并对接收的响应解码（膨胀）—— 和客户端使用的方式一致，这种操作随着压缩级别的增长会变得更加倾向于CPU密集型。

默认情况下 —— 如果通过 [setCompressionSupported](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerOptions.html#setCompressionSupported-boolean-) 方法启用压缩，Vert.x 将使用 6 作为压缩级别，但是该参数可通过 [setCompressionLevel](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpServerOptions.html#setCompressionLevel-int-) 方法来更改。

### 创建 HTTP 客户端

您可通过以下方式创建一个具有默认配置的 HttpClient 实例：

```java
HttpClient client = vertx.createHttpClient();
```

若您想要配置客户端选项，可按以下方式创建：

```java
HttpClientOptions options = new HttpClientOptions().setKeepAlive(false);
HttpClient client = vertx.createHttpClient(options);
```

Vert.x 支持基于 TLS h2 和 TCP h2c 的 HTTP/2 协议。

默认情况下，HTTP 客户端会发送 HTTP/1.1 请求。若要执行 HTTP/2 请求，则必须调用 [setProtocolVersion](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpClientOptions.html#setProtocolVersion-io.vertx.core.http.HttpVersion-) 方法将版本设置成 [HTTP_2](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpVersion.html#HTTP_2)。

对于 h2 请求，必须使用应用层协议协商(ALPN)启用TLS：

```java
HttpClientOptions options = new HttpClientOptions().
    setProtocolVersion(HttpVersion.HTTP_2).
    setSsl(true).
    setUseAlpn(true).
    setTrustAll(true);

HttpClient client = vertx.createHttpClient(options);
```

对于 h2c 请求，TLS必须禁用，客户端将执行 HTTP/1.1 请求并尝试升级到 HTTP/2：

```java
HttpClientOptions options = new HttpClientOptions().setProtocolVersion(HttpVersion.HTTP_2);

HttpClient client = vertx.createHttpClient(options);
```

`h2c`连接也可以直接建立，如连接可以使用前文提到的方式创建，当[`setHttp2ClearTextUpgrade`](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpClientOptions.html#setHttp2ClearTextUpgrade-boolean-)选项设置为 false 时：建立连接后，客户端将发送 HTTP/2 连接前缀，并期望从服务端接收相同的连接偏好。

HTTP 服务端可能不支持 HTTP/2，当响应到达时，可以使用 [version](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpClientResponse.html#version--) 方法检查响应实际HTTP版本。

当客户端连接到 HTTP/2 服务端时，它将向服务端发送其[初始设置](http://vertx.io/docs/apidocs/io/vertx/core/http/HttpClientOptions.html#getInitialSettings--)。设置定义服务器如何使用连接、客户端的默认初始设置是由 HTTP/2 RFC定义的。

### 记录客户端网络活动

为了进行调试，可以记录网络活动：

```java
HttpClientOptions options = new HttpClientOptions().setLogActivity(true);
HttpClient client = vertx.createHttpClient(options);
```

详情请参阅 [记录网络活动](https://vertxchina.github.io/vertx-translation-chinese/core/Core.html#记录网络活动) 章节。









