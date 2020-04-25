---
layout: post
comments: true
title: centos7中rc.local不起作用
date: 2017-08-26 22:23:01
tags:
categories:
- linux
---

### 背景

今天收到公司报警短信，提示在线应用resin的8080端口连接失败。最后知道是应用所在的机器被重启了。但是奇怪的是重启后我们的应用并没有启动。

### 问题原因

我们的应用部署环境时，在`rc.local`中添加了应用启动命令，难道是没有生效？带着这个疑问查看了配置文件`/etc/init.d/rc.local` 发现文件中确实已经添加了对应的应用启动命令。

求助Google后找到了问题的真正原因。

<!-- more -->

```shell
#!/bin/bash
# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
#
# It is highly advisable to create own systemd services or udev rules
# to run scripts during boot instead of using this file.
#
# In constrast to previous versions due to parallel execution during boot
# this script will NOT be run after all other services.
#
# Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
# that this script will be executed during boot.
```

翻译过来就是：

```
#这个文件是为了兼容性目的而添加的。
#
#强烈建议创建自己的systemd服务或udev规则来在开机时运行脚本而不是使用这个文件。
#
#与以前的版本相比较，在系统启动过程的并行执行中，这个脚本将不会在其他所有的服务启动后执行。
#
#请注意，你必须执行`chmod +x /etc/rc.d/rc.local`来确保确保这个脚本在系统启动时执行。
```

### systemd

关于systemd的前世今生可以阅读大神耗子个的文章[LINUX PID 1 和 SYSTEMD](https://coolshell.cn/articles/17998.html)

### 创建systemd服务

第一步：在`/usr/lib/systemd/system`目录添加一个配置文件。例如`tomcat.service`

第二步：执行命令`sudo systemctl enable tomcat.service`

第三步：启动服务`sudo systemctl start tomcat.service`

### systemd服务描述文件格式

我们还是以`tomcat.service`文件为例子

```shell
[Unit]
Description=Tomcat service 
Documentation=http://tomcat.apache.org
After=nginx.service

[Service]
EnvironmentFile=-/usr/local/tomcat/bin/env.sh
ExecStart=/usr/local/tomcat/bin/start.sh
Type=simple

[Install]
WantedBy=multi-user.target
```

### systemd常用命令

查看服务状态：`systemctl status 服务名`

启动服务：`systemctl start 服务名`

停止服务：`systemctl stop 服务名`

重启服务：`systemctl restart 服务名`

杀死服务：`systemctl kill 服务名`

查看服务配置：`systemctl cat 服务名`


### 参考资料

[Systemd 入门教程：命令篇](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html)
[Systemd 入门教程：实战篇](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-part-two.html)


