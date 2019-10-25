## 前言

RocketMQ算是当前国内MQ领域内的明星产品。网上关于它文章非常的多。但是总觉得任何一项你想要在生产环境使用的技术，你必须亲自深入学习一遍才能更好的使用它，出了问题也好定位。在学习源码的过程中可能有许多意想不到的收获，也是提示技术能力的一种方式。

### RocketMQ 文件目录

RocketMQ 作为一个具有持久化功能的消息中间件，必然需要强大的存储功能。那么首先需要了解的就是它的存储架构。

RocketMQ 存储目录结构

```java
.
├── abort
├── checkpoint
├── commitlog
│   ├── 00000000000000000000
│   └── 00000000001073741824
├── config
│   ├── consumerFilter.json
│   ├── consumerFilter.json.bak
│   ├── consumerOffset.json
│   ├── consumerOffset.json.bak
│   ├── delayOffset.json
│   ├── delayOffset.json.bak
│   ├── subscriptionGroup.json
│   ├── topics.json
│   └── topics.json.bak
├── consumequeue
│   ├── rmq-test-topic
│   │   ├── 0
│   │   │   └── 00000000000000000000
│   │   ├── 1
│   │   │   └── 00000000000000000000
│   │   ├── 2
│   │   │   └── 00000000000000000000
│   │   ├── 3
│   │   │   └── 00000000000000000000
│   │   ├── 4
│   │   │   └── 00000000000000000000
│   │   ├── 5
│   │   │   └── 00000000000000000000
│   │   ├── 6
│   │   │   └── 00000000000000000000
│   │   └── 7
│   │       └── 00000000000000000000
│   └── SCHEDULE_TOPIC_XXXX
│       └── 2
│           └── 00000000000000000000
├── index
│   └── 20191014212222129
└── lock
```

 RocketMQ 整体存储结构![img](https://upload-images.jianshu.io/upload_images/4325076-6021ce04990eef9e.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1146/format/webp) 

上图是测试环境搭建的一个单机集群（只有一个NameServer和Broker）的目录结构。

### commitlog

commitlog 文件夹下面的每个文件大小默认 1G，内容是每条消息的完整内容。

>  需要注意的是：不可能所有的消息加起来恰好等于1G, 所以文件的尾部可能会有填充。

### consumequeue

consumequeue 文件夹下面是每个Topic的消费队列数据。

每个消费队列文件中保存的待消费的消息：结构如下：

```java
offset:length:taghash
```

### config

config 目录保存的是一些配置信息

#### consumerFilter.json

该文件保存的是每个Topic中消息的过滤逻辑。

#### consumerOffset.json

该文件保存每个consumer group 的消费进度

#### delayOffset.json

该文件保存构建延迟消息的进度

#### subscriptionGroup.json

该文件保存consumerGroup和Topic的订阅关系

#### topics.json

该文件保存Topic信息

### Broker消息保存流程

#### 1. Rpc请求处理

```java
class NettyServerHandler extends SimpleChannelInboundHandler<RemotingCommand> {

        @Override
        protected void channelRead0(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
            processMessageReceived(ctx, msg);
        }
}

public void processMessageReceived(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
    final RemotingCommand cmd = msg;
    if (cmd != null) {
        switch (cmd.getType()) {
            case REQUEST_COMMAND:
                processRequestCommand(ctx, cmd);
                break;
            case RESPONSE_COMMAND:
                processResponseCommand(ctx, cmd);
                break;
            default:
                break;
        }
    }
}

/**
     * Process incoming request command issued by remote peer.
     *
     * @param ctx channel handler context.
     * @param cmd request command.
     */
public void processRequestCommand(final ChannelHandlerContext ctx, final RemotingCommand cmd) {
    // 不同任务的请求是由不同的线程池和Processor进行处理的
    // 这样可以实现请求隔离，防止意外请求导致整个服务不可用
    final Pair<NettyRequestProcessor, ExecutorService> matched = this.processorTable.get(cmd.getCode());
    
    final Pair<NettyRequestProcessor, ExecutorService> pair = null == matched ? this.defaultRequestProcessor : matched;
    final int opaque = cmd.getOpaque();
	
    if (pair != null) {
    	// 构造异步请求任务，准备投递到对于的线程池中    
        Runnable run = new Runnable() {
            @Override
            public void run() {
                try {
                    // 执行钩子程序
                    RPCHook rpcHook = NettyRemotingAbstract.this.getRPCHook();
                    if (rpcHook != null) {
 rpcHook.doBeforeRequest(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd);
                    }
					// 真正进行请求的处理， SendMessageProcessor
                    final RemotingCommand response = pair.getObject1().processRequest(ctx, cmd);
                    if (rpcHook != null) {
                    // 执行钩子程序    rpcHook.doAfterResponse(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd, response);
                    }

                    if (!cmd.isOnewayRPC()) {
                        if (response != null) {
                            response.setOpaque(opaque);
                            response.markResponseType();
                            try {
                                ctx.writeAndFlush(response);
                            } catch (Throwable e) {
                                log.error("process request over, but response failed", e);
                                log.error(cmd.toString());
                                log.error(response.toString());
                            }
                        } else {

                        }
                    }
                } catch (Throwable e) {
                    log.error("process request exception", e);
                    log.error(cmd.toString());

                    if (!cmd.isOnewayRPC()) {
                        final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_ERROR,
                                                                                               RemotingHelper.exceptionSimpleDesc(e));
                        response.setOpaque(opaque);
                        ctx.writeAndFlush(response);
                    }
                }
            }
        };
        // 请求处理线程池拒绝服务，直接返回
        if (pair.getObject1().rejectRequest()) {
            final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_BUSY,
                                                                                   "[REJECTREQUEST]system busy, start flow control for a while");
            response.setOpaque(opaque);
            ctx.writeAndFlush(response);
            return;
        }

        try {
            //投递到线程池进行处理
            final RequestTask requestTask = new RequestTask(run, ctx.channel(), cmd);
            pair.getObject2().submit(requestTask);
        } catch (RejectedExecutionException e) {
            if ((System.currentTimeMillis() % 10000) == 0) {
                log.warn(RemotingHelper.parseChannelRemoteAddr(ctx.channel())
                         + ", too many requests and system thread pool busy, RejectedExecutionException "
                         + pair.getObject2().toString()
                         + " request code: " + cmd.getCode());
            }
			// 返回系统繁忙，客户端可以重试
            if (!cmd.isOnewayRPC()) {
                final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_BUSY,
                                                                                       "[OVERLOAD]system busy, start flow control for a while");
                response.setOpaque(opaque);
                ctx.writeAndFlush(response);
            }
        }
    } else {
        // 不支持的请求CODE
        String error = " request type " + cmd.getCode() + " not supported";
        final RemotingCommand response =
            RemotingCommand.createResponseCommand(RemotingSysResponseCode.REQUEST_CODE_NOT_SUPPORTED, error);
        response.setOpaque(opaque);
        ctx.writeAndFlush(response);
        log.error(RemotingHelper.parseChannelRemoteAddr(ctx.channel()) + error);
    }
}
```

#### 2. 消息处理器SendMessageProcessor

```java
@Override
public RemotingCommand processRequest(ChannelHandlerContext ctx,
                                      RemotingCommand request) throws RemotingCommandException {
    SendMessageContext mqtraceContext;
    switch (request.getCode()) {
        case RequestCode.CONSUMER_SEND_MSG_BACK:
            // 处理消费者ack
            return this.consumerSendMsgBack(ctx, request);
        default:
            SendMessageRequestHeader requestHeader = parseRequestHeader(request);
            if (requestHeader == null) {
                return null;
            }

            mqtraceContext = buildMsgContext(ctx, requestHeader);
            // 执行钩子程序
            this.executeSendMessageHookBefore(ctx, request, mqtraceContext);

            String topic = requestHeader.getTopic();
            this.checkTopicAuthority(ctx, topic);

            RemotingCommand response;
            if (requestHeader.isBatch()) {
                response = this.sendBatchMessage(ctx, request, mqtraceContext, requestHeader);
            } else {
                response = this.sendMessage(ctx, request, mqtraceContext, requestHeader);
            }
			// 执行钩子程序
            this.executeSendMessageHookAfter(response, mqtraceContext);
            return response;
    }
}

private RemotingCommand sendMessage(final ChannelHandlerContext ctx,
                                    final RemotingCommand request,
                                    final SendMessageContext sendMessageContext,
                                    final SendMessageRequestHeader requestHeader) throws RemotingCommandException {

    final RemotingCommand response = RemotingCommand.createResponseCommand(SendMessageResponseHeader.class);
    final SendMessageResponseHeader responseHeader = (SendMessageResponseHeader)response.readCustomHeader();

    response.setOpaque(request.getOpaque());

    response.addExtField(MessageConst.PROPERTY_MSG_REGION, this.brokerController.getBrokerConfig().getRegionId());
    response.addExtField(MessageConst.PROPERTY_TRACE_SWITCH, String.valueOf(this.brokerController.getBrokerConfig().isTraceOn()));

    log.debug("receive SendMessage request command, {}", request);

    //Broker还没有开始服务
    final long startTimestamp = this.brokerController.getBrokerConfig().getStartAcceptSendRequestTimeStamp();
    if (this.brokerController.getMessageStore().now() < startTimestamp) {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark(String.format("broker unable to service, until %s", UtilAll.timeMillisToHumanString2(startTimestamp)));
        return response;
    }

    response.setCode(-1);
    super.msgCheck(ctx, requestHeader, response);
    if (response.getCode() != -1) {
        return response;
    }

    final byte[] body = request.getBody();
	// 获取 队列id，判断是否是发送到指定的队列
    int queueIdInt = requestHeader.getQueueId();
    TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());
    // 随机选择一个队列进行写入
    if (queueIdInt < 0) {
        queueIdInt = Math.abs(this.random.nextInt() % 99999999) % topicConfig.getWriteQueueNums();
    }

    MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
    msgInner.setTopic(requestHeader.getTopic());
    msgInner.setQueueId(queueIdInt);
    // 处理重试消息
    if (!handleRetryAndDLQ(requestHeader, response, request, msgInner, topicConfig)) {
        return response;
    }
    // 省略部分代码
    String traFlag = oriProps.get(MessageConst.PROPERTY_TRANSACTION_PREPARED);
    // 事务消息处理
    if (traFlag != null && Boolean.parseBoolean(traFlag)) {
        transHalfMsg = true;
        if (this.brokerController.getBrokerConfig().isRejectTransactionMessage()) {
            response.setCode(ResponseCode.NO_PERMISSION);
            response.setRemark(
                "the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1()
                + "] sending transaction message is forbidden");
            return response;
        }
        putMessageResult = this.brokerController.getTransactionalMessageService().prepareMessage(msgInner);
    } else {
        // schedule message don't support transaction message now
        int delayInSeconds = msgInner.getDelayTimeInSeconds();
        int delayLevel = msgInner.getDelayTimeLevel();
        if (delayInSeconds != 0 && delayLevel != 0) {
            response.setCode(ResponseCode.MESSAGE_ILLEGAL);
            response.setRemark("delayInSeconds and delayTimeLevel should not be specified at the same time.");
            return response;
        }
        // 延迟消息处理
        if (delayInSeconds != 0) {
            putMessageResult = this.brokerController.getScheduledMessageStore().putMessage(msgInner);
        } else {
            // 普通消息处理
            putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);
        }
    }
    if (sendMessageContext != null) {
        sendMessageContext.buildTrackMsgType(transHalfMsg);
    }
    return handlePutMessageResult(putMessageResult, response, request, msgInner, responseHeader, sendMessageContext, ctx, queueIdInt);
}
```

#### 普通消息保存

普通消息保存是由 `DefaultMessageStore.putMessage`来实现消息具体的保存逻辑。

```java
public PutMessageResult putMessage(MessageExtBrokerInner msg) {
    if (this.shutdown) {
        log.warn("message store has shutdown, so putMessage is forbidden");
        return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
    }
    // 省略部分代码

    // 默认情况下，获取锁后，消息写入MappedFile的时间超过1s，表示OS Page Cache 忙
    // 此处应该是异步刷盘，同步刷盘超时概率变大
    if (this.isOSPageCacheBusy()) {
        return new PutMessageResult(PutMessageStatus.OS_PAGECACHE_BUSY, null);
    }

    long beginTime = this.getSystemClock().now();
    // CommitLog 真正写入消息
    PutMessageResult result = this.commitLog.putMessage(msg);

    long eclipseTime = this.getSystemClock().now() - beginTime;
    if (eclipseTime > 500) {
        log.warn("putMessage not in lock eclipse time(ms)={}, bodyLength={}", eclipseTime, msg.getBody().length);
    }
    // 监控消息写入性能 和 失败率
    this.storeStatsService.setPutMessageEntireTimeMax(eclipseTime);
    if (null == result || !result.isOk()) {
        this.storeStatsService.getPutMessageFailedTimes().incrementAndGet();
    }

    return result;
}
```

#### CommitLog写入消息

`CommitLog`表示RocketMQ Broker上保存的所有消息。 消息分布在多个`MappedFile`文件中，所有的`MappedFile`组成一个`MappedFileQueue`。

##### MappedFile 

顾名思义`MappedFile`表示一个内存映射文件，通过下面的代码创建：

```java
this.fileChannel = new RandomAccessFile(this.file, "rw").getChannel();
this.mappedByteBuffer = this.fileChannel.map(MapMode.READ_WRITE, 0, fileSize);
```

为什么用内存映射文件呢？

答案是：效率。消息写入直接写入内存页，由OS负责刷新内存中的内容到磁盘 或 通过API `MappedByteBuffer.force` 和 `FileChannel.force`方法来手动刷盘。

##### putMessage

```java
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {
    // 省略部分代码
    String topic = msg.getTopic();
    int queueId = msg.getQueueId();

    final int tranType = MessageSysFlag.getTransactionValue(msg.getSysFlag());
    // commit 消息 或 非事务消息
    if (tranType == MessageSysFlag.TRANSACTION_NOT_TYPE
        || tranType == MessageSysFlag.TRANSACTION_COMMIT_TYPE) {
        // Delay Delivery
        if (msg.getDelayTimeLevel() > 0) {
            if (msg.getDelayTimeLevel() > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {
                msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel());
            }
            // 写入到固定的Topic SCHEDULE_TOPIC_XXXX
            topic = ScheduleMessageService.SCHEDULE_TOPIC;
            // 根据延迟时间登记，选择一个队列
            queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

            // Backup real topic, queueId
            MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
            MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
            msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));

            msg.setTopic(topic);
            msg.setQueueId(queueId);
        }
    }

    long eclipseTimeInLock = 0;
    MappedFile unlockMappedFile = null;
    // 获取最后一个使用的MappedFile
    MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();

    putMessageLock.lock(); //spin or ReentrantLock ,depending on store config
    try {
        long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
        this.beginTimeInLock = beginLockTimestamp;

        // Here settings are stored timestamp, in order to ensure an orderly
        // global
        msg.setStoreTimestamp(beginLockTimestamp);
        // 创建第一个文件 或 创建下一个使用的文件
        if (null == mappedFile || mappedFile.isFull()) {
            mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
        }
        if (null == mappedFile) {
            log.error("create mapped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
            beginTimeInLock = 0;
            return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null);
        }
		// 写入共享内存
        result = mappedFile.appendMessage(msg, this.appendMessageCallback);
        switch (result.getStatus()) {
            case PUT_OK:
                break;
            case END_OF_FILE:
                // 文件满了，创建新文件来保存消息
                unlockMappedFile = mappedFile;
                // Create a new file, re-write the message
                mappedFile = this.mappedFileQueue.getLastMappedFile(0);
                if (null == mappedFile) {
                    // XXX: warn and notify me
                    log.error("create mapped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                    beginTimeInLock = 0;
                    return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result);
                }
                result = mappedFile.appendMessage(msg, this.appendMessageCallback);
                break;
            case MESSAGE_SIZE_EXCEEDED:
            case PROPERTIES_SIZE_EXCEEDED:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result);
            case UNKNOWN_ERROR:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
            default:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
        }

        eclipseTimeInLock = this.defaultMessageStore.getSystemClock().now() - beginLockTimestamp;
        beginTimeInLock = 0;
    } finally {
        putMessageLock.unlock();
    }

    if (eclipseTimeInLock > 500) {
        log.warn("[NOTIFYME]putMessage in lock cost time(ms)={}, bodyLength={} AppendMessageResult={}", eclipseTimeInLock, msg.getBody().length, result);
    }

    if (null != unlockMappedFile && this.defaultMessageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
        this.defaultMessageStore.unlockMappedFile(unlockMappedFile);
    }

    PutMessageResult putMessageResult = new PutMessageResult(PutMessageStatus.PUT_OK, result);

    // Statistics
    storeStatsService.getSinglePutMessageTopicTimesTotal(msg.getTopic()).incrementAndGet();
    storeStatsService.getSinglePutMessageTopicSizeTotal(topic).addAndGet(result.getWroteBytes());

    handleDiskFlush(result, putMessageResult, msg);
    handleHA(result, putMessageResult, msg);

    return putMessageResult;
}
```

##### MappedFileQueue.getLastMappedFile()

`MappedFileQueue`是由多个`MappedFile`组成。 获取最近一次使用的`MappedFile`时，存在2种情况。

1. 队列为空，创建第一个MappedFile
2. 不为空，返回最近一次使用的MappedFile

```java
public MappedFile getLastMappedFile() {
    MappedFile mappedFileLast = null;
	
    //有并发问题，此处简化处理了
    while (!this.mappedFiles.isEmpty()) {
        try {
            mappedFileLast = this.mappedFiles.get(this.mappedFiles.size() - 1);
            break;
        } catch (IndexOutOfBoundsException e) {
            //continue;
        } catch (Exception e) {
            log.error("getLastMappedFile has exception.", e);
            break;
        }
    }
    // 队列为空，直接返回null, 下面的方法会创建一个新的MappedFile 
    return mappedFileLast;
}
```

##### MappedFileQueue.getLastMappedFile(final long startOffset, boolean needCreate)

```java
public MappedFile getLastMappedFile(final long startOffset) {
        return getLastMappedFile(startOffset, true);
}

public MappedFile getLastMappedFile(final long startOffset, boolean needCreate) {
    long createOffset = -1;
    MappedFile mappedFileLast = getLastMappedFile();

    //第一个文件
    if (mappedFileLast == null) {
        createOffset = startOffset - (startOffset % this.mappedFileSize);
    }
	// 满了， 创建一个新文件，新文件的名称 = createOffset
    if (mappedFileLast != null && mappedFileLast.isFull()) {
        createOffset = mappedFileLast.getFileFromOffset() + this.mappedFileSize;
    }
	// 新创建的文件,名称必须是20个0， 或者 mappedFileSize的整数倍(格式化后为20位)
    if (createOffset != -1 && needCreate) {
        String nextFilePath = this.storePath + File.separator + UtilAll.offset2FileName(createOffset);
        String nextNextFilePath = this.storePath + File.separator
            + UtilAll.offset2FileName(createOffset + this.mappedFileSize);
        MappedFile mappedFile = null;
		// 交给 MappedFile 分配服务，异步创建。
        if (this.allocateMappedFileService != null) {
            mappedFile = this.allocateMappedFileService.putRequestAndReturnMappedFile(nextFilePath,
                                                                                      nextNextFilePath, this.mappedFileSize);
        } else {
            try {
                //直接创建MappedFile
                mappedFile = new MappedFile(nextFilePath, this.mappedFileSize);
            } catch (IOException e) {
                log.error("create mappedFile exception", e);
            }
        }

        if (mappedFile != null) {
            if (this.mappedFiles.isEmpty()) {
                mappedFile.setFirstCreateInQueue(true);
            }
            this.mappedFiles.add(mappedFile);
        }

        return mappedFile;
    }

    return mappedFileLast;
}
```



## 延迟消息实现

### 概述

RocketMQ延迟消息的实现，是通过将延迟消息根据延迟的时间长度不同，写入到不同时间粒度的Block中。

满足即将要发送（可能是一个小时内要发送的消息）的延迟消息会加载到内存中，通过TimeWheel来发送到目标Topicz中。通过后台线程不断查询满足条件的消息，及时加载到TimeWheel中。

### RocketMQ 目录和文件

#### 目录

数据存储根目录：user.home /store 

commit log 存储目录：user.home /store/commitlog

延迟消息存储目录：user.home /store/scheduledtempdata

#### 文件信息



消费者队列文件每个文件大小 30w * 20 byte

user.home / store / abort.scheduled  文件存在表示上次是正常关机，否则是异常关机（延迟消息恢复时使用）。

### ScheduledMessageStore

ScheduledMessageStore 提供访问延迟消息的API。

1. load
2. start

### Block 介绍

Block是一个逻辑概念，底层是由多个大小相同的log文件组成。时间粒度一般为一个小时。

Block文件目录结构：

```java
user.home/store/scheduledtempdata/20190910
    /log/000000xxxxxx // log目录下每个文件大小相同都是128M, 顺序写
    /queue/000000xxxxx // 每个文件大小为 30w * 24 字节
    /checkpoint
```

### ScheduledTempBlock

保存特定时间范围内的延迟消息，提供对于的操作API。

### 延迟消息加载逻辑

RocketMQ启动时，需要恢复延迟消息到上次关机是的状态。

1. 检查是否正常关机并打印日志。是否正常关机通过文件:`user.home / store / abort.scheduled`是否存在来判断。
2. 加载Block。遍历保存延迟消息的Block目录，通过每个`ScheduledTempBlock.load`方法来load各个Block。
3. 每个Block是由大小相同的文件组成，这些文件组成`MappedFileQueue`，每个Block load时，底层其实是通过`MappedFileQueue`来恢复，`MappedFileQueue.load`时，会把该Block对应的所有文件进行`mmap`，并恢复`MappedFile`的状态。
4. 恢复Block。遍历保存延迟消息的Block目录，通过每个`ScheduledTempBlock.revover`方法进行状态恢复。底层实现是通过`ScheduledTempQueue.recover`和`ScheduledTempLog.recoverNormally`来实现恢复。如果Queue 文件个数小于等于3,则从第一个Queue文件恢复，否则从倒数第三个开始恢复。
5. TimeWheelService 加载延迟消息。

### Consumer Queue 文件结构

ConsumeQueue中每个消息时20Byte长。结构为

![img](https://upload-images.jianshu.io/upload_images/5149787-be103f4e0b434fe5.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

## 参考资料

<https://www.jianshu.com/p/453c6e7ff81c>

 https://www.jianshu.com/p/ccdf6fc710b0 