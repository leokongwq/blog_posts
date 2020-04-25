---
layout: post
comments: true
title: kafka入门教程
date: 2017-02-06 22:25:30
tags:
- kafka
categories:
- MQ
---

本文翻译自：apache_kafka_tutorial.pdf

### kafka背景

Kafka最初是由LinkedIn开发，并于2011年成为Apache的一个开源项目，随后在2012变为Apache的一个顶级项目。kafka是由Scala和Java语言编写的的。Kafka是一个基于发布-订阅，具有容错的消息系统。具有高性能，可扩展，分布式等特性。

<!-- more -->

### kafka介绍

在大数据领域中，我们会使用了大量的数据。 关于大数据，我们有两个主要挑战。第一个挑战是如何收集大量的数据，第二个挑战是分析收集到的数据。为了克服这些挑战，你需要一个消息系统。Kafka的设计目标就是称为一个分布式，高吞吐量的消息系统。Kafka也可以作为一个传统的消息中间件的替代品。 与其他消息系统相比，Kafka具有更好的吞吐量，内置分区，复制和固有的容错能力，这使得它非常适合大规模消息处理应用程序。

#### 什么是消息系统？

消息系统负责将数据从一个应用程序传输到另一个应用程序，因此应用程序可以专注于数据，但不用关心消息如何共享。 分布式消息传递基于可靠消息队列的概念。 消息在客户端应用程序和消息传递系统之间异步排队。 有两种类型的消息模式可用：一种是点对点，另一种是发布-订阅（pub-sub）消息系统。 大多数消息模式遵循pub-sub。

##### 点对点消息系统
在点对点消息系统中，消息被保留在队列中。 一个或多个消费者可以消费队列中的消息，但是特定消息最多只能由一个消费者消费。 一旦消费者读取队列中的消息，它就从该队列中消失。 该系统的典型示例是订单处理系统，其中每个订单将由一个订单处理器处理，但多个订单处理器也可以同时工作。 下图描述了结构。

{% asset_img point-to-point-model.jpeg %}

##### 发布-订阅消息系统
在发布-订阅系统中，消息被保留在一个主题中。与点对点系统不同，消费者可以订阅一个或多个主题并使用该主题中的所有消息。在发布-订阅系统中，消息生产者称为发布者，消息使用者称为订阅者。 一个现实生活的例子是Dish电视，它发布不同的渠道，如运动，电影，音乐等，任何人都可以订阅自己的频道集，并在这些订阅的频道可用时获得它们。

{% asset_img pub-sub-model.jpeg %}

#### kafka是什么？

Apache Kafka是一个分布式发布 - 订阅消息系统和一个强大的队列，可以处理大量的数据，并使您能够将消息从一个端点传递到另一个端点。 Kafka适合离线和在线消息消费。 Kafka消息保留在磁盘上，并在群集内复制以防止数据丢失。 Kafka构建在ZooKeeper同步服务之上。 它与Apache Storm和Spark非常好地集成，用于实时流式数据分析。

##### 优点
以下是Kafka的几个优点：
可靠性 - Kafka 是一个分布式，消息分区的，拥有消息复制和容错的系统。扩展性 - Kafka消息系统可以实现在线扩展可用性 - Kafka 使用 "分布式的commit log"，这也意味着消息会尽可能快的持久化到磁盘。高性能 - Kafka 针对消息发送和消费都有很高的吞吐量。 即使保存了TB级别的数据性能也不会下降。

Kafka 非常快并保证零宕机和零消息丢失。##### 用例kafka通常用于下面的使用场景：- 监控    
Kafka通常用于监控数据的操作。 这涉及聚合来自分布式应用程序的统计信息，以产生集中化的操作数据。
- 日志聚合方案 
kafka可以用来收集跨组织的多个服务的日志，并将这些日志转为统一的格式供消费者使用。- 流式处理
流行的实时计算框架，如`Storm`和`Spark`流式读取`topic`中的数据进行处理，然后将处理的结果写入新的，用户或应用需要使用的`topic`中。Kafka强大的持久性功能在流式处理上下文中也是非常有用的。

#### need for Kafka

Kafka是一个统一的平台，用于处理所有实时数据Feed。 Kafka支持低延迟消息传递，并在出现机器故障时提供对容错的保证。它具有处理大量不同消费者的能力。 Kafka非常快，执行2百万w/s。 Kafka将所有数据保存到磁盘，这实质上意味着所有写入都会进入操作系统（RAM）的页面缓存。 这使得将数据从页面缓存传输到网络套接字非常有效。

###  Kafka基础概念

在深入了解Kafka之前，你应该先了解一些基本的术语。像`topics`,`brokers`,`producers`,`consumers`。下图说明了主要的术语并且表格对这些术语进行了详细的解释。

{% asset_img kafka-foundamental.jpeg %}

在上图中一个`topic`被配置为拥有3个分区，分区1包含两个偏移因子0和1。分区2包含4个偏移因子0, 1, 2 和 3。分区3包含1个偏移因子0。 分区副本的id和该副本所在的broker的id一致。

