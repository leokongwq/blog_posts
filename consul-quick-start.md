---
layout: post
comments: true
title: consul快速启动
date: 2017-12-31 09:25:12
tags:
- consul
categories:
---

### 前言

该文章翻译自 [https://www.consul.io/intro/index.html](https://www.consul.io/intro/index.html)

这篇文章的目标是对Consul进行一个简单的介绍。包括Consul是什么，它解决了什么问题，它有哪些功能。通过这篇文章能让你对Consul能有一个大致的了解，和基本的使用。

### What is Consul?

-------

Consul 有许多组件，但是作为一个整理来看，它为你的基础设施提供了服务发现和统一配置功能。它提供如下的主要的功能：

<!-- more -->

- 服务发现: Consul 的客户端可以在Consul上注册服务，如 API 或 mysql。并且其它的客户端可以通过Consul来发现这些服务。通过 DNS 或 HTTP接口，应用程序可以很容易的发现它们依赖的服务。
- 健康检查: Consul客户端可以提供任何数量的健康检查，或者与给定的服务相关联（“Web服务器返回200 OK”），或者与本地节点（“内存利用率低于90％”）相关联。 操作员可以使用此信息来监视群集运行状况，服务发现组件使用此信息将流量从不健康的主机中引导出去。
- KV 存储: 应用可以通过使用Consul提供的具有继承结构的 `Key/Value` 存储来实现任何需要的功能，包括动态配置， 特征标记，协调，领导选举等等。 Consul提供了简单的HTTP API是该功能非常容易使用。
- 多数据中心: Consul支持多个数据中心。 这意味着Consul的用户不必担心构建额外的抽象层以扩展到多个区域。

Consul设计的目标是对DevOps社区和应用程序开发人员友好，使其成为现代化，弹性基础架构的完美选择。

### Basic Architecture of Consul

-------

Consul是一个分布式，高可用的系统。本节将介绍基础知识，故意省略一些不必要的细节，以便你快速了解Consul的工作原理。有关更多详细信息，请参阅[深入了解Consul架构细节](https://www.consul.io/docs/internals/architecture.html)。

向Consul提供服务的每个节点都运行一个Consul Agent。运行一个Consul代理不是用来发现其他服务或操作键值对数据。不需要运行代理。该代理的目的是负责对该结点提供的服务和其本身进行健康检查。

该Consul Agent与一个或多个Consul Server进行通信。Consul Server是数据存储和复制的地方。服务器自己选出一位Leader。虽然Consul可以在一台服务器上运行，但建议部署3个或5个结点，以避免数据丢失故障的发生。每个数据中心都建议使用一组Consul服务器。

需要发现其它服务或节点的基础架构组件可以查询任何Consul服务器或任何Consul代理。代理自动将查询转发到服务器。

每个数据中心都运行Consul服务器集群。当发生跨数据中心服务发现或配置请求时，本地Consul服务器将请求转发到远程数据中心并返回结果。

### Installing Consul

-------

安装Consul非常简单。[下载](https://www.consul.io/downloads.html)目标平台的安装文件，解压或安装到指定目录，并添加到系统PATH中就可以了。

#### Verifying the Installation

安装完Consul收，在命令行执行`consul`，如果有如下的输出，则表示consul安装成功了。

```shell
$ consul
usage: consul [--version] [--help] <command> [<args>]

Available commands are:
    agent          Runs a Consul agent
    event          Fire a new event

# ...
```

### Run the Consul Agent

For more detail on bootstrapping a datacenter, see this guide.

Consul安装完成后，必须启动Consul Agent。该agent可以允许在`Server`模式和`Client`模式。 每一个数据中心至少需要一个Server，但是推荐一个Consul集群至少包含3个或5个Consul Server结点。只有一个Consul Server结点是不推荐的，因为很可能丢失数据，在开发环境可以这么做。

所有其它的agent运行在client模式。 一个client模式的agent是一个非常轻量级的进程，它会注册服务，执行监控检测，并将请求转发到Server。组成集群的每个结点都必须运行一个agent。

在一个数据中心搭建Consul集群可以参考[https://www.consul.io/docs/guides/bootstrapping.html](https://www.consul.io/docs/guides/bootstrapping.html)

#### Starting the Agent

-------

为了简单起见，我们现在将以开发模式启动Consul agent。 这种模式对于快速简单地启动单节点Consul环境非常有用。 它并不打算在生产中使用，因为它不会持续任何状态。

```shell
-$ consul agent -dev
==> Starting Consul agent...
==> Starting Consul agent RPC...
==> Consul agent running!
           Version: 'v0.7.0'
         Node name: 'Armons-MacBook-Air'
        Datacenter: 'dc1'
            Server: true (bootstrap: false)
       Client Addr: 127.0.0.1 (HTTP: 8500, HTTPS: -1, DNS: 8600, RPC: 8400)
      Cluster Addr: 127.0.0.1 (LAN: 8301, WAN: 8302)
    Gossip encrypt: false, RPC-TLS: false, TLS-Incoming: false
             Atlas: <disabled>

==> Log data will now stream in as it occurs:

2016/09/15 10:21:10 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:127.0.0.1:8300 Address:127.0.0.1:8300}]
2016/09/15 10:21:10 [INFO] raft: Node at 127.0.0.1:8300 [Follower] entering Follower state (Leader: "")
2016/09/15 10:21:10 [INFO] serf: EventMemberJoin: Armons-MacBook-Air 127.0.0.1
2016/09/15 10:21:10 [INFO] serf: EventMemberJoin: Armons-MacBook-Air.dc1 127.0.0.1
2016/09/15 10:21:10 [INFO] consul: Adding LAN server Armons-MacBook-Air (Addr: tcp/127.0.0.1:8300) (DC: dc1)
2016/09/15 10:21:10 [INFO] consul: Adding WAN server Armons-MacBook-Air.dc1 (Addr: tcp/127.0.0.1:8300) (DC: dc1)
2016/09/15 10:21:13 [DEBUG] http: Request GET /v1/agent/services (180.708µs) from=127.0.0.1:52369
2016/09/15 10:21:13 [DEBUG] http: Request GET /v1/agent/services (15.548µs) from=127.0.0.1:52369
2016/09/15 10:21:17 [WARN] raft: Heartbeat timeout from "" reached, starting election
2016/09/15 10:21:17 [INFO] raft: Node at 127.0.0.1:8300 [Candidate] entering Candidate state in term 2
2016/09/15 10:21:17 [DEBUG] raft: Votes needed: 1
2016/09/15 10:21:17 [DEBUG] raft: Vote granted from 127.0.0.1:8300 in term 2. Tally: 1
2016/09/15 10:21:17 [INFO] raft: Election won. Tally: 1
2016/09/15 10:21:17 [INFO] raft: Node at 127.0.0.1:8300 [Leader] entering Leader state
2016/09/15 10:21:17 [INFO] consul: cluster leadership acquired
2016/09/15 10:21:17 [DEBUG] consul: reset tombstone GC to index 3
2016/09/15 10:21:17 [INFO] consul: New leader elected: Armons-MacBook-Air
2016/09/15 10:21:17 [INFO] consul: member 'Armons-MacBook-Air' joined, marking health alive
2016/09/15 10:21:17 [INFO] agent: Synced service 'consul'
```    

如你所见， Consul agent已经启动了并输出一些日志数据。从日志数据可以知道该agent运行在`Server`模式并且是整个集群的Leader。此外，本地成员被标记为集群中的监控结点。

> Note for OS X Users: Consul uses your hostname as the default node name. If your hostname contains periods, DNS queries to that node will not work with Consul. To avoid this, explicitly set the name of your node with the -node flag.

### Cluster Members

在一个终端窗口执行命令：`consul members`，你可以知道Consul集群的所有成员信息。在后面的章节中我们会介绍如何加入集群，但是现在，你只能看到一个结点，那就是你自己。

```shell
$ consul members
Node                Address            Status  Type    Build     Protocol  DC
Armons-MacBook-Air  172.20.20.11:8301  alive   server  0.6.1dev  2         dc1
```

命令`consul members`的输出显示了我们自己的节点，它正在运行的地址，运行状况，在集群中的角色以及一些版本信息。 额外的元数据可以通过提供`-detailed`标志来查看。

`consul members`命令的输出基于[gossip协议](https://www.consul.io/docs/internals/gossip.html)，并且是最终一致的。 也就是说，在任何时候，本地agent所看到的数据可能与服务器上的状态不完全一致。 要获得一致的数据，可以通过HTTP API 查询 Consul Server结点。

```shell
$ curl localhost:8500/v1/catalog/nodes
[{"Node":"Armons-MacBook-Air","Address":"127.0.0.1","TaggedAddresses":{"lan":"127.0.0.1","wan":"127.0.0.1"},"CreateIndex":4,"ModifyIndex":110}]
```

除了HTTP API， 也可以通过DNS接口来查询结点的信息。但需要注意的是你的查询需要指定Consul的DNS服务器信息（默认端口是8600）。

```shell
$ dig @127.0.0.1 -p 8600 Armons-MacBook-Air.node.consul
...

;; QUESTION SECTION:
;Armons-MacBook-Air.node.consul.    IN  A

;; ANSWER SECTION:
Armons-MacBook-Air.node.consul. 0 IN    A   127.0.0.1
»
```

### Stopping the Agent

你可以使用`Ctrl-C`(中断信号)来优雅的关闭agent。在中断agent后，你可以看到该agent已经离开了集群并且处于关闭状态。

通过优雅地离开，Consul通知其它集群成员该节点离开。 如果你强行杀死了代理进程，则群集的其它成员将检测到该节点失败。 成员离开时，它的服务和检查信息将从`catalog`中删除。 当一个成员失败时，其健康被简单地标记为关键，但不会从目录中删除。 Consul将自动尝试重新连接到失败的节点，使其能够从特定的网络条件恢复，直到该结点再也联系不到。

此外，如果一个agent是作为`server`模式运行，那么优雅的离开是非常重要的，因为这会导致[共识协议](https://www.consul.io/docs/internals/consensus.html)潜在的不可用。 有关如何安全地添加和删除服务器的详细信息，请参阅[指南](https://www.consul.io/docs/guides/index.html)部分。

### Registering Services

#### Defining a Service

-------

服务的注册既可以通过提供服务定义，也可以通过调用Consul提供的服务注册HTTP API。

服务定义是注册服务最常用的方式，因此我们将在这一步中使用该方法。

首先，为Consul配置创建一个目录。 Consul将所有配置文件加载到配置目录中，因此Unix系统上的一个通用约定是将目录命名为`/etc/consul.d`（.d后缀意味着“该目录包含一组配置文件”）。

```shell
$ sudo mkdir /etc/consul.d
```

下面我们创建一个服务定义文件。我们假设一个名为`web`的服务运行在`80`端口。此外我们还给该服务指定了一个标签： 

```shell
$ echo '{"service": {"name": "web", "tags": ["rails"], "port": 80}}' | sudo tee /etc/consul.d/web.json
```

现在，我们重启agent，并指定Consul的配置文件目录

```shell
$ consul agent -dev -config-dir=/etc/consul.d
==> Starting Consul agent...
...
    [INFO] agent: Synced service 'web'
...
```

在启动的日志输出中你会看到`synced the web service`字样。这就意味着该agent从服务定义配置文件中加载了服务定义，并且成功注册到服务的`catalog`中。

如果你想要注册多个服务，你可以创建多个服务定义文件在Consul的配置目录中。

### Querying Services

-------

一旦Agent启动并且服务注册成功，我们就可以通过DNS和HTTP API来查询服务信息。

#### DNS API

```shell
$ dig @127.0.0.1 -p 8600 web.service.consul
...

;; QUESTION SECTION:
;web.service.consul.        IN  A

;; ANSWER SECTION:
web.service.consul. 0   IN  A   172.20.20.11
```

如你所见，返回了一个可以用的，带有结点IP地址的记录。 一条记录只能有一个IP地址。

```shell
$ dig @127.0.0.1 -p 8600 web.service.consul SRV
...

;; QUESTION SECTION:
;web.service.consul.        IN  SRV

;; ANSWER SECTION:
web.service.consul. 0   IN  SRV 1 1 80 Armons-MacBook-Air.node.dc1.consul.

;; ADDITIONAL SECTION:
Armons-MacBook-Air.node.dc1.consul. 0 IN A  172.20.20.11
```

也可以通过DNS查询指定标签的服务：

```shell
$ dig @127.0.0.1 -p 8600 rails.web.service.consul
...

;; QUESTION SECTION:
;rails.web.service.consul.      IN  A

;; ANSWER SECTION:
rails.web.service.consul.   0   IN  A   172.20.20.11
```

`rails`就是服务`web`的标签。

#### HTTP API

```shell
$ curl http://localhost:8500/v1/catalog/service/web
[{"Node":"Armons-MacBook-Air","Address":"172.20.20.11","ServiceID":"web", \
    "ServiceName":"web","ServiceTags":["rails"],"ServicePort":80}]
```

catalog API 给出了指定服务的所有结点信息。就像我们随后看到的健康检查一样，你通常只想查询健康的服务实例。

```shell
$ curl 'http://localhost:8500/v1/health/service/web?passing'
[{"Node":"Armons-MacBook-Air","Address":"172.20.20.11","Service":{ \
    "ID":"web", "Service":"web", "Tags":["rails"],"Port":80}, "Checks": ...}]
```

### Updating Services

-------

服务定义的修改可以通过修改服务定义文件，并给agent发送`SIGHUP`信号即可。这样你可以不停机，同时不影响服务查询的情况下修改服务定义。此外，也可以通过HTTP API 来动态添加，删除，修改服务。

### Consul Cluster

我们已经启动了我们第一个agent，并在该agent上进行了服务注册和查询。这表明了Consul的使用是多么的简单，但是没有展示如何基于该agent进行扩展为生产环境级别的服务发现基础设施。该该小节，我们将搭建一个真正的集群。

当一个agent启动后，它是不知道其它agent的存在的：它自己处于一个隔离的只有一个结点的集群。如果要了解其它节点的信息，则该agent必须加入已有的集群。为了加入已有的集群，它只需要知道已有集群中一个结点的信息即可。加入集群后，它会通过`gossip`协议快速发现集群中的其它结点。一个 Consul agent 可以加入任何其它的agent，而不仅仅运行在server模式。

#### Starting the Agents

分别在两台机器上执行下面的命令

10.13.40.95

```shell
agent -server -bootstrap-expect=1 -data-dir=/tmp/consul -node=agent-one -bind=10.13.40.95 -enable-script-checks=true -config-dir=/usr/local/consul/conf
```

10.110.82.169 

```shell
agent -server -bootstrap-expect=1 -data-dir=/tmp/consul -node=agent-two -bind=10.110.82.169 -enable-script-checks=true -config-dir=/usr/local/consul/conf
```

到目前为止我们拥有了2个集群，每个集群都只有一个结点。相互之间并不知道彼此的存在。

#### Joining a Cluster

-------

通过下面的命令来组件集群：

```shell
/usr/local/consul/bin/consul join 10.110.82.169
Successfully joined cluster by contacting 1 nodes.
```

通过命令`members`来检测集群是否搭建成功：

```shell
/usr/local/consul/bin/consul members
Node       Address             Status  Type    Build  Protocol  DC   Segment
agent-one  10.13.40.95:8301    alive   server  1.0.2  2         dc1  <all>
agent-two  10.110.82.169:8301  alive   server  1.0.2  2         dc1  <all>
```

#### Auto-joining a Cluster on Start

-------

Consul facilitates auto-join by enabling the auto-discovery of instances in AWS, Google Cloud or Azure with a given tag key/value. To use the integration, add the retry_join_ec2, retry_join_gce or the retry_join_azure nested object to your Consul configuration file. This will allow a new node to join the cluster without any hardcoded configuration. Alternatively, you can join a cluster at startup using the -join flag or start_join setting with hardcoded addresses of other known Consul agents.

实践中，当在数据中心启动一个新的Consul结点，它应该自动加入Consul集群而不应该通过人工的方式来加入。

#### Querying Nodes

-------

和查询服务一样，Consul提供了API来查询结点的信息，你可以通过DNS和HTTP API来进行查询。

针对DNS API, 结点的查询命令结构为:`NAME.node.consul` 或 `NAME.node.DATACENTER.consul`。如果不知道数据中心信息，那么该查询只会查询本数据中心内的结点。

例如，在结点`agent-one`，我们可以查询结点`agent-two`的信息:

```shell
dig @127.0.0.1 -p 8600 agent-two.node.consul
...

;; QUESTION SECTION:
;agent-two.node.consul. IN  A

;; ANSWER SECTION:
agent-two.node.consul.  0 IN    A   172.20.20.11
```

#### Leaving a Cluster

-------

otherwise, other nodes will detect it as having failed. The difference is covered in more detail here.

退出Consul集群，你可以优雅的退出（`Ctrl-C`）agent 进程 或 kill agent进程。优雅的退出集群会允许结点慢慢进入离开状态，其它结点也会收到通知信息。虽然其它结点也会检查到离开的结点，但是两者还是有不同的。具体可以参考[https://www.consul.io/intro/getting-started/agent.html#stopping](https://www.consul.io/intro/getting-started/agent.html#stopping)

### Health Checks

我们已经看到了使用Consul时多么的简单，添加集群结点，添加服务，查询结点和服务的信息都很容易。在该小节，我们将继续添加结点和服务的健康检查机制。健康检查是服务发现的关键组件，可以防止使用不健康的服务。

#### Defining Checks

Similar to a service, a check can be registered either by providing a check definition or by making the appropriate calls to the HTTP API.

We will use the check definition approach because, just like with services, definitions are the most common way to set up checks.

与服务定义类似，可以通过提供`检查定义`或通过调用对应的HTTP API来注册`检查`。

我们将使用`检查定义`方法，因为就像服务一样，定义是设置检查最常用的方法。

在Consul的配置文件目录创建2定义文件：

```shell
~$ echo '{"check": {"name": "ping",
  "script": "ping -c1 google.com >/dev/null", "interval": "30s"}}' \
  >/etc/consul.d/ping.json

~$ echo '{"service": {"name": "web", "tags": ["rails"], "port": 80,
  "check": {"script": "curl localhost >/dev/null 2>&1", "interval": "10s"}}}' \
  >/etc/consul.d/web.json
```

第一个定义添加了一个名为`ping`的主机级别的检查。 此检查运行间隔为30秒，调用`ping -c1 google.com`。 在基于脚本的运行状况检查上，检查以与启动Consul进程相同的用户身份运行。 如果该命令以非零退出码退出，则该节点将被标记为不健康。 这是任何基于脚本的健康检查的共同约定。

第二个命令修改名为web的服务，添加一个检查，每隔10秒通过curl发送一个请求，以验证Web服务器是否可访问。 与主机级运行状况检查一样，如果脚本以非零退出代码退出，服务将被标记为不健康。

现在重启第二个agent， 可以通过`consule reload` 或 通过给agent发送`SIGHUP`信号。你应该看到下面的日志信息： 

```shell
==> Starting Consul agent...
...
    [INFO] agent: Synced service 'web'
    [INFO] agent: Synced check 'service:web'
    [INFO] agent: Synced check 'ping'
    [WARN] Check 'service:web' is now critical
```

前几行表示代理已经同步了新的定义。 最后一行表明我们为Web服务添加的检查是至关重要的。 这是因为我们实际上没有运行Web服务器，所以`curl`测试失败了！

#### Checking Health Status

现在我们已经添加了一些简单的检查，我们可以使用HTTP API来检查它们。 首先，我们可以使用这个命令查找任何失败的检查（注意，这可以在任一节点上运行）：

```shell
~$ curl http://localhost:8500/v1/health/state/critical
[{"Node":"agent-two","CheckID":"service:web","Name":"Service 'web' check","Status":"critical","Notes":"","ServiceID":"web","ServiceName":"web","ServiceTags":["rails"]}]
```

我们可以看到，只有一个检查，我们的Web服务检查，在临界状态。

另外，我们可以尝试使用DNS查询Web服务。 由于服务不健康，Consul 不会返回任何结果：

```shell
dig @127.0.0.1 -p 8600 web.service.consul
...

;; QUESTION SECTION:
;web.service.consul.        IN  A
```
    
### KV Data

Consul除了提供服务发现和综合健康检查之外，还提供一个易于使用的KV存储。 这可以用来保存动态配置，协助服务协调，建立领导者选举，并使开发人员可以考虑构建的任何东西。

这一步假定您至少有一个Consul代理已经在运行。

#### Simple Usage

为了演示开始有多简单，我们将操作K / V存储中的几个键。 有两种方式与Consul KV存储进行交互：通过HTTP API和Consul KV CLI。 以下示例显示使用Consul KV CLI，因为它是最容易入门的。 对于更高级的集成，您可能需要使用Consul KV HTTP API

首先让我们探索KV存储。 我们可以向Consul询问名为redis / config / minconns的路径上的密钥的值：

```shell
$ consul kv get redis/config/minconns
Error! No key exists at: redis/config/minconns
```

As you can see, we get no result, which makes sense because there is no data in the KV store. Next we can insert or "put" values into the KV store.

```shell
$ consul kv put redis/config/minconns 1
Success! Data written to: redis/config/minconns

$ consul kv put redis/config/maxconns 25
Success! Data written to: redis/config/maxconns

$ consul kv put -flags=42 redis/config/users/admin abcd1234
Success! Data written to: redis/config/users/admin
```

Now that we have keys in the store, we can query for the value of individual keys:

```shell
$ consul kv get redis/config/minconns
1
```

Consul retains additional metadata about the field, which is retrieved using the -detailed flag:

```shell
$ consul kv get -detailed redis/config/minconns
CreateIndex      207
Flags            0
Key              redis/config/minconns
LockIndex        0
ModifyIndex      207
Session          -
Value            1
```

For the key "redis/config/users/admin", we set a flag value of 42. All keys support setting a 64-bit integer flag value. This is not used internally by Consul, but it can be used by clients to add meaningful metadata to any KV.

It is possible to list all the keys in the store using the recurse options. Results will be returned in lexicographical order:

```shell
$ consul kv get -recurse
redis/config/maxconns:25
redis/config/minconns:1
redis/config/users/admin:abcd1234
```

To delete a key from the Consul KV store, issue a "delete" call:

```shell
$ consul kv delete redis/config/minconns
Success! Deleted key: redis/config/minconns
```

It is also possible to delete an entire prefix using the recurse option:

```shell
$ consul kv delete -recurse redis
Success! Deleted keys with prefix: redis
```

To update the value of an existing key, "put" a value at the same path:

```shell
$ consul kv put foo bar

$ consul kv get foo
bar

$ consul kv put foo zip

$ consul kv get foo
zip
```

Consul can provide atomic key updates using a Check-And-Set operation. To perform a CAS operation, specify the -cas flag:

```shell
$ consul kv put -cas -modify-index=123 foo bar
Success! Data written to: foo

$ consul kv put -cas -modify-index=123 foo bar
Error! Did not write to foo: CAS failed
```

In this case, the first CAS update succeeds because the index is 123. The second operation fails because the index is no longer 123.


### Consul Web UI

Consul comes with support for beautiful, functional web UIs out of the box. UIs can be used for viewing all services and nodes, for viewing all health checks and their current status, and for reading and setting key/value data. The UIs automatically support multi-datacenter.

{% asset_img consul_web_ui-3a1e7bf9.png %}

To set up the self-hosted UI, start the Consul agent with the -ui parameter:

```shell
$ consul agent -ui
...
```

The UI is available at the /ui path on the same port as the HTTP API. By default this is http://localhost:8500/ui.

You can view a live demo of the Consul Web UI here.



    










