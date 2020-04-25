---
layout: post
comments: true
title: RocketMQ学习总结
date: 2019-11-28 23:21:09
tags:
    - MQ 
categories:
---

## 前言

RocketMQ算是当前国内MQ领域内的明星产品。网上关于它文章非常的多。但是总觉得任何一项你想要在生产环境使用的技术，你必须亲自深入学习一遍才能更好的使用它，出了问题也好定位。在学习源码的过程中可能有许多意想不到的收获，也是提示技术能力的一种方式。

### RocketMQ 文件目录

RocketMQ 作为一个具有持久化功能的消息中间件，必然需要强大的存储功能。那么首先需要了解的就是它的存储架构。

RocketMQ 存储目录结构

<!-- more -->

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

#### abort文件

abort 文件是在`DefaultMessageStore`启动时创建在，在正常关机的情况下会删除。

```java
private void createTempFile() throws IOException {
    String fileName = StorePathConfigHelper.getAbortFile(this.messageStoreConfig.getStorePathRootDir());
    File file = new File(fileName);
    MappedFile.ensureDirOK(file.getParent());
    boolean result = file.createNewFile();
    log.info(fileName + (result ? " create OK" : " already exists"));
}
```

#### checkpoint 文件

文件内容：

```java
public StoreCheckpoint(final String scpPath) throws IOException {
    File file = new File(scpPath);
    MappedFile.ensureDirOK(file.getParent());
    boolean fileExists = file.exists();

    this.randomAccessFile = new RandomAccessFile(file, "rw");
    this.fileChannel = this.randomAccessFile.getChannel();
    this.mappedByteBuffer = fileChannel.map(MapMode.READ_WRITE, 0, MappedFile.OS_PAGE_SIZE);

    if (fileExists) {
        log.info("store checkpoint file exists, " + scpPath);
        this.physicMsgTimestamp = this.mappedByteBuffer.getLong(0);
        this.logicsMsgTimestamp = this.mappedByteBuffer.getLong(8);
        this.indexMsgTimestamp = this.mappedByteBuffer.getLong(16);
    } else {
        log.info("store checkpoint file not exists, " + scpPath);
    }
}
```



### MappedFile & MappedFileQueue & AllocateMappedFileService

#### MappedFile 

MappedFile ：顾名思义就是一个内存映射文件。内存映射文件写入时，首先写入`OS Page Cache`中，OS会定期刷新文件内容到磁盘上。也可以通过如下的方法强制写入磁盘：

1. FileChannel.force
2. MappedByteBuffer.force

MappedFile 是最底层，真正用来保存消息的，主要的保存消息方法如下：

