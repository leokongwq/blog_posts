---
layout: post
comments: true
title: kafka发送消息过程分析
date: 2017-02-28 23:08:31
tags:
- MQ
- kafka
categories:
- 消息中间件
---

## 前言

前面的一篇文章分析了消息发送时topic分区选择的问题，本文就分析一下后续的发送逻辑。

## 消息发送涉及的类

### Producer

`Producer`类是Kafka消息的发送的入口

### ProducerFactory

`ProducerFactory` 是一个策略接口，用来创佳`Producer`, 默认的实现是：`DefaultKafkaProducerFactory`,其创建
的`Producer`是`CloseSafeProducer`, `CloseSafeProducer`是`KafkaProducer`的一个代理类。

`DefaultKafkaProducerFactory` 的构造方法接收一个Map对象，用来指的创建`Producer`的配置。

```java
public DefaultKafkaProducerFactory(Map<String, Object> configs) {
	this(configs, null, null);
}

public DefaultKafkaProducerFactory(Map<String, Object> configs, Serializer<K> keySerializer,
		Serializer<V> valueSerializer) {
	this.configs = new HashMap<>(configs);
	this.keySerializer = keySerializer;
	this.valueSerializer = valueSerializer;
}
```

### KafkaTemplate

`KafkaTemplate`是一个模板类，提供了操作Kafka的高级Api。

```java
private KafkaTemplate<Integer, String> createTemplate() {
	Map<String, Object> senderProps = senderProps();
	ProducerFactory<Integer, String> pf =
			new DefaultKafkaProducerFactory<>(senderProps);
	KafkaTemplate<Integer, String> template = new KafkaTemplate<>(pf);
	return template;
}
```

## 发送消息实例

```java
// 生产者配置项
private Map<String, Object> senderProps() {
	Map<String, Object> props = new HashMap<>();
	props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
	props.put(ProducerConfig.RETRIES_CONFIG, 0);
	props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
	props.put(ProducerConfig.LINGER_MS_CONFIG, 1);
	props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);
	props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, IntegerSerializer.class);
	props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
	return props;
} 
// 创建生产者工厂实例
ProducerFactory<Integer, String> pf = new DefaultKafkaProducerFactory<>(senderProps);

 @Test
public void testSendMsgSysn() throws Exception {
	KafkaTemplate<Integer, String> kafkaTemplate = createTemplate();
	for (int i = 0; i < 10; i++) {
		// 同步发送
		//kafkaTemplate.send(topic1, "hello ===> " + i).get();
		// 异步发送
		kafkaTemplate.send(topic1, "hello ===> " + i);
	}
}
```

## 消息发送过程分析

`Producer` 的一个重要方法就是`send`, 方法签名如下：

```java
public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback);
```

从方法的签名我们能够推断出，Kafka的消息发送是异步的。 这个从后面的分析可以得证。

<!-- more -->

`Producer`的实现类是：`KafkaProducer`, 它对`send`方法的实现如下：

```java
@Override
public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
	// intercept the record, which can be potentially modified; this method does not throw exceptions
	ProducerRecord<K, V> interceptedRecord = this.interceptors == null ? record : this.interceptors.onSend(record);
	return doSend(interceptedRecord, callback);
}
```

该方法会对将要发送的消息进行个性化修改。是Kafka提供的插件机制。可以通过`interceptor.classes`配置型来配置。

```java
/**
* 异步发送一条记录到topic, 等价于　send(record, null)
* See {@link #send(ProducerRecord, Callback)} for details.
*/
private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
	TopicPartition tp = null;
	try {
		// 省略大部分代码
		RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey, serializedValue, interceptCallback, remainingWaitMs);
		//当记录累加器满了　或　创建了心的发送批次，此时会唤醒发送线程来真正通过网络发送消息
		if (result.batchIsFull || result.newBatchCreated) {
			log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
			// 唤醒发现线程
			this.sender.wakeup();
		}
		return result.future;
	}　
	// 省略异常处理代码
}
```

