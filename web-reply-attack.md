---
layout: post
comments: true
title: web重放攻击介绍和防御方法
date: 2016-10-12 16:34:22
tags:
    - web
    - web安全
categories:
    - web
    
---

                        
### 重放攻击定义

> 所谓重放攻击就是攻击者发送一个目的主机已接收过的包，来达到欺骗系统的目的，主要用于身份认证过程。攻击者利用网络监听或者其他方式盗取认证凭据，之后再把它重新发给认证服务器。从这个解释上理解，加密可以有效防止会话劫持，但是却防止不了重放攻击。重放攻击任何网络通讯过程中都可能发生。

<!-- more -->

### 分类

**根据消息的来源**： 

*协议轮内攻击*：一个协议轮内消息重放 
*协议轮外攻击*：一个协议不同轮次消息重放 

**根据消息的去向**： 
**偏转攻击**：改变消息的去向 
**直接攻击**：将消息发送给意定接收方
其中偏转攻击分为： 
**反射攻击**：将消息返回给发送者 
**第三方攻击**：将消息发给协议合法通信双方之外的任一方

### 重放攻击与cookie

截获的http数据传输中的敏感数据大多数就是存放在cookie中的数据。其实在web安全中的通过其他方式（非网络监听）盗取cookie与提交cookie也是一种重放攻击。我们有时候可以轻松的复制别人的cookie直接获得相应的权限。

### 重放攻击的防御方案

1. 时间戳
    客户端提交的请求必须包含时间戳字段，且该字段的值距离服务器接收到请求的时刻比较接近。缺少时间戳字段，时间戳字段的值大于服务器的当前时间或是具体当前时间的间隔比较大的请求都是不合法的请求。这个时间间隔可以随意调整。间隔过小可能误杀正常的请求，过大则会减弱保护效果。这种防御方式需要通信的各方保持时钟同步。
 
2. 序号
    通信双方通过消息中的序列号来判断消息的新鲜性要求通信双方必须事先协商一个初始序列号，并协商递增方法。该序列号可以专门采用一个序号生成服务（提供时间戳服务），在客户端请求时请求该服务拿到序号，和请求参数一起提交给服务端。服务端校验该序号的有效性。使用时间戳的好处是：1.自增；2.服务器端时钟同步好处理。如果使用客户端生成，每个客户端的时钟不能确保同步。

### 总结
    根据重放攻击的定义:重复发送服务器已接处理过的请求数据；防御方案的关键点在于：

 1. 除了接口所需要的数据外，需要添加额外的变量，确保每次的完成相同业务功能的请求数据不同（对请求数据+变量进行签名）。
 2. 该变量不能被攻击者获取。                        
                    
                    