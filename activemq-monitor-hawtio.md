---
layout: post
comments: true
title: activemq监控之hawtio
date: 2017-12-21 20:38:12
tags:
- ActiveQM
categories:
- MQ
---

### 背景

要对系统进行压测，而系统依赖的消息中间件是Apache ActiveMQ。 在压测中需要衡量队列的堆积和消费情况。ActiveMQ自己的console不够友好，直观，需要不停的F5才能看到队列的堆积情况，而且只能反映瞬时的值，不能够知道整个压测过程过ActiveMQ的状态。

<!-- more -->

Google后发现ActiveMQ的监控有多种方式，具体可以参考官网地址：[How can I monitor ActiveMQ](http://activemq.apache.org/how-can-i-monitor-activemq.html)

生产环境使用的是ActiveMQ版本是5.8，默认是集成了[Jolokia](http://www.jolokia.org/)。这样我们就可以通过[hawt.io](http://hawt.io/)这个开源的web console来对ActiveMQ进行监控。

### 过程

第一步：下载hawtio 地址为：[http://hawt.io/getstarted/index.html](http://hawt.io/getstarted/index.html)

我使用的是`jar`方式。

第二部：启动

```shell
java -jar hawtio-app-1.5.6.jar --port 9999 
```

启动后打开地址：[http://localhost:9999/hawtio](http://localhost:9999/hawtio)

点击右上角的connect菜单：

{% asset_img connect.jpg %}

输入下面的参数：

- 项目连接名称(用于保存连接使用)
- ActiveMQ主机ip(或主机名)
- ActiveMQ端口号(即ActiveMQ的jetty.xml文件中配置的管理页面端口号)
- jolokia路径名(对ActiveMQ，使用/api/jolokia)

保存后，点击`connect to remote server`。

在输出的页面中输入ActiveMQ Web Console的用户名和密码即可。

{% asset_img hawtio-index.png %}


### 注意

hawtio 从版本 `1.5.0`开始，在连接远程JVM时，你需要在hawtio的启动JVM参数中添加：`hawtio.proxyWhitelist`。 格式是以','分割的域名或IP

具体参考：[http://hawt.io/configuration/index.html](http://hawt.io/configuration/index.html)

### 参考

[http://hawt.io/getstarted/index.html](http://hawt.io/getstarted/index.html)
[https://stackoverflow.com/questions/43206932/unable-to-connect-to-remote-server-from-hawtio-dashboard](https://stackoverflow.com/questions/43206932/unable-to-connect-to-remote-server-from-hawtio-dashboard)






