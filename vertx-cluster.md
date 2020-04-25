---
layout: post
comments: true
title: vertx集群模式
date: 2017-12-06 19:34:26
tags:
- vert.x
categories:
---

### vert.x 集群简介

在 Vert.x 中，集群化与高可用均是开箱即用的。Vert.x 通过可插拔的集群管理器（cluster manager）来实现集群管理。在 Vert.x 中，采用 Hazelcast 作为默认的集群管理器。

### vert.x 集群器管理的作用

在 Vert.x 中，集群管理器可用于各种功能，包括：

- 对集群中 `Vert.x`结点进行分组，服务注册和发现
- 维护集群范围中的主题订阅者列表（所以我们可知道哪些节点对哪个Event Bus地址感兴趣）
- 分布式Map的支持
- 分布式锁
- 分布式计数器

集群管理器不处理Event Bus节点之间的传输，这是由 Vert.x 直接通过TCP连接完成。

<!-- more -->

### vert.x集群管理器设置

Vert.x发行版中使用的默认集群管理器是使用的Hazelcast集群管理器，但是它可以轻松被替换成实现了Vert.x集群管理器接口的不同实现，因为Vert.x集群管理器可替换的。

集群管理器必须实现`ClusterManager`接口，Vert.x在运行时使用Java的服务加载器（`Service Loader`）功能查找集群管理器，以便在类路径中查找`ClusterManager`的实例。

#### 命令行设置

若您在命令行中使用Vert.x并要使用集群，则应确保Vert.x安装的lib目录包含您的集群管理器jar。

#### 依赖设置

若您在 Maven/Gradle 项目使用Vert.x，则只需将集群管理器jar作为项目依赖添加。

maven

```java
<dependency>
  <groupId>io.vertx</groupId>
  <artifactId>vertx-hazelcast</artifactId>
  <version>3.5.0</version>
</dependency>
```

gradle

```java
compile 'io.vertx:vertx-hazelcast:3.5.0'
```

#### 编程设置

您也可以以编程的方式在嵌入Vert.x 时使用`setClusterManager`指定集群管理器。

```java
//创建ClusterManger对象
ClusterManager mgr = new HazelcastClusterManager();
//设置到Vertx启动参数中
VertxOptions options = new VertxOptions().setClusterManager(mgr);

Vertx.clusteredVertx(options, res -> {
  if (res.succeeded()) {
    Vertx vertx = res.result();
  } else {
    // failed!
  }
});
```

#### 集群管理器设置顺序

```java VertxImpl
private ClusterManager getClusterManager(VertxOptions options) {
    //是否启用集群模式
    if (options.isClustered()) {
        // 编程设置优先
        if (options.getClusterManager() != null) {
            return options.getClusterManager();
        } else {
            ClusterManager mgr;
            //命令行参数
            String clusterManagerClassName =    System.getProperty("vertx.cluster.managerClass");
            if (clusterManagerClassName != null) {
                // We allow specify a sys prop for the cluster manager factory which 
                // overrides ServiceLoader
                try {
                    Class<?> clazz = Class.forName(clusterManagerClassName);
                    mgr = (ClusterManager) clazz.newInstance();
                } catch (Exception e) {
                    throw new IllegalStateException("Failed to instantiate " +  clusterManagerClassName, e);
                }
            } else {
                // 服务查找
                mgr = ServiceHelper.loadFactoryOrNull(ClusterManager.class);
                if (mgr == null) {
                    throw new IllegalStateException("No ClusterManagerFactory instances found on classpath");
                }
            }
            return mgr;
        }
    } else {
        return null;
    }
}
```

总结一下查找顺序：

1. 首先通过 VertxOptions 对象获取集群管理器
2. 通过命令行参数指定 集群管理器的实现类
3. 通过Java的`service load`方式加载