```java
public AppendMessageResult appendMessage(final MessageExtBrokerInner msg, final AppendMessageCallback cb) {
    return appendMessagesInner(msg, cb);
}

public AppendMessageResult appendMessages(final MessageExtBatch messageExtBatch, final AppendMessageCallback cb) {
    return appendMessagesInner(messageExtBatch, cb);
}
/**
* Broker 写入消息时调用的方法
*/
public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb) {
    assert messageExt != null;
    assert cb != null;
	// 当前写入MappedFile文件的位置
    int currentPos = this.wrotePosition.get();

    if (currentPos < this.fileSize) {
        // writeBuffer 本身也是一个DirectBuffer, 默认大小1G 和 MappedFile大小相同
        // 每次slice 都会得到position == 0, limit 和 capacity 等于1G 的影子ByteBuffer
        ByteBuffer byteBuffer = writeBuffer != null ? writeBuffer.slice() : this.mappedByteBuffer.slice();
        // 通过设置position， 使得上面获取的影子ByteBuffer写入时不会覆盖原有数据
        // 只能从底层真实的ByteBuffer的position处开始写数据
        // 因为 writeBuffer 和 mappedByteBuffer 的position一直保持不变
        // 所以设置position的方法不会抛出异常
        byteBuffer.position(currentPos);
        AppendMessageResult result = null;
        if (messageExt instanceof MessageExtBrokerInner) {
            result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBrokerInner) messageExt);
        } else if (messageExt instanceof MessageExtBatch) {
            result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBatch) messageExt);
        } else {
            return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
        }
        this.wrotePosition.addAndGet(result.getWroteBytes());
        this.storeTimestamp = result.getStoreTimestamp();
        return result;
    }
    log.error("MappedFile.appendMessage return null, wrotePosition: {} fileSize: {}", currentPos, this.fileSize);
    return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
}

public boolean appendMessage(final byte[] data) {
    int currentPos = this.wrotePosition.get();

    if ((currentPos + data.length) <= this.fileSize) {
        try {
            this.fileChannel.position(currentPos);
            this.fileChannel.write(ByteBuffer.wrap(data));
        } catch (Throwable e) {
            log.error("Error occurred when append message to mappedFile.", e);
        }
        this.wrotePosition.addAndGet(data.length);
        return true;
    }

    return false;
}

private boolean isAbleToFlush(final int flushLeastPages) {
    //上次刷盘的位置
    int flush = this.flushedPosition.get();
    // 当前的写入位置
    int write = getReadPosition();
	//如果文件写满了，那么需要刷盘
    if (this.isFull()) {
        return true;
    }
	//计算是否满足最少刷盘页数的的要求
    if (flushLeastPages > 0) {
        // 不可以用(write - flush) / OS_PAGE_SIZE 吗？
        // OS_PAGE_SIZE 写死为 4k, 如果启用大页， 吞吐量不是更高吗？
        return ((write / OS_PAGE_SIZE) - (flush / OS_PAGE_SIZE)) >= flushLeastPages;
    }
	// 只要上次刷盘后有新的数据写入，就可以刷盘
    return write > flush;
}

/**
 * @return The current flushed position
*/
public int flush(final int flushLeastPages) {
    if (this.isAbleToFlush(flushLeastPages)) {
        if (this.hold()) {
            int value = getReadPosition();

            try {
                //We only append data to fileChannel or mappedByteBuffer, never both.
                if (writeBuffer != null || this.fileChannel.position() != 0) {
                    this.fileChannel.force(false);
                } else {
                    this.mappedByteBuffer.force();
                }
            } catch (Throwable e) {
                log.error("Error occurred when force data to disk.", e);
            }

            this.flushedPosition.set(value);
            this.release();
        } else {
            log.warn("in flush, hold failed, flush offset = " + this.flushedPosition.get());
            // 如果 使用了writeBuffer, getReadPosition返回的是当前写入的position
            // 写入的position 是大于等于 刷盘的位置的，没有刷盘就更新flushedPosition合适吗？
            this.flushedPosition.set(getReadPosition());
        }
    }
    return this.getFlushedPosition();
}

protected boolean isAbleToCommit(final int commitLeastPages) {
    int flush = this.committedPosition.get();
    int write = this.wrotePosition.get();

    if (this.isFull()) {
        return true;
    }

    if (commitLeastPages > 0) {
        return ((write / OS_PAGE_SIZE) - (flush / OS_PAGE_SIZE)) >= commitLeastPages;
    }

    return write > flush;
}

public int commit(final int commitLeastPages) {
    // writeBuffer 为 NULL 表示消息是直接写入MappedByteBuffer的
    if (writeBuffer == null) {
        //no need to commit data to file channel, so just regard wrotePosition as committedPosition.
        return this.wrotePosition.get();
    }
    if (this.isAbleToCommit(commitLeastPages)) {
        if (this.hold()) {
            commit0(commitLeastPages);
            this.release();
        } else {
            log.warn("in commit, hold failed, commit offset = " + this.committedPosition.get());
        }
    }

    // All dirty data has been committed to FileChannel.
    if (writeBuffer != null && this.transientStorePool != null && this.fileSize == this.committedPosition.get()) {
        this.transientStorePool.returnBuffer(writeBuffer);
        this.writeBuffer = null;
    }

    return this.committedPosition.get();
}
```



#### MappedFileQueue 

MappedFileQueue  就是由多个`MappedFile`组成的一个队列。由它来管理多个分配的内存映射文件。多个文件组成一个逻辑上的单一文件，供上层的业务使用。

```java
public class MappedFileQueue {
    // 保存所有分配的MappedFile
	private final CopyOnWriteArrayList<MappedFile> mappedFiles = new CopyOnWriteArrayList<MappedFile>();
    
    // 保存刷盘位置, 该字段的值是在Broker启动时，恢复状态时计算出来的。
    // 正常情况下，是CommmitLog在磁盘上最后一个MappedFile中最后一个正常消息的偏移量
    private long flushedWhere = 0;
    // 写入到PageCache的位置， 在Broker启动时，值和flushedWhere计算逻辑一致。
    private long committedWhere = 0;
    
    // 执行刷盘操作，由刷盘后台线程调用
	public boolean flush(final int flushLeastPages) {
        boolean result = true;
        MappedFile mappedFile = this.findMappedFileByOffset(this.flushedWhere, this.flushedWhere == 0);
        if (mappedFile != null) {
            long tmpTimeStamp = mappedFile.getStoreTimestamp();
            // 最底层的刷盘操作，返回MappedFile中已经落盘的数据量
            int offset = mappedFile.flush(flushLeastPages);
            // 计算并更新下次刷盘的位置
            long where = mappedFile.getFileFromOffset() + offset;
            result = where == this.flushedWhere;
            this.flushedWhere = where;
            if (0 == flushLeastPages) {
                this.storeTimestamp = tmpTimeStamp;
            }
        }

        return result;
    }
}    
```



#### AllocateMappedFileService

MappedFile 的创建有2种方式：

1. 通过MappedFile的构造方法进行创建。
2. 通过AllocateMappedFileService进行异步创建。(内部有一个创建的`MappedFile`的请求队列，由独立的线程进行`MappedFile`创建)

`MappedFileQueue` 在进行消息保存，需要创建新的`MappedFile`时，内部是通过 `DefaultMessageStore`创建时创建的`AllocateMappedFileService`进行`MappedFile`创建的。

