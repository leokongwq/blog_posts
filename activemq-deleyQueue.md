---
layout: post
comments: true
title: ActiveMQ 延迟消息
date: 2018-06-27 23:08:31
tags:
- MQ
- ActiveMQ
categories:
- 消息中间件
---

### 前言

有非常多的业务场景需要用到延迟消息功能。例如：订单待支付提醒，订单超时取消等。但并不是所有的消息中间件都支持延迟消息的功能，为了实现延迟消息，各路大神也是创造了很多的方案。
我们的系统使用的是ActiveMQ，ActiveMQ从版本`5.4`开始提供了持久化的延迟消息功能。下文就ActiveMQ提供的延迟消息功能进行介绍。

### 启用ActiveMQ定时消息功能

为了使用ActiveMQ的延迟消息功能，我们需要修改ActiveMQ的配置文件`activemq.xml`，
在broker节点上添加`schedulerSupport="true"`，如下所示:

<!-- more -->

```xml
<broker xmlns="http://activemq.apache.org/schema/core" schedulerSupport="true" >
</broker>
```

### 创建延迟消息并发送

```java
jmsTemplate.send(orderCreatedTopic, session -> {
	MapMessage orderCreatedMsg = session.createMapMessage();
	orderCreatedMsg.setString("orderCode", orderCode);
	//延迟10分钟进行投递
	long time = 10 * 60 * 1000;
	orderCreatedMsg.setLongProperty(ScheduledMessage.AMQ_SCHEDULED_DELAY, time);
	return orderCreatedMsg;
});
```

### 延时消息属性总结

ActiveM提供了很多高级的延时消息配置属性，解释如下：

| 属性 | 类型 | 描述 |
| ---- | --- | ---- |
|　AMQ_SCHEDULED_DELAY | long | broker在投递该消息前等待的毫秒数 |
|　AMQ_SCHEDULED_PERIOD | long | 每次重新投递该消息的时间间隔 |
|　AMQ_SCHEDULED_REPEAT | int | 重复投递该消息的次数  |
|　AMQ_SCHEDULED_CRON　| String | 使用一个cron表达式来表示消息投递的策略 |


例子:

```java
MessageProducer producer = session.createProducer(destination);
TextMessage message = session.createTextMessage("test msg");
message.setStringProperty(ScheduledMessage.AMQ_SCHEDULED_CRON, "0 * * * *");
message.setLongProperty(ScheduledMessage.AMQ_SCHEDULED_DELAY, 1000);
message.setLongProperty(ScheduledMessage.AMQ_SCHEDULED_PERIOD, 1000);
message.setIntProperty(ScheduledMessage.AMQ_SCHEDULED_REPEAT, 9);
producer.send(message);
```

### 参考资料

[http://activemq.apache.org/delay-and-schedule-message-delivery.html](http://activemq.apache.org/delay-and-schedule-message-delivery.html)