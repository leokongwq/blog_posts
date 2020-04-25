---
layout: post
comments: true
title: consul 命令介绍
date: 2018-07-29 00:25:40
tags:
- consul
categories:
- 分布式
---

## consul 命令用法

安装完consul后，通过在控制台直接实现`consul`命令了解consul命令行的用法，输出如下：

```shell
Usage: consul [--version] [--help] <command> [<args>]

Available commands are:
agent          Runs a Consul agent
catalog        Interact with the catalog
event          Fire a new event
exec           Executes a command on Consul nodes
force-leave    Forces a member of the cluster to enter the "left" state
info           Provides debugging information for operators.
join           Tell Consul agent to join cluster
keygen         Generates a new encryption key
keyring        Manages gossip layer encryption keys
kv             Interact with the key-value store
leave          Gracefully leaves the Consul cluster and shuts down
lock           Execute a command holding a lock
maint          Controls node or service maintenance mode
members        Lists the members of a Consul cluster
monitor        Stream logs from a Consul agent
operator       Provides cluster-level tools for Consul operators
reload         Triggers the agent to reload configuration files
rtt            Estimates network round trip time between nodes
snapshot       Saves, restores and inspects snapshots of Consul server state
validate       Validate config files/directories
version        Prints the Consul version
watch          Watch for changes in Consul
```

如上所示， consul提供的命令很多，下面就逐个学习下每个命令的作用和用法。

<!-- more -->

## agent

`agent` 命令的作用是启动一个`consul` 进程， 该进程的具体作用和该命令的参数有关。

通过 `consul agent --help` 可以获取该命令的详细用法


### datacenter

该参数指定该`agent`所在的数据中心名称。格式如下：

```
-datacenter=<value>
```

### advertise

该参数用来设置`advertise`所使用的地址。默认值和`-bind`指定的地址一样

```
-advertise=<value>
```

### advertise-wan

设置广域网环境的的`advertise`地址。
     
```
-advertise-wan=<value>
```

### -bind

设置集群通信所使用的IP地址。

```
-bind=<value>
```

### bootstrap

将Server设置为`boostrap`模式。
     
```
-bootstrap
```

### bootstrap-expect

将Server设置为`expect boostrap`模式。
     
```
-bootstrap-expect=<value>
```

### client

设置客户端访问所使用的IP地址。 该地址可用于RPC, DNS,HTTP,HTTPS通信使用。
     
```
-client=<value>
```

### config-dir

设置配置文件所在的目录的路径。consul会读取该目录下所有以`.json`结尾的文件作为配置文件，并且以文件名字典序来应用这些配置文件。

该参数可以多次使用，来指定多个配置文件目录。
     
```
-config-dir=<value>
```

### config-file

指定配置文件路径，可以多次多次使用，用来指定多个配置文件。

```     
-config-file=<value>
```

### config-format

设置配置文件的格式。 `json`或`hcl`

```
-config-format=<value>
```

### data-dir

设置保存`agent`状态数据的目录。
     
```
-data-dir=<value>
```

### dev

该`agent`以开发模式运行。会输出详细的日子信息

```
-dev
```

### disable-host-node-id

该该选项设置为true, 则consul不会使用所在主机的信息生成集群结点id。 从而每次都生成一个随机值。
建议不要开始该选项。默认的结点id将会是主机名`hostname`

```
-disable-host-node-id
```

### disable-keyring-file

禁止将`keyring`备份到文件中。

```
-disable-keyring-file
```

### dns-port
     
设置DNS 查询所使用的端口。

```     
-dns-port=<value>
```

### domain

DNS 接口所使用的域名。

```
-domain=<value>
```

### enable-script-checks

启用健康检测脚本。

```
-enable-script-checks
```

### encrypt

提供`gossip`广播加密数据所使用的key.

```
-encrypt=<value>
```

### hcl

`hcl` 配置片段。 可以多次使用。

```
-hcl=<value>
```

### http-port

设置http API 服务监听的端口。

```
-http-port=<value>
```

### join

设置该`agent`启动时加入所在集群的成员IP地址。可以多次使用。

```
-join=<value>
```
    
### join-wan

设置该`agent`启动时加入所在跨IDC集群的成员IP地址。可以多次使用。

```
-join-wan=<value>
```

### log-level

设置`agent`的日志输出级别。

```
-log-level=<value>
```

### node

设置该结点在集群中的名称，必须唯一。

```
-node=<value>
```

### node-id

设置该结点在集群中永恒的ID，默认是一个随机生成的值，并保存的数据目录中。