AllocateMappedFileService 会使用`MappedFile`如下的构造方法进行创建：

```java
public MappedFile(final String fileName, final int fileSize,
                  final TransientStorePool transientStorePool) throws IOException {
    init(fileName, fileSize, transientStorePool);
}
public void init(final String fileName, final int fileSize,
                 final TransientStorePool transientStorePool) throws IOException {
    init(fileName, fileSize);
    // 消息会先写入ByteBuffer中，然后再写入FileChannel
    // writeBuffer 大小默认是1G, 和 MappedFile的文件大小相同
    this.writeBuffer = transientStorePool.borrowBuffer();
    this.transientStorePool = transientStorePool;
}
```

### DefaultMessageStore 启动流程

#### DefaultMessageStore 初始化

初始化过程最重要的是加载消息文件

```java
public boolean load() {
    boolean result = true;

    try {
        // 判断是否上次是否是正常退出。 正常退出是没有abort文件的
        boolean lastExitOK = !this.isTempFileExist();
        log.info("last shutdown {}", lastExitOK ? "normally" : "abnormally");
	    // 先加载延迟消息		
        if (null != scheduleMessageService) {
            result = result && this.scheduleMessageService.load();
        }

        // load Commit Log
        // 加载 Commit Log 文件，其实是通过 MappedFileQueue加载每一个MappedFile
        result = result && this.commitLog.load();

        // load Consume Queue
        // 实现机制和 Commit Log 一样
        result = result && this.loadConsumeQueue();

        if (result) {
            // 初始化checkpoint
            this.storeCheckpoint =
                new StoreCheckpoint(StorePathConfigHelper.getStoreCheckpoint(this.messageStoreConfig.getStorePathRootDir()));
			// 加载索引文件
            this.indexService.load(lastExitOK);
			// 进行状态恢复， 底层还是通过
            this.recover(lastExitOK);

            log.info("load over, and the max phy offset = {}", this.getMaxPhyOffset());
        }
    } catch (Exception e) {
        log.error("load exception", e);
        result = false;
    }

    if (!result) {
        this.allocateMappedFileService.shutdown();
    }

    return result;
}

private void recover(final boolean lastExitOK) {
    // 恢复消费队列状态
    this.recoverConsumeQueue();
	// 恢复Commit Log 状态
    if (lastExitOK) {
        this.commitLog.recoverNormally();
    } else {
        this.commitLog.recoverAbnormally();
    }

    this.recoverTopicQueueTable();
}
```



#### DefaultMessageStore 启动

### CommitLog 启动流程

#### CommitLog 加载

```java
public boolean load() {
    boolean result = this.mappedFileQueue.load();
    log.info("load commit log " + (result ? "OK" : "Failed"));
    return result;
}
```

加载的过程就是对构成的Commit Log的所有磁盘文件，创建对应的MappedFile，并通过MappedFileQueue进行组织的过程。再次过程，申请了虚拟内存空间，但是并没有读取磁盘文件。

#### CommitLog 启动

```java
public void start() {
    // 启动消息 刷盘后台服务线程
    this.flushCommitLogService.start();
	// 如果启用了异步刷盘，则启动 消息commit后台服务线程
    if (defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
        this.commitLogService.start();
    }
}
```

#### CommitLog  恢复

恢复分为正常恢复和异常恢复

##### 正常恢复

```java
public void recoverNormally() {
    // 是否校验数据完成性，这个会导致启动变慢
    boolean checkCRCOnRecover = this.defaultMessageStore.getMessageStoreConfig().isCheckCRCOnRecover();
    final List<MappedFile> mappedFiles = this.mappedFileQueue.getMappedFiles();
    if (!mappedFiles.isEmpty()) {
        // 从最后三个文件进行恢复
        int index = mappedFiles.size() - 3;
        if (index < 0)
            index = 0;
		
        MappedFile mappedFile = mappedFiles.get(index);
        ByteBuffer byteBuffer = mappedFile.sliceByteBuffer();
        //本文件的起始偏移量，也就是文件名称
        long processOffset = mappedFile.getFileFromOffset();
        // 从0开始校验消息文件
        long mappedFileOffset = 0;
        while (true) {
            DispatchRequest dispatchRequest = this.checkMessageAndReturnSize(byteBuffer, checkCRCOnRecover);
            int size = dispatchRequest.getMsgSize();
            // Normal data
            if (dispatchRequest.isSuccess() && size > 0) {
                mappedFileOffset += size;
            }
            // Come the end of the file, switch to the next file Since the
            // return 0 representatives met last hole,
            // this can not be included in truncate offset
            else if (dispatchRequest.isSuccess() && size == 0) {
                // 所有文件都验证完成
                index++;
                if (index >= mappedFiles.size()) {
                    // Current branch can not happen
                    log.info("recover last 3 physics file over, last mapped file " + mappedFile.getFileName());
                    break;
                } else {
                    // 切换下一个文件进行校验
                    mappedFile = mappedFiles.get(index);
                    byteBuffer = mappedFile.sliceByteBuffer();
                    processOffset = mappedFile.getFileFromOffset();
                    mappedFileOffset = 0;
                    log.info("recover next physics file, " + mappedFile.getFileName());
                }
            }
            // Intermediate file read error
            else if (!dispatchRequest.isSuccess()) {
                log.info("recover physics file end, " + mappedFile.getFileName());
                break;
            }
        }
		// processOffset 就是有效消息的位置
        processOffset += mappedFileOffset;
        this.mappedFileQueue.setFlushedWhere(processOffset);
        this.mappedFileQueue.setCommittedWhere(processOffset);
        // 丢弃broken的消息
        this.mappedFileQueue.truncateDirtyFiles(processOffset);
    }
}
```