消息记录累加器： RecordAccumulator

```java RecordAccumulator
/**
 * 该类扮演一个队列的角色，用来将消息累积到`MemoryRecords`中
 * 该类使用一个有上限的内存缓存区来保存消息，默认情况下，调用添加消息的方法append时，如果缓存区满了会阻塞调用线程　(除非手动关闭这一行为)
 */
public final class RecordAccumulator {
	/**
     * 将一条记录添加到该累加器中，并返回添加的结果
     * 返回的结果包含了FutureRecordMetadata 和一个表示批次是否已经满了的标志位，一个批次是否是新创建的标志位
     * @param tp　该记录将要发送到的Topic的Partition信息
     * @param timestamp The timestamp of the record
     * @param key The key for the record
     * @param value The value for the record
     * @param callback 用户提供的回调函数
     * @param maxTimeToBlock 等待内存缓存区有可用空间的最大时间，单位是毫秒
     */
    public RecordAppendResult append(TopicPartition tp,
                                     long timestamp,
                                     byte[] key,
                                     byte[] value,
                                     Callback callback,
                                     long maxTimeToBlock) throws InterruptedException {
        // We keep track of the number of appending thread to make sure we do not miss batches in
        // abortIncompleteBatches().
        appendsInProgress.incrementAndGet();
        try {
            // 获取或创建一个指定partion对于的Deque
            Deque<RecordBatch> dq = getOrCreateDeque(tp);
            synchronized (dq) {
                if (closed)
                    throw new IllegalStateException("Cannot send after the producer is closed.");
                RecordAppendResult appendResult = tryAppend(timestamp, key, value, callback, dq);
                if (appendResult != null)
                    return appendResult;
            }

            //batchSize 的默认值是：16384
            int size = Math.max(this.batchSize, Records.LOG_OVERHEAD + Record.recordSize(key, value));
            log.trace("Allocating a new {} byte message buffer for topic {} partition {}", size, tp.topic(), tp.partition());
            // 分配一块缓存区，来保存消息
			ByteBuffer buffer = free.allocate(size, maxTimeToBlock);
            synchronized (dq) {
                // Need to check if producer is closed again after grabbing the dequeue lock.
                if (closed)
                    throw new IllegalStateException("Cannot send after the producer is closed.");

                RecordAppendResult appendResult = tryAppend(timestamp, key, value, callback, dq);
                if (appendResult != null) {
                    // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
                    free.deallocate(buffer);
                    return appendResult;
                }
				// 创建一个内存缓存区来缓存记录 
                MemoryRecords records = MemoryRecords.emptyRecords(buffer, compression, this.batchSize);
                RecordBatch batch = new RecordBatch(tp, records, time.milliseconds());
				// 将记录添加到内存缓存区
                FutureRecordMetadata future = Utils.notNull(batch.tryAppend(timestamp, key, value, callback, time.milliseconds()));
				// 将批次放入队列中
                dq.addLast(batch);
				// 将批次放入待确认队列
                incomplete.add(batch);
				// 第一条消息肯定是 新创建的， 会立刻发送。
                return new RecordAppendResult(future, dq.size() > 1 || batch.records.isFull(), true);
            }
        } finally {
            appendsInProgress.decrementAndGet();
        }
    }
}
```

消息批次： RecordBatch

```java RecordBatch
/**
 * 该类表示一个将要被发送的消息批次
 * 该类不是线程安全的，因此在修改该类的实例对象状态时需要外部同步
 */
public final class RecordBatch {
}
```

消息发送线程：Sender

