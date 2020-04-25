---
layout: post
comments: true
title: vertx命令行选项解析
date: 2017-12-05 17:25:59
tags:
- vert.x
categories:
---

### 背景

我们在开发，测试，生产环境需要设置Vert.x不同的`VertxOptions`，通常可以通过自定义`Launcher`，复写`beforeStartingVertx`方法来设置`VertxOptions`。 eg.

```java
@Override
public void beforeStartingVertx(VertxOptions options) {
    options.setEventLoopPoolSize(Runtime.getRuntime().availableProcessors() * 2);
    options.setWorkerPoolSize(Integer.getInteger(VERTX_WORKER_POOL_SIZE, DEFAULT_WORKER_POOL_SIZE));
    System.setProperty("IGNITE_NO_SHUTDOWN_HOOK", "true");
}
```

<!-- more -->

其实 Vert.x 也提供了直接在命令行设置`VertxOptions`的能力。


### Vert.x RunCommand 启动Verticle

```java RunCommand.run
@Override
public void run() {
    if (redeploy == null || redeploy.isEmpty()) {
        JsonObject conf = getConfiguration();
        if (conf == null) {
            conf = new JsonObject();
        }
        afterConfigParsed(conf);
        super.run(this::afterStoppingVertx);
}      
```

BareCommand.run

```java BareCommand.run
public void run(Runnable action) {
    this.finalAction = action;
    vertx = startVertx();
}

@SuppressWarnings("ThrowableResultOfMethodCallIgnored")
protected Vertx startVertx() {
    MetricsOptions metricsOptions = getMetricsOptions();
    options = new VertxOptions().setMetricsOptions(metricsOptions);
    //读取命令行配置
    configureFromSystemProperties(options, VERTX_OPTIONS_PROP_PREFIX);
    beforeStartingVertx(options);
}
```

> 需要注意的值：命令行配置选项的前缀是`vertx.options.`。例如你可以通过参数`-Dvertx.options.eventLoopPoolSize` 来指定 `eventLoopPoolSize`的大小


推而广之：

可以通过：`vertx.metrics.options.` ， `vertx.deployment.options.` 为前缀的JVM参数来设置相关的配置项。

```java
-Dvertx.metrics.options.jmxEnabled=true -Dvertx.metrics.options.jmxDomain=vertx
```