##### 异常恢复

```java
public void recoverAbnormally() {
    // recover by the minimum time stamp
    boolean checkCRCOnRecover = this.defaultMessageStore.getMessageStoreConfig().isCheckCRCOnRecover();
    final List<MappedFile> mappedFiles = this.mappedFileQueue.getMappedFiles();
    if (!mappedFiles.isEmpty()) {
        // 从倒数第一个开始恢复， 在load的时候，按文件名称排序的
        int index = mappedFiles.size() - 1;
        MappedFile mappedFile = null;
        for (; index >= 0; index--) {
            mappedFile = mappedFiles.get(index);
            //查找第一个需要需要恢复的文件
            if (this.isMappedFileMatchedRecover(mappedFile)) {
                log.info("recover from this mapped file " + mappedFile.getFileName());
                break;
            }
        }
		// 没有满足恢复查找条件的问题件，那么从第一个文件开始恢复
        if (index < 0) {
            index = 0;
            mappedFile = mappedFiles.get(index);
        }

        ByteBuffer byteBuffer = mappedFile.sliceByteBuffer();
        long processOffset = mappedFile.getFileFromOffset();
        long mappedFileOffset = 0;
        while (true) {
            DispatchRequest dispatchRequest = this.checkMessageAndReturnSize(byteBuffer, checkCRCOnRecover);
            int size = dispatchRequest.getMsgSize();

            // Normal data
            if (size > 0) {
                mappedFileOffset += size;
				// 是否允许重复消息
                if (this.defaultMessageStore.getMessageStoreConfig().isDuplicationEnable()) {
                    // getCommitLogOffset 返回的是 消息在Commit Log中的偏移量
                    if (dispatchRequest.getCommitLogOffset() < this.defaultMessageStore.getConfirmOffset()) {
                        this.defaultMessageStore.doDispatch(dispatchRequest);
                    }
                } else {
                    // 构建索引 和 消费队列
                    this.defaultMessageStore.doDispatch(dispatchRequest);
                }
            }
            // Intermediate file read error
            else if (size == -1) {
                log.info("recover physics file end, " + mappedFile.getFileName());
                break;
            }
            // Come the end of the file, switch to the next file
            // Since the return 0 representatives met last hole, this can
            // not be included in truncate offset
            else if (size == 0) {  // 文件末尾
                index++;
                if (index >= mappedFiles.size()) { // 最后一个文件
                    // The current branch under normal circumstances should
                    // not happen
                    log.info("recover physics file over, last mapped file " + mappedFile.getFileName());
                    break; 
                } else { // 切换下一个文件进行恢复
                    mappedFile = mappedFiles.get(index);
                    byteBuffer = mappedFile.sliceByteBuffer();
                    processOffset = mappedFile.getFileFromOffset();
                    mappedFileOffset = 0;
                    log.info("recover next physics file, " + mappedFile.getFileName());
                }
            }
        }
		// 设置初始状态
        processOffset += mappedFileOffset;
        this.mappedFileQueue.setFlushedWhere(processOffset);
        this.mappedFileQueue.setCommittedWhere(processOffset);
        this.mappedFileQueue.truncateDirtyFiles(processOffset);

        // Clear ConsumeQueue redundant data
        this.defaultMessageStore.truncateDirtyLogicFiles(processOffset);
    }
    // Commitlog case files are deleted
    else {
        this.mappedFileQueue.setFlushedWhere(0);
        this.mappedFileQueue.setCommittedWhere(0);
        this.defaultMessageStore.destroyLogics();
    }
}
```



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
	// 通知刷盘线程进行刷盘操作
    handleDiskFlush(result, putMessageResult, msg);
    // 主从复制处理
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

### CommitLog Flush 流程

消息刷盘的过程是由 `FlushCommitLogService`的3GroupCommitService个子类完成的。

```java
if (FlushDiskType.SYNC_FLUSH == defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
    this.flushCommitLogService = new GroupCommitService();
} else {
    this.flushCommitLogService = new FlushRealTimeService();
}

this.commitLogService = new CommitRealTimeService();
```



#### CommitRealTimeService

CommitRealTimeService 该后台服务线程起作用的前提是`MappedFile`写入数据时使用了`WriteBuffer`，