假设，如果一个`topic`配置的副本因子为3，则kafka会针对该`topic`的每个分区创建3个独立的副本并将这些副本尽量均匀分散到集群的每个节点上。为了在集群的节点间进行负载，每一个`broker`都会保存一个或多个这样的分区。多个`producer`和`consumer`可以同时发布或获取消息。

|Components | Description |
|-----|----|
|topics  | 隶属于特定分类的消息流称为`topic`。数据保存在topic中||partitions | `topics`会被切分为分区。针对每一个主题，kafka最少保持一个分区。每一个这样的分区以顺序不可变的方式保存消息。一个分区有一个或多个大小相同的`segment`文件组成。`Topics`拥有多个分区，因此可以保存大量的数据 | | Partition offset | 每个分区中的消息拥有一个唯一的序列id,被称为`offset` || Replicas of partition | 分区副本仅仅是分区的备份，不会对副本分区进行读写操作，只是用来防止数据丢失。|| Brokers | 1. Brokers 是维护发布消息的系统。每个broker针对每个topic可能包含0个或多个该topic的分区。假设，一个topic拥有N个分区，并且集群拥有N个broker,则每个broker会负责一个分区。2. 假设，一个topic拥有N个分区，并且集群拥有N+M个broker,则前N个broker每个处理一个分区，剩余的M个broker则不会处理任何分区 3。 假设，一个topic拥有N个分区，并且集群拥有M个broker（M < N），则这些分区会在所有的broker中进行均匀分配。每个broker可能会处理一个或多个分区。这种场景不推荐使用，因为会导致热点问题和负载不均衡问题。|
| Kafka Cluster | 由多个broker组成的kafka被称为kafka集群。一个kafaka集群在不停机扩展。集群负载所有消息的持久化和副本处理。|| Producers | Producers 是向一个或多个Kafka中topic发布消息的发布者。Producers 将消息发送到 Kafka 的 brokers中。任意时刻 producer 发布到broker中的消息都会被追加到某个分区的最后一个segment文件的最后。 Producer 也可以选择消息发送到指定的分区 || Consumers | Consumers 从broker读取数据。Consumers 订阅一个或多个 topic，并通过pull方式从broker拉取订阅的数据。 || Leader | `Leader`是负责某个分区数据读写操作的节点。每个分区都有一个`leader`|| Follower | 跟随`leader`操作的节点被称为follower。如果leader节点不可用，则会从所有的fellower中挑选一个作为新的leader节点。一个follower节点作为leader节点一个普通的消费者，拉取leader数据并更新自己的数据存储。|

### Kafka集群架构

看看下面的插图。 它显示了Kafka的集群工作原理。

{% asset_img kafka-cluster.jpeg %}

下面的表格描述了在上图中提到的每个组件的详细信息。

|Components|Description|
|---|---|
|Broker| Kafka集群通常使用多个Broker来实现集群的负载均衡。 Kafka brokers 是无状态的，因为它们使用 ZooKeeper 来保持它们的集群信息。 单个Kafka Broker 每秒可以处理数十万的读写请求，即使保存了TB级的数据也不会影响性能。Kafka broker de leader  选举是通过Zookeeper实现的。|
|ZooKeeper| ZooKeeper是用来管理和协调Kafka broker 的。ZooKeeper 服务主要用来通知 producer 和 consumer 关于任何新加入Kafka集群或某个Kafka Broker宕机退出集群的消息。 根据收到Zookeeper的关于Broker的存在或失败的消息通知，然后生产者和消费者采取决定，并开始与其它Broker协调它们的任务。|
|Producers | producer将数据推送给Broker。 当新Broker启动时，所有生产者搜索它并自动发送消息到该新Broker。 Kafka Producer不等待来自Broker的确认，并以Broker可以处理的速度发送消息。|
|Consumers | 由于 Kafka brokers 是无状态的， 因此需要Consumer来维护根据`partition offset`已经消费的消息数量信息。 如果 consumer 确认了一个指定消息的`offset`，那也就意味着 consumer 已经消费了该`offset`之前的所有消息。Consumer可以向Broker异步发起一个拉取消息的请求来缓存待消费的消息。consumers 也可以通过提供一个指定的`offset`值来回溯或跳过Partition中的消息。Consumer 消费消息的`offset`值是保存在ZooKeeper中的。|

### Kafka工作流程

到目前为止，我们讨论了Kafka的核心该概念。现在让我们将目光转向Kafka的工作流程上。Kafka是由分裂为一个或多个partition的topic的集合。 Kafka中的partition可以认为是消息的线性排序序列，其中每个消息由它们的索引（称为offset）来标识。 Kafka集群中的所有数据是每个partition数据分区的并集。 新写入的消息写在分区的末尾，消息由消费者顺序读取。通过将消息复制到不同的Broker来提供持久性。Kafka以快速，可靠，持久，容错和零停机的方式提供基于`pub-sub`和队列模型的消息系统。 在这两种情况下，生产者只需将消息发送到topic，消费者可以根据自己的需要选择任何一种类型的消息传递系统。 让我们按照下一节中的步骤来了解消费者如何选择他们选择的消息系统。

