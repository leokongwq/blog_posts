---
layout: post
comments: true
title: vert.x之Vertx类解析
date: 2017-11-22 12:32:58
tags:
- vert.x
categories:
---

### Vertx

Vertx 是整个Vert.x核心API的入口，利用该对象可以创建，组合一个完整的Vert.x应用。

一个Vertx实例有如下的核心方法：

#### static Vertx vertx() 

该方法通过VertxFactory工厂类来创建一个Vertx实例。

#### static Vertx vertx(VertxOptions options)

用参数VertxOptions指定的配置项创建一个Vertx的实例。

####  static void clusteredVertx(VertxOptions options, Handler<AsyncResult<Vertx>> resultHandler)

用参数VertxOptions指定的配置项创建一个支持集群的Vertx的实例。

<!-- more -->

### vert.x 创建分析

Vertx 是通过 VertxFactory 创建的。

```java VertxFactoryImpl
public class VertxFactoryImpl implements VertxFactory {
  @Override
  public Vertx vertx() {
    return new VertxImpl();
  }

  @Override
  public Vertx vertx(VertxOptions options) {
    if (options.isClustered()) {
      throw new IllegalArgumentException("Please use Vertx.clusteredVertx() to create a clustered Vert.x instance");
    }
    return new VertxImpl(options);
  }
}
```

创建 VertxImpl 实例过程分析，目前省略了集群和其它相关的代码，只留下容易立即的代码和创建HttpServer,NetServer时会用到的代码。

```java VertxImpl
VertxImpl(VertxOptions options, Handler<AsyncResult<Vertx>> resultHandler) {
    //创建EventLoop线程阻塞或执行时间过长检测线程（实质上是一个Timer）
    checker = new BlockedThreadChecker(options.getBlockedThreadCheckInterval(), options.getWarningExceptionTime());
    //创建EventLoop线程池
    eventLoopThreadFactory = new VertxThreadFactory("vert.x-eventloop-thread-", checker, false, options.getMaxEventLoopExecuteTime());
    eventLoopGroup = transport.eventLoopGroup(options.getEventLoopPoolSize(), eventLoopThreadFactory, NETTY_IO_RATIO);
    
    //创建Acceptor线程池
    ThreadFactory acceptorEventLoopThreadFactory = new VertxThreadFactory("vert.x-acceptor-thread-", checker, false, options.getMaxEventLoopExecuteTime());
    // The acceptor event loop thread needs to be from a different pool otherwise can get lags in accepted connections
    // under a lot of load
    acceptorEventLoopGroup = transport.eventLoopGroup(1, acceptorEventLoopThreadFactory, 100);
   //创建worker线程池 
   ExecutorService workerExec = Executors.newFixedThreadPool(options.getWorkerPoolSize(),
   new VertxThreadFactory("vert.x-worker-thread-", checker, true, options.getMaxWorkerExecuteTime()));
   //创建内部阻塞操作线程池。
   ExecutorService internalBlockingExec = Executors.newFixedThreadPool(options.getInternalBlockingPoolSize(),
        new VertxThreadFactory("vert.x-internal-blocking-", checker, true, options.getMaxWorkerExecuteTime()));
    PoolMetrics internalBlockingPoolMetrics = metrics != null ? metrics.createMetrics(internalBlockingExec, "worker", "vert.x-internal-blocking", options.getInternalBlockingPoolSize()) : null;
    internalBlockingPool = new WorkerPool(internalBlockingExec, internalBlockingPoolMetrics);
}
```

VertxOptions

```java VertxOptions
/**
* 默认的EventLoop线程池大小为 CPU的核数 * 2
*/
public static final int DEFAULT_EVENT_LOOP_POOL_SIZE = 2 * CpuCoreSensor.availableProcessors();

/**
* 默认的WORKER线程池大小为 20 
*/
public static final int DEFAULT_WORKER_POOL_SIZE = 20;

/**
* 内部执行一些阻塞操作的线程池默认大小是20
*/
public static final int DEFAULT_INTERNAL_BLOCKING_POOL_SIZE = 20;
```

###  Vertx发布Verticle

在Vert.x应用中，我们的所有业务逻辑都是靠精心组织的Verticle来实现的。那么如何发布一个Verticle，发布的过程是如何进行的？这对我们来说是非常有必要了解的。

先看一个简单的代码：

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

上面的代码中我们发布了一个Verticle，该Verticle中创建了一个Http服务器。先来分析下Verticle的发布流程：

