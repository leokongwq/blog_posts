---
layout: post
comments: true
title: 分布式系统一致性
date: 2018-01-19 10:08:35
tags:
- 架构
categories:
---

### CAP定理

在理论计算机科学中，`CAP定理`（CAP theorem），又被称作`布鲁尔定理`（Brewer's theorem），它指出对于一个`分布式存储系统`来说，不可能同时满足以下三点:

- 一致性（Consistence） 每次读取都能获取最新的写入或一个`ERROR`
- 可用性（Availability）（每次请求都能获取到非`ERROR`的响应 -- 但是不保证获取的数据为最新数据）
- 分区容错性（Network partitioning）（以实际效果而言，分区相当于对通信的时限要求。系统如果不能在时限内达成数据一致性，就意味着发生了分区的情况，必须就当前操作在C和A之间做出选择。）

根据定理，分布式系统只能满足三项中的两项而不可能满足全部三项。理解CAP理论的最简单方式是想象两个节点分处分区两侧。允许至少一个节点更新状态会导致数据不一致(`全部不更新能保证一致性`)，即丧失了C性质。如果为了保证数据一致性，将分区一侧的节点设置为不可用，那么又丧失了A性质。除非两个节点可以互相通信，才能既保证C又保证A，这又会导致丧失P性质。

<!-- more -->

#### 自我理解

- 一致性: 指一旦数据写入成功，随后所有的读取请求都能获取写入结果。单节点系统，如：一个节点的DBMS能保证一致性，因为没有副本。`这里的一致性指的是强一致性`。
- 可用性: 每次请求都能获取到的响应（合理的时间内，超时不可以），但是不保证获取的数据为最新数据。
- 分区容错性（Network partitioning）（以实际效果而言，分区相当于对通信的时限要求。系统如果不能在时限内达成数据一致性，就意味着发生了分区的情况，必须就当前操作在C和A之间做出选择[3]。）

一个节点的存储系统一致性能保证，但是可用性不能保证。为什么？ 原因是：网络不总是可靠的，在发生网络分区或大量丢包，大量延迟的情况下，请求不能在合理的时间内结束，如此可用性是不能保证的（单点故障）。要消灭单点故障，就需要对数据做备份。数据多副本的情况下就有不一致的风险。 因此在设计系统时，首先要考虑的就是：在网络故障的前提下如何对可用性和一致性做取舍。通常情况下我们都会选择可用性，舍弃强一致性。

CAP是针对分布式系统的，对集中式系统不起作用。例如：应用和数据存储（DBMS）在一台机器上，没有副本就没有一致性的问题，没有网络就没有网络分区的问题。只要DBMS进程存在就是可用的。

CAP 同时是针对分布式存储状态或数据的系统的，如果分布式系统的每个节点都是纯粹的计算节点，那么根本就存在上述的各种问题。

### BASE理论

`BASE`是`Basically Available`（基本可用）、`Soft state`（软状态）和`Eventually consistent`（最终一致性）三个短语的简写，是由来自eBay的架构师`Dan Pritchett`在其文章[BASE: An Acid Alternative](https://queue.acm.org/detail.cfm?id=1394128) 中第一次明确提出的。BASE是对CAP中一致性和可用性权衡的结果，其来源于对大规模互联网系统分布式实践的总结，是基于CAP定理逐步演化而来的，其核心思想是即使无法做到强一致性(Strong consistency)，但每个应用都可以根据自身的业务特点，采用适当的方式来使系统达到最终一致性(Eventual consistency)。

### 基本可用

基本可用是指分布式系统在出现不可预知故障的时候，允许损失部分可用性——但请注意，这绝不等价于系统不可用。以下两个就是“基本可用”的典型例子。

响应时间上的损失：正常情况下，一个在线搜索引擎需要在0.5秒之内返回给用户相应的查询结果，但由于出现故障（比如系统部分机房发生断电或断网故障），查询结果的响应时间增加到了1～2秒。

功能上的损失：正常情况下，在一个电子商务网站上进行购物，消费者几乎能够顺利地完成每一笔订单，但是在一些节日大促购物高峰的时候，由于消费者的购物行为激增，为了保护购物系统的稳定性，部分消费者可能会被引导到一个降级页面。

#### 弱状态

弱状态也称为软状态，和硬状态相对，是指允许系统中的数据存在中间状态，并认为该中间状态的存在不会影响系统的整体可用性，即允许系统在不同节点的数据副本之间进行数据同步的过程存在延时。

#### 最终一致性

最终一致性强调的是系统中所有的数据副本，在经过一段时间的同步后，最终能够达到一个一致的状态。因此，最终一致性的本质是需要系统保证最终数据能够达到一致，而不需要实时保证系统数据的强一致性。

### 一致性模型

- 强一致性  ：系统任何的数据写入成功后，随后在所有结点所有读取操作都能获取到该写入结果。
- 弱一致性  ：系统在数据写入成功后，不能保证随后立即可以读取到该写入结果，但是会保证一段时间窗口后肯定能读取到该结果。
- 最终一致性 ：最终一致性是一种特殊的弱一致性，它是指在上线数据写入成功后，如果没有再对该数据有写入操作，那么在一段时间窗口后所有客户端端都能读取到一致的数据。

### 最终一致性分类

亚马逊首席技术官Werner Vogels在于2008年发表的一篇经典文章 [Eventually Consistent-Revisited](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html)中，对最终一致性进行了非常详细的介绍。他认为最终一致性是一种特殊的弱一致性：系统能够保证在没有其它新的更新操作的情况下，数据最终一定能够达到一致的状态，因此所有客户端对系统的数据访问都能够获取到最新的值。同时，在没有发生故障的前提下，数据达到一致状态的时间延迟，取决于网络延迟、系统负载和数据复制方案设计等因素。

在实际工程实践中，最终一致性存在以下五类主要变种。 

#### 因果一致性（Causal consistency）

因果一致性是指，如果进程A在更新完某个数据项后通知了进程B，那么进程B之后对该数据项的访问都应该能够获取到进程A更新后的最新值，并且如果进程B要对该数据项进行更新操作的话，务必基于进程A更新后的最新值，即不能发生丢失更新情况。与此同时，与进程A无因果关系的进程C的数据访问则没有这样的限制。

#### 读己之所写（Read your writes）

读己之所写是指，进程A更新一个数据项之后，它自己总是能够访问到更新过的最新值，而不会看到旧值。也就是说，对于单个数据获取者来说，其读取到的数据，一定不会比自己上次写入的值旧。因此，读己之所写也可以看作是一种特殊的因果一致性。

#### 会话一致性（Session consistency）

会话一致性将对系统数据的访问过程框定在了一个会话当中：系统能保证在同一个有效的会话中实现“读己之所写”的一致性，也就是说，执行更新操作之后，客户端能够在同一个会话中始终读取到该数据项的最新值。

#### 单调读一致性（Monotonic read consistency）

单调读一致性是指如果一个进程从系统中读取出一个数据项的某个值后，那么系统对于该进程后续的任何数据访问都不应该返回更旧的值。

#### 单调写一致性（Monotonic write consistency）

单调写一致性是指，一个系统需要能够保证来自同一个进程的写操作被顺序地执行。

### 一致性实现方式

### 参考

[CAP定理](https://zh.wikipedia.org/wiki/CAP%E5%AE%9A%E7%90%86)
[better-explaining-cap-theorem](https://dzone.com/articles/better-explaining-cap-theorem)
[understanding-the-cap-theorem](https://dzone.com/articles/understanding-the-cap-theorem)
[eventually_consistent](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html)