也就是配置项`transientStorePoolEnable` 的值为true，并且刷盘模式是`ASYNC_FLUSH`， 而且是Master。

作用是：消息先写入DirectByteBuffer中，然后通过该后台服务线程写入到FileChannel （）。

```java
class CommitRealTimeService extends FlushCommitLogService {

    private long lastCommitTimestamp = 0;

    @Override
    public String getServiceName() {
        return CommitRealTimeService.class.getSimpleName();
    }

    @Override
    public void run() {
        CommitLog.log.info(this.getServiceName() + " service started");
        while (!this.isStopped()) {
            // 默认值200ms, 只有在启用了 TransientStorePool 的时候使用
            // 刷新数据到FileChannel
            int interval = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitIntervalCommitLog();
			// 默认的页数 4页
            int commitDataLeastPages = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitCommitLogLeastPages();
			// 默认值 200ms
            int commitDataThoroughInterval =
                CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitCommitLogThoroughInterval();

            long begin = System.currentTimeMillis();
            if (begin >= (this.lastCommitTimestamp + commitDataThoroughInterval)) {
                this.lastCommitTimestamp = begin;
                commitDataLeastPages = 0;
            }

            try {
                boolean result = CommitLog.this.mappedFileQueue.commit(commitDataLeastPages);
                long end = System.currentTimeMillis();
                if (!result) {
                    this.lastCommitTimestamp = end; // result = false means some data committed.
                    //now wake up flush thread.
                    flushCommitLogService.wakeup();
                }

                if (end - begin > 500) {
                    log.info("Commit data to file costs {} ms", end - begin);
                }
                this.waitForRunning(interval);
            } catch (Throwable e) {
                CommitLog.log.error(this.getServiceName() + " service has exception. ", e);
            }
        }

        boolean result = false;
        for (int i = 0; i < RETRY_TIMES_OVER && !result; i++) {
            result = CommitLog.this.mappedFileQueue.commit(0);
            CommitLog.log.info(this.getServiceName() + " service shutdown, retry " + (i + 1) + " times " + (result ? "OK" : "Not OK"));
        }
        CommitLog.log.info(this.getServiceName() + " service end");
    }
}
```



#### GroupCommitService

如果配置了同步刷盘模式，则使用`GroupCommitService`

```java
class GroupCommitService extends FlushCommitLogService {
    private volatile List<GroupCommitRequest> requestsWrite = new ArrayList<GroupCommitRequest>();
    private volatile List<GroupCommitRequest> requestsRead = new ArrayList<GroupCommitRequest>();
	
    // 提交同步刷盘请求
    public synchronized void putRequest(final GroupCommitRequest request) {
        synchronized (this.requestsWrite) {
            this.requestsWrite.add(request);
        }
        if (hasNotified.compareAndSet(false, true)) {
            waitPoint.countDown(); // notify
        }
    }

    private void swapRequests() {
        List<GroupCommitRequest> tmp = this.requestsWrite;
        this.requestsWrite = this.requestsRead;
        this.requestsRead = tmp;
    }

    private void doCommit() {
        synchronized (this.requestsRead) {
            if (!this.requestsRead.isEmpty()) {
                for (GroupCommitRequest req : this.requestsRead) {
                    // There may be a message in the next file, so a maximum of
                    // two times the flush
                    boolean flushOK = false;
                    for (int i = 0; i < 2 && !flushOK; i++) {
                        // 判断是否已经刷盘了
                        flushOK = CommitLog.this.mappedFileQueue.getFlushedWhere() >= req.getNextOffset();
                        if (!flushOK) {
                            // 刷盘页数为0表示直接调用force方法
                            // 不再判断commited 和 write直接的差值
                            CommitLog.this.mappedFileQueue.flush(0);
                        }
                    }
					// 唤醒等待的消息写入线程
                    req.wakeupCustomer(flushOK);
                }

                long storeTimestamp = CommitLog.this.mappedFileQueue.getStoreTimestamp();
                if (storeTimestamp > 0) {
                    CommitLog.this.defaultMessageStore.getStoreCheckpoint().setPhysicMsgTimestamp(storeTimestamp);
                }

                this.requestsRead.clear();
            } else {
                // Because of individual messages is set to not sync flush, it
                // will come to this process
                CommitLog.this.mappedFileQueue.flush(0);
            }
        }
    }

    public void run() {
        CommitLog.log.info(this.getServiceName() + " service started");

        while (!this.isStopped()) {
            try {
                // 没有被通知刷盘，则等待10ms
                this.waitForRunning(10);
                // 执行刷盘操作
                this.doCommit();
            } catch (Exception e) {
                CommitLog.log.warn(this.getServiceName() + " service has exception. ", e);
            }
        }
		// 优雅停机时，执行下面的操作
        // Under normal circumstances shutdown, wait for the arrival of the
        // request, and then flush
        try {
            Thread.sleep(10);
        } catch (InterruptedException e) {
            CommitLog.log.warn("GroupCommitService Exception, ", e);
        }

        synchronized (this) {
            this.swapRequests();
        }

        this.doCommit();

        CommitLog.log.info(this.getServiceName() + " service end");
    }

    @Override
    protected void onWaitEnd() {
        this.swapRequests();
    }

    @Override
    public String getServiceName() {
        return GroupCommitService.class.getSimpleName();
    }

    @Override
    public long getJointime() {
        return 1000 * 60 * 5;
    }
}
```



