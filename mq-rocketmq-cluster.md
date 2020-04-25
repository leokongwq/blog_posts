---
layout: post
comments: true
title: rocketmq集群搭建
date: 2017-01-12 18:21:26
tags:
- MQ
categories:
- MQ
---

上篇[rocketmq快速入门](/2017/01/12/mq-rocketmq-quickstart.html)讲了rocketmq的单机部署和测试，这篇主要记录下RocketMQ的集群搭建。

<!-- more -->

在正式部署之前我们需要明白几个概念

1. `Name Server`是一个几乎无状态节点,可集群部署,节点之间无任何信息同步
2. `Broker`分为Master与Slave,一个 Master 可以对应多个Slave,但是一个 Slave 只能对应一个 Master,Master与Slave的对应关系通过指定相同的`BrokerName`,不同的`BrokerId`来定义,BrokerId 为 0 表示 Master,非 0 表示 Slave。Master 也可以部署多个。每个 Broker 与 Name Server 集群中的所有节点建立长连接,定时注册Topic信息到所有`Name Server`

### 启动NameServer

为了避免NameServer的单点问题，在一个集群中至少需要2个节点。因为是单机模拟集群，所以我本地就先启动一个节点。

#### NameServer的配置

命令

    ./mqnamesrv -h
    
输出信息如下：

```
-c,--configFile <arg>    指定NameServer的配置文件
-h,--help                打印帮助信息
-n,--namesrvAddr <arg>   NameServer地址信息列表，如：192.168.0.1:9876;192.168.0.2:9876
-p,--printConfigItem     打印所有的配置项
```

NameServer有如下的配置项(这些都是默认值`./mqnamesrv -p`命令的输出)：

rocketmqHome=/usr/local/rocketmq
kvConfigPath=/Users/zhangyanghong/namesrv/kvConfig.json
productEnvName=center
clusterTest=false
orderMessageEnable=false
listenPort=9876
serverWorkerThreads=8
serverCallbackExecutorThreads=0
serverSelectorThreads=3
serverOnewaySemaphoreValue=256
serverAsyncSemaphoreValue=64
serverChannelMaxIdleTimeSeconds=120
serverSocketSndBufSize=4096
serverSocketRcvBufSize=4096
serverPooledByteBufAllocatorEnable=true
useEpollNativeSelector=false

#### 启动NameServer

如果不需要更改默认的配置项，则可以通过如下命令启动：

    ./mqnamesrv -c 
    
注意该启动方式仅仅用了开发环境。生产环境还是需要使用如下的命令启动

    ./mqnamesrv -c configFilePath -n nameServerAddrList
        
### 启动Broker集群

#### Broker的配置

命令

    ./mqbroker -h
    
输出信息如下：    

```
-c,--configFile <arg>       Broker 配置文件
-h,--help                   打印帮助信息
-m,--printImportantConfig   打印重要的配置项信息
-n,--namesrvAddr <arg>      NameServer地址列表192.168.0.1:9876;192.168.0.2:9876
-p,--printConfigItem        打印所有的配置项
```

Broker所有配置项如下：

rocketmqHome=/usr/local/rocketmq
namesrvAddr=
brokerIP1=172.18.48.79
brokerIP2=172.18.48.79
brokerName=jiexiu'Mac
brokerClusterName=DefaultCluster
brokerId=0
brokerPermission=6
defaultTopicQueueNums=8
autoCreateTopicEnable=true
clusterTopicEnable=true
brokerTopicEnable=true
autoCreateSubscriptionGroup=true
messageStorePlugIn=
sendMessageThreadPoolNums=32
pullMessageThreadPoolNums=24
adminBrokerThreadPoolNums=16
clientManageThreadPoolNums=16
flushConsumerOffsetInterval=5000
flushConsumerOffsetHistoryInterval=60000
rejectTransactionMessage=false
fetchNamesrvAddrByAddressServer=false
sendThreadPoolQueueCapacity=10000
pullThreadPoolQueueCapacity=10000
filterServerNums=0
longPollingEnable=true
shortPollingTimeMills=1000
notifyConsumerIdsChangedEnable=true
highSpeedMode=false
commercialEnable=true
commercialTimerCount=1
commercialTransCount=1
commercialBigCount=1
transferMsgByHeap=true
maxDelayTime=40
regionId=DefaultRegion
registerBrokerTimeoutMills=6000
slaveReadEnable=false
disableConsumeIfConsumerReadSlowly=false
consumerFallbehindThreshold=0
waitTimeMillsInSendQueue=200
startAcceptSendRequestTimeStamp=0
listenPort=10911
serverWorkerThreads=8
serverCallbackExecutorThreads=0
serverSelectorThreads=3
serverOnewaySemaphoreValue=256
serverAsyncSemaphoreValue=64
serverChannelMaxIdleTimeSeconds=120
serverSocketSndBufSize=131072
serverSocketRcvBufSize=131072
serverPooledByteBufAllocatorEnable=true
useEpollNativeSelector=false
clientWorkerThreads=4
clientCallbackExecutorThreads=4
clientOnewaySemaphoreValue=65535
clientAsyncSemaphoreValue=65535
connectTimeoutMillis=3000
channelNotActiveInterval=60000
clientChannelMaxIdleTimeSeconds=120
clientSocketSndBufSize=131072
clientSocketRcvBufSize=131072
clientPooledByteBufAllocatorEnable=false
clientCloseSocketIfTimeout=false
storePathRootDir=/Users/zhangyanghong/store
storePathCommitLog=/Users/zhangyanghong/store/commitlog
mapedFileSizeCommitLog=1073741824
mapedFileSizeConsumeQueue=6000000
flushIntervalCommitLog=1000
flushCommitLogTimed=false
flushIntervalConsumeQueue=1000
cleanResourceInterval=10000
deleteCommitLogFilesInterval=100
deleteConsumeQueueFilesInterval=100
destroyMapedFileIntervalForcibly=120000
redeleteHangedFileInterval=120000
deleteWhen=04
diskMaxUsedSpaceRatio=75
fileReservedTime=72
putMsgIndexHightWater=600000
maxMessageSize=4194304
checkCRCOnRecover=true
flushCommitLogLeastPages=4
flushLeastPagesWhenWarmMapedFile=4096
flushConsumeQueueLeastPages=2
flushCommitLogThoroughInterval=10000
flushConsumeQueueThoroughInterval=60000
maxTransferBytesOnMessageInMemory=262144
maxTransferCountOnMessageInMemory=32
maxTransferBytesOnMessageInDisk=65536
maxTransferCountOnMessageInDisk=8
accessMessageInMemoryMaxRatio=40
messageIndexEnable=true
maxHashSlotNum=5000000
maxIndexNum=20000000
maxMsgsNumBatch=64
messageIndexSafe=false
haListenPort=10912
haSendHeartbeatInterval=5000
haHousekeepingInterval=20000
haTransferBatchSize=32768
haMasterAddress=
haSlaveFallbehindMax=268435456
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
syncFlushTimeout=5000
messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
flushDelayOffsetInterval=10000
cleanFileForciblyEnable=true
warmMapedFileEnable=false
offsetCheckInSlave=false
debugLockEnable=false
duplicationEnable=false
diskFallRecorded=true
osPageCacheBusyTimeOutMills=1000
defaultQueryMaxNum=32