#### Pub-Sub 消息模型工作流程下面是 Pub-Sub 消息模型的工作流程

- 生产者定期向topic发送消息。- Kafka broker 根据配置将topic的消息存储到指定的partition上。Kafka确保所有的消息均匀分布在topic的所有partition上。如果producer发送了两条消息，并且该topic有两个partition，则每个partition会有一条消息。- Consumer 订阅指定的topic。- 一旦消费者订阅了topic，Kafka将向消费者提供topic的当前`offset`，并且还将`offset`保存在Zookeeper中。- 消费者将定期请求Kafka（如100 Ms）新消息。- 一旦Kafka从生产者接收到消息，它将这些消息转发给消费者。- 消费者将收到消息并进行处理。- 一旦消息被处理，消费者将向Kafka broker发送确认。- 一旦Kafka收到确认，它将`offset`更改为新值，并在Zookeeper中更新它。 由于`offset`在Zookeeper中被维护，消费者可以正确地读取下一条消息，即使服务器宕机后重启。- 以上流程将重复，直到消费者停止请求。
- 消费者可以随时回退/跳转到某个topic的期望`offset`处，并读取所有后续消息。

#### 队列消息模型工作流程 & Consumer Group

在基于队列的消息系统中，取代单个消费者的是订阅了相同topic的一群拥有相同`Group ID`的消费者集群。简单来说，订阅具有相同“组ID”的主题的消费者被认为是单个组，并且消息在它们之间共享。 让我们检查这个系统的实际工作流程。

- 生产者定期向topic发送消息。- Kafka broker 根据配置将topic的消息存储到指定的partition上。
- 单个consumer以名为`Group-1`的`Group ID` 订阅名为`Topic-01`的topic。- Kafka 会以和Pub-Sub消息模型相同的方式和consumer进行交互知道新的消费者以同样的Group ID加入到消费者分组中。- 一旦新的消费者加入后，Kafka将操作切换到共享模式，将所有topic的消息在两个消费者间进行均衡消费。这种共享行为直到加入的消费者结点数目达到该topic的分区数。- 一旦消费者的数目大于topic的分区数，则新的消费者不会收到任何消息直到已经存在的消费者取消订阅。出现这种情况是因为Kafka中的每个消费者将被分配至少一个分区，并且一旦所有分区被分配给现有消费者，新消费者将必须等待。该功能被称为 "Consumer Group"。以同样的方式，Kafka将以非常简单和高效的方式提供这两种系统功能。
  
#### ZooKeeper的角色