```java VertxImpl
@Override
public void deployVerticle(Supplier<Verticle> verticleSupplier, DeploymentOptions options, Handler<AsyncResult<String>> completionHandler) {
    boolean closed;
    synchronized (this) {
        closed = this.closed;
    }
    if (closed) {
        if (completionHandler != null) {
            completionHandler.handle(Future.failedFuture("Vert.x closed"));
        }
    } else {
        deploymentManager.deployVerticle(verticleSupplier, options, completionHandler);
    }
}
```

DeploymentManager 执行发布

```java DeploymentManager
public void deployVerticle(Supplier<Verticle> verticleSupplier, DeploymentOptions options, Handler<AsyncResult<String>> completionHandler) {
    //创建上下文
    ContextImpl currentContext = vertx.getOrCreateContext();
    ClassLoader cl = getClassLoader(options, currentContext);
    int nbInstances = options.getInstances();
    //省略掉创建多个Verticle实例的代码
    Verticle[] verticlesArray = verticles.toArray(new Verticle[verticles.size()]);
    String verticleClass = verticlesArray[0].getClass().getName();
    doDeploy("java:" + verticleClass, generateDeploymentID(), options, currentContext, currentContext, completionHandler, cl, verticlesArray);
}
//执行真正的发布逻辑
private void doDeploy(String identifier, String deploymentID, DeploymentOptions options,
                        ContextImpl parentContext,
                        ContextImpl callingContext,
                        Handler<AsyncResult<String>> completionHandler,
                        ClassLoader tccl, Verticle... verticles) {
    JsonObject conf = options.getConfig() == null ? new JsonObject() : options.getConfig().copy(); // Copy it
    String poolName = options.getWorkerPoolName();

    Deployment parent = parentContext.getDeployment();
    // deploymentID 就是 UUID.randomUUID().toString()
    // identifier 就是Verticle的类名
    DeploymentImpl deployment = new DeploymentImpl(parent, deploymentID, identifier, options);

    AtomicInteger deployCount = new AtomicInteger();
    AtomicBoolean failureReported = new AtomicBoolean();
    for (Verticle verticle: verticles) {
      // 创建执行阻塞，异步任务的线程池， 如果poolName 为空，则会使用创建Vertx实例时创建的Worker线程池
      WorkerExecutorImpl workerExec = poolName != null ? vertx.createSharedWorkerExecutor(poolName, options.getWorkerPoolSize()) : null;
      WorkerPool pool = workerExec != null ? workerExec.getPool() : null;
      //创建上下文对象。
      ContextImpl context = options.isWorker() ? vertx.createWorkerContext(options.isMultiThreaded(), deploymentID, pool, conf, tccl) :
        vertx.createEventLoopContext(deploymentID, pool, conf, tccl);
      if (workerExec != null) {
        context.addCloseHook(workerExec);
      }
      context.setDeployment(deployment);
      deployment.addVerticle(new VerticleHolder(verticle, context));
      //这个操作非常重要， verticle的初始化，启动都是在这里进行的。
      context.runOnContext(v -> {
        try {
          verticle.init(vertx, context);
          Future<Void> startFuture = Future.future();
          verticle.start(startFuture);
          startFuture.setHandler(ar -> {
            if (ar.succeeded()) {
              if (parent != null) {
                parent.addChild(deployment);
                deployment.child = true;
              }
              VertxMetrics metrics = vertx.metricsSPI();
              if (metrics != null) {
                metrics.verticleDeployed(verticle);
              }
              deployments.put(deploymentID, deployment);
              if (deployCount.incrementAndGet() == verticles.length) {
                reportSuccess(deploymentID, callingContext, completionHandler);
              }
            } else if (failureReported.compareAndSet(false, true)) {
              context.runCloseHooks(closeHookAsyncResult -> reportFailure(ar.cause(), callingContext, completionHandler));
            }
          });
        } catch (Throwable t) {
          if (failureReported.compareAndSet(false, true))
            context.runCloseHooks(closeHookAsyncResult -> reportFailure(t, callingContext, completionHandler));
        }
      });
    }
}  
```

VertxImpl 创建EventLoop上下文

```java VertxImpl
public EventLoopContext createEventLoopContext(String deploymentID, WorkerPool workerPool, JsonObject config, ClassLoader tccl) {
    return new EventLoopContext(this, internalBlockingPool, workerPool != null ? workerPool : this.workerPool, deploymentID, config, tccl);
}
```

ContextImpl.runOnContext

