---
layout: post
comments: true
title: consul之acl配置
date: 2018-07-08 23:08:31
tags:
- consul
---

## 前言

## 配置项

### acl_datacenter 

该配置项指定了对ACL信息具有权威性的数据中心。 必须提供它才能启用ACL。 所有Server实例和数据中心必须就ACL数据中心达成一致。 
必须在整个集群的Server节点上设置该配置项。但是针对API来说，如果要使客户端的请求能被正确转发，那么也需要在客户端节点设置。 
在Consul 0.8及更高版本中，还可以启用代理级别的ACL。 有关详细信息，请参阅[ACL指南](https://www.consul.io/docs/guides/acl.html)。

### acl_default_policy

该配置项可选的值为`allow`和`deny`。默认值为'allow'。默认的策略控制token在没有匹配的规则时的行为。
在`allow`模式下，ACL规则是一个黑名单：任何没有被禁止的操作都是可以执行的。
在`deny`模式下，ACL规则是一个白名单，任何没有明确指定的操作都是被禁止的。

> 注意：该配置项只有在配置了`acl_datacenter`后才起作用

<!-- more -->

### acl_down_policy

该配置项可选的值为`allow`，`deny`或`extend-cache`，　默认值是`extend-cache`。假如不能从`acl_datacenter`或`Leader`节点获取一个token的acl信息，则该配置项指定的策略被使用。

- `allow`模式下，所有的操作都允许。 
- `deny`模式下，所有的操作都被禁止。 
- `extend-cache`模式下,使用缓存的 ACL规则并忽略这些规则的过期时间。如果一个不可缓存的ＡＣＬ的规则被使用，则当作`deny`策略来处理。

### acl_master_token

该配置项只能用在`acl_datacenter`中的Server节点上。 如果该令牌不存在，则将使用管理级权限创建该令牌。 它允许操作员使用众所周知的令牌ID来引导ACL系统。

仅当服务器获得集群领导时才安装acl_master_token。 如果要安装或更改acl_master_token，请在所有服务器的配置中为acl_master_token设置新值。 完成此操作后，重新启动当前领导者以强制进行领导者选举。 如果未提供acl_master_token，则服务器不会创建主令牌。 提供值时，它可以是任何字符串值。 使用UUID可以确保它看起来与其他令牌相同，但并不是绝对必要的。

### acl_agent_master_token

用作特殊访问 token，在配置它的每个 agent 上具有 agent ACL 策略 write 特权。 只有当 Consul server 不可用于解析 ACL token 时， 此 tokone 才应由运维人员在服务中断期间使用。 应用程序应在正常操作期间使用常规 ACL token。该配置项从版本`0.7.2`开始使用，并且只有当配置项`acl_enforce_version_8`设置为true时才起作用。更多信息参考[ ACL Agent Master Token ](https://www.consul.io/docs/guides/acl.html#acl-agent-master-token)


### acl_agent_token

由客户端和`Server`节点执行内部操作使用。如果没有设置该配置项的值，则`acl_token`配置项的被使用。该配置项从版本`0.7.2`开始使用。

此令牌必须至少具有对其将注册的节点名称的写访问权，以便设置目录中的任何节点级信息，例如元数据或节点的标记地址。 使用此令牌还有其他地方，请参阅[ACL代理令牌](https://www.consul.io/docs/guides/acl.html#acl-agent-token)以获取更多详细信息。

### acl_enforce_version_8

用于客户端和服务器，以确定是否应在Consul 0.8之前预览新的ACL策略。 在Consul 0.7.2中添加，在0.8之前的Consul版本中默认为false，在Consul 0.8及更高版本中默认为true。 通过在实施开始之前允许策略到位，这有助于简化向新ACL功能的过渡。 有关详细信息，请参阅ACL指南。

### acl_replication_token

仅用于运行Consul 0.7或更高版本的`acl_datacenter`之外的服务器。 提供时，这将启用使用此令牌进行ACL复制，以检索ACL并将其复制到非权威的本地数据中心。 在Consul 0.9.1和更高版本中，您可以使用`enable_acl_replication`启用ACL复制，然后稍后使用每个服务器上的代理令牌API设置令牌。 如果在配置文件中设置了`acl_replication_token` ，它将自动将`enable_acl_replication`设置为true以实现向后兼容性。

如果存在影响权威数据中心的分区或其他中断，并且`acl_down_policy`设置为`extend-cache`，则可以在中断期间使用复制的ACL集解析不在缓存中的令牌。 有关更多详细信息，请参阅[ACL指南复制部分](https://www.consul.io/docs/guides/acl.html#replication)。

### acl_token

如果设置了该配置项，代理将在向Consul服务器发出请求时使用此令牌。 客户端可以通过提供`？token`查询参数，基于每个请求覆盖此令牌。 如果未提供，则使用映射到“匿名”ACL策略的空标记。

### acl_ttl

用于控制ACL缓存的过期时间。 默认是30秒。 此设置会对性能产生重大影响：减少它会导致更频繁的刷新，增加它会减少刷新次数。 但是，由于缓存未被主动失效，因此ACL策略可能会过时到TTL值。