```java
/**
 * 该类表示一个后台线程，用来将消息发送到kafka集群中。
 */
public class Sender implements Runnable {
	 /**
     * The main run loop for the sender thread
     */
    public void run() {
        log.debug("Starting Kafka producer I/O thread.");

        // 主循环，获取待发送的消息，并进行发送
        while (running) {
            try {
                run(time.milliseconds());
            } catch (Exception e) {
                log.error("Uncaught error in kafka producer I/O thread: ", e);
            }
        }

        log.debug("Beginning shutdown of Kafka producer I/O thread, sending remaining records.");

        // 停止接收发送消息的请求，但是会将缓存区的消息发送完毕，病处理服务端返回的ack
        while (!forceClose && (this.accumulator.hasUnsent() || this.client.inFlightRequestCount() > 0)) {
            try {
                run(time.milliseconds());
            } catch (Exception e) {
                log.error("Uncaught error in kafka producer I/O thread: ", e);
            }
        }
		// 强制关闭
        if (forceClose) {
            // We need to fail all the incomplete batches and wake up the threads waiting on
            // the futures.
            this.accumulator.abortIncompleteBatches();
        }
		// 关闭网络客户端
        try {
            this.client.close();
        } catch (Exception e) {
            log.error("Failed to close network client", e);
        }

        log.debug("Shutdown of Kafka producer I/O thread has completed.");
    }

	 /**
     * Run a single iteration of sending
     * @param now　The current POSIX time in milliseconds
     */
    void run(long now) {
		// 获取元数据
        Cluster cluster = metadata.fetch();
        // 获取可以发送数据的 Partition
        RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);

        // 如果任何Partition的Leader不能确定，则需要强制获取最新的Partition的Leader信息。
        if (result.unknownLeadersExist)
            this.metadata.requestUpdate();

        // 剔除不可用的节点
        Iterator<Node> iter = result.readyNodes.iterator();
        long notReadyTimeout = Long.MAX_VALUE;
        while (iter.hasNext()) {
            Node node = iter.next();
            if (!this.client.ready(node, now)) {
                iter.remove();
                notReadyTimeout = Math.min(notReadyTimeout, this.client.connectionDelay(node, now));
            }
        }

        // create produce requests
        Map<Integer, List<RecordBatch>> batches = this.accumulator.drain(cluster,
                                                                         result.readyNodes,
                                                                         this.maxRequestSize,
                                                                         now);
        //是否需要保持消息的顺序，max.in.flight.requests.per.connection 的值　＝１　可以保证
		if (guaranteeMessageOrder) {
            // Mute all the partitions drained
            for (List<RecordBatch> batchList : batches.values()) {
                for (RecordBatch batch : batchList)
                    this.accumulator.mutePartition(batch.topicPartition);
            }
        }

        List<RecordBatch> expiredBatches = this.accumulator.abortExpiredBatches(this.requestTimeout, now);
        // update sensors
        for (RecordBatch expiredBatch : expiredBatches)
            this.sensors.recordErrors(expiredBatch.topicPartition.topic(), expiredBatch.recordCount);

        sensors.updateProduceRequestMetrics(batches);
		// 创建需要发送的请求包对象
        List<ClientRequest> requests = createProduceRequests(batches, now);
        // If we have any nodes that are ready to send + have sendable data, poll with 0 timeout so this can immediately
        // loop and try sending more data. Otherwise, the timeout is determined by nodes that have partitions with data
        // that isn't yet sendable (e.g. lingering, backing off). Note that this specifically does not include nodes
        // with sendable data that aren't ready to send since they would cause busy looping.
        long pollTimeout = Math.min(result.nextReadyCheckDelayMs, notReadyTimeout);
        if (result.readyNodes.size() > 0) {
            log.trace("Nodes with data ready to send: {}", result.readyNodes);
            log.trace("Created {} produce requests: {}", requests.size(), requests);
            pollTimeout = 0;
        }
		// 将请求包暂存在待发送队列中
        for (ClientRequest request : requests)
            client.send(request, now);

        // if some partitions are already ready to be sent, the select time would be 0;
        // otherwise if some partition already has some data accumulated but not ready yet,
        // the select time will be the time difference between now and its linger expiry time;
        // otherwise the select time will be the time difference between now and the metadata expiry time;
        // 执行真正的网络ＩＯ操作，发送数据，读取响应，执行请求的回调函数
        this.client.poll(pollTimeout, now);
    }
}
```

