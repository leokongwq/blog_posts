---
layout: post
comments: true
title: vert.x run 命令解析
date: 2017-12-01 11:30:08
tags:
- vert.x
categories:
---

### 背景

在学习vert.x [蓝图](http://vertx.io/blog/vert-x-blueprint-tutorials/)系列工程是，发现项目的配置文件是通过启动参数`-conf`来指定的。在项目中没有发现解析该参数的代码，很是疑惑！那究竟是谁来处理该参数的呢？经过20分钟的努力终于找到了答案:`RunCommand`。耗时的原因是对Vert.x还不是很熟悉，知识点都没有串起来。

`RunCommand`提供了很多参数来启动Vert.x应用。下面我就一一分析下。

<!-- more -->

### 答案解析过程

RunCommand命令的`run`方法会将`--conf`参数指定配置文件进行解析，并调用`DeploymentOptions.setConfig`
设置该配置。

```java RunCommand
@Override
public void run() {
    //解析配置文件
    JsonObject conf = getConfiguration();  
    
    deploymentOptions = new DeploymentOptions();
    //读取系统数据，并设置
    configureFromSystemProperties(deploymentOptions, DEPLOYMENT_OPTIONS_PROP_PREFIX);
    //设置 DeploymentOptions 实例的 config属性
    deploymentOptions.setConfig(conf).setWorker(worker).setHa(ha).setInstances(instances);
}
```

创建`Context`时，设置`Context`的`config`属性:

```java DeploymentManager.doDeploy
private void doDeploy(String identifier, String deploymentID, DeploymentOptions options,
                        ContextImpl parentContext,
                        ContextImpl callingContext,
                        Handler<AsyncResult<String>> completionHandler,
                        ClassLoader tccl, Verticle... verticles) {
    //获取DeploymentOptions的config属性，并复制一份                        
    JsonObject conf = options.getConfig() == null ? new JsonObject() :  options.getConfig().copy(); // Copy it
    ...... 省略一些代码
    //创建Context时，传入Context的构造函数
    vertx.createEventLoopContext(deploymentID, pool, conf, tccl);
}    
```

### RunCommand 命令选项。

下面都是介绍该命令的选项的，可以通过Vert.x的命令行工具获取：

```shell
cd $VERTX_HOME/bin
./vertx run -h
```

### -cp,--classpath

指定额外的类路径。

### --cluster

如果指定了该参数，则该Vert.x实例会通过网络和其它Vert.x实例组成一个集群。

### --cluster-port

Vert.x应用集群的通信端口，默认为0。 随机使用一个端口。

### conf 参数

该参数通过`--conf`来指定，可以是一个json格式的文件，也可以是json字符串。该参数用来设置主Verticle的配置。

在idea开发环境中，我们可以将自定义的`Launcher`作为主类，通过`--conf`指定配置文件。默认配置文件是从项目的根目录查找的。

```java RunCommand
@Override
public void run() {
    //读取配置文件并解析
    JsonObject conf = getConfiguration();
    //创建发布选项 并 和命令行参数进行合并
    deploymentOptions = new DeploymentOptions();
    configureFromSystemProperties(deploymentOptions, DEPLOYMENT_OPTIONS_PROP_PREFIX);
    //将配置文件内容，设置到 发布选项对象中。        deploymentOptions.setConfig(conf).setWorker(worker).setHa(ha).setInstances(instances);
}    
```

DeploymentManager.doDeploy

```java
private void doDeploy(String identifier, String deploymentID, DeploymentOptions options,
                        ContextImpl parentContext,
                        ContextImpl callingContext,
                        Handler<AsyncResult<String>> completionHandler,
                        ClassLoader tccl, Verticle... verticles) {
    JsonObject conf = options.getConfig() == null ? new JsonObject() : options.getConfig().copy(); // Copy it
    String poolName = options.getWorkerPoolName();
    
    // 创建Verticle上下文对象， 将conf参数传入，以后就可以通过上下文对象随时访问
    ContextImpl context = options.isWorker() ? vertx.createWorkerContext(options.isMultiThreaded(), deploymentID, pool, conf, tccl) :
        vertx.createEventLoopContext(deploymentID, pool, conf, tccl);
}    
```    

### --ha

是否以高可用模式运行，如果指定了该参数，那么失败的Verticle会自动转移到同一个高可用组里面。

### --hagroup 

该参数和 `--ha` 配合使用，指定高可用的组名称。默认是`__DEFAULT__`

### --instances

指定发布的Verticle的数量，默认为1。

### --on-redeploy 

指定一个可以的shell命令，在重新发布Verticle的时候会被调用。该参数通常和`--ha`一起使用，当某个Verticle被重新发布到新的节点上时，我们可以获取一个通知。

### --quorum

和`--ha`配合使用，指定组成HA集群的最小节点个数。默认为1。

### --redeploy

启用自动重新发布功能。该参数需要指定一些被监控的文件，多个以逗号分隔。

### --redeploy-grace-period

如果启用了自动重新发布功能，该参数用来指定两次发布的时间间隔。单位是毫秒，默认值为1000。

### --redeploy-scan-period

如果启用了自动重新发布功能，该选项用来配置文件系统的扫描文件改变的间隔频率。默认是 250毫秒。

### --redeploy-termination-period

如果启用了自动重新发布功能，该选项用来指定重新发布前，确认当前应用是否停止的等待时间。该选项在`Windows`上非常有用，这是因为`terminate`命令会等一段时间才执行。 默认是0毫秒。
                                             
### --worker

如果知道了该参数，则该Verticle 以Worker模式运行。


