---
layout: post
comments: true
title: kafka 消费消息过程分析
date: 2017-02-28 23:08:31
tags:
- MQ
- kafka
- springboot
categories:
- 消息中间件
---

## 前言

Springboot目前非常火热，如果在Springboot环境下使用Kafka也变得非常重要。本文就Springboot环境下如何配置
和使用kafka进行一下介绍。

### Producer 发送消息确认策略

Kafka在消息的发送端提供了三种确认策略，改策略是通过是通过创建`KafkaProducer`的配置项`ack`来进行设置的。
它的取值有下面所示的三种值：

1. 0 不等待确认。此时 retries 重试配置不起作用。
2. 1 Partition 的Leader进行确认就可以了。 但此时消息没有被同步到其它副本，有丢失消息的风险。
3. all 等待所有副本都确认了才行。 改机制能保证很强的数据安全性。


## Consumer

`Consumer`是消费Kafka消息的接口，主要的实现类是`KafkaConsumer`

### ConsumerFactory

`ConsumerFactory` 是一个策略接口，用来创佳`Consumer`, 默认的实现是：`DefaultKafkaConsumerFactory`，返回一个`KafkaConsumer`
的实例。

```java
public DefaultKafkaConsumerFactory(Map<String, Object> configs) {
	this(configs, null, null);
}

public DefaultKafkaConsumerFactory(Map<String, Object> configs,
		Deserializer<K> keyDeserializer,
		Deserializer<V> valueDeserializer) {
	this.configs = new HashMap<>(configs);
	this.keyDeserializer = keyDeserializer;
	this.valueDeserializer = valueDeserializer;
}

// 消费者配置项
private Map<String, Object> consumerProps() {
	Map<String, Object> props = new HashMap<>();
	props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
	props.put(ConsumerConfig.GROUP_ID_CONFIG, group);
	props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
	props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "100");
	props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, "15000");
	props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, IntegerDeserializer.class);
	props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
	return props;
}

// 消费者工厂实例
DefaultKafkaConsumerFactory<Integer, String> cf = new DefaultKafkaConsumerFactory<>(props);
```

### MessageListenerContainer

`MessageListenerContainer`是Spring框架内部对消息消费的一个抽象，一般不应该被外面的类进行实现。
它用来表示一个消息监听者容器。

它有两个重要的实现类：`KafkaMessageListenerContainer` 和 `ConcurrentMessageListenerContainer`。
前者是单线程的，后者支持多线程。

`MessageListenerContainer` 在启动的时候会主动通过`Consumer`来获取Kafka上面的消息，并调用消息监听器
来处理消息。

### KafkaDataListener

`KafkaDataListener`是一个标识接口，表示一个消息监听器`AbstractMessageListenerContainer`在启动时会检查
它所包含的监听器的类型。

### 消费消息例子

```java
 @Test
public void testAutoCommit() throws Exception {
	logger.info("Start auto");
	ContainerProperties containerProps = new ContainerProperties(topic1);
	final CountDownLatch latch = new CountDownLatch(4);
	//设置消息监听器
	containerProps.setMessageListener((MessageListener<Integer, String>) message -> {
		logger.info("received: " + message);
		latch.countDown();
	});
	KafkaMessageListenerContainer<Integer, String> container = createContainer(containerProps);
	container.setBeanName("testAuto");
	container.start();
	Thread.sleep(1000); // wait a bit for the container to start
	//send msg
	testSendMsgSysn();

	assertTrue(latch.await(60, TimeUnit.SECONDS));
	container.stop();
	logger.info("Stop auto");
}
```

### Kafka消息消费确认类型

#### 确认类型
Kafka 中消息消费的offset的commit类型由枚举类`AckMode`来定义，具体如下：

### RECORD

每处理一条commit一次

#### BATCH

每次poll的时候批量提交一次，频率取决于每次poll的调用频率

#### TIME
 
每次间隔ackTime的时间去commit

#### COUNT

累积达到ackCount次的ack去commit

#### COUNT_TIME

ackTime或ackCount哪个条件先满足，就commit

#### MANUAL

用户负负责ack，但是背后也是批量上去。此时的监听器类型必须是`AcknowledgingMessageListener`

#### MANUAL_IMMEDIATE

listner负责ack，每调用一次，就立即commit。此时的监听器类型必须是`AcknowledgingMessageListener`

> ContainerProperties 中ackMode的默认值为：AckMode.BATCH

### Kafka获取消息逻辑分析

#### 获取消息过程

