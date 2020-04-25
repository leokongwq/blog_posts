---
layout: post
comments: true
title: centos下安装crond
date: 2017-12-20 15:05:50
tags:
- linux
- centos
categories:
---

### 背景

在测试环境执行命令`crontab -l`，系统提示找不到该命令。很奇怪？印象中这中情况是第一次遇到。Google了一下了解到不是所有系统或某个发行版的所有版本都会默认安装该命令。

<!-- more -->

### centos6安装 crond

```shell
yum -y install vixie-cron

service crond start

chkconfig crond on
```

### centos7安装 crond

```shell
yum install cronie
```

### cron 服务

cron 是linux的内置服务，但它不是自动起来，可以用以下的方法启动、关闭这个服务：

`/sbin/service crond start` //启动服务
`/sbin/service crond stop` //关闭服务
`/sbin/service crond restart` //重启服务
`/sbin/service crond reload` //重新载入配置

查看crontab服务状态：`service crond status`

查看crontab服务是否已设置为开机启动，执行命令：`ntsysv`

加入开机自动启动:

```shell
chkconfig --level 35 crond on
```

