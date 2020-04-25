---
layout: post
comments: true
title: vertx创建http服务器分析
date: 2017-11-22 21:41:28
tags:
- vert.x
categories:
---

### 创建HTTP服务器

从前面的文章我们知道Vert.x创建一个HttpServer的API是非常简单的，但需要注意的是里面的机制是非常复杂的，现在我们就来仔细分析一下Vert.x是如何一步步创建一个HttpServer的。

创建http服务器代码

```java
public class MyVerticle extends AbstractVerticle {
    @Override
    public void start() throws Exception {
        vertx.createHttpServer().requestHandler(req -> {
            res.response().end("hello Vert.x");
        }).listen(8888);
    }
}     
vertx.deployVerticle(MyVerticle.class.getName());
```

Vertx发布Verticle的分析参考:[vert.x之Vertx类解析](/2017/11/22/vertx-Vertx.html)

<!-- more -->

创建HttpServer

HttpServerOptions 默认的一些选项

```java HttpServerOptions
public static final int DEFAULT_PORT = 80; 
```

VertxImpl创建HttpServer

```java
public HttpServer createHttpServer(HttpServerOptions serverOptions) {
    return new HttpServerImpl(this, serverOptions);
}
```

HttpServerImpl 构造

```java HttpServerImpl
 public HttpServerImpl(VertxInternal vertx, HttpServerOptions options) {
    this.options = new HttpServerOptions(options);
    // 创建该HttpServer的Vertx实例
    this.vertx = vertx;
    // 创建该HttpServer的Verticle发布时的上下文
    this.creatingContext = vertx.getContext();
    if (creatingContext != null) {
      if (creatingContext.isMultiThreadedWorkerContext()) {
        throw new IllegalStateException("Cannot use HttpServer in a multi-threaded worker verticle");
      }
      creatingContext.addCloseHook(this);
    }
    this.sslHelper = new SSLHelper(options, options.getKeyCertOptions(), options.getTrustOptions());
    this.logEnabled = options.getLogActivity();
}
```

设置请求处理器，这个后面详细分析

```java
@Override
public synchronized HttpServer requestHandler(Handler<HttpServerRequest> handler) {
    requestStream.handler(handler);
    return this;
}
```

### 监听端口：

