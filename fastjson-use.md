---
layout: post
comments: true
title: fastjson使用简介
date: 2016-10-17 10:58:32
tags:
categories:
- java
---

### fastjson简介

> fastjson是一个用java语言编写的json处理器(解析和生成)。

### 特性

 - FAST (官方说法，比任何json处理器都快，包括jackson。)
 - Powerful (支持普通JDK类，包括任意Java Bean Class、Collection、Map、Date或enum)
 - Zero-dependency (零依赖，除了JDK)

<!-- more -->

### maven 依赖

    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>VERSION_CODE</version>
    </dependency>


### 官方 Benchmark

时间是纳秒, 单位是字节

    create     ser   +same   deser   +shal   +deep   total   size  +dfl
    kryo                                119    1335    1273    1562    1709    1689    3024    223   140
    java-manual                         119    2129    2055    1005    1031    1062    3191    255   147
    json/fastjson/databind              117    1809    1687    1741    1786    1907    3715    486   262
    msgpack                             117    1827    1582    1985    2020    2118    3945    233   146
    json/jackson/manual                 118    1672    1501    2235    2232    2365    4037    468   253
    protobuf                            227    2939    1362    1626    1661    1965    4904    239   149
    thrift                              242    3157    2898    2031    2116    2194    5351    349   197
    json/jackson/databind               119    2851    2726    3941    4008    4154    7005    485   261
    hessian                             117    6135    5513   10082   10204   10311   16446    501   313
    json/google-gson/databind           117   11309   11304    7913    7967    8076   19385    486   259
    json/json-lib-databind              119   45547   45114  145951  145866  146582  192129    485   263

## 使用

### 普通实体类

```java
    public class Person {
        private String name;
        private String age;
        
        public Person(){
        }
        public Person(String name, String age){
            this.name = name;
            this.age = age;
        }
        // setter, getter省略
    }
    //对象序列化
    Person person = new Person("sky", 24);
    String json =  JSON.toJSONString(person);
    //对象序列化并格式化
    System.out.println(JSON.toJSONString(person, true));
    // 对象反序列化
    Person lily = JSON.parseObject(json, Person.class);
```

### 泛型对象

```java
    List<Person> list = new ArrayList<Person>();
    list.add(new Person("sky", 24));
    //序列化
    String json = JSON.toJSONString(list);
    System.out.println(json);
    //反序列化
    List<Person> list2 = JSON.parseObject(json, new TypeReference<List<Person>>(){});
```
                
                    
                    
                    