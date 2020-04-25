---
layout: post
comments: true
title: Linux中如何清除DNS缓存
date: 2017-08-30 15:30:29
tags:
- DNS
categories:
- linux
---

### 背景

今天一个同事在服务器上修改了`/etc/hosts`文件，将一个远程的域名解析到了本机`remote.host.name 127.0.0.1`。此时一切正常，所有请求都路由到了本机。但是将`/etc/hosts`文件中的配置删除后，域名还是解析到本机。此时很容易想到是DNS缓存。只要清除了DNS缓存就好了。Google了一下，发现好多文章都没有解决我的问题。最终找到此篇文章：[How to Clear/flush DNS Cache on Linux](http://www.2daygeek.com/flush-clear-dns-cache-on-ubuntu-centos-debian-fedora-mint-rhel-opensuse/#)，总算解决了问题。算是有学习到一点知识。

将本篇文章翻译如下，让更多需要的同学查看。

<!-- more -->

有些时候，由于DNS问题你不能访问某些网站。这很有可能是你的本地DNS缓存有问题。 对于这种情况，你需要重新启动操作系统DNS缓存服务。

DNS缓存代表域名缓存系统，是由计算机操作系统维护的临时数据库，其中包含你最近浏览的网站的IP地址。

我建议你了解学习一下Linux中可用的DNS工具程序，你可以利用这些工具检查所有DNS记录和相关信息，以进一步排除故障。 DNS实用程序有`nslookup`，`dig`和`host`。

下面列出的是在各Linux发行版中主要使用的DNS缓存服务。

- nscd DNS cache
- dnsmasq dns cache
- BIND server dns cache

### **nscd DNS Cache **

nscd 是 `name service cache daemon`的缩写，`Nscd`是一个守护经常，提供最普通的域名请求的缓存服务。 
默认的配置文件位于`/etc/nscd.conf`。

### ** dnsmasq DNS Cache **

`Dnsmasq`是一个轻量的，小巧的，易于配置的DNS转发器和DHCP服务器。 它旨在向小型网络提供DNS和可选的DHCP，适用于资源受限的路由器和防火墙。 它可以服务于不在全局DNS中的本地计算机的名称。 它专为个人计算机使用和小型网络而设计，而不是大型网络。

### ** BIND Server DNS Cache **

`BIND`是`Berkeley Internet Name Domain` 的缩写，是使用最为广泛的域名服务软件。`BIND`是实现互联网域名系统（DNS）协议的开源软件。 BIND是迄今为止在互联网上使用最广泛的DNS软件，提供强大而稳定的平台。

### Flush DNS cache on Ubuntu/Debian/LinuxMint

在`Ubuntu/Debian/Mint`系统中可以使用下面的命令来清除DNS缓存：

```shell
$ sudo /etc/init.d/dns-clean start
[sudo] password for magesh: [Enter your root password]
* Restoring resolver state...          [ OK ] 
```

### Flush BIND server dns cache

使用下面的命令来清除`BIND` DNS服务器的DNS缓存：

```shell
# /etc/init.d/named restart
Stopping named: .                                          [  OK  ]
Starting named:                                            [  OK  ]

# service named restart
Stopping named: .                                          [  OK  ]
Starting named:                                            [  OK  ]
```

### Flush nscd DNS cache

使用下面的命令来清除`nscd`服务的DNS缓存：

```shell
# /etc/init.d/nscd restart
# service nscd restart
# service nscd reload
# nscd -i hosts
```

### Flush dnsmasq dns cache

使用下面的命令来清除`dnsmasq`服务的DNS缓存：

```shell
# /etc/init.d/dnsmasq restart
```

### Flush dns cache in windows

使用下面的命令来清除windows系统的DNS缓存：

```shell
# ipconfig /flushdns
Windows IP Configuration
Successfully flushed the DNS Resolver Cache.
```

我们正在准备所有文章，深入了解所有级别/阶段的Linux管理员。 如果文章对您有用，那么请花一点时间在我们的评论部分分享您的宝贵意见。





 