```java HttpServerImpl
private final VertxEventLoopGroup availableWorkers = new VertxEventLoopGroup();

public synchronized HttpServer listen(int port, String host, Handler<AsyncResult<HttpServer>> listenHandler) {
    // 获取上下文对象，
    listenContext = vertx.getOrCreateContext();
    listening = true;
    
    synchronized (vertx.sharedHttpServers()) {
      this.actualPort = port; // Will be updated on bind for a wildcard port
      id = new ServerID(port, host);
      // 如果发布多个会创建监听同一个端口号的HttpServer，我们发现并不会导致端口冲突，原因就在这里。
      // Vert.x 会做判断
      HttpServerImpl shared = vertx.sharedHttpServers().get(id);
      if (shared == null || port == 0) {
        serverChannelGroup = new DefaultChannelGroup("vertx-acceptor-channels", GlobalEventExecutor.INSTANCE);
        //这里是就是Netty的API了
        ServerBootstrap bootstrap = new ServerBootstrap();
        // boss EventLoopGroup 就是创建Vertx时指定的, 
        // WorkerEventLoopGroup 是Vert.x自己实现的（继承自Netty AbstractEventExecutorGroup）
        bootstrap.group(vertx.getAcceptorEventLoopGroup(), availableWorkers);
        // 设置参数（一些TCP参数，如发送和接收缓存区大小等）
        applyConnectionOptions(bootstrap);
        // 设置 WorkerEventLoop 事件处理器。
        bootstrap.childHandler(new ChannelInitializer<Channel>() {
            @Override
            protected void initChannel(Channel ch) throws Exception {
                if (requestStream.isPaused() || wsStream.isPaused()) {
                    ch.close();
                    return;
                }
                // 省略https和http2判断和初始化逻辑，我们只关心http1
                ChannelPipeline pipeline = ch.pipeline();
                configureHttp1(pipeline);
            }
        });
        添加处理器，这里会设置我们通过 HttpServer.requestHandler 设置的处理器
        addHandlers(this, listenContext);
        try {
          // 绑定端口号 
          bindFuture = AsyncResolveConnectHelper.doBind(vertx, SocketAddress.inetSocketAddress(port, host), bootstrap);
          // 添加绑定结果监听器
          bindFuture.addListener(res -> {
            if (res.failed()) {
              //失败
              vertx.sharedHttpServers().remove(id);
            } else {
              // 成功，  
              Channel serverChannel = res.result();
              HttpServerImpl.this.actualPort = ((InetSocketAddress)serverChannel.localAddress()).getPort();
              serverChannelGroup.add(serverChannel);
              VertxMetrics metrics = vertx.metricsSPI();
              this.metrics = metrics != null ? metrics.createMetrics(this, new SocketAddressImpl(port, host), options) : null;
            }
          });
        } catch (final Throwable t) {
          // Make sure we send the exception back through the handler (if any)
          if (listenHandler != null) {
            vertx.runOnContext(v -> listenHandler.handle(Future.failedFuture(t)));
          } else {
            // No handler - log so user can see failure
            log.error(t);
          }
          listening = false;
          return this;
        }
        vertx.sharedHttpServers().put(id, this);
        actualServer = this;
      } else {
        // Server already exists with that host/port - we will use that
        actualServer = shared;
        this.actualPort = shared.actualPort;
        addHandlers(actualServer, listenContext);
        VertxMetrics metrics = vertx.metricsSPI();
        this.metrics = metrics != null ? metrics.createMetrics(this, new SocketAddressImpl(port, host), options) : null;
      }
      actualServer.bindFuture.addListener(future -> {
        if (listenHandler != null) {
          final AsyncResult<HttpServer> res;
          if (future.succeeded()) {
            res = Future.succeededFuture(HttpServerImpl.this);
          } else {
            res = Future.failedFuture(future.cause());
            listening = false;
          }
          // 执行listen方法调用是传入的回调函数
          listenContext.runOnContext((v) -> listenHandler.handle(res));
        } else if (future.failed()) {
          listening  = false;
          // No handler - log so user can see failure
          log.error(future.cause());
        }
      });
    }
    return this;
}

// http1 协议请求处理配置
private void configureHttp1(ChannelPipeline pipeline) {
    //解码器和编码器设置
    pipeline.addLast("httpDecoder", new HttpRequestDecoder(options.getMaxInitialLineLength()
        , options.getMaxHeaderSize(), options.getMaxChunkSize(), false, options.getDecoderInitialBufferSize()));
    pipeline.addLast("httpEncoder", new VertxHttpResponseEncoder());
    //空闲连接检查
    if (options.getIdleTimeout() > 0) {
      pipeline.addLast("idle", new IdleStateHandler(0, 0, options.getIdleTimeout()));
    }
     HandlerHolder<HttpHandlers> holder = httpHandlerMgr.chooseHandler(pipeline.channel().eventLoop());
    ServerHandler handler;
    // 判断是否支持WebSocket
    if (DISABLE_WEBSOCKETS) {
      // As a performance optimisation you can set a system property to disable websockets altogether which avoids
      // some casting and a header check
      // 普通的Http请求
      handler = new ServerHandler(sslHelper, options, serverOrigin, holder, metrics);
    } else {
      handler = new ServerHandlerWithWebSockets(sslHelper, options, serverOrigin, holder, metrics);
    }
    handler.addHandler(conn -> {
      connectionMap.put(pipeline.channel(), conn);
    });
    handler.removeHandler(conn -> {
      connectionMap.remove(pipeline.channel());
    });
    // 这个就非常重要了，经过前面的解码处理，和其它 Handler 的处理，最终的请求会由该Handler处理
    pipeline.addLast("handler", handler);
}

private void addHandlers(HttpServerImpl server, ContextImpl context) {
    server.httpHandlerMgr.addHandler(
        new HttpHandlers(
            requestStream.handler(),
            wsStream.handler(),
            connectionHandler,
            exceptionHandler == null ? DEFAULT_EXCEPTION_HANDLER : exceptionHandler)
        , context);
}
```