```
-node-id=<value>
```

### node-meta

给该结点设置元数据。

```
-node-meta=<key:value>
```

### non-voting-server

该选项使该Server不参于Raft协议的选举流程，仅仅进行数据同步。这样角色的结点主要用来扩展consul集群的读能力。

```
-non-voting-server
```

### pid-file

指定保存pid的文件路径。

```
-pid-file=<value>
```

### protocol

设置协议版本。默认使用最新版本。

```
-protocol=<value>
```

### raft-protocol

设置Raft协议的版本。默认使用最新版本。

```
-raft-protocol=<value>
```

### recursor

上游DNS服务器的地址。可以多次使用，指定多个地址。

```
-recursor=<value>
```

### rejoin

忽略上次的离开，并尝试从新加入集群。

```
-rejoin
```

### retry-interval

每次尝试重试加入集群的重试时间间隔。

```
-retry-interval=<value>
```

### retry-interval-wan

同样用来指定加入集群的重试时间间隔，不过针对的广域网。

```
-retry-interval-wan=<value>
```

### retry-join

Address of an agent to join at start time with retries enabled. Can
be specified multiple times.

```
-retry-join=<value>
```

### retry-join-wan

Address of an agent to join -wan at start time with retries
enabled. Can be specified multiple times.

```
-retry-join-wan=<value>
```

### retry-max

设置重试的最大次数。默认值是`0`, 意味着一直进行尝试。

```
-retry-max=<value>
```

### retry-max-wan

设置重试的最大次数。针对的是广域网。默认值是`0`, 意味着一直进行尝试。

```
-retry-max-wan=<value>
```

### segment 

(企业级版本才有的功能) 设置加入的网络段。

```
-segment=<value>
```

### serf-lan-bind

Address to bind Serf LAN listeners to.

```
-serf-lan-bind=<value>
```

### serf-wan-bind

Address to bind Serf WAN listeners to.

```
-serf-wan-bind=<value>
```

### -server

以`server`模式启动，默认是`client`

### syslog

将日志输出到syslog.

```
-syslog
```

### ui

启用consul内置的静态wei UI服务器。

```
-ui
```

### ui-dir

指定包含web界面的资源目录。

```
-ui-dir=<value>
```

## catalog

此命令具有与Consul目录交互的子命令。该目录不应与代理混淆，尽管API和回应可能类似。

以下是一些简单示例，并提供了更详细的示例在子命令或文档中。

列出所有数据中心

```
$ consul catalog datacenters
```

列出所有结点

```
$ consul catalog nodes
```

列出所有服务

```
$ consul catalog services
```

## event

跨数据中心发布自定义用户事件。 必须提供事件名称，但事件的内容是可选的。 
  
支持通过正则表达式，节点名称，服务名称，或标签进行过滤。

### name

事件的名称

```
-name=<string>
```

### node

用来指定通过正则表达式过滤的节点名称。
     
```
-node=<string>
```

### service 

用来指定通过正则表达式过滤的服务名称。

```
-service=<string>
```

### tag

用来指定通过正则表达式过滤的服务的标签名称。必须和`-service`选项一起使用。

```
-tag=<string>
```

## exec

在远程Consul节点上执行命令。 节点的响应内容可以通过正则表达式进行过滤。

可用的选项如下：

```
-node=<string>

Regular expression to filter on node names.

-prefix=<string>

Prefix in the KV store to use for request data.

-service=<string>

Regular expression to filter on service instances.

-shell

Use a shell to run the command.

-tag=<string>

Regular expression to filter on service tags. Must be used with
-service.

-verbose

Enables verbose output.

-wait=<duration>

Period to wait with no responses before terminating execution.

-wait-repl=<duration>

Period to wait for replication before firing event. This is an
optimization to allow stale reads to be performed.
```

## leave

将一个结点优雅进入`leave`状态，并关闭。

## force-leave

强制一个集群的节点进入`left`状态。

## info

提供操作命令的调试信息。

## join 

使一个运行中的agent加入到集群中。

### wan

```
-wan
```

## keygen
  
生成一个新的加密用的key，用来供agent对网络流量就行加密。  

## keyring

管理用来加密`gossip`消息的秘钥。`Gossip`加密是可以选的。当启用`Gossip`加密，该命令可以用来检查集群中处于激活状态下的加密key，添加新的key，删除老的key。把这些功能结合起来，就可以实现不破坏集群的前提下，更新加密秘钥的功能。