#### FlushRealTimeService

如果配置了异步刷盘模式，则使用`FlushRealTimeService`。

```java
class FlushRealTimeService extends FlushCommitLogService {
    private long lastFlushTimestamp = 0;
    private long printTimes = 0;

    public void run() {
        CommitLog.log.info(this.getServiceName() + " service started");

        while (!this.isStopped()) {
            //是否定时进行CommitLog的刷盘操作, 默认是False (也就是说默认同步刷盘)
            boolean flushCommitLogTimed = CommitLog.this.defaultMessageStore.getMessageStoreConfig().isFlushCommitLogTimed();
			// 刷盘的时间间隔， 默认是 500 毫秒
            int interval = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getFlushIntervalCommitLog();
            // 每次刷盘至少需要刷盘的页数，默认是4
            int flushPhysicQueueLeastPages = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getFlushCommitLogLeastPages();
			// 默认是 10 * 1000
            int flushPhysicQueueThoroughInterval =
 CommitLog.this.defaultMessageStore.getMessageStoreConfig().getFlushCommitLogThoroughInterval();

            boolean printFlushProgress = false;

            // Print flush progress
            long currentTimeMillis = System.currentTimeMillis();
            if (currentTimeMillis >= (this.lastFlushTimestamp + flushPhysicQueueThoroughInterval)) {
                this.lastFlushTimestamp = currentTimeMillis;
                flushPhysicQueueLeastPages = 0;
                printFlushProgress = (printTimes++ % 10) == 0;
            }

            try {
                // 如果是异步刷盘，则休眠 500 毫秒后进行刷盘操作
                if (flushCommitLogTimed) {
                    Thread.sleep(interval);
                } else {
                    // 同步刷盘，则等待通知
                    // CommitLog 完成消息写入后，通过方法handleDiskFlush来唤醒刷盘线程
                    this.waitForRunning(interval);
                }
				// 输出刷盘信息， 源码输出日志被注释
                if (printFlushProgress) {
                    this.printFlushProgress();
                }

                long begin = System.currentTimeMillis();
                // 真正执行刷盘
                CommitLog.this.mappedFileQueue.flush(flushPhysicQueueLeastPages);
                long storeTimestamp = CommitLog.this.mappedFileQueue.getStoreTimestamp();
                if (storeTimestamp > 0) {
                    CommitLog.this.defaultMessageStore.getStoreCheckpoint().setPhysicMsgTimestamp(storeTimestamp);
                }
                long past = System.currentTimeMillis() - begin;
                if (past > 500) {
                    log.info("Flush data to disk costs {} ms", past);
                }
            } catch (Throwable e) {
                CommitLog.log.warn(this.getServiceName() + " service has exception. ", e);
                this.printFlushProgress();
            }
        }

        // Normal shutdown, to ensure that all the flush before exit
        boolean result = false;
        for (int i = 0; i < RETRY_TIMES_OVER && !result; i++) {
            result = CommitLog.this.mappedFileQueue.flush(0);
            CommitLog.log.info(this.getServiceName() + " service shutdown, retry " + (i + 1) + " times " + (result ? "OK" : "Not OK"));
        }

        this.printFlushProgress();

        CommitLog.log.info(this.getServiceName() + " service end");
    }

    @Override
    public String getServiceName() {
        return FlushRealTimeService.class.getSimpleName();
    }

    private void printFlushProgress() {
        // CommitLog.log.info("how much disk fall behind memory, "
        // + CommitLog.this.mappedFileQueue.howMuchFallBehind());
    }

    @Override
    public long getJointime() {
        return 1000 * 60 * 5;
    }
}
```



### 队列索引构建流程

所有消息都是先写入CommitLog文件中(不同的Topic消息也是在一个文件中)，然后异步构建消息队列 和 消息索引。

#### ReputMessageService

该类是一个后台服务线程。它在Broker启动时，从CommitLog文件指定的offset处开始读取消息，然后通过`CommitLogDispatcher`进行消息分发。

> 注意：Broker启动时应该从哪个位置读取消息进行分发是非常重要的。通常有2个选择：
>
> 1. 从CommitLog文件最大的可读数据点开始分发（可以是最后写入消息的位置 或 已经commit的消息位置）。
> 2. 从CommitLog确认点开放分发。（如果开启了 duplicationEnable = true 配置项）



DefaultMessageStore 在启动时的代码，计算reput的位置。

```java
if (this.getMessageStoreConfig().isDuplicationEnable()) {
    this.reputMessageService.setReputFromOffset(this.commitLog.getConfirmOffset());
} else {
    this.reputMessageService.setReputFromOffset(this.commitLog.getMaxOffset());
}
```





