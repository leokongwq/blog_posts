---
layout: post
comments: true
title: pm2使用简介
date: 2016-10-17 10:58:40
tags:
- node
categories:
- nodejs
---

### pm2是什么

> pm2是一个用于生产环境的Nodejs程序进程管理器。它内置了请求的负载均衡功能；可以让你的进程永远运行；不停机的服务重载和日常的系统管理任务。

### 安装

    npm install pm2 -g

### 使用

    pm2 start app.js 

<!-- more -->

### pm2模块系统
PM2 内置了一个简单，功能强大的模块系统。通过下面的命令安装pm2模块：

    pm2 install <module_name>

下面是一些PM2的模块：    

[pm2-logrotate][1] : 自动切分pm2和应用的日志
[pm2-webshell][2] ： 提供一个功能完备的浏览器终端

### 升级pm2

    # Install latest pm2 version
    $ npm install pm2 -g
    # Save process list, exit old PM2 & restore all processes
    $ pm2 update

### 主要功能

#### 主要命令

    $ npm install pm2 -g            # Install PM2
    $ pm2 start app.js              # Start, Daemonize and auto restart application
    $ pm2 start app.js -i 4         # Start 4 instances of application in cluster mode
                                # it will load balance network queries to each app
    $ pm2 start app.js --name="api" # Start application and name it "api"
    $ pm2 start app.js --watch      # Restart application on file change
    $ pm2 start script.sh           # Start bash script
    $ pm2 list                      # List all processes started with PM2
    $ pm2 monit                     # Display memory and cpu usage of each app
    $ pm2 show [app-name]           # Show all informations about application
    $ pm2 logs                      # Display logs of all apps
    $ pm2 logs [app-name]           # Display logs for a specific app
    $ pm2 flush
    $ pm2 stop all                  # Stop all apps
    $ pm2 stop 0                    # Stop process with id 0
    $ pm2 restart all               # Restart all apps
    $ pm2 reload all                # Reload all apps in cluster mode
    $ pm2 gracefulReload all        # Graceful reload all apps in cluster mode
    $ pm2 delete all                # Kill and delete all apps
    $ pm2 delete 0                  # Delete app with id 0
    $ pm2 scale api 10              # Scale app with name api to 10 instances
    $ pm2 reset [app-name]          # Reset number of restart for [app-name]
    $ pm2 startup                   # Generate a startup script to respawn PM2 on boot
    $ pm2 save                      # Save current process list
    $ pm2 resurrect                 # Restore previously save processes
    $ pm2 update                    # Save processes, kill PM2 and restore processes
    $ pm2 generate                  # Generate a sample json configuration file
    $ pm2 deploy app.json prod setup    # Setup "prod" remote server
    $ pm2 deploy app.json prod          # Update "prod" remote server
    $ pm2 deploy app.json prod revert 2 # Revert "prod" remote server by 2
    $ pm2 module:generate [name]    # Generate sample module with name [name]
    $ pm2 install pm2-logrotate     # Install module (here a log rotation system)
    $ pm2 uninstall pm2-logrotate   # Uninstall module
    $ pm2 publish                   # Increment version, git push and npm publish

### 服务的不同启动方式

    $ pm2 start app.js --watch      # Restart application on file change
    $ pm2 start script.sh           # Start bash script
    $ pm2 start app.js -- -a 34     # Start app and pass option -a 34
    $ pm2 start app.json            # Start all applications declared in app.json
    $ pm2 start my-python-script.py --interpreter python

### 进程管理

进程启动后，可以很容易的列出和管理所有通过pm2启动的进程

{% asset_img pm2-readme/pm2-list.png%}

**列出所有启动的进程**

    pm2 list

**通过命令管理进程**    
    
    $ pm2 stop     <app_name|id|'all'|json_conf>
    $ pm2 restart  <app_name|id|'all'|json_conf>
    $ pm2 delete   <app_name|id|'all'|json_conf>

**获取进程信息**

    $ pm2 describe <id|app_name>

### 负载均衡 和 0秒重载

当一个应用通过`-i`选项启动，则会启用应用集群模式。

该功能支持主要的`Node.js`框架和任何`Node.js`程序。
    
**警告**: 如果你想使用pm2内置的负载功能，推荐使用`node#0.12.0+` 或 `node#0.11.16+`。pm2不在支持`node#0.10.*`版本的集群模式。

在集群模式下, PM2 支持所有的应用进程使用该主机的所有CPU资源。每一个 HTTP/TCP/UDP 请求会被一个选择的服务进程处理。

    $ pm2 start app.js -i 0  # Enable load-balancer and cluster features

    $ pm2 reload all           # Reload all apps in 0s manner

    $ pm2 scale <app_name> <instance_number> # Increase / Decrease process number

### CPU / 内存 监控

{% asset_img pm2-readme/pm2-monit.png %}

监控所有运行中的进程:

     pm2 monit

### 日志功能

{% asset_img pm2-readme/pm2-logs.png %}

实时显示所有日志或指定进程的日志

pm2 logs ['all'|'PM2'|app_name|app_id] [--err|--out] [--lines <n>] [--raw] [--timestamp [format]]

例子:

    $ pm2 logs
    $ pm2 logs WEB-API --err
    $ pm2 logs all --raw
    $ pm2 logs --lines 5
    $ pm2 logs --timestamp "HH:mm:ss"
    $ pm2 logs WEB-API --lines 0 --timestamp "HH:mm" --out
    $ pm2 logs PM2 --timestamp
    $ pm2 flush          # Clear all the logs

### 生成启动脚本
PM2 可以生成和配置一个启动脚本，该脚本用来保证在系统重启时启动pm2和应用进程。

    $ pm2 startup
    # auto-detect platform
    $ pm2 startup [platform]
    # render startup-script for a specific platform, the [platform] could be one of:
    #   ubuntu|centos|redhat|gentoo|systemd|darwin|amazon

可以通过下面的命令来保存进程列表

    $ pm2 save

### Keymetrics monitoring

{% asset_img pm2-readme/keymetrics.jpg %}

如果你通过PM2管理Nodejs程序, Keymetrics 可以通过服务端来使监控和管理应用程序变的更简单。 

### 更多

Watch & Restart
Application Declaration via JS files
PM2 API
Deploying workflow
PM2 and Heroku/Azure/App Engine
PM2 auto completion



                    