---
layout: post
comments: true
title: redis哨兵功能简介
date: 2016-10-17 10:28:08
tags:
- redis
categories:
- 缓存
---

### Redis Sentinel 简介

Redis Sentinel 是redis官方提供的redis高可用方案，它可以大幅度减少生产环境中人为干涉主从实例down机导致错误的次数。Redis Sentinel 主要有以下的功能：

 - **监控** Sentinel 会不断的检查主从实例是否正常工作。
 - **通知** Sentinel 可以通过API通知系统管理员或其它机器上的进程,告诉它们哪个被监控的进程有错误。
 - **自动容灾** 如果主进程不能按照预期进行工作,Sentinel会提升一个从进程变为主进程,如果有多个从实例，则其它的从实例会重新将最新的主进程作为master进程。并且使用redis服务的应用程序在和redis服务器建立连接时会得到新的master地址。
 - **配置提供者** Sentinel作为客户端发现redis服务地址的权威来源：客户端通过连接到Sentinels来发现指定服务当前的master进程地址。
 
<!-- more -->
 
### Sentinel的分布式特性

Redis Sentinel 是一个分布式系统:

使用单个sentinel进程来监控redis集群是不可靠的，当该sentinel进程宕掉后，如果集群发生错误，则整个集群就不能实现容灾了。所以有必要将sentinel集群，这样有几个好处：

 - 即使有一些sentinel进程宕掉了，依然可以进行redis集群的主备切换；
 - 如果有多个sentinel，redis的客户端可以随意地连接任意一个sentinel来获得关于redis集群中的信息。
 - 有多个sentinel进程，可以更精确的判断redis进程的健康状态，多个进程可以进行有效的投票表决。
 
### Sentinel版本选择

当前的sentinel被称为sentinel2, 最新的稳定版的sentinel跟随Redis 2.8 和 3.0发布。强烈建议使用2.8及以上版本。

### 运行 Sentinel

Sentinel 跟随redis一起安装的，可以在编译后的src目录下找到sentinel的可执行文件，按照如下方式启动：

    redis-sentinel /path/to/sentinel.conf 或
    redis-server /path/to/sentinel.conf --sentinel

上面两种方式的功能是一样的，必须指定配置文件，sentinel默认监听26379端口；由于该配置文件在运行的过程中会应该redis进程的主从切换而变化，所有需要保证启动进程的用户需要有该配置文件的写权限。

### Sentinel的配置文件解析

    sentinel monitor 138redis 10.13.128.138 6379 1
    sentinel down-after-milliseconds 138redis 60000
    sentinel config-epoch 138redis 0
    sentinel leader-epoch 138redis 0
    
    sentinel known-slave 138redis 10.13.128.137 6380
    sentinel monitor 137redis 10.13.128.137 6379 1
    sentinel down-after-milliseconds 137redis 10000
    sentinel parallel-syncs 137redis 5

上面是测试环境的一个sentinel的配置文件，下面解释一下配置项的含义：

    sentinel monitor 138redis 10.13.128.138 6379 2

上面的配置表示健康一个名为`138redis` 的主进程，ip为`10.13.128.138`, 端口为`6379`, 2表示sentinel集群中如果有2个进程认为该master已经死了，则整个集群就都认为该master死了。

除了监控master的配置，剩下的配置项格式都是统一的：

    sentinel <option_name> <master_name> <option_value>

 - down-after-milliseconds

> sentinel会向master发送心跳PING来确认master是否存活，如果master在“一定时间范围”内不回应PONG 或者是回复了一个错误消息，那么这个sentinel会主观地(单方面地)认为这个master已经不可用了(subjectively down, 也简称为SDOWN)。而这个down-after-milliseconds就是用来指定这个“一定时间范围”的，单位是毫秒。
 
 - parallel-syncs
 
> 在发生failover主备切换时，这个选项指定了最多可以有多少个slave同时对新的master进行同步，这个数字越小，完成failover所需的时间就越长，但是如果这个数字越大，就意味着越多的slave因为replication而不可用。可以通过将这个值设为 1 来保证每次只有一个slave处于不能处理命令请求的状态。

其他配置项在sentinel.conf中都有很详细的解释。
所有的配置都可以在运行时用命令SENTINEL SET command动态修改。

### Sentinel的“仲裁会”

前面我们谈到，当一个master被sentinel集群监控时，需要为它指定一个参数，这个参数指定了当需要判决master为不可用，并且进行failover时，所需要的sentinel数量，本文中我们暂时称这个参数为票数