HandlerManager 添加处理器

```java
public synchronized void addHandler(T handler, ContextImpl context) {
    // 这个地方很重要，这个Worker就是创建 Context 时获取到的Netty EventLoop
    EventLoop worker = context.nettyEventLoop();
    availableWorkers.addWorker(worker);
    Handlers<T> handlers = new Handlers<>();
    Handlers<T> prev = handlerMap.putIfAbsent(worker, handlers);
    if (prev != null) {
      handlers = prev;
    }
    handlers.addHandler(new HandlerHolder<>(context, handler));
    hasHandlers = true;
}
```

### 请求处理

从前面的分析过程我们知道http请求最终是由：ServerHandler进行处理

```java VertxHandler
public abstract class   VertxHandler<C extends ConnectionBase> extends ChannelDuplexHandler {
  @Override
  public void channelRead(ChannelHandlerContext chctx, Object msg) throws Exception {
    Object message = decode(msg, chctx.alloc());
    ContextImpl context;
    context = conn.getContext();
    context.executeFromIO(() -> {
      conn.startRead();
      // 抽象方法，由子类实现， 我们这里关心的是ServerHandler如何处理
      handleMessage(conn, context, chctx, message);
    });
  }
}
```

ServerHandler 处理消息

```java
public class ServerHandler extends VertxHttpHandler<ServerConnection> {
    @Override
    protected void handleMessage(ServerConnection conn, ContextImpl context, ChannelHandlerContext chctx, Object msg) throws Exception {
        conn.handleMessage(msg);
    }
}
```

ServerConnection

```java ServerConnection
public class ServerConnection extends Http1xConnectionBase implements HttpConnection {
    synchronized void handleMessage(Object msg) {
        if (queueing) {
          enqueue(msg);
        } else {
          processMessage(msg);
        }
    }
    // 真正处理请求消息的地方
    private void processMessage(Object msg) {
        if (msg instanceof HttpRequest) {
              if (pendingResponse != null) {
                enqueue(msg);
                return;
              }
              HttpRequest request = (HttpRequest) msg;
              if (request.decoderResult().isFailure()) {
                handleError(request);
                return;
              }
              if (options.isHandle100ContinueAutomatically() && HttpUtil.is100ContinueExpected(request)) {
                write100Continue();
              }
              HttpServerResponseImpl resp = new HttpServerResponseImpl(vertx, this, request);
              HttpServerRequestImpl req = new HttpServerRequestImpl(this, request, resp);
              currentRequest = req;
              pendingResponse = resp;
              if (METRICS_ENABLED && metrics != null) {
                requestMetric = metrics.requestBegin(metric(), req);
              }
              // 这个地方就会调用我们在创建HttpServer时设置的Handler，后续的业务逻辑处理就由我们自己处理
              requestHandler.handle(req);
        } else if (msg == LastHttpContent.EMPTY_LAST_CONTENT) {
              handleLastHttpContent();
        } else if (msg instanceof HttpContent) {
              HttpContent content = (HttpContent) msg;
              handleContent(content);
        } else {
              WebSocketFrameInternal frame = (WebSocketFrameInternal) msg;
              handleWsFrame(frame);
        }
        checkNextTick();
    }
}
```

### 总结

1. 创建一个HttpServer和使用Netty本身创建Http服务器核心逻辑都是相似的(其实创建TCP服务器也一样)，并且整个的创建过程都是在对应的Verticle的start方法中（当然了，init方法也行），本质上都是被对应的EventLoop线程所执行。
2. http新建连接是通过公用的Acceptor线程来处理的。每个连接的IO操作都是由EventLoop线程进行处理，解码后，最终会调用我们设置请求处理器Handler。
3. 因为请求最终由EventLoop线程进行处理，所有我们需要特别注意，在请求处理器中不能违反Vert.x的`金科玉律`：不要阻塞EventLoop线程。


  

