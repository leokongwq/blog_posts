---
layout: post
comments: true
title: Linux Netcat 命令用法简介
date: 2016-10-25 21:22:03
tags:
categories:
- linux
---

> netcat(简写是nc)是linux上非常有用的网络工具，它能通过TCP和UDP在网络中读写数据。通过配合使用其他工具和重定向，可以在脚本中以多种方式使用它。netcat所做的就是在两台电脑之间建立链接并返回两个数据流，在这之后所能做的事就看你的想像力了。
<!-- more -->

### 用法简介

nc [-hlnruz][-g<网关...>][-G<指向器数目>][-i<延迟秒数>][-o<输出文件>][-p<通信端口>][-s<来源位址>][-v...][-w<超时秒数>][主机名称][通信端口...]

### 参数说明

-  -g <网关>   设置路由器跃程通信网关，最丢哦可设置8个。
-  -G <指向器数目>   设置来源路由指向器，其数值为4的倍数。
-  -h   在线帮助。
-  -i <延迟秒数>   设置时间间隔，以便传送信息及扫描通信端口。
-  -l   使用监听模式，管控传入的资料。
-  -n   直接使用IP地址，而不通过域名服务器。
-  -o <输出文件>   指定文件名称，把往来传输的数据以16进制字码倾倒成该文件保存。
-  -p <通信端口>   设置本地主机使用的通信端口。
-  -r   乱数指定本地与远端主机的通信端口。
-  -s <来源位址>   设置本地主机送出数据包的IP地址。
-  -u   使用UDP传输协议。
-  -v   显示指令执行过程。
-  -w <超时秒数>   设置等待连线的时间。
-  -z   使用0输入/输出模式，只在扫描通信端口时使用。

### 端口扫描

    nc -v -w 2 192.168.2.34 -z 21-24
    //输出
    nc: connect to 192.168.2.34 port 21 (tcp) failed: Connection refused
    Connection to 192.168.2.34 22 port [tcp/ssh] succeeded!
    nc: connect to 192.168.2.34 port 23 (tcp) failed: Connection refused
    nc: connect to 192.168.2.34 port 24 (tcp) failed: Connection refused
    
### 文件拷贝
    
    在192.168.2.34上： nc -l 1234 > test.txt
    在192.168.2.33上： nc 192.168.2.34 < test.txt

### 简单聊天工具
    
    在192.168.2.34上： nc -l 1234
    在192.168.2.33上： nc 192.168.2.34 1234
    
这样，双方就可以相互交流了。使用ctrl+C(或D）退出。
    
### 用nc命令操作memcached

1. 存储数据：printf “set key 0 10 6rnresultrn” |nc 192.168.2.34 11211
2. 获取数据：printf “get keyrn” |nc 192.168.2.34 11211
3. 删除数据：printf “delete keyrn” |nc 192.168.2.34 11211
4. 查看状态：printf “statsrn” |nc 192.168.2.34 11211
5. 模拟top命令查看状态：watch “echo stats” |nc 192.168.2.34 11211
6. 清空缓存：printf “flush_allrn” |nc 192.168.2.34 11211 (小心操作，清空了缓存就没了）
    
### 连接web服务器
    
    //建立从本地1234端口到host.example.com的80端口连接，5秒超时
    nc -p 1234 -w 5 host.example.com 80
    
### udp连接服务器
    
    nc -u host.example.com 53

### 连接服务器并执行 

    echo -n "GET / HTTP/1.0"r"n"r"n" | nc host.example.com 80    
    
### 指定TCP协议版本

netcat 的 -4 和 -6 参数用来指定 IP 地址类型，分别是 IPv4 和 IPv6：

    //server
    nc -4 -l 2389
    //client
    nc -4 localhost 2389
    
### 禁止从标准输入中读取数据

该功能使用 -d 参数，请看下面例子：
    
    //server side 
    $ nc -l 2389
    
    //client side
    nc -d localhost 2389

### 强制 Netcat 服务器端保持启动状态

通常情况下如果连接到服务端的客户端连接断开,服务端进程也会退出, 如果想让服务端保持存活可以加一个`-k`的参数
        
        //server side
        $ nc -k -l 2389
        
        //client side
        nc localhost 2389
        

### netcat 主页
        
更多的用法可以参考:[http://netcat.sourceforge.net/](http://netcat.sourceforge.net/)
    
    
    
    
        
    