不过，当failover主备切换真正被触发后，failover并不会马上进行，还需要sentinel中的大多数sentinel授权后才可以进行failover。
当ODOWN时，failover被触发。failover一旦被触发，尝试去进行failover的sentinel会去获得“大多数”sentinel的授权（如果票数比大多数还要大的时候，则询问更多的sentinel)
这个区别看起来很微妙，但是很容易理解和使用。例如，集群中有5个sentinel，票数被设置为2，当2个sentinel认为一个master已经不可用了以后，将会触发failover，但是，进行failover的那个sentinel必须先获得至少3个sentinel的授权才可以实行failover。
如果票数被设置为5，要达到ODOWN状态，必须所有5个sentinel都主观认为master为不可用，要进行failover，那么得获得所有5个sentinel的授权。

### 配置版本号

为什么要先获得大多数sentinel的认可时才能真正去执行failover呢？

当一个sentinel被授权后，它将会获得宕掉的master的一份最新配置版本号，当failover执行结束以后，这个版本号将会被用于最新的配置。因为大多数sentinel都已经知道该版本号已经被要执行failover的sentinel拿走了，所以其他的sentinel都不能再去使用这个版本号。这意味着，每次failover都会附带有一个独一无二的版本号。我们将会看到这样做的重要性。

而且，sentinel集群都遵守一个规则：如果sentinel A推荐sentinel B去执行failover，B会等待一段时间后，自行再次去对同一个master执行failover，这个等待的时间是通过failover-timeout配置项去配置的。从这个规则可以看出，sentinel集群中的sentinel不会再同一时刻并发去failover同一个master，第一个进行failover的sentinel如果失败了，另外一个将会在一定时间内进行重新进行failover，以此类推。

redis sentinel保证了活跃性：如果大多数sentinel能够互相通信，最终将会有一个被授权去进行failover.
redis sentinel也保证了安全性：每个试图去failover同一个master的sentinel都会得到一个独一无二的版本号。

### 配置传播

一旦一个sentinel成功地对一个master进行了failover，它将会把关于master的最新配置通过广播形式通知其它sentinel，其它的sentinel则更新对应master的配置。

一个faiover要想被成功实行，sentinel必须能够向选为master的slave发送SLAVE OF NO ONE命令，然后能够通过INFO命令看到新master的配置信息。

