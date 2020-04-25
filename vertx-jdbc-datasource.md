---
layout: post
comments: true
title: vertx对jdbc数据源的支持
date: 2017-12-01 22:30:18
tags:
- vert.x
categories:
---

### 前言

用Vert.x开发微服务少不了对各种jdbc数据源的支持，Vert.x对jdbc数据源的支持是通过SPI的方式支持的。这样我们可以通过配置选择Vert.x官方提供的jdbc数据源，也可以指定自己的数据源。

<!-- more -->

### Vert.x对jdbc数据源的SPI支持


```java
public JDBCClientImpl(Vertx vertx, JsonObject config, String datasourceName) {
    Objects.requireNonNull(vertx);
    Objects.requireNonNull(config);
    Objects.requireNonNull(datasourceName);
    this.vertx = vertx;
    this.holder = lookupHolder(datasourceName, config);
    this.exec = holder.exec();
    this.ds = holder.ds();
    this.metrics = holder.metrics;
    this.helper = new JDBCStatementHelper(config);
    setupCloseHook();
}

private DataSourceHolder lookupHolder(String datasourceName, JsonObject config) {
    synchronized (vertx) {
      LocalMap<String, DataSourceHolder> map = vertx.sharedData().getLocalMap(DS_LOCAL_MAP_NAME);
      DataSourceHolder theHolder = map.get(datasourceName);
      if (theHolder == null) {
        theHolder = new DataSourceHolder((VertxInternal) vertx, config, map, datasourceName);
      } else {
        theHolder.incRefCount();
      }
      return theHolder;
    }
}

private class DataSourceHolder implements Shareable {
    DataSourceProvider provider;
    JsonObject config;
    DataSource ds;
    synchronized DataSource ds() {
        //配置指定的数据源SPI实现类
        String providerClass = config.getString("provider_class");
        if (providerClass == null) {
            // 默认是C3P0
            providerClass = DEFAULT_PROVIDER_CLASS;
        }
        //通过配置的DataSourceProvider类名来获取数据源。
        Class clazz = Thread.currentThread().getContextClassLoader().loadClass(providerClass);
            provider = (DataSourceProvider) clazz.newInstance();
            //通过DataSourceProvider来获取DataSource
            ds = provider.getDataSource(config);
    }
}
```

### provider_class 的取值

`provider_class`配合项的取值有：

- io.vertx.ext.jdbc.spi.impl.C3P0DataSourceProvider
- io.vertx.ext.jdbc.spi.impl.HikariCPDataSourceProvider
- io.vertx.ext.jdbc.spi.impl.BoneCPDataSourceProvider
- io.vertx.ext.jdbc.spi.impl.AgroalCPDataSourceProvider

默认是：`io.vertx.ext.jdbc.spi.impl.C3P0DataSourceProvider`


### 不同数据源的配置参数

#### C3P0DataSourceProvider

- url
- user
- password
- max_pool_size
- min_pool_size
- initial_pool_size
- max_statements
- max_statements_per_connection
- max_idle_time

如果如何设置额外C3P0连接池的参数，可以通过在classpath中添加文件`c3p0.properties`

#### HikariCPDataSourceProvider

- dataSourceClassName
- jdbcUrl
- driverClassName
- username
- password
- autoCommit
- connectionTimeout
- idleTimeout
- maxLifetime
- connectionTestQuery
- minimumIdle
- maximumPoolSize
- metricRegistry
- healthCheckRegistry
- poolName
- initializationFailFast
- isolationInternalQueries
- allowPoolSuspension
- readOnly
- registerMBeans
- catalog
- connectionInitSql
- transactionIsolation
- validationTimeout
- leakDetectionThreshold

#### BoneCPDataSourceProvider

BoneCPDataSourceProvider 的配置参考 BoneCPDataSource本身的配置，参数名保持一致。

#### AgroalCPDataSourceProvider

- jdbcUrl
- driverClassName
- principal
- credential
- minSize
- maxSize
- initialSize
- acquisitionTimeout
- connectionReapTimeout
- connectionLeakTimeout
- connectionValidationTimeout

### 一个数据源配置例子

```json
{
  "provider_class":"io.vertx.ext.jdbc.spi.impl.HikariCPDataSourceProvider",
  "jdbcUrl": "jdbc:mysql://localhost:3306/vertx_blueprint?characterEncoding=UTF-8&useSSL=false",
  "driverClassName": "com.mysql.cj.jdbc.Driver",
  "username": "hello",
  "password": "world",
  "connectionTestQuery": "select 1 from dual",
  "maximumPoolSize":100,
  "connectionTimeout":3000,
  "idleTimeout":1800,
  "poolName":"HikariCPDataSource-Order"
}
```