其中重要的配置信息如下：

namesrvAddr=
brokerIP1=172.18.48.79
brokerName=jiexiu'Mac
brokerClusterName=DefaultCluster
brokerId=0
autoCreateTopicEnable=true
autoCreateSubscriptionGroup=true
rejectTransactionMessage=false
fetchNamesrvAddrByAddressServer=false
storePathRootDir=/Users/zhangyanghong/store
storePathCommitLog=/Users/zhangyanghong/store/commitlog
flushIntervalCommitLog=1000
flushCommitLogTimed=false
deleteWhen=04
fileReservedTime=72
maxTransferBytesOnMessageInMemory=262144
maxTransferCountOnMessageInMemory=32
maxTransferBytesOnMessageInDisk=65536
maxTransferCountOnMessageInDisk=8
accessMessageInMemoryMaxRatio=40
messageIndexEnable=true
messageIndexSafe=false
haMasterAddress=
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
cleanFileForciblyEnable=true

#### Broker 默认提供的配置文件

在RocketMQ的安装目录的conf子目录下有下面几个文件夹：

    2m-2s-async
    2m-2s-sync
    2m-noslave

2m-2s-async 里面存放的是搭建两个Master两个Slave并异步进行数据的集群。

2m-2s-sync 里面存放的是搭建两个Master两个Slave并同步进行数据的集群。

2m-noslave 里面存放的是搭建两个Master没有Slave的集群。

下面我们就以2m-noslave里面的配置文件搭建集群。

首先看下这个目录下的两个文件中的内容

broker-a.properties

    namesrvAddr=127.0.0.1:9876
    brokerClusterName=DefaultCluster
    brokerName=broker-a
    brokerId=0
    deleteWhen=04
    fileReservedTime=48
    brokerRole=ASYNC_MASTER
    flushDiskType=ASYNC_FLUSH

broker-b.properties

    namesrvAddr=127.0.0.1:9876
    brokerClusterName=DefaultCluster
    brokerName=broker-b
    brokerId=0
    deleteWhen=04
    fileReservedTime=48
    brokerRole=ASYNC_MASTER
    flushDiskType=ASYNC_FLUSH
    #该配置是我自己添加的，因为是本机，否则会端口冲突,这里需要注意的是有一个配置 项haListenPort=10912， 也不能和该端口冲突
    listenPort=12911

这两个文件就表明通过这2个配置文件启动的两个节点同属于一个集群`DefaultCluster`, 每个节点都是Master， 异步同步数据，异步刷盘。

#### 启动节点

./mqbroker -c ../conf/2m-noslave/broker-a.properties &

./mqbroker -c ../conf/2m-noslave/broker-b.properties &    

### 验证

执行如下命令进行集群验证：

    /mqadmin clusterList -n 127.0.0.1:9876

输出信息如下：

```
#Cluster Name     #Broker Name            #BID  #Addr                  #Version                #InTPS(LOAD)       #OutTPS(LOAD) #PCWait(ms) #Hour #SPACE
DefaultCluster    broker-a                0     172.18.48.79:10911     V3_5_8                   0.00(0,0ms)         0.00(0,0ms)          0 412299.47 0.5476
DefaultCluster    broker-b                0     172.18.48.79:12911     V3_5_8                   0.00(0,0ms)         0.00(0,0ms)          0 412299.47 0.5476
```

### 总结

总体来说搭建集群还算不复杂。重要的是需要将这些信息自动化。应该需要一个自动化的脚本或运维系统将变化的信息控制起来。手动运维一个大的集群肯定是不合适的。