Apache Kafka的一个关键依赖是Apache Zookeeper，它是一个分布式配置和同步服务。 Zookeeper是Kafka代理和消费者之间的协调接口。 Kafka服务器通过Zookeeper集群共享信息。 Kafka在Zookeeper中存储基本元数据，例如关于主题，代理，消费者偏移（队列读取器）等的信息。由于所有关键信息存储在Zookeeper中，并且它通常在其整个集群上复制此数据，因此Zookeeper的故障不会影响Kafka集群的状态。一旦Zookeeper重新启动， Kafka将恢复状态。 这为Kafka带来了零停机时间。 Kafka代理之间的领导者选举也通过使用Zookeeper在领导失败的情况下完成。
要了解更多关于Zookeeper，请参考[http://www.tutorialspoint.com/zookeeper/](http://www.tutorialspoint.com/zookeeper/)

### Kafka安装步骤 

下面是在你机器上安装Java的步骤：
#### 步骤 1: 校验Java是否安装

但愿你的机器上已经安装了Java环境，因此你可以通过下面的命令检查Java是否已经安装。

```
$ java -version
```

如果你的机器上已经安装了Java，则你会看到Java的版本信息。##### 步骤 1.1: 下载 JDK
如果你还没有下载JDK，则可以通过下面的链接下载最新版本的JDK。[http://www.oracle.com/technetwork/java/javase/downloads/index.html](http://www.oracle.com/technetwork/java/javase/downloads/index.html)目前最新的JDK版本是`JDK 8u 60` 文件名为： “jdk-8u60-linux-x64.tar.gz”。##### 步骤 1.2: 解压文件
通常，正在下载的文件存储在下载文件夹中，使用下面的命令进行文件校验和解压。

```
$ cd /go/to/download/path$ tar -zxf jdk-8u60-linux-x64.gz```
#### 步骤 1.3: 进入Opt文件夹
为了让Java对所有人都是可用的，需要将解压后的内容移动到`/usr/local/java`文件夹下

```
$ supassword: (type password of root user) $ mkdir /opt/jdk$ mv jdk-1.8.0_60 /opt/jdk/```
#### 步骤 1.4: 设置path环境变量
可以同步下面的命令设置`path`和`JAVA_HOME`环境变量

    export JAVA_HOME =/usr/jdk/jdk-1.8.0_60    export PATH=$PATH:$JAVA_HOME/bin

是上面的配置生效：

```
$ source ~/.bashr
```

#### Step 1.5: Java AlternativesUse the following command to change Java Alternatives.

```
update-alternatives --install /usr/bin/java java /opt/jdk/jdk1.8.0_60/bin/java 100`
```

#### Step 1.6: Now verify java using verification command (java -version) explained in Step 1. 

#### 步骤 2: ZooKeeper 框架安装

##### 步骤 2.1: 下载 ZooKeeper
通过下面的链接下载zookeeper并安装[http://zookeeper.apache.org/releases.html](http://zookeeper.apache.org/releases.html)到目前为止最新的 ZooKeeper 版本是 3.4.6 (ZooKeeper-3.4.6.tar.gz).

##### 步骤 2.2: 解压 tar 文件
通过下面的命令解压文件：

```
$ cd opt/$ tar  -zxf  zookeeper-3.4.6.tar.gz$ cd zookeeper-3.4.6$ mkdir data
```

##### 步骤 2.3: 创建配置文件
用`vi`打开名为`conf/zoo.cfg`的配置文件并在配置文件中添加如下的内容：

```
$ vi conf/zoo.cfgtickTime=2000dataDir=/path/to/zookeeper/dataclientPort=2181initLimit=5syncLimit=2
```

完成配置文件的创建后你就可以启动Zookeeper服务器了。

##### 步骤 2.4: 启动 ZooKeeper Server

```
$ bin/zkServer.sh start
```

在该命令执行后你可以看到下面的反馈信息：

```
$ JMX enabled by default$ Using config: /Users/../zookeeper-3.4.6/bin/../conf/zoo.cfg $ Starting zookeeper ... STARTED
```

##### 步骤 2.5: 启动命令行工具

```
$ bin/zkCli.sh
```
在输入上面的命令后，你会连接到Zookeeper服务器并得到下面的响应信息：
```
Connecting to localhost:2181................................................Welcome to ZooKeeper!................................WATCHER::WatchedEvent state:SyncConnected type: None path:null [zk: localhost:2181(CONNECTED) 0]
```

##### 步骤 2.6: 停止 Zookeeper Server
在连接到Zookeeper并执行了一些操作后你可以通过下面的命令来停止Zookeeper。

    $ bin/zkServer.sh stop

现在你已经在你的机器上成功安装了Java和Zookeeper，接下来我们将安装Kafka。

#### 步骤 3: 安装Apache Kafka

##### 步骤 3.1: 下载 Kafka
点击下面的链接来下载kafka: 
[https://www.apache.org/dyn/closer.cgi?path=/kafka/0.9.0.0/kafka_2.11-0.9.0.0.tgz](https://www.apache.org/dyn/closer.cgi?path=/kafka/0.9.0.0/kafka_2.11-0.9.0.0.tgz)

##### 步骤 3.2: 解压tar文件
使用下面的命令解压下载的文件：

    $ cd opt/    $ tar  -zxf kafka_2.11.0.9.0.0 tar.gz    $ cd kafka_2.11.0.9.0.0
    
##### Step 3.3: 启动 Server
你可以通过下面的命令启动Server:

    $ bin/kafka-server-start.sh config/server.properties
    
在服务启动后，你回在屏幕上看到如下的信息：

    $ bin/kafka-server-start.sh config/server.properties    [2016-01-02 15:37:30,410] INFO KafkaConfig values:    request.timeout.ms = 30000    log.roll.hours = 168 inter.broker.protocol.version = 0.9.0.X    log.preallocate = false security.inter.broker.protocol = PLAINTEXT       ....................................................       ....................................................

##### 步骤 4: 停止 the Server

在执行完所有的操作后你可以通过下面的命令停止服务：

    $ bin/kafka-server-stop.sh config/server.properties
目前我们已经完成了Kafka的安装，接下来我们来看看kafka的基本操作。

### Kafka基本操作

首先让我们实现单个结点的Broker，随后看看如何搭建多结点的Kafka集群。
希望你已经安装了Java，Zookeeper, Kafka。在搭建Kafka集群前你应该先启动ZooKeeper ，因为Kafka集群依赖ZooKeeper。

#### Start ZooKeeper
打开一个新的终端并输入下面的命令：

    bin/zookeeper-server-start.sh config/zookeeper.properties
    输入下面的命令启动Kafka Server:

    bin/kafka-server-start.sh config/server.properties
    
在启动Kafka Broker后，在启动Zookeeper的终端窗口输入`jps`命令你会看到如下的输出：

    821 QuorumPeerMain    928 Kafka    931 Jps    
    
现在你应该可以在终端看到两个守护进程，QuorumPeerMain 是 ZooKeeper 另一个是Kafka。
#### 单个Broker结点配置

在这个配置中，你有一个ZooKeeper和一个Broker实例。 以下是配置步骤：
*创建Kafka主题*：Kafka提供了一个名为“kafka-topics.sh”的命令行实用程序来在服务器上创建主题。 打开新终端并键入以下示例。

*例子*

```shell
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic Hello-Kafka
```
    
我们刚才创建了一个名为`Hello-Kafka`,一个分区和一个副本的topic。上面命令的输出信息如下：

*输出*: `Created topic “Hello-Kafka”`


##### List of Topics为了获取Kafka服务器上的topic列表，你可以使用下面的命令：*例子*

```shell
bin/kafka-topics.sh --list --zookeeper localhost:2181```
    
*输出*

```shell
Hello-Kafka
```
    由于我们只创建了一个名为`Hello-Kafka`的topic，则只会有上面的输出。如果你创建了很多topic则你回获取更过topic名称列表。

##### 启动Producer发送消息

*语法*

```shell
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic topic-name
```

从上面的语法我们可以看到有两个重要参数是必须的：`Broker-list` - 指定我们要发送消息到的Broker列表。在这个例子中我们只有一个Broker。配置文件 `server.properties`中指定了Broker的端口号，由于我们已经指定了Broker的端口号所以我们可以直接指定；`topic-name`：我们发送消息所属的topic。

*例子*

```shell
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic Hello-Kafka
```

生产者将等待来自标准输入的输入并发布到Kafka集群。 默认情况下，每个新行都作为一条新消息发布，生产者使用配置文件`config/producer.properties`中指定缺省属性。 现在你可以在终端中键入几行消息，如下所示。

*输出*

```shell
$ bin/kafka-console-producer.sh --broker-list localhost:9092 --topic Hello-Kafka    [2016-01-16 13:50:45,931] WARN property topic is not valid (kafka.utils.Verifia- bleProperties)HelloMy first message
My second message
```

##### 启动消费者接收消息
和生产者类似，消费者默认的属性配置在配置文件`config/consumer.properties`文件中。开发一个新的终端，输入如下的命令来消费消息：

*语法*

```shelll
bin/kafka-console-consumer.sh --zookeeper localhost:2181 —topic topic-name --from- beginning
```
    
*例子*

```shelll
bin/kafka-console-consumer.sh --zookeeper localhost:2181 —topic Hello-Kafka --from- beginning    
```
    
*输出*    

```shelll
HelloMy first messageMy second message
```
    
最后，你可以从生产者的终端输入消息，并看到它们出现在消费者的终端。 到目前为止，你对具有单个broker的单节点群集有非常好的了解。 现在让我们继续讨论多个代理配置。    

#### 多节点配置

在搭建多结点Kafka节点集群时首先启动你的Zookeeper服务。
*创建多节点Kafka Broker* – 我们已经拥有了一个Kafka Broker实例，它使用的配置文件是`config/server.properties`中。 现在我们需要多个代理实例，因此将现有的`server.proties`文件复制到两个新的配置文件中，并将其重命名为`server-one.properties`和`server-two.properties`。 然后编辑这两个新文件并分配以下更改：*config/server-one.properties*

    # The id of the broker. This must be set to a unique integer for    each broker. 
    broker.id=1    # The port the socket server listens on    port=9093    # A comma seperated list of directories under which to store log files 
    log.dirs=/tmp/kafka-logs-1

*config/server-two.properties*

    # The id of the broker. This must be set to a unique integer for    each broker. 
    broker.id=2    # The port the socket server listens on    port=9094    # A comma seperated list of directories under which to store log files 
    log.dirs=/tmp/kafka-logs-2


*启动多个Broker* – 在完成上面的修改后，打开三个终端来启动三个Broker节点。

    Broker1    bin/kafka-server-start.sh config/server.properties    Broker2    bin/kafka-server-start.sh config/server-one.properties    Broker3    bin/kafka-server-start.sh config/server-two.properties

现在我们用了三个不同Broker结点在同一个机器上，你可以通过`jps`命令来校验，你会看到对应的输出。

##### 创建topic

让我们将副本因子设为3，因为我们拥有三个Broker实例，如果你拥有两个Broker则你设置为2。*语法*

```shell
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic topic-name
```
    
*例子*    

```shell
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 -partitions 1 --topic Multibrokerapplication
```

*输出*

```shell  
created topic “Multibrokerapplication”
```

`Describe` 命令用来检查指定topic的详细信息，如下：

```shell
bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic Multibrokerapplication
```
    
*输出*    

```shell
Topic:Multibrokerapplication PartitionCount:1 ReplicationFactor:3 Configs:
Topic:Multibrokerapplication Partition:0 Leader:0 Replicas:0,2,1 Isr:0,2,1 
```
    
从上面的输出，我们可以得出结论，第一行给出所有分区的摘要，显示topic名称，分区数量和我们已经选择的复制因子。 在第二行中，每个节点将是分区的随机选择的领导者。
在我们的例子中，我们看到我们的第一个broker（with broker.id 0）是领导者。 然后Replicas：0,2,1意味着所有的Broker已经完成topic的复制。`Isr`是`in-sync`副本的集合。这是副本的子集，当前活着的并和leader保持同步的结点。       

##### 启动生产者发送消息
这个生产者和前面提到的单个Broder结点中的生产者是一样的。

*例子*

```shell
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic Multibrokerapplication    
```
    
*输出*    

```shell
This is single node-multi broker demoThis is the second message
```

##### 启动消费者接收消息
这个消费者和前面提到的单个Broder结点中的消费者是一样的。

*例子*

```shell
bin/kafka-console-consumer.sh --zookeeper localhost:2181 —topic Multibrokerapplication --from-beginning
```
    
*输出*    

```shell
bin/kafka-console-consumer.sh --zookeeper localhost:2181 —topic Multibrokerapplication —from-beginningThis is single node-multi broker demoThis is the second message
```

#### 基本操作

该本章节我们来讨论Kafka的各种基本操作。##### 修改Topic
由于你已经在Kafka集群创建了topic，现在让我们修改已经创建的topic的信息。

*语法*

```shell
bin/kafka-topics.sh --zookeeper localhost:2181 --alter --topic topic_name --partitions count
```

*例子*

```shell
We have already created a topic “Hello-Kafka” with single partition count and one replica factor. Now using “alter” command we have changed the partition count.    
bin/kafka-topics.sh --zookeeper localhost:2181 --alter --topic Hello-kafka --parti- tions 2
```
    
*输出*    

```shell
WARNING: If partitions are increased for a topic that has a key, the partition logic or ordering of the messages will be affected Adding partitions succeeded!
```
    
##### 删除topic
可以通过下面的语法删除topic

*语法*

```shell
bin/kafka-topics.sh --zookeeper localhost:2181 --delete --topic topic_name
```
    
*例子*    

```shell
bin/kafka-topics.sh --zookeeper localhost:2181 --delete --topic Hello-kafka
```

*输出*

```shell
Topic Hello-kafka marked for deletion
```
     
**注意**： 该操作只有在Broker的配置文件中设置了选项`delete.topic.enable=true`才可以。

### Kafka–简单生产者示例

让我们使用Java客户端创建一个用于发布和使用消息的应用程序。 Kafka生产者客户端包括以下API。

#### KafkaProducer API

让我们了解本节中最重要的一组Kafka生产者API。 KafkaProducer API的中心部分是“KafkaProducer”类。 KafkaProducer类提供了一个选项，用于将构造函数中的Kafka代理连接到以下方法。

- KafkaProducer 类提供了`send`方法来异步发送消息，`send`方法的签名如下：
        producer.send(new ProducerRecord<byte[],byte[]>(topic, partition, key1, value1) , callback);
    - ProducerRecord - 生产者管理等待发送的记录的缓冲器。􏰂 
    - Callback-一个用户提供的回调对象，当消息被Broker确认后会回调该对象的回调方法。如果为null则意味着没有回调。
- KafkaProducer 类提供了一个`flush`方法来保证当前所有消息已经被真正的发送到Broker了。该方法的签名是：public void flush()
- KafkaProducer 类提供了一个`partitionFor`方法， 用来获取指定topic的分区原信息。这个可以被用来客户端自定义分区。该方法的签名如下：`public partitionsFor(string topic)`
- KafkaProducer 类提供了一个`metrics`方法，该方法返回一个由生产者维护的metrics信息的Map。该方法的签名是： `public Map metrics()`。
- public void close() – KafkaProducer 类提供的`close`方法会等所有的消息发送完成后关闭Producer。

#### Producer API

生产者API的中心部分是“生产者”类。 生产者类提供了一个选项，通过以下方法在其构造函数中连接Kafka代理。

##### The Producer Class
生产者类提供send方法，使用以下签名向单个或多个主题发送消息。

```java
public void send(KeyedMessage<k,v> message)public void send(List<KeyedMessage<k,v>> messages) 
Properties prop = new Properties();prop.put(producer.type,”async”)ProducerConfig config = new ProducerConfig(prop);
```
    
有两种类型的生产者-同步和异步。相同的API配置也适用于“同步”生产者。 它们之间的区别是同步生产者直接发送消息，但是在后台发送消息。 当您想要更高的吞吐量时，异步生成器是首选。 在以前的版本，如0.8，一个异步生产者没有回调send（）注册错误处理程序。 这仅在当前版本0.9中可用。    public void close()
Producer 类提供的`close`方法可以关闭生产者和Broker之间的连接池。

##### Configuration Settings    

下表中列出了Producer API的主要配置设置，以便更好地理解：

|配置项|说明|
|---|---|
|client.id | 标志生产者应用程序|
|producer.type | 可以是 sync 或 async |
|acks | 这个配置可以设定发送消息后是否需要Broker端返回确认，0: 不需要进行确认，速度最快。存在丢失数据的风险。1: 仅需要Leader进行确认，不需要ISR进行确认。是一种效率和安全折中的方式。all: 需要ISR中所有的Replica给予接收确认，速度最慢，安全性最高，但是由于ISR可能会缩小到仅包含一个Replica，所以设置参数为all并不能一定避免数据丢失。 |
|retries | 如果生产者发送消息时被后会通过该选择指定的值进行重试|
|bootstrap.servers | 生产者启动的Broker结点列表 |
|linger.ms| Producer默认会把两次发送时间间隔内收集到的所有Requests进行一次聚合然后再发送，以此提高吞吐量，而linger.ms则更进一步，这个参数为每次发送增加一些delay，以此来聚合更多的Message |
|key.serializer| 消息 Key 值序列化接口|
|value.serializer| 消息 value 的序列化接口|
|batch.size | Producer会尝试去把发往同一个Partition的多个Requests进行合并，batch.size指明了一次Batch合并后Requests总大小的上限。如果这个值设置的太小，可能会导致所有的Request都不进行Batch |
|buffer.memory| 在Producer端用来存放尚未发送出去的Message的缓冲区大小。缓冲区满了之后可以选择阻塞发送或抛出异常，由block.on.buffer.full的配置来决定 |

#### ProducerRecord API

ProducerRecord 是一个键值对，用来封装发送到Kafka集群消息。构造函数的签名如下：

```java
public ProducerRecord (string topic, int partition, k key, v value)
```
    
- Topic - 用户定义的Topic名称。
- Partition - 发送到那个Partition
- Key - 消息的Key值
- Value - 消息体数据

没有指定Partition的构造函数签名

```java
public ProducerRecord (string topic, k key, v value)*
```

- Topic - 用户定义的Topic名称
- Key - 消息的Key值
- Value - 消息体数据

只有topic名称和消息体的构造函数

```java
public ProducerRecord (string topic, v value)
```
- Topic - 用户定义的Topic名称
- Value - 消息体数据
    
ProducerRecord 类提供的方法如下：

|方法|说明|    
|---|---|
|public string topic()|Topic will append to the record.|
|public K key()|Key that will be included in the record. If no such key, null will be re- turned here.|
|public V value()|Record contents.|
|partition()|Partition count for the record|    

#### SimpleProducer 应用程序

在创建应用程序前，首先需要启动ZooKeeper和Kafka Broker并通过创建topic的命令创建指定的Topic。

```java
import java.util.Properties;
//import util.properties packages
import org.apache.kafka.clients.producer.Producer; //import simple producer packages
import org.apache.kafka.clients.producer.KafkaProducer; //import KafkaProducer packages
import org.apache.kafka.clients.producer.ProducerRecord; //import ProducerRecord packages
public class SimpleProducer {
    //Create java class named “SimpleProducer”
    public static void main(String[] args) throws Exception {
        // Check arguments length value
        if(args.length == 0) {
            System.out.println("Enter topic name”);
            return;
        }
        String topicName = args[0].toString();
        //Assign topicName to string variable
        Properties props = new Properties();
        // create instance for properties to access producer configs
        props.put("bootstrap.servers", "localhost:9092"); //Assign localhost id
        props.put("acks", "all");
        //Set acknowledgements for producer requests.
        props.put("retries", 0);
        //If the request fails, the producer can automatically retry,
        props.put("batch.size", 16384);
        //Specify buffer size in config
        props.put("linger.ms", 1);
        //Reduce the no of requests less than 0
        props.put("buffer.memory", 33554432);
        //The buffer.memory controls the total amount of memory available to the producer for buffering.
        props.put("key.serializer", "org.apache.kafka.common.serializa- tion.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serializa-tion.StringSerializer");
        Producer<String, String> producer = new KafkaProducer<String,String>(props);
        for(int i = 0; i < 10; i++){
            producer.send(new ProducerRecord<String, String>(topicName, Integer.toString(i), Integer.toString(i)));
            System.out.println("Message sent successfully");
        }
        producer.close();
    }
}
```
            
是用下面的命令编译应用程序：

```java
javac -cp “/path/to/kafka/kafka_2.11-0.9.0.0/lib/*” *.java
```

使用下面的命令执行应用程序：

```java
java -cp “/path/to/kafka/kafka_2.11-0.9.0.0/lib/*”:. SimpleProducer <topic-name>
```

输出：
```java
Message sent successfullyTo check the above output open new terminal and type Consumer CLI command to receive messages.>> bin/kafka-console-consumer.sh --zookeeper localhost:2181 —topic <topic-name> — from-beginning1 
2 
3 
4 
5 
6 
7 
8 
9 
10
```

#### 简单的 Consumer 例子

到目前为止，我们已经创建了一个发送消息到Kafka集群的生产者。 现在让我们创建一个消费者来消费Kafka集群中的消息。 KafkaConsumer API用于消费来自Kafka集群的消息。 KafkaConsumer类的构造函数定义如下。

```java
public KafkaConsumer(java.util.Map<java.lang.String,java.lang.Object> configs)
```

configs - 一个包含Consumer配置项的Map
KafkaConsumer 类提供下面表格所示的方法列表。

| 方法签名 | 描述 |
|:------- |:-------|
|public java.util.Set<TopicPartition> assignment() | 获取当前消费者对应的Partition信息|
|public string subscription()| 获取订阅的信息 |
|public void subscribe(java.util.List<java.lang.String> topics, ConsumerRebalanceListener listener) | 批量订阅Topic|
|public void unsubscribe() |取消指定的订阅|
|public void subscribe(java.util.List<java.lang.String> topics)|批量订阅，没有回调|
|public void subscribe(java.util.regex.Pattern pattern, | 按照正则匹配取消订阅|
|public void assign(java.util.List<TopicPartition> partitions)|手动指定消费者消费的分区信息|
|poll()| 获取订阅的topic消息|
|public void commitSync() | 同步更新最后获取的消息的offset信息|
|public void seek(TopicPartition partition, long offset)| 跳转到指定分区的指定偏移位置，下次从该分区的偏移位置获取消息|
|public void resume()| 恢复暂停的分区|
|public void wakeup()| 唤醒消费者|

##### ConsumerRecord API

The ConsumerRecord API is used to receive records from the Kafka cluster. This API consists of a topic name, partition number, from which the record is being received and an offset that points to the record in a Kafka partition. ConsumerRecord class is used to create a consumer record with specific topic name, partition count and <key, value> pairs. It has the following signature.

```java
public ConsumerRecord(string topic,int partition, long offset,K key, V value)
```

- Topic - The topic name for consumer record received from the Kafka cluster.
- 􏰁Partition - Partition for the topic.
- 􏰁Key - The key of the record, if no key exists null will be returned.
- Value - Record contents.

##### ConsumerRecords API
ConsumerRecords API acts as a container for ConsumerRecord. This API is used to keep the list of ConsumerRecord per partition for a particular topic. Its Constructor is defined below.

```java
public ConsumerRecords(java.util.Map<TopicPartition,java.util.List<Consumer- Record<K,V>>> records)
```

- TopicPartition - Return a map of partition for a particular topic.
- Records - Return list of ConsumerRecord.

ConsumerRecords class has the following methods defined.

|方法签名|说明|
|---|---|
|public int count()|The number of records for all the topics.|
|public Set partitions()|The set of partitions with data in this record set (if no data was returned then the set is empty).|
|public Iterator iterator()|Iterator enables you to cycle through a collection, obtaining or re- moving elements.|
|public List records()|Get list of records for the given partition.|

##### Configuration Settings

The configuration settings for the Consumer client API main configuration settings are listed below:

|配置项|说明|
|---|---|
|bootstrap.servers|Bootstrapping list of brokers.|
|group.id|Assigns an individual consumer to a group.|
|enable.auto.commit|enable.auto.commitEnable auto commit for offsets if the value is true, otherwise not committed.|
|auto.commit.interval.ms|Return how often updated consumed offsets are written to ZooKeeper.|
|session.timeout.ms|Indicates how many milliseconds Kafka will wait for the ZooKeeper to respond to a request (read or write) before giving up and continuing to consume messages.|

##### SimpleConsumer Application

The producer application steps remain the same here. First, start your ZooKeeper and Kafka broker. Then create a “SimpleConsumer” application with the java class named “SimpleCon- sumer.java” and type the following code.

```java
package com.leokongwq.kafka;

import java.util.Properties;
import java.util.Arrays;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.ConsumerRecord;

public class SimpleConsumer {
    public static void main(String[] args) {
        if(args.length == 0){
            System.out.println("Enter topic name");
            return;
        }
        String topicName = args[0].toString();
        Properties props = new Properties();
        //Kafka consumer configuration settings
        props.put("bootstrap.servers", "localhost:9092");
        props.put("group.id", "test");
        props.put("enable.auto.commit", "true");
        props.put("auto.commit.interval.ms", "1000");
        props.put("session.timeout.ms", "30000");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);
        consumer.subscribe(Arrays.asList(topicName));
        //KafkaConsumer subscribes list of topics here. System.out.println("Subscribed to topic " + topicName); //print the
        int i = 0;

        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(100);
            for (ConsumerRecord<String, String> record : records){
                // print the offset,key and value for the consumer records.
                System.out.printf("offset = %d, key = %s, value = %s\n", record.offset(), record.key(), record.value());
            }
        }
    }
}
```

编译：

```java
javac -cp “/path/to/kafka/kafka_2.11-0.9.0.0/lib/*” *.java
```

运行：

```java
java -cp “/path/to/kafka/kafka_2.11-0.9.0.0/lib/*”:. SimpleConsumer <topic-name>
```

*Input* – Open the producer CLI and send some messages to the topic. You can put the smple input as ‘Hello Consumer’.*Output* – Following will be the output.

```
Subscribed to topic Hello-Kafkaoffset = 3, key = null, value = Hello Consumer
```

### Kafka–ConsumerGroupExample


    


    
    

    

     
     


