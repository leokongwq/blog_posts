---
layout: post
comments: true
title: ActiveMQ 消息重发策略
date: 2018-05-19 09:24:28
tags:
- ActiveMQ
categories:
- web
---

### 背景

在使用ActiveMQ时，配置了消息重发策略。 但因为对配置项的理解不够深刻，导致虽然消息重新被投递了，单因为时间间隔太小，最终被放入DLQ中。

> 注意： 我使用的ActiveMQ版本是5.8

### 错误配置

```java
@Bean
public RedeliveryPolicy redeliveryPolicy() {
   RedeliveryPolicy redeliveryPolicy = new RedeliveryPolicy();
   //是否在每次尝试重新发送失败后,增长这个等待时间
   redeliveryPolicy.setUseExponentialBackOff(true);
   //重发次数,默认为6次   这里设置为10次
   redeliveryPolicy.setMaximumRedeliveries(10);
   //重发时间间隔,默认为1秒
   redeliveryPolicy.setInitialRedeliveryDelay(1);
   //第一次失败后重新发送之前等待1秒,第二次失败再等待1 * 2秒,这里的2就是value
   redeliveryPolicy.setBackOffMultiplier(2);
   //是否避免消息碰撞
   redeliveryPolicy.setUseCollisionAvoidance(false);
   //设置重发最大拖延时间-1 表示没有拖延只有UseExponentialBackOff(true)为true时生效
   redeliveryPolicy.setMaximumRedeliveryDelay(-1);
   return redeliveryPolicy;
}
```

<!-- more -->

上面配置有如下问题：

1. 第一次投递延时`(initialRedeliveryDelay)`为1毫秒, 而不是注释里面说的1秒。这是第一个致命错误
2. 最大重投次数`(maximumRedeliveries)` 为10。 加上第一个配置，导致短时间内消息被重新投递多次，一般来说消费者肯定不能成功消费的。因此会导致消息被放入DLQ中，业务丢失了消息。
这是第二个错误，重投次数有点小。对于非常重要的消息，可以适当调大该配置值。
3. 是否启用重投时延指数增长策略`(useExponentialBackOff) 默认是false`。如何理解呢？ActiveMQ 会在延迟`initialRedeliveryDelay` 指定的时间后发起**第一次**重新投递，之后根据是否设置了`useExponentialBackOff=true`来判断是否需要递增每次投递的时延。如果设置了`useExponentialBackOff=true`，那么每次重新投递的时间会延迟`redeliveryDelay * backOffMultiplier`
    
由此可以退出每次消息重新投递的延时为：

| 次数 | 延时 |
| --- | --- |
| 1 | 1毫秒 |
| 2 | 2 毫秒 |
| 3 | 4 毫秒 |
| 4 | 8 毫秒 |
| 5 | 16 毫秒 |
| 6 | 32 毫秒 |
| 7 | 64 毫秒 |
| 8 | 128 毫秒 |
| 9 | 256 毫秒 |
| 10 | 512 毫秒 |

从上表可以看出，完全没有达到需要的效果。痛定思痛，翻看官方文档和源代码，将ActiveMQ消息重复策略总结如下：

### 消息重发时机

1．在使用事务的Session中，调用rollback()方法；
2．在使用事务的Session中，调用commit()方法之前就关闭了Session;
3．在Session中使用CLIENT_ACKNOWLEDGE签收模式，并且调用了`recover()`方法。

可以通过设置`ActiveMQConnectionFactory`和`ActiveMQConnection`来定制想要的再次传送策略。

### 消息重发配置项

#### collisionAvoidanceFactor	

默认值 0.15	 

设置防止冲突范围的正负百分比，只有启用useCollisionAvoidance参数时才生效。

#### maximumRedeliveries	 

默认值：6

最大重传次数。 达到最大重连次数后抛出异常。为-1时不限制次数，为0时表示不进行重传。

#### maximumRedeliveryDelay	 

默认值 -1	

