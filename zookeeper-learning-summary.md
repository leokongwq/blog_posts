---
layout: post
comments: true
title: Zookeeper简介
date: 2018-05-24 20:46:53
tags:
- zookeeper
categories:
- 架构
---

### zookeeper 是什么？

官方解释如下

> ZooKeeper是一个开源的，专门服务于分布式应用的分布式协调服务。 它提供了一组简单的`原语`，分布式应用程序可以利用这些`原语`来实现更高级别的服务，以实现同步，配置维护以及组和命名。 它被设计为易于编程，并使用类似大家所熟悉的文件系统目录树结构的数据模型。

众所周知，协调服务很难正确实现。 它们特别容易出现诸如竞态条件和死锁等错误。 ZooKeeper背后的动机是减轻分布式应用程序从头开始实施协调服务的责任。

<!-- more -->

### 设计目标

#### 简单

ZooKeeper允许分布式进程通过与标准文件系统组织相似的共享分层命名空间相互协调。 命名空间由称为znode的数据节点组成，按ZooKeeper的说法 - 这些节点类似于文件和目录。 与典型的专为存储功能而设计的文件系统不同，ZooKeeper中的数据保存在内存中，这意味着ZooKeeper可以实现高吞吐量和低延迟。

ZooKeeper的实现关注与高性能，高可用性，和严格有序的访问。 ZooKeeper的高性能意味着它可以用于大型分布式系统。 可靠性使它不会造成单点故障。 严格的顺序意味着可以在客户端实现复杂的同步原语。

#### 多副本 

像ZooKeeper所协调的分布式进程一样，ZooKeeper本身也倾向于部署多个实例，组成一个整体。

{% asset_img zkservice.jpg %}

组成ZooKeeper服务的服务器必须全都彼此了解。它们保持状态的内存映像，以及持久存储中的事务日志和快照。 只要大部分服务器都可用，ZooKeeper服务将可用。

客户端只会连接到整个集群中的一台ZooKeeper服务器。 客户端会维护一个TCP连接，通过该连接发送请求，获取响应，获取监视事件并发送心跳。 如果到服务器的TCP连接中断，则客户端将连接到不同的服务器。

#### 顺序性

ZooKeeper使用一个反映所有ZooKeeper事务顺序的数字来标记每个更新。 后续操作可以使用该顺序来实现更高级别的抽象，例如同步原语。

#### 高性能

zookeeper在读多写少的场景下速度非常快。ZooKeeper 服务可以运行在数千台机器上，在读取操作远多于写操作的场景下性能表现很好。读的性能可能10倍于写性能

#### 数据模型和继承结构的命名空间

ZooKeeper提供的名称空间非常类似于标准文件系统。 名称是由斜线`/`分隔的一系列路径元素。 ZooKeeper名称空间中的每个节点都由一个路径标识。

{% asset_img zknamespace.jpg %}

#### 节点和临时节点

与标准文件系统不同，ZooKeeper命名空间中的每个节点都可以拥有与其相关的数据。 这就像有一个文件系统，允许一个节点既是文件也是目录。 （ZooKeeper被设计用于存储协调数据：状态信息，配置，位置信息等，因此存储在每个节点的数据通常很小，在字节到千字节范围内。）我们使用术语znode来说明我们正在谈论ZooKeeper数据节点。

Znodes维护一个统计结构，包括数据更改的版本号，ACL更改和时间戳，以允许缓存验证和协调更新。 每次znode的数据更改时，版本号都会增加。 例如，每当客户端检索数据时，客户端也会收到数据的版本。

存储在名称空间中每个节点上的数据都是以原子方式读取和写入的。 读取获取与znode关联的所有数据字节，写入将替换所有数据。 每个节点都有一个访问控制列表（ACL），限制谁可以做什么。

ZooKeeper也有临时节点的概念。 只要创建znode的会话处于活动状态，这些znode就会存在。 当会话结束时，znode被删除。 当你想实现[tbd]时，临时节点很有用。

#### 条件更新和观察者

ZooKeeper支持观察者的概念。 客户可以在znode上设置观察者。 当znode改变时，观察者将被`触发并移除`。 当观察者被触发时，客户端会收到一个数据包，说明znode已经改变。 如果客户端和其中一个ZooKeeper服务器之间的连接中断，客户端将收到本地通知。 这些可用于[tbd]。

#### 保证

ZooKeeper非常快速且非常简单。 由于其目标是构建更复杂的服务（如同步）的基础，因此它提供了一组保证。 这些是：

- 顺序一致性  来自客户端的更新将按照它们发送的顺序被执行。
- 原子性  更新或者成功或失败，不存在中间状态
- 单系统映像  无论客户端连接到哪个服务器，客户端都会看到相同的服务视图。
- 可靠性 一旦更新被成功执行，则该更新结果会从该时刻被持久化，直到客户端下一次更新它为止。
- 及时性 - 系统的客户视图保证在一定的时间范围内保持最新状态。


#### 简单的 API

Zookeeper的一个设计目标就是提供非常简单的编程接口。因此，它只提供了下面的操作：

##### create

创建一个节点

##### delete

删除一个节点

##### exists

判断一个节点是否存在

##### get data

获取节点的数据

##### set data

设置节点的数据

#####  get children

获取节点的子节点列表

##### sync

等待数据传播

#### 实现

ZooKeeper组件显示了ZooKeeper服务的高级组件。 除请求处理器外，构成ZooKeeper服务的每个服务器都复制其各个组件的副本。

{% asset_img zkcomponents.jpg %}

副本集数据库是一个包含整个数据树的内存数据库。更新操作会被写入磁盘以实现故障恢复，写操作会先序列化到磁盘，然后在内存数据库中执行。

每个ZooKeeper服务器都为客户提供服务。客户端连接至一台服务器以提交irequest。读请求的数据是从每个服务器数据库的本地副本获取。需改服务状态的请求，也就是写请求是通过一致性协议进行处理的。

作为一致性协议的一部分，所有来自客户端的写入请求都被转发到一台称为`Leader`的服务器。 ZooKeeper服务的其余节点服务器（称为follower）接收Leader发出的消息提议，并就消息传递达成一致。消息传递层负责在Leader宕机后选择新的Leader，并与Leader数据保存同步。

ZooKeeper使用自定义的原子消息传递协议`(ZAB)`。由于消息传递层是原子的，因此ZooKeeper可以保证本地数据副本永不过期。当Leader接收到一个写请求时，它会计算在写操作被执行时系统所处的状态，并将其转换成一个捕获这个新状态的事务

### zookeeper的一致性如何保证

### zookeeper如何选主

### zookeeper写数据的过程

### 参考资料

[zookeeperOver.html](http://zookeeper.apache.org/doc/r3.5.4-beta/zookeeperOver.html)