```java
// 只要reput的位置后有新写入的消息，则可以构建消费队里和消息索引
private boolean isCommitLogAvailable() {
	return this.reputFromOffset < DefaultMessageStore.this.commitLog.getMaxOffset();
}

private void doReput() {
    for (boolean doNext = true; this.isCommitLogAvailable() && doNext; ) {
		// 
        if (DefaultMessageStore.this.getMessageStoreConfig().isDuplicationEnable()
            && this.reputFromOffset >= DefaultMessageStore.this.getConfirmOffset()) {
            break;
        }

        SelectMappedBufferResult result = DefaultMessageStore.this.commitLog.getData(reputFromOffset);
        if (result != null) {
            try {
                this.reputFromOffset = result.getStartOffset();

                for (int readSize = 0; readSize < result.getSize() && doNext; ) {
                    DispatchRequest dispatchRequest =
                        DefaultMessageStore.this.commitLog.checkMessageAndReturnSize(result.getByteBuffer(), false, false);
                    int size = dispatchRequest.getMsgSize();

                    if (dispatchRequest.isSuccess()) {
                        if (size > 0) {
                            DefaultMessageStore.this.doDispatch(dispatchRequest);

                            if (BrokerRole.SLAVE != DefaultMessageStore.this.getMessageStoreConfig().getBrokerRole()
                                && DefaultMessageStore.this.brokerConfig.isLongPollingEnable()) {
                                DefaultMessageStore.this.messageArrivingListener.arriving(dispatchRequest.getTopic(),
                                                                                          dispatchRequest.getQueueId(), dispatchRequest.getConsumeQueueOffset() + 1,
                                                                                          dispatchRequest.getTagsCode(), dispatchRequest.getStoreTimestamp(),
                                                                                          dispatchRequest.getBitMap(), dispatchRequest.getPropertiesMap());
                            }

                            this.reputFromOffset += size;
                            readSize += size;
                            if (DefaultMessageStore.this.getMessageStoreConfig().getBrokerRole() == BrokerRole.SLAVE) {
                                DefaultMessageStore.this.storeStatsService
                                    .getSinglePutMessageTopicTimesTotal(dispatchRequest.getTopic()).incrementAndGet();
                                DefaultMessageStore.this.storeStatsService
                                    .getSinglePutMessageTopicSizeTotal(dispatchRequest.getTopic())
                                    .addAndGet(dispatchRequest.getMsgSize());
                            }
                        } else if (size == 0) {
                            this.reputFromOffset = DefaultMessageStore.this.commitLog.rollNextFile(this.reputFromOffset);
                            readSize = result.getSize();
                        }
                    } else if (!dispatchRequest.isSuccess()) {

                        if (size > 0) {
                            log.error("[BUG]read total count not equals msg total size. reputFromOffset={}", reputFromOffset);
                            this.reputFromOffset += size;
                        } else {
                            doNext = false;
                            if (DefaultMessageStore.this.brokerConfig.getBrokerId() == MixAll.MASTER_ID) {
                                log.error("[BUG]the master dispatch message to consume queue error, COMMITLOG OFFSET: {}",
                                          this.reputFromOffset);

                                this.reputFromOffset += result.getSize() - readSize;
                            }
                        }
                    }
                }
            } finally {
                result.release();
            }
        } else {
            doNext = false;
        }
    }
}
```





CommitLogDispatcherBuildConsumeQueue 此类用来构建队列。

```java
class CommitLogDispatcherBuildConsumeQueue implements CommitLogDispatcher {

    @Override
    public void dispatch(DispatchRequest request) {
        final int tranType = MessageSysFlag.getTransactionValue(request.getSysFlag());
        switch (tranType) {
            case MessageSysFlag.TRANSACTION_NOT_TYPE:
            case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
                // 从方法名称能看出，是保存消息位置信息
                // 队列中保存的是消息的offset，size, tag
                DefaultMessageStore.this.putMessagePositionInfo(request);
                break;
            case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
            case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
                break;
        }
    }
}
```

CommitLogDispatcherBuildIndex 此类用来构建消息索引。

