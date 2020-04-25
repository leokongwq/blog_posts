---
layout: post
comments: true
title: linux网络编程backlog和somaxconn
date: 2016-10-24 21:35:41
tags:
categories:
- linux
---

### 前言

> 学习过的知识只要用的机会不多,多半过段时间就会忘记.如果能反复学习或者记笔记则会记得更牢固一点.以后也可以直接查看复习.

**以下内容基于Linux 2.6.18内核**

### listen方法传入的backlog参数

    #include <sys/socket.h>

    int listen(int sockfd, int backlog);
    
在上面的代码中我们看到listen函数的第二个参数为backlog. 这个参数的意义在不同的Linux内核版本或操作系统定义是不同的.

<!-- more -->

### tcp状态转化图

{% asset_img tcp-state-diagram.png %}

建立Tcp连接需要3次握手, 因此在一个连接的状态变为`ESTABLISHED`之前,它会有一个过渡的中间状态`SYN RECEIVED `
因此TCP/IP协议栈就有2种方法来实现一个处于listen状态SOCKET连接.

1. 只使用一个队列, 也就是说未成功建立连接的和已经成功建立连接的socket都放入一个队列.但是只有处于状态`ESTABLISHED`的socket才能被应用程序调用`accept`获取.
2. 使用2个队列, 一个队列保存处于状态`SYN RECEIVED`的socket, 一个队列保存已经成功建立连接的socket.在历史上BSD采用这种实现方式.

### linux 上的backlog

在Linux2.2以前的版本中, backlog的值是未完成连接建立的socket队列长度. 具体描述可以参考[http://man7.org/linux/man-pages/man2/listen.2.html](http://man7.org/linux/man-pages/man2/listen.2.html);在后来的版本中,backlog的值就是指代已经完成连接建立, 等待`accept`调用的socket队列最大长度.未完成连接建立的队列最大长度可以由`/proc/sys/net/ipv4/tcp_max_syn_backlog`配置来控制.


### 已完成队列满后

通常未完成队列的长度大于已完成队列.已完成队列满后, 当服务器收到来自客户端的ACK包时如果 `/proc/sys/net/ipv4/tcp_abort_on_overflow` 设为 1, 直接回RST包,结束连接.否则忽视ACK包.

内核有定时器管理未完成队列,对于由于网络原因没收到ACK包或是收到ACK包后被忽视的SYN_RCVD连接重发SYN+ACK包, 最多重发次数由`/proc/sys/net/ipv4/tcp_synack_retries` 设定.

backlog 即上述已完成队列的大小, 这个设置是个参考值,不是精确值. 内核会做些调整, 大于/proc/sys/net/core/somaxconn, 则取somaxconn的值

### 未完成队列满后

如果启用syncookies (net.ipv4.tcp_syncookies = 1),新的连接不进入未完成队列,不受影响.否则,服务器不在接受新的连接.

### SYN 洪水攻击(syn flood attack)

通过伪造IP向服务器发送SYN包,塞满服务器的未完成队列,服务器发送SYN+ACK包 没回复,反复SYN+ACK包,使服务器不可用.

启用syncookies 是简单有效的抵御措施.
启用syncookies,仅未完成队列满后才生效.


### 参考

[http://veithen.github.io/2014/01/01/how-tcp-backlog-works-in-linux.html](http://veithen.github.io/2014/01/01/how-tcp-backlog-works-in-linux.html)
[http://www.jianshu.com/p/fe2228a77429](http://www.jianshu.com/p/fe2228a77429)





  






