---
layout: post
comments: true
title: raft协议简介
date: 2017-03-13 18:27:06
tags:
categories:
- 架构
---

### raft是什么？

raft是一个分布式一致性算法。

### raft 系统结点分类

- Follower
- Leader
- Candidate

### Leader选举

raft系统结点启动时的身份是：`Follower`。 如果Follower结点不能和Leader结点进行通信则会变为Candidate结点，然后Candidate结点请求其它结点进行投票。其它结点会以它们的投票结果进行响应。如果Candidate获取了大多数的投票则会变为Leader结点。该过程称为Leader选举。

<!-- more -->

在Raft中有两个超时时间来控制Leader结点的选举。第一个是`election timeout`, 该时间是指follower变为candidate前的等待时间。该时间的取值是`150ms 到 300ms`间的一个随机数。该`election timeout`时间后follower结点会变为candidate结点，然后发起新一轮的将自身选为leader的选择操作，它会请求其它结点进行投票。如果收到请求的结点在本次选举中还没有投票则它就会给该candidate进行投票，然后该结点重置它的`election timeout`。一旦candidate获取了大多数的投票，则它就会变为新的Leader结点。然后Leader结点就会向follower结点发送`Append Entries`消息， 该类型消息的发送频率由`heartbeat timeout`来指定。follower结点会响应每一个`Append Entries`消息。该选举动作会一直持续到某个follower结点停止接收`heartbeats`后变为candidate结束。需要大多数的投票数保证了只有一个节点能被选为Leader。如果同一时刻有两个节点变为`candidate`则会发生投票分裂。这两个结点会同时发起Leader选举，对每个follower结点来说选举Leader的消息总有一个会先于另一个收到。现在每个`candidate`结点都获取了同样的票数：两票，并且在此轮选举中不会再收到投票消息。此时会重新发起新的一轮选举，获取票数多的结点会变为Leader节点。

### 写入操作步骤

所有改变系统状态的操作都必须通过Leader结点。每个改变都会变为Leader结点日志中的一个`entry`。该日志`entry`还没有被提交，因此它不会改变结点的值。在提交这个`entry`前，Leader结点需要将该`entry`复制到所有的结点中。Leader结点会等大多数的`follower`结点写入该`entry`后，然后在自身提交该`entry`。在Leader自身成功提交了`entry`后，leader会通知`follower`结点该`entry`已经提交了，`follower`结点收到通知后提交自身的数据。 然后整个集群就处于一致状态了。该过程被称为`Log Replication`。

### 日志复制

一旦我们的集群发生了Leader选举，则我们需要将我们系统的所有改变复制到集群的所有节点中。该操作是同样用于心跳的`Append Entries`消息来完成的。步骤如下：

1. 客户端将修改操作发送给Leader节点
2. 该操作会被添加到Leader节点的LOG中
3. 然后在下一个心跳时，将该修改操作发送给所有的follower节点
4. Leader节点`entry`日志的提交依赖于大多数follower结点确认收到该改变消息。
5. 当Leader节点收到大多数follower的确认消息后会给客户端进行响应

### 网络分区

Raft协议在网络发生分区后也能保持在一致性状态。因为网络分区，所以可能导致出现两个Leader节点。

当一个分区的Leader结点收到修改请求时，因为不能获取大多数follower结点的确认消息，则它本身的`entry`日志是处于`uncommitted`状态的。然而另一个分区的结点数占了整个未分区集群的大多数，对这样的集群中Leader结点的修改操作会成功执行。

当网络分区恢复后，结点数少的分区的Leader结点和follower节点会发现有更高Leader选举的发生，因此它们会回滚它们未提交的`entry`日志并匹配新的Leader结点的日志。这样整个集群的状态又会变为一致的状态。




