---
layout: post
comments: true
title: vertx中vertx-jdbc-client使用问题总结
date: 2017-12-01 22:55:24
tags:
- vert.x
categories:
---

### 背景

在使用vertx-jdbc-client中发现了一些问题，此文就是将我遇到的问题进行总结记录。希望能帮助同样遇到问题想找答案的人。

### SQLOperations.querySingleWithParams 空指针

在使用该方法想要返回一条记录时，如果数据库没有对应的数据，那么会报空指针。具体原因如下：

```java
return queryWithParams(sql, arguments, execute -> {
    if (execute.failed()) {
        handler.handle(Future.failedFuture(execute.cause()));
    } else {
        final ResultSet rs = execute.result();
        if (rs == null) {
            handler.handle(Future.succeededFuture());
        } else {
            //这里的判断不严谨，应用是==null || results.size() == 0
            List<JsonArray> results = rs.getResults();
            if (results == null) {
                handler.handle(Future.succeededFuture());
            } else {
               handler.handle(Future.succeededFuture(results.get(0)));
            }
        }
    }
});
```
<!-- more -->

### SQLOperations.updateXXX方法返回自增主键的值

我们在开发中通常想要获取新插入记录的注解ID值，在以前使用类似hibernate,mybatis等ORM框架时，只需要通过简单的配置，就可以获取插入的ID。

在使用vert-jdbc-client时，根据updateXX方法的返回值`UpdateResult`的描述信息:

```java UpdateResult
//SQL影响的行数
private int updated;
// insert操作产生的自增ID值集合
private JsonArray keys;

/**
* Constructor
*
* @param updated  number of rows updated
* @param keys  any generated keys
*/
public UpdateResult(int updated, JsonArray keys) {
    this.updated = updated;
    this.keys = keys;
}
```

但是在实际的使用过程中，如果我们什么都不做，插入数据后，调用`UpdateResult`的`getKeys`方法是不会拿到自增的主键值的。具体原因在于我们没有对表示数据库连接的`SQLConnection`对象进行配置。

来看一个例子：

```java
private Future<UpdateResult> doUpdate(String sql, JsonArray param) {
   return getConnection().compose(connection -> {
       // 这句话至关重要   
       connection.setOptions(new SQLOptions().setAutoGeneratedKeys(true));
       Future<UpdateResult> future = Future.future();
       connection.updateWithParams(sql, param, ar -> {
           if (ar.succeeded()) {
               future.complete(ar.result());
           } else {
               future.fail(ar.cause());
           }
           connection.close();
       });
       return future;
   });
}    
```

如果我们设置了`SQLOptions.autoGeneratedKeys`数据的值，那我们就能正确的获取到新增记录产生的增长ID值。

更近异步探究，我们发现`SQLOptions`还有许多其它配置，

```java SQLOptions
// connection
private boolean readOnly;
private String catalog;
private TransactionIsolation transactionIsolation;
private ResultSetType resultSetType;
private ResultSetConcurrency resultSetConcurrency;
// backwards compatibility
private boolean autoGeneratedKeys = true;
private JsonArray autoGeneratedKeysIndexes;
private String schema;
// statement
private int queryTimeout;
// resultset
private FetchDirection fetchDirection;
private int fetchSize;
```

以后如果遇到其它和预期不一致的行为，我们都可以在这里找一找是否可指定配置。

### 相关文章

[https://groups.google.com/forum/#!topic/vertx/ZFUSx3_HNsw](https://groups.google.com/forum/#!topic/vertx/ZFUSx3_HNsw)