该命令提供的所有操作都只能在`Server`模式的结点上执行， 并且影响范围包括`LAN`和`WAN`。

当所有结点正确返回时，该命令输出:0, 否则其他任何情况都输出1。

该命令的选项如下：

### install

安装一个新的秘钥，并广播到集群的所有结点中。秘钥的格式是base64的。
     
```
 -install=<string>
 ```
 
### list

列出所有正在集群中使用的秘钥。

```
-list
```
     
### relay-factor

Setting this to a non-zero value will cause nodes to relay their
response to the operation through this many randomly-chosen other
nodes in the cluster. The maximum allowed value is 5.
     
```
-relay-factor=<int>
```

### remove   

从集群中删除秘钥。 该操作只能针对哪些当前不是主秘钥的秘钥。
     
```
-remove=<string>
```
  
### use

Change the primary encryption key, which is used to encrypt
messages. The key must already be installed before this operation
can succeed.

```
-use=<string>
```
    
## kv

该命令包含一些操作Consul K-V 存储的子命令。

下面是一些简单的例子：

创建或更新一个名称为`redis/config/connections`，值为5的键值对。

```
$ consul kv put redis/config/connections 5
```

获取该键值对的值

```
$ consul kv get redis/config/connections
```

获取该键值对的详细信息

```
$ consul kv get -detailed redis/config/connections
```

删除该键值对

```
$ consul kv delete redis/config/connections
```

### 子命令

delete    Removes data from the KV store
export    Exports a tree from the KV store as JSON
get       Retrieves or lists data from the KV store
import    Imports a tree stored as JSON to the KV store
put       Sets or updates data in the KV store
    
## lock

Acquires a lock or semaphore at a given path, and invokes a child process
when successful. The child process can assume the lock is held while it
executes. If the lock is lost or communication is disrupted the child
process will be sent a SIGTERM signal and given time to gracefully exit.
After the grace period expires the process will be hard terminated.

For Consul agents on Windows, the child process is always hard terminated
with a SIGKILL, since Windows has no POSIX compatible notion for SIGTERM.

When -n=1, only a single lock holder or leader exists providing mutual
exclusion. Setting a higher value switches to a semaphore allowing multiple
holders to coordinate.

The prefix provided must have write privileges.
  
## maint

该命令的作用是将节点或服务置于维护模式。在维护模式下，无论通过DNS查询还是HTTP接口，都不再返回节点或服务的信息。命令有效地将其从可用池中取出节点。该命令的执行原理是通过注册节点或服务的健康检查来完成的。

为节点或服务启用维护模式时，你可以选择指定一个字符串来表明原因。该字符串将出现在“Notes”字段中
对节点或注册的重要健康检查服务。如果没有提供原因，将使用默认值。

维护模式是持久的，并且会在agent重启启动的时候恢复。 因此，在将给定节点或服务将被放回可以池中之前需要禁用维护模式。

默认情况下，我们将一个节点作为一个整体进行操作。通过指定`-service`参数，可以操作具体的服务。

如果没有给出参数，将显示agent的维护状态。如果当前没有任何维护，则返回空白。

  
### disable

禁用维护模式

```
-disable
```

### enable

启用维护模式

```
-enable
```

### reason

```
-reason=<string>
```

指定本次操作的原因描述信息

### service

指定操作的服务ID。

```
-service=<string>
```

## members

该命令的作用是：输出集群成员信息

可用的参数有：

### -detailed

输出更详细的节点信息。

```
consul members -detailed
Node   Address         Status  Tags
bogon  127.0.0.1:8301  alive   build=1.0.2:b55059f,dc=dc1,id=fc1742c4-d97b-8a5b-ff3c-ab11941e2fea,port=8300,raft_vsn=3,role=consul,segment=<all>,vsn=2,vsn_max=3,vsn_min=2,wan_join_port=8302
```

### segment

```
-segment=<string>
```

企业级版本才有的功能。 可以只输出所属segment的结点信息。

### status

```
-status=<string>
```

根据状态进行过滤需要输出的结点。

### wan

如果agent允许在server模式下，该选项可以输出其它在`WAN`广域网的节点信息。

```
-wan
```

## monitor

显示Consul agent 的最近日志消息，并连接到agent，实时输出agent上日志消息。 而且可以通过参数来指定需要查看的日志级别。默认输出的是DEBUG级别的日志。

## operator

该命令提供了Consul集群级别的操作能力， 在使用时需要非常小心。使用不当有可能造成，节点数据过期或数据丢失。

子命令如下：

### autopilot