```java ContextImpl
private final EventLoop eventLoop;

// Run the task asynchronously on this same context
@Override
public void runOnContext(Handler<Void> task) {
    try {
        // 子类实现具体的内容
        executeAsync(task);
    } catch (RejectedExecutionException ignore) {
      // Pool is already shut down
    }
}

//从EventLoop线程池中获取一个EventLoop线程
private static EventLoop getEventLoop(VertxInternal vertx) {
    EventLoopGroup group = vertx.getEventLoopGroup();
    if (group != null) {
      return group.next();
    } else {
      return null;
    }
}
  
protected ContextImpl(VertxInternal vertx, WorkerPool internalBlockingPool, WorkerPool workerPool, String deploymentID, JsonObject config,
                        ClassLoader tccl) {
    // 这里很重要， getEventLoop方法会获取一个空闲Netty EventLoop线程
    this(vertx, getEventLoop(vertx), internalBlockingPool, workerPool, deploymentID, config, tccl);
}

//获取创建上下文时获取的EventLoop线程
public EventLoop nettyEventLoop() {
    return eventLoop;
}

//创建一个异步任务
protected Runnable wrapTask(ContextTask cTask, Handler<Void> hTask, boolean checkThread, PoolMetrics metrics) {
    Object metric = metrics != null ? metrics.submitted() : null;
    return () -> {
      Thread th = Thread.currentThread();
      if (!(th instanceof VertxThread)) {
        throw new IllegalStateException("Uh oh! Event loop context executing with wrong thread! Expected " + contextThread + " got " + th);
      }
      VertxThread current = (VertxThread) th;
      if (THREAD_CHECKS && checkThread) {
        if (contextThread == null) {
          contextThread = current;
        } else if (contextThread != current && !contextThread.isWorker()) {
          throw new IllegalStateException("Uh oh! Event loop context executing with wrong thread! Expected " + contextThread + " got " + current);
        }
      }
      if (metrics != null) {
        metrics.begin(metric);
      }
      if (!DISABLE_TIMINGS) {
        current.executeStart();
      }
      try {
        setContext(current, ContextImpl.this);
        if (cTask != null) {
          cTask.run();
        } else {
          // 回调 DeployManager的doDeploy()方法中 context.runOnContext(Handler)指定的回调Handler, 也就是说 Verticle的初始化和启动都是异步进行的。   
          hTask.handle(null);
        }
        if (metrics != null) {
          metrics.end(metric, true);
        }
      } catch (Throwable t) {
        log.error("Unhandled exception", t);
        Handler<Throwable> handler = this.exceptionHandler;
        if (handler == null) {
          handler = owner.exceptionHandler();
        }
        if (handler != null) {
          handler.handle(t);
        }
        if (metrics != null) {
          metrics.end(metric, false);
        }
      } finally {
        // We don't unset the context after execution - this is done later when the context is closed via
        // VertxThreadFactory
        if (!DISABLE_TIMINGS) {
          current.executeEnd();
        }
      }
    };
}
```

EventLoopContext(父类是ContextImpl)

```java EventLoopContext
public EventLoopContext(VertxInternal vertx, WorkerPool internalBlockingPool, WorkerPool workerPool, String deploymentID, JsonObject config,
                          ClassLoader tccl) {
    //调用 ContextImpl 的方法                     
    super(vertx, internalBlockingPool, workerPool, deploymentID, config, tccl);
}

// DeploymentManager中调用Context的runOnContext方法时，具体的实现是由Context的子类来实现的。 
// 在此处就是 EventLoopContext  
public void executeAsync(Handler<Void> task) {
    // No metrics, we are on the event loop.
    // nettyEventLoop()方法获取创建该EventLoopContext上下文时获取到的EventLoop线程
    nettyEventLoop().execute(wrapTask(null, task, true, null));
}
```

### Vertx执行异步/阻塞代码

在Vert.x执行异步/阻塞代码注意是通过Vertx类中的三个方法来实现：

```java
<T> void executeBlocking(Handler<Future<T>> blockingCodeHandler, boolean ordered, Handler<AsyncResult<T>> resultHandler);

<T> void executeBlocking(Handler<Future<T>> blockingCodeHandler, Handler<AsyncResult<T>> resultHandler);
```

阻塞代码的执行都是在Worker线程池中进行的。

### 总结

1. Vert.x的Acceptor线程只有一个，并且是和EventLoop线程池分开的。（为的是在高负载新的连接不能被accept）
2. 默认的EventLoop线程数=CPU的核心数 * 2
3. 默认的Worker线程数为20
4. 默认的内部线程数是20
5. Verticle的初始化和启动是异步的，并且是通过EventLoop进行的。
6. 一个Verticle只能和一个EventLoop线程进行绑定，但是一个EventLoop线程可以被多个Verticle绑定。
7. 每一个Verticle都拥有一个独立的Context上下文。并且该上下文是核EventLoop线程绑定的。