```java
@Override
public void run() {
	if (this.autoCommit && this.theListener instanceof ConsumerSeekAware) {
		((ConsumerSeekAware) this.theListener).registerSeekCallback(this);
	}
	this.count = 0;
	this.last = System.currentTimeMillis();
	if (isRunning() && this.definedPartitions != null) {
		initPartitionsIfNeeded();
		// we start the invoker here as there will be no rebalance calls to
		// trigger it, but only if the container is not set to autocommit
		// otherwise we will process records on a separate thread
		if (!this.autoCommit) {
			startInvoker();
		}
	}
	long lastReceive = System.currentTimeMillis();
	long lastAlertAt = lastReceive;
	while (isRunning()) {
		try {
			// 不是 autoCommit，则处理消费进度 
			if (!this.autoCommit) {
				processCommits();
			}
			processSeeks();
			if (this.logger.isTraceEnabled()) {
				this.logger.trace("Polling (paused=" + this.paused + ")...");
			}
			// 拉取待消费的消息, 默认一秒拉取一次
			ConsumerRecords<K, V> records = this.consumer.poll(this.containerProperties.getPollTimeout());
			if (records != null && this.logger.isDebugEnabled()) {
				this.logger.debug("Received: " + records.count() + " records");
			}
			if (records != null && records.count() > 0) {
				if (this.containerProperties.getIdleEventInterval() != null) {
					lastReceive = System.currentTimeMillis();
				}
				// 如果autoCommit设置为true，则在该线程进行消息的处理
				if (this.autoCommit) {
					invokeListener(records);
				}
				// 否则将记录提交到一个缓存队列中
				else {
					if (sendToListener(records)) {
						if (this.assignedPartitions != null) {
							// avoid group management rebalance due to a slow
							// consumer
							this.consumer.pause(this.assignedPartitions);
							this.paused = true;
							this.unsent = records;
						}
					}
				}
			}
			// 处理idle事件
			else {
				if (this.containerProperties.getIdleEventInterval() != null) {
					long now = System.currentTimeMillis();
					if (now > lastReceive + this.containerProperties.getIdleEventInterval()
							&& now > lastAlertAt + this.containerProperties.getIdleEventInterval()) {
						publishIdleContainerEvent(now - lastReceive);
						lastAlertAt = now;
						if (this.theListener instanceof ConsumerSeekAware) {
							seekPartitions(getAssignedPartitions(), true);
						}
					}
				}
			}
			this.unsent = checkPause(this.unsent);
		}
		catch (WakeupException e) {
			this.unsent = checkPause(this.unsent);
		}
		catch (Exception e) {
			if (this.containerProperties.getGenericErrorHandler() != null) {
				this.containerProperties.getGenericErrorHandler().handle(e, null);
			}
			else {
				this.logger.error("Container exception", e);
			}
		}
	}
	if (this.listenerInvokerFuture != null) {
		stopInvokerAndCommitManualAcks();
	}
	try {
		this.consumer.unsubscribe();
	}
	catch (WakeupException e) {
		// No-op. Continue process
	}
	this.consumer.close();
	if (this.logger.isInfoEnabled()) {
		this.logger.info("Consumer stopped");
	}
}
```

### kafka 消息确认之autoCommit原理分析

如果配置了`enable.auto.commit=true`， 那么在创建`ConsumerCoordinator`时，会提交一个定时任务，
每隔固定的时间`auto.commit.interval.ms`就会通过后台线程`AutoCommitTask`来完成消费进度的提交。

```java
private class AutoCommitTask implements DelayedTask {
	public void run(final long now) {
		//异步提交 消费进度， subscriptions.allConsumed() 获取每个Partition的消费进度
		commitOffsetsAsync(subscriptions.allConsumed(), new OffsetCommitCallback() {
			@Override
			public void onComplete(Map<TopicPartition, OffsetAndMetadata> offsets, Exception exception) {
				if (exception == null) {
					reschedule(now + interval);
				} else {
					log.warn("Auto offset commit failed for group {}: {}", groupId, exception.getMessage());
					reschedule(now + interval);
				}
			}
		});
	}
}
```

SubscriptionState

```java SubscriptionState
/* the list of partitions currently assigned */
// TopicPartitionState 里面包含了消费的进度， 它的值是在每次拉取数据时更新的。
private final Map<TopicPartition, TopicPartitionState> assignment;
```

Fetcher

```Fetcher
private int append(Map<TopicPartition, List<ConsumerRecord<K, V>>> drained,
                       PartitionRecords<K, V> partitionRecords,
                       int maxRecords) {
	// 更新最新的消费进度						   
	subscriptions.position(partitionRecords.partition, nextOffset);						   
}						   
```

### kafka 消息非autoCommit实现分析

