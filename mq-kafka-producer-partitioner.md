---
layout: post
comments: true
title: kafka发送消息分区选择策略详解
date: 2017-02-27 12:09:35
tags:
- kafka
- 面试
categories:
- MQ
---

### 背景

面试被问到kafka消息发送是分区选择的策略，当时回答说是随机选择一个分区；或者通过消息key的hash值和分区数计算出分区。当时只是猜测的，并没有查看过kafka的源代码来证实，今天就通过源码来证实一下。

<!-- more -->

### 发送消息的一个简单例子

```java
 private static void sendMsg() {
   String topicName = "kafka-test";
   // 设置配置属性
   Properties props = new Properties();
   props.put("metadata.broker.list", "127.0.0.1:2181");
   props.put("bootstrap.servers", "127.0.0.1:9092");
   props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
   props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
   props.put("request.required.acks", "1");
   // 创建producer
   Producer<String, String> producer = new KafkaProducer<String, String>(props);

   for (int i = 0; i < 10; i++) {
       producer.send(new ProducerRecord<String, String>(topicName, Integer.toString(i), Integer.toString(i)));
   }

   System.out.println("Message sent successfully");
   producer.close();
}
```

### kafka 生产者发送消息分区选择策略

要知道kafka发送消息的分片选择策略，我们就从`send`方法入手，通过跟踪send方法，发现`KafkaProducer`是通过内部的私有方法`doSend`来发送消息的，里面有一行代码：

```java
int partition = partition(record, serializedKey, serializedValue, cluster);
```
这行代码的功能其实就是选择分区，partition方法的代码逻辑如下：

```java
private int partition(ProducerRecord<K, V> record, byte[] serializedKey, byte[] serializedValue, Cluster cluster) {
   Integer partition = record.partition();
   return partition != null ?partition :partitioner.partition(record.topic(),   record.key(), serializedKey, record.value(), serializedValue, cluster);
}
```

从上面的代码逻辑我们可以看出，如果`record`指定了分区则指定的分区会被使用，如果没有则使用`partitioner`分区器来选择分区。如果我们不在创建`KafkaProducer`对象的配置项中指定配置项:

```java
partitioner.class
```

的值的话，那默认使用的分区选择器实现类是：`DefaultPartitioner.class`, 该类的分区选择策略如下：

```java
/**
* Compute the partition for the given record.
*
* @param topic The topic name
* @param key The key to partition on (or null if no key)
* @param keyBytes serialized key to partition on (or null if no key)
* @param value The value to partition on or null
* @param valueBytes serialized value to partition on or null
* @param cluster The current cluster metadata
*/
public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
   List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
   int numPartitions = partitions.size();
   if (keyBytes == null) {
       int nextValue = nextValue(topic);
       List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
       if (availablePartitions.size() > 0) {
           int part = Utils.toPositive(nextValue) % availablePartitions.size();
           return availablePartitions.get(part).partition();
       } else {
           // no partitions are available, give a non-available partition
           return Utils.toPositive(nextValue) % numPartitions;
       }
   } else {
       // hash the keyBytes to choose a partition
       return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
   }
}
private int nextValue(String topic) {
   AtomicInteger counter = topicCounterMap.get(topic);
   if (null == counter) {
       counter = new AtomicInteger(new Random().nextInt());
       AtomicInteger currentCounter = topicCounterMap.putIfAbsent(topic, counter);
       if (currentCounter != null) {
           counter = currentCounter;
       }
   }
   return counter.getAndIncrement();
}
```  

分区选择策略分为两种：

- 消息的key为null

如果key为null，则先根据topic名获取上次计算分区时使用的一个整数并加一。然后判断topic的可用分区数是否大于0，如果大于0则使用获取的`nextValue`的值和可用分区数进行取模操作。 如果topic的可用分区数小于等于0，则用获取的`nextValue`的值和总分区数进行取模操作（其实就是随机选择了一个不可用分区）。
 
- 消息的key不为null 
    
不为null的选择策略很简单，就是根据hash算法`murmur2`就算出key的hash值，然后和分区数进行取模运算。    

### 总结

1. 如果不手动指定分区选择策略类，则会使用默认的分区策略类。
2. 如果不指定消息的key，则消息发送到的分区是随着时间不停变换的。
3. 如果指定了消息的key，则会根据消息的hash值和topic的分区数取模来获取分区的。
4. 如果应用有消息顺序性的需要，则可以通过指定消息的key和自定义分区类来将符合某种规则的消息发送到同一个分区。同一个分区消息是有序的，同一个分区只有一个消费者就可以保证消息的顺序性消费。

### 参考资料

[http://blog.csdn.net/ouyang111222/article/details/51086037](http://blog.csdn.net/ouyang111222/article/details/51086037)
[http://colobu.com/2015/01/22/which-kafka-partition-will-keyedMessages-be-sent-to-if-key-is-null/](http://colobu.com/2015/01/22/which-kafka-partition-will-keyedMessages-be-sent-to-if-key-is-null/)

