---
layout: post
comments: true
title: 常见MQ产品比较
date: 2016-10-12 18:00:34
tags:
categories:
- MQ
---

### 常见MQ产品比较

1. 介绍
 2. kafka
 3. rabbitmq
 4. rocketmq
 5. activemq
 6. zeromq
 7. 选型总结

<!-- more -->

### 1. 介绍

选择哪个mq取决于具体的应用场景。为了方便看清这些mq的区别，接下来将会主要罗列一下各个MQ最核心的一些特性。

本文的一个基本约定：当某个MQ罗列出一个核心特性，该特性往往是别的MQ没有的，或者是做的相对不够好的

### 2. kafka

 - 高吞吐量
 - 强消息堆积能力
 - 流处理(0.10.x支持 )

### 3. rabbitmq

遵循AMQP协议，借助erlang的特性在可靠性、稳定性和实时性上比别的MQ做得更好
当需要经过复杂的路由才将消息给consumer的情况也使用于rabbitmq(也就是说在复杂网络情况下)

### 4. rocketmq

 - 分布式事务消息支持
 - 消费失败定时重试
 - 严格消息顺序（kafka在某个broker挂了之后，在全局上看会出现消费乱序，解决的话要在consumer上下功夫）
 - 消息查询、消息轨迹、消息过滤
用其他MQ要做消息过滤查询可以使用Storm处理或者Cassandra做查询缓存

### 5. activemq

 - 完全遵循JMS规范
 - 比较重量级
                        
好像没啥特别好的也没啥特别不好的

### 6. zeromq

感觉这个MQ比较小众。不太适合分布式场景，没有一些高可用的保证措施，不过吞吐量还是比较大的

### 7. 选型总结

MQ现在感觉比较流行和成熟的应该是kafka和rabbitmq。MQ选型可以从以下两个角度去考虑选择KAFKA还是RABBITMQ

 - 开发语言：两者分别使用scala和erlang，如果需要对源码做修改和扩展，相信使用kafka会好点
 - 消息量：10万/秒以上的消息量则用kafka，2万/秒左右的消息量用rabbitmq                    

#### 参考资料：

[https://yq.aliyun.com/articles/25385](https://yq.aliyun.com/articles/25385)
[https://www.quora.com/What-are-the-differences-between-Apache-Kafka-and-RabbitMQ](https://www.quora.com/What-are-the-differences-between-Apache-Kafka-and-RabbitMQ)
                    