最大传送延迟，只在useExponentialBackOff为true时有效（V5.5），假设首次重连间隔为10ms，倍数为2，那么第二次重连时间间隔为 20ms，第三次重连时间间隔为40ms，当重连时间间隔大的最大重连时间间隔时，以后每次重连时间间隔都为最大重连时间间隔。

#### initialRedeliveryDelay	 

默认值 1000L	 

初始重发延迟时间

#### redeliveryDelay	 

默认值：1000L	 

重发延迟时间，当initialRedeliveryDelay=0时生效（v5.4）

#### useCollisionAvoidance	 

默认值 false	 

启用防止冲突功能，因为消息接收时是可以使用多线程并发处理的，应该是为了重发的安全性，避开所有并发线程都在同一个时间点进行消息接收处理。所有线程在同一个时间点处理时会发生什么问题呢？应该没有问题，只是为了平衡broker处理性能，不会有时很忙，有时很空闲。

#### useExponentialBackOff	

默认值 false	 	 

启用指数倍数递增的方式增加延迟时间。

#### backOffMultiplier	 

默认值 5	

重连时间间隔递增倍数，只有值大于1和启用`useExponentialBackOff`参数时才生效。

### 重发源码分析

#### 重发时机

```java ActiveMQMessageConsumer.rollback
final int currentRedeliveryCount = lastMd.getMessage().getRedeliveryCounter();
 if (currentRedeliveryCount > 0) {
     // 获取下次被重新投递的延迟时间
     redeliveryDelay = redeliveryPolicy.getNextRedeliveryDelay(redeliveryDelay);
 } else {
     //  第一被重新投递的延迟时间 
     redeliveryDelay = redeliveryPolicy.getInitialRedeliveryDelay();
 }
 // 当前消息被重新投递的此时大于配置的值，此时消息会被发送到DLQ
 if (redeliveryPolicy.getMaximumRedeliveries() != RedeliveryPolicy.NO_MAXIMUM_REDELIVERIES
                    && lastMd.getMessage().getRedeliveryCounter() > redeliveryPolicy.getMaximumRedeliveries()) {
// We need to NACK the messages so that they get sent to the
// DLQ.
// Acknowledge the last message.

MessageAck ack = new MessageAck(lastMd, MessageAck.POSION_ACK_TYPE, deliveredMessages.size());
 } else {
     // only redelivery_ack after first delivery
     if (currentRedeliveryCount > 0) {
         MessageAck ack = new MessageAck(lastMd, MessageAck.REDELIVERED_ACK_TYPE, deliveredMessages.size());
         ack.setFirstMessageId(firstMsgId);
         session.sendAck(ack,true);
     }
}
```

#### 重新投递的延迟时间计算

```java RedeliveryPolicy.getNextRedeliveryDelay
public long getNextRedeliveryDelay(long previousDelay) {
   long nextDelay = redeliveryDelay;
   //启用了 指数级延迟时间递增策略
   if (previousDelay > 0 && useExponentialBackOff && backOffMultiplier > 1) {
       nextDelay = (long) (previousDelay * backOffMultiplier);
       if(maximumRedeliveryDelay != -1 && nextDelay > maximumRedeliveryDelay) {
           // in case the user made max redelivery delay less than redelivery delay for some reason.
           nextDelay = Math.max(maximumRedeliveryDelay, redeliveryDelay);
       }
   }
   //  启用防止冲突功能, 计算一个随机的延迟时间
   if (useCollisionAvoidance) {
       /*
        * First random determines +/-, second random determines how far to
        * go in that direction. -cgs
        */
       Random random = getRandomNumberGenerator();
       double variance = (random.nextBoolean() ? collisionAvoidanceFactor : -collisionAvoidanceFactor) * random.nextDouble();
       nextDelay += nextDelay * variance;
   }
   //每次重新投递的延迟时间是固定的 
   return nextDelay;
}
```

### 参考资料

[http://activemq.apache.org/redelivery-policy.html](http://activemq.apache.org/redelivery-policy.html)

[http://activemq.apache.org/message-redelivery-and-dlq-handling.html](http://activemq.apache.org/message-redelivery-and-dlq-handling.html)