创建客户端请求对象

```java
// 创建客户端请求对象
private ClientRequest produceRequest(long now, int destination, short acks, int timeout, List<RecordBatch> batches) {
	Map<TopicPartition, ByteBuffer> produceRecordsByPartition = new HashMap<TopicPartition, ByteBuffer>(batches.size());
	final Map<TopicPartition, RecordBatch> recordsByPartition = new HashMap<TopicPartition, RecordBatch>(batches.size());
	for (RecordBatch batch : batches) {
		TopicPartition tp = batch.topicPartition;
		produceRecordsByPartition.put(tp, batch.records.buffer());
		recordsByPartition.put(tp, batch);
	}
	ProduceRequest request = new ProduceRequest(acks, timeout, produceRecordsByPartition);
	RequestSend send = new RequestSend(Integer.toString(destination),
										this.client.nextRequestHeader(ApiKeys.PRODUCE),
										request.toStruct());
	RequestCompletionHandler callback = new RequestCompletionHandler() {
		public void onComplete(ClientResponse response) {
			handleProduceResponse(response, recordsByPartition, time.milliseconds());
		}
	};
	return new ClientRequest(now, acks != 0, send, callback);
}
```

ClientRequest

```java
public final class ClientRequest {
    private final long createdTimeMs;
	// 需要进行确认
    private final boolean expectResponse;
    private final RequestSend request;
	// 回调
    private final RequestCompletionHandler callback;
    private final boolean isInitiatedByNetworkClient;
    private long sendTimeMs;
}	
```	

## Producer不丢失消息配置

- 同步发送。 类似：producer.send(record).get();
- block.on.buffer.full = true
- acks = all
- retries = MAX_VALUE
- max.in.flight.requests.per.connection = 1
- 使用KafkaProducer.send(record, callback)
- callback逻辑中显式关闭producer：close(0) 
- replication.factor >= 3
- min.insync.replicas > 1 消息至少要被写入到这么多副本才算成功，也是提升数据持久性的一个参数。与acks配合使用
保证replication.factor > min.insync.replicas  如果两者相等，当一个副本挂掉了分区也就没法正常工作了。通常设置replication.factor = min.insync.replicas + 1即可

解释：

1. block.on.buffer.full ： 使得producer将一直等待缓冲区直至其变为可用
2. acks ： 所有follower都响应了才认为消息提交成功，即"committed"
3. retries = MAX ： 无限重试
4. max.in.flight.requests.per.connection=1 ： 限制客户端在单个连接上能够发送的未响应请求的个数。
设置此值是1表示kafka broker在响应请求之前client不能再向同一个broker发送请求。注意：设置此参数是为了避免消息乱序
5. 使用`KafkaProducer.send(record, callback)`而不是`send(record)`方法   自定义回调逻辑处理消息发送失败
callback逻辑中最好显式关闭`producer：close(0)`
6. unclean.leader.election.enable=false  关闭unclean leader选举，即不允许非ISR中的副本被选举为leader，以避免数据丢失
7. replication.factor >= 3 业界通用的设置，每个Partition的副本大于等于3
8. min.insync.replicas > 1 消息至少要被写入到这么多副本才算成功，也是提升数据持久性的一个参数。与acks配合使用
保证replication.factor > min.insync.replicas  如果两者相等，当一个副本挂掉了分区也就没法正常工作了。通常设置replication.factor = min.insync.replicas + 1即可

## 总结

1. Kafka的消息发送是异步的。
2. Kafka提供了插件机制来扩展消息的发送流程，可以通过配置`interceptor.classes`来指定消息过滤器，在消息发送前对消息做特殊处理。
3. Kafka消息在通过网络发送前会累积在内存中（通过`RecordAccumulator`）。
4. 通过配置`max.in.flight.requests.per.connection = 1` 可以让Kafka一次只发送一条消息，只有在上条消息ack后才发送下一条消息