```java
class CommitLogDispatcherBuildIndex implements CommitLogDispatcher {
     @Override
     public void dispatch(DispatchRequest request) {
         if (DefaultMessageStore.this.messageStoreConfig.isMessageIndexEnable()) {
             DefaultMessageStore.this.indexService.buildIndex(request);
         }
     }
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

## 相关知识总结

### JVM shutdownhook

JVM 在在shutdown时， 会响应如下两种事件：

1. 进程正常退出（最后一个非Daemon线程退出） 或 通过`System.exit(int status)`方法 或 这等价方法退出。
2. 用户中断进程(例如：Ctrl + C)， 或OS系统事件(用户退出， 系统关机)

shutdownhook 本质上是一个已经初始化，但是未启动的线程。

JVM 在进入shutdown 流程后，多个shutdownhook会并发执行。所有的shutdownhook执行完成后，如果开启了`finalization-on-exit`时，JVM 还会执行所有未被调用的`finalizers`。 

> 注意：如果是通过`System.exit`触发的shutdown流程，则在shutdown流程中，Daemon 和 非Daemon线程还在继续执行。
>
> shutdown流程一旦开启，则是不可逆的，除非通过调用`Runtime.halt`方法，该方法会强制关闭JVM。
>
> shutdown流程执行中，不能再添加或删除shutdownhook，否则会抛出IllegalStateException异常。
>
> shutdownhook 代码需要精心编写：例如是线程安全，不会导致死锁，不会有异常等。
>
> JVM 不能响应 kill -9 （SIGKILL）信号

 [JVM对于signal的处理及案例分析](https://www.jianshu.com/p/3cb9aacc26a2) 

### JNA 

在RocketMQ实现中，为了实现高性能，避免进程内存页被swap，它通过调用本地C库API来锁定内存页。这里面就涉及到了Java如何调用本地库的实现。在[JNA]( https://github.com/java-native-access/jna )之前，我们只能通过[JNI]( https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/jniTOC.html )进行跨语言的相互调用，非常的繁琐。幸亏有了JNA，使我们得以非常快捷的实现跨语言调用。

RocketMQ中的实现。

#### 定义本地调用接口

```java
public interface LibC extends Library {
    LibC INSTANCE = (LibC) Native.loadLibrary(Platform.isWindows() ? "msvcrt" : "c", LibC.class);

    int MADV_WILLNEED = 3;
    int MADV_DONTNEED = 4;

    int MCL_CURRENT = 1;
    int MCL_FUTURE = 2;
    int MCL_ONFAULT = 4;

    /* sync memory asynchronously */
    int MS_ASYNC = 0x0001;
    /* invalidate mappings & caches */
    int MS_INVALIDATE = 0x0002;
    /* synchronous memory sync */
    int MS_SYNC = 0x0004;

    int mlock(Pointer var1, NativeLong var2);

    int munlock(Pointer var1, NativeLong var2);

    int madvise(Pointer var1, NativeLong var2, int var3);

    Pointer memset(Pointer p, int v, long len);

    int mlockall(int flags);

    int msync(Pointer p, NativeLong length, int flags);
}
```

> 本地库接口一般继承`Library` ，在接口中定义你需要调用的本地API对应的Java方法。这里需要非常注意Jav、a中的数据类型和本地接口的数据类型匹配问题。此外还需要注意一定订单一个`public static`的变量
>
> 该变量就是你调用本地API的入口对象。



#### 内存锁定

有了和本地API进行交互的能力后，就可以方便的利用Java SDK中不具备，只能通过本地接口提供的功能了。

##### mlock

mlock 方法接受2个参数，一个是要锁定内存的起始地址，一个是锁定的大小。但是需要注意的是你锁定的页面当前页可能并不在物理内存中，因此你需要对要锁定的内存区域写入数据。RocketMQ就是在创建MappedFile并warm的时候写入0进行内存完全占用的。

内存锁定是危险操作，需要具有ROOT权限。

##### munlock

作用同mlock相反，用来取消内存锁定。

## 个人思考

1. OS_PAGE_SIZE 在刷盘时，是按照页大小4K进行的，如果操作系统(应用进程)启用了大页，是不是可以通过大页作为单位呢？

2. 刷盘时，最少页数的计算 ((write / OS_PAGE_SIZE) - (flush / OS_PAGE_SIZE)) >= flushLeastPages; 是不是可以通过 (write - flush) / OS_PAGE_SIZE >= flushLeastPages 呢？

3. 同步刷盘需要处理一个刷盘请求队列，队列里面有多个请求，当刷盘请求较密集时(还有线程切换原因)，可能导致多次调用force方法，但是每次刷盘的数据其实不会很多。现有的逻辑是每次等待10ms来达到类似MySQL redo log组提交的效果，其实 10ms 这个值可以通过 磁盘的IOPS 和 配置参数进行调节，这样刷盘效果可能更好。

4. 感觉消息持久化逻辑和InnoDB的redo log 刷盘逻辑很类型。

5. NUMA 架构下的优化

6. MappedFile wrotePosition, commitedPosition, flushedPosition。 

   1. wrotePosition 是写入到MappedByteBuffer的位置
   2. commitedPosition 是启用了TransientStorePool后，写入到DirectByteBuffer的位置。
   3. flushedPosition 是最后刷盘的位置
   4. wrotePosition  >= commitedPosition >= flushedPosition

   

## 参考资料

<https://www.jianshu.com/p/453c6e7ff81c>

 https://www.jianshu.com/p/ccdf6fc710b0 

 https://www.jianshu.com/p/6d0c118c17de 

 http://jm.taobao.org/2017/03/23/20170323/ 

 https://www.cnblogs.com/lanxuezaipiao/p/3635556.html 

 https://github.com/OpenHFT/Java-Thread-Affinity 