当将一个slave选举为master并发送SLAVE OF NO ONE`后，即使其它的slave还没针对新master重新配置自己，failover也被认为是成功了的，然后所有sentinels将会发布新的配置信息。

新配在集群中相互传播的方式，就是为什么我们需要当一个sentinel进行failover时必须被授权一个版本号的原因。

每个sentinel使用##发布/订阅##的方式持续地传播master的配置版本信息，配置传播的##发布/订阅##管道是：__sentinel__:hello。

因为每一个配置都有一个版本号，所以以版本号最大的那个为标准。

举个栗子：假设有一个名为mymaster的地址为192.168.1.50:6379。一开始，集群中所有的sentinel都知道这个地址，于是为mymaster的配置打上版本号1。一段时候后mymaster死了，有一个sentinel被授权用版本号2对其进行failover。如果failover成功了，假设地址改为了192.168.1.50:9000，此时配置的版本号为2，进行failover的sentinel会将新配置广播给其他的sentinel，由于其他sentinel维护的版本号为1，发现新配置的版本号为2时，版本号变大了，说明配置更新了，于是就会采用最新的版本号为2的配置。

这意味着sentinel集群保证了第二种活跃性：一个能够互相通信的sentinel集群最终会采用版本号最高且相同的配置。

### SDOWN和ODOWN的更多细节

sentinel对于不可用有两种不同的看法，一个叫主观不可用(SDOWN),另外一个叫客观不可用(ODOWN)。SDOWN是sentinel自己主观上检测到的关于master的状态，ODOWN需要一定数量的sentinel达成一致意见才能认为一个master客观上已经宕掉，各个sentinel之间通过命令SENTINEL is_master_down_by_addr来获得其它sentinel对master的检测结果。

从sentinel的角度来看，如果发送了PING心跳后，在一定时间内没有收到合法的回复，就达到了SDOWN的条件。这个时间在配置中通过is-master-down-after-milliseconds参数配置。

当sentinel发送PING后，以下回复之一都被认为是合法的：

PING replied with +PONG.
PING replied with -LOADING error.
PING replied with -MASTERDOWN error.
其它任何回复（或者根本没有回复）都是不合法的。

从SDOWN切换到ODOWN不需要任何一致性算法，只需要一个gossip协议：如果一个sentinel收到了足够多的sentinel发来消息告诉它某个master已经down掉了，SDOWN状态就会变成ODOWN状态。如果之后master可用了，这个状态就会相应地被清理掉。

正如之前已经解释过了，真正进行failover需要一个授权的过程，但是所有的failover都开始于一个ODOWN状态。

ODOWN状态只适用于master，对于不是master的redis节点sentinel之间不需要任何协商，slaves和sentinel不会有ODOWN状态。

### Sentinel之间和Slaves之间的自动发现机制

虽然sentinel集群中各个sentinel都互相连接彼此来检查对方的可用性以及互相发送消息。但是你不用在任何一个sentinel配置任何其它的sentinel的节点。因为sentinel利用了master的发布/订阅机制去自动发现其它也监控了统一master的sentinel节点。

通过向名为__sentinel__:hello的管道中发送消息来实现。

同样，你也不需要在sentinel中配置某个master的所有slave的地址，sentinel会通过询问master来得到这些slave的地址的。

每个sentinel通过向每个master和slave的发布/订阅频道__sentinel__:hello每秒发送一次消息，来宣布它的存在。
每个sentinel也订阅了每个master和slave的频道__sentinel__:hello的内容，来发现未知的sentinel，当检测到了新的sentinel，则将其加入到自身维护的master监控列表中。
每个sentinel发送的消息中也包含了其当前维护的最新的master配置。如果某个sentinel发现
自己的配置版本低于接收到的配置版本，则会用新的配置更新自己的master配置。

在为一个master添加一个新的sentinel前，sentinel总是检查是否已经有sentinel与新的sentinel的进程号或者是地址是一样的。如果是那样，这个sentinel将会被删除，而把新的sentinel添加上去。

### 网络隔离时的一致性

redis sentinel集群的配置的一致性模型为最终一致性，集群中每个sentinel最终都会采用最高版本的配置。然而，在实际的应用环境中，有三个不同的角色会与sentinel打交道：

Redis实例.
Sentinel实例.
客户端.
为了考察整个系统的行为我们必须同时考虑到这三个角色。

下面有个简单的例子，有三个主机，每个主机分别运行一个redis和一个sentinel:

             +-------------+
             | Sentinel 1  | <--- Client A
             | Redis 1 (M) |
             +-------------+
                         |
                         |
     +-------------+     |                     +------------+
     | Sentinel 2  |-----+-- / partition / ----| Sentinel 3 | <--- Client B
     | Redis 2 (S) |                           | Redis 3 (M)|
     +-------------+                           +------------+
     
在这个系统中，初始状态下redis3是master, redis1和redis2是slave。之后redis3所在的主机网络不可用了，sentinel1和sentinel2启动了failover并把redis1选举为master。

Sentinel集群的特性保证了sentinel1和sentinel2得到了关于master的最新配置。但是sentinel3依然持着的是就的配置，因为它与外界隔离了。

当网络恢复以后，我们知道sentinel3将会更新它的配置。但是，如果客户端所连接的master被网络隔离，会发生什么呢？

客户端将依然可以向redis3写数据，但是当网络恢复后，redis3就会变成redis的一个slave，那么，在网络隔离期间，客户端向redis3写的数据将会丢失。

也许你不会希望这个场景发生：

如果你把redis当做缓存来使用，那么你也许能容忍这部分数据的丢失。
但如果你把redis当做一个存储系统来使用，你也许就无法容忍这部分数据的丢失了。
因为redis采用的是异步复制，在这样的场景下，没有办法避免数据的丢失。然而，你可以通过以下配置来配置redis3和redis1，使得数据不会丢失。

min-slaves-to-write 1
min-slaves-max-lag 10
通过上面的配置，当一个redis是master时，如果它不能向至少一个slave写数据(上面的min-slaves-to-write指定了slave的数量)，它将会拒绝接受客户端的写请求。由于复制是异步的，master无法向slave写数据意味着slave要么断开连接了，要么不在指定时间内向master发送同步数据的请求了(上面的min-slaves-max-lag指定了这个时间)。

### Sentinel状态持久化

snetinel的状态会被持久化地写入sentinel的配置文件中。每次当收到一个新的配置时，或者新创建一个配置时，配置会被持久化到硬盘中，并带上配置的版本戳。这意味着，可以安全的停止和重启sentinel进程。

    

                    