`autopilot` 命令用于与Consul的`autopilot`子系统进行交互。 该命令可用于查看或修改当前配置。

#### get-config

显示当前的自动驾驶仪配置

```
consul operator autopilot get-config
CleanupDeadServers = true
LastContactThreshold = 200ms
MaxTrailingLogs = 250
ServerStabilizationTime = 10s
RedundancyZoneTag = ""
DisableUpgradeMigration = false
UpgradeVersionTag = ""
```

#### set-config

修改当前的自动驾驶仪配置

### raft

#### list-peers

Display the current Raft peer configuration

#### remove-peer 

Remove a Consul server from the Raft configuration

## reload

该命令的作用是重新加载配置文件，以此来替代`SIGHUP`信号。

## rtt

`rtt`命令的用法如下：

```
Usage: consul rtt [options] node1 [node2]
```

该命令的作用是：预估2个结点间一次通信的耗时状况的。至少需要提供一个节点名称。如果只提供了一个节点的名称，
则第二个结点的名称默认就是agent所在的结点名称。

> 需要注意的是：这些节点名称和`consul members`输出中的结点名称相同，不是一个IP地址。

默认情况下，都是假设2个节点在同一个数据中心内，并使用局域网的网络协调器。如果有`-wan`参数，那么使用格式如下：

```
consul rtt -wan bogon.dc1
Estimated bogon.dc1 <-> bogon.dc1 rtt: 0.020 ms (using WAN coordinates)
```

节点的名称后面需要添加所属数据中心的名称。

该命令不能用来测量局域网和广域网2个节点间的网络通信耗时，因为他们位于不同的`gossip`域。

该命令可以使用的参数如下：

### ca-file

指定TLS通信是使用的`CA`文件。 该选项的值也可以通过环境变量`CONSUL_CACERT`的值获取。
     
```
-ca-file=<value>
```
 
### ca-path
   
指定TLS通信是使用的`CA`文件所在的目录。 该选项的值也可以通过环境变量`CONSUL_CAPATH`的值获取。
      
```
-ca-path=<value>
```

### client-cert   

指定在使用TLS进行通信时，并且启用了`verify_incoming`功能时，客户端所使用的证书文件，
也可以通过环境变量`CONSUL_CLIENT_CERT` 来指定。

```
-client-cert=<value>
```
  
### client-key
  
指定在使用TLS进行通信时，并且启用了`verify_incoming`功能时，客户端所使用的key文件。也可以通过环境变量`CONSUL_CLIENT_KEY` 来指定。

```
-client-key=<value>
```
  
### http-addr

指定提供http服务的Consul agent 的IP地址和端口号。也可以通过环境变量`CONSUL_HTTP_ADDR`的值来指定。

默认的值是：`http://127.0.0.1:8500`。 也可以通过设置环境变量`CONSUL_HTTP_SSL=true`来启用https进行通信。

```
-http-addr=<address>
```

### tls-server-name

指定当通过TLS进行通信时作为`SNI`主机的服务器名称。也可以通过`CONSUL_TLS_SERVER_NAME`环境变量来指定。

```
-tls-server-name=<value>
```

### token

```
-token=<value>
```

指定请求中使用的ACL token的值。也可以通过环境变量`CONSUL_HTTP_TOKEN`来指定。如果没有指定，则使用通过http 访问Consul agent 所使用的token值。

### wan

```
-wan 
```

使用广域网协调器来替代局域网协调器。
     

## snapshot
  
该命令包含一些子命令，作用是保存，重新加载 或在故障恢复时检查 Consul Server 的状态。

这些操作都是原子性的，保存当前时间点 包括`键/值条目`，`服务目录`，`准备好的查询`，`会话和ACL`信息的快照。

如果启用了`ACLS`功能，那么在使用这些子命令时，需要提供管理token来进行快照的操作。
  
### 创建一个快照

  
```
$ consul snapshot save backup.snap
```      

### 从快照中恢复

```
$ consul snapshot restore backup.snap
```

先启动一个没有任何数据的agent,然后执行该命令，加载指定的快照信息。

### 检查快照信息

```
$ consul snapshot inspect backup.snap
```

输出：

```
leo@bogon consul snapshot inspect a.snap
ID           2-110-1532832078752
Size         1175
Index        110
Term         2
Version      1
```

可以允许一个Consul agent, 每小时保存一次快照信息。 不过该功能只有企业级版本才有。

## validate

校验consul配置文件，或配置文件目录下配置文件的正确性

## version

打印consul的版本

## watch



     