```java processCommits
private void processCommits() {
	handleAcks();
	this.count += this.acks.size();
	long now;
	AckMode ackMode = this.containerProperties.getAckMode();
	if (!this.isManualImmediateAck) {
		//这里有重复更新保存消费进度数据的Map
		if (!this.isManualAck) {
			updatePendingOffsets();
		}
		boolean countExceeded = this.count >= this.containerProperties.getAckCount();
		if (this.isManualAck || this.isBatchAck || this.isRecordAck
				|| (ackMode.equals(AckMode.COUNT) && countExceeded)) {
			if (this.logger.isDebugEnabled() && ackMode.equals(AckMode.COUNT)) {
				this.logger.debug("Committing in AckMode.COUNT because count " + this.count
						+ " exceeds configured limit of " + this.containerProperties.getAckCount());
			}
			commitIfNecessary();
			this.count = 0;
		}
		// 处理 AckMode.TIME 或 AckMode.COUNT_TIME
		else {
			now = System.currentTimeMillis();
			boolean elapsed = now - this.last > this.containerProperties.getAckTime();
			//  如果 当前时间距离上次ack的时间间隔大于配置项ackTime设置的值
			if (ackMode.equals(AckMode.TIME) && elapsed) {
				if (this.logger.isDebugEnabled()) {
					this.logger.debug("Committing in AckMode.TIME " +
							"because time elapsed exceeds configured limit of " +
							this.containerProperties.getAckTime());
				}
				// 提交保存在 offsets 中的数据 （同步或异步）
				commitIfNecessary();
				this.last = now;
			}
			// 如果 当前时间距离上次ack的时间间隔大于配置项 ackTime 设置的值 或 待ack的记录数大于
			// ackCount设置的值 
			else if (ackMode.equals(AckMode.COUNT_TIME) && (elapsed || countExceeded)) {
				if (this.logger.isDebugEnabled()) {
					if (elapsed) {
						this.logger.debug("Committing in AckMode.COUNT_TIME " +
								"because time elapsed exceeds configured limit of " +
								this.containerProperties.getAckTime());
					}
					else {
						this.logger.debug("Committing in AckMode.COUNT_TIME " +
								"because count " + this.count + " exceeds configured limit of" +
								this.containerProperties.getAckCount());
					}
				}
				// 提交保存在 offsets 中的数据 （同步或异步）
				commitIfNecessary();
				this.last = now;
				this.count = 0;
			}
		}
	}
}
```

####  handleAcks

逐条处理每一个待ack的记录

```java handleAcks
private void handleAcks() {
	ConsumerRecord<K, V> record = this.acks.poll();
	while (record != null) {
		if (this.logger.isTraceEnabled()) {
			this.logger.trace("Ack: " + record);
		}
		processAck(record);
		record = this.acks.poll();
	}
}

private void processAck(ConsumerRecord<K, V> record) {
	// 如果AckMode的值是`AckMode.MANUAL_IMMEDIATE`
	if (ListenerConsumer.this.isManualImmediateAck) {
		try {
			// 则直接同步更新（syncCommits默认值为true） 或 异步更新 zookeeper中保存的offset的值
			ackImmediate(record);
		}
		catch (WakeupException e) {
			// ignore - not polling
		}
	}
	// 如果不是 `AckMode.MANUAL_IMMEDIATE` 则只是将消费进度offset的值保存到一个Map中
	else {
		addOffset(record);
	}
}
```		

### @EnableKafka

`@EnableKafka`注解配合`@Configuration`注解一起只用，会激活Kafka的基于注解驱动的消息消费功能。

```java EnableKafka
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(KafkaBootstrapConfiguration.class)
public @interface EnableKafka {
}
```

### KafkaBootstrapConfiguration

`KafkaBootstrapConfiguration` 的主要功能有两个：

 1. 注册`KafkaListenerAnnotationBeanPostProcessor`， 该Bean主要用来处理`@KafkaListener`注解
 2. 注册`KafkaListenerEndpointRegistry`， 该Bean用于管理`MessageListenerContainer`

```java KafkaBootstrapConfiguration
@Configuration
public class KafkaBootstrapConfiguration {

	@SuppressWarnings("rawtypes")
	@Bean(name = KafkaListenerConfigUtils.KAFKA_LISTENER_ANNOTATION_PROCESSOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public KafkaListenerAnnotationBeanPostProcessor kafkaListenerAnnotationProcessor() {
		return new KafkaListenerAnnotationBeanPostProcessor();
	}

	@Bean(name = KafkaListenerConfigUtils.KAFKA_LISTENER_ENDPOINT_REGISTRY_BEAN_NAME)
	public KafkaListenerEndpointRegistry defaultKafkaListenerEndpointRegistry() {
		return new KafkaListenerEndpointRegistry();
	}
}
```

### 总结

1. `ackOnError` 的默认值为true， 非autoCommit情况下，消费失败的记录会被添加到待ack的队列中。
2. `ListenerConsumer` 的 `errorHandler` 默认值是 `LoggingErrorHandler`，只会打印日志
3. 
4. 
5. 