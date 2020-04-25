---
layout: post
comments: true
title: linux工具之traceroute
date: 2017-05-12 13:58:17
tags:
- linux
categories:
- linux
---

### 背景

在一次面试题中到`ping`和`traceroute`的区别，今天就详细总结一下`traceroute`命令的使用。

<!-- more -->

### 介绍

`traceroute`的`man`说明是：

> print the route packets take to network host.

意思就是输出数据包到目标主机的路径信息。

`traceroute`的工作机制主要是利用使用`ICMP`报文和和`IP`首部中的`TTL`字段来实现的。在网络数据包的传输过程中，每个处理处理数据包的路由器都要讲数据包的`TTL`值减1或者减去数据报在路由器中停留的秒数，由于大多数的路由器转发数据报的延时都小于1秒钟，因此TTL最终成为一个跳站的计数器，所经过的每个路由器都将其值减1。`TTL`字段的目的是防止数据报在网络中无休止的流动。当路由器收到一份IP数据报，如果TTL字段是0或者1，则路由器不转发该数据报（接收到这种数据报的目的主机可以将它交给应用程序，这是因为不需要转发该数据报。但是，在通常情况下系统不应该接收TTL字段为0的数据报）。通常情况下是，路由器将该数据报丢弃，并给信源主机发送一份`ICMP`超时信息。`tracerouter`程序的关键在于，这份`ICMP`超时信息包含了该路由器的地址。

`tracerouter`利用网络协议的这种机制，`TTL`值从1开始每次发送一个`TTL`等于上次值加一的数据包，直到收到目的主机的响应才停止。这样就能拿到数据包经过路径上的每个路由器的地址信息，从而打印路由信息。

### 使用

`tracerouter`命令的使用格式如下：

```shell
Usage: traceroute [-adDeFInrSvx] [-A as_server] [-f first_ttl] [-g gateway] [-i iface]
	[-M first_ttl] [-m max_ttl] [-p port] [-P proto] [-q nqueries] [-s src_addr]
	[-t tos] [-w waittime] [-z pausemsecs] host [packetlen]
```

### 命令参数：

- -d 使用Socket层级的排错功能。
- -f 设置第一个检测数据包的存活数值TTL的大小。
- -F 设置勿离断位。
- -g 设置来源路由网关，最多可设置8个。
- -i 使用指定的网卡地址送出数据包。
- -I 使用ICMP回应取代UDP资料信息。
- -m 设置检测数据包的最大存活数值TTL的大小。
- -n 直接使用IP地址而非主机名称。
- -p 设置UDP传输协议的通信端口。
- -r 忽略普通的Routing Table，直接将数据包送到远端主机上。
- -s 设置本地主机送出数据包的IP地址。
- -t 设置检测数据包的TOS数值。
- -v 详细显示指令的执行过程。
- -w 设置等待远端主机回报的时间。
-x 开启或关闭数据包的正确性检验。





