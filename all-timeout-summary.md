---
layout: post
comments: true
title: 关于timeout你必须了解
date: 2019-11-30 13:14:37
tags:
- 网络
categories:
---

### 前言

有经验的开发同学都知道访问依赖的服务服务时需要设置超时时间。这些外部服务可能是一个http接口，RPC接口，获取分布式锁等等。没有合理的超时时间设置，你的系统可能出现雪崩。但是在工作中发现大部分同学，包括我自己在内对如何合理的设置timeout没有形成一个完整的知识链条，这就会导致你可能设置了timeout，但系统并不会像你想象中的正常工作。

下面以一个简单的访问数据库的HTTP请求来串起整个理论。可能理解或实践有误，还请发现的同学留言斧正。

> 注意：所有代码例子都是在Linux 3.10.0 测试上通过，使用Java语言编写的。

<!-- more -->

### connect timeout 

我们使用最为广泛的数据库驱动底层是通过TCP来完成Client和Server通信的，在通信前必须建立网络连接。

{% asset_img tcp-connect.jpg %}

如图所示，Client 发送的syn包如果在一定的时间内没有收到Server的响应，那么Client就会报`ConnectException`

#### 防火墙设置

为了模拟syn丢包，我们通过在Server上添加如下的防火墙规则：

```shell
iptables -I INPUT -s 10.110.82.169 -j DROP 
```

客户端IP : 10.110.82.169 进来的包都会被DROP

#### 测试代码

Server

```java
import java.net.InetSocketAddress;
import java.net.ServerSocket;

/**
 * @author jiexiu
 * created 2019/11/30 - 13:38
 */
public class Server {

    public static void main(String[] args) throws Exception {
        ServerSocket socketServer = new ServerSocket();
        InetSocketAddress socketAddress = new InetSocketAddress("10.13.40.95", 3333);
        socketServer.bind(socketAddress, 10);
        System.out.println("Server started.");
        Thread.sleep(1000 * 100000);
    }
}
```

Client 代码

```java
/**
 * @author jiexiu
 * created 2019/11/30 - 13:41
 */
public class Client {

    public static void main(String[] args) throws Exception {
        Socket client = new Socket();
        InetSocketAddress socketAddress = new InetSocketAddress("10.13.40.95", 3333);
        client.connect(socketAddress);
        System.out.println("Client connect Server success.");
        client.setTcpNoDelay(true);
        OutputStream outputStream = client.getOutputStream();
        outputStream.write(1);
    }
}
```

#### 运行结果

执行命令：

```shell
date ; java Client; date
```
 
输出如下：

```java
Sat Nov 30 15:39:49 CST 2019
Exception in thread "main" java.net.ConnectException: Connection timed out (Connection timed out)
        at java.net.PlainSocketImpl.socketConnect(Native Method)
        at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)
        at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)
        at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)
        at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
        at java.net.Socket.connect(Socket.java:589)
        at java.net.Socket.connect(Socket.java:538)
        at Client.main(Client.java:15)
Sat Nov 30 15:39:52 CST 2019
```

可以看到超时时间大概是：`3s`

那为什么是3s呢？这是因为Linux内核关于TCP协议栈的配置。

执行命令

```
sysctl -a | grep retries
```

输出如下：
 
```
net.dccp.default.request_retries = 6
net.dccp.default.retries1 = 3
net.dccp.default.retries2 = 15
net.ipv4.tcp_orphan_retries = 0
net.ipv4.tcp_retries1 = 3
net.ipv4.tcp_retries2 = 15
# 下面2个配置很重要
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_synack_retries = 1
```

可以看到机器配置的`net.ipv4.tcp_syn_retries`和`net.ipv4.tcp_synack_retries`都是1。

注意: 1表示的是重试的次数，每次重试的间隔都是2的N次幂。例如：1, 2, 4, 8, 16, 32, 64。

将重试次数设置为7

```
 echo 7 > /proc/sys/net/ipv4/tcp_syn_retries 
```

重新测试，结果如下：

```java
date ; java Client; date
Sat Nov 30 16:16:27 CST 2019
Exception in thread "main" java.net.ConnectException: Connection timed out (Connection timed out)
        at java.net.PlainSocketImpl.socketConnect(Native Method)
        at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)
        at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)
        at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)
        at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
        at java.net.Socket.connect(Socket.java:589)
        at java.net.Socket.connect(Socket.java:538)
        at Client.main(Client.java:15)
Sat Nov 30 16:20:34 CST 2019
```

耗时大概是：4分7秒 = 247 < 1+2+4+8+16+32+64+128 = 255 (结果表明这个时间有误差) 

将重试次数设置为8

```
 echo 8 > /proc/sys/net/ipv4/tcp_syn_retries 
```

重新测试，结果如下：

```
date ; java Client; date                    
Sat Nov 30 16:26:14 CST 2019
Exception in thread "main" java.net.ConnectException: Connection timed out (Connection timed out)
        at java.net.PlainSocketImpl.socketConnect(Native Method)
        at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)
        at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)
        at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)
        at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
        at java.net.Socket.connect(Socket.java:589)
        at java.net.Socket.connect(Socket.java:538)
        at Client.main(Client.java:15)
Sat Nov 30 16:32:22 CST 2019
```

耗时是 6分8秒 = 368 约等于 1+2+4+8+16+32+64+128 + 128 = 383

man page 解释如下：

```
tcp_syn_retries (integer; default: 5; since Linux 2.2)
      The maximum number of times initial SYNs for an active TCP
      connection attempt will be retransmitted.  This value should
      not be higher than 255.  The default value is 5, which
      corresponds to approximately 180 seconds.
```

默认重试5次，最大值不应该大于255。 在默认情况下大概是180秒左右超时。

再次设置为5，测试结果如下：

```
 java Client; date                    
Sat Nov 30 16:47:14 CST 2019
Exception in thread "main" java.net.ConnectException: Connection timed out (Connection timed out)
        at java.net.PlainSocketImpl.socketConnect(Native Method)
        at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)
        at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)
        at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)
        at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
        at java.net.Socket.connect(Socket.java:589)
        at java.net.Socket.connect(Socket.java:538)
        at Client.main(Client.java:15)
Sat Nov 30 16:48:18 CST 2019
```

超时时间是：63秒。 这个值和文档的值有出入，可能是原因是新的内核版本对这部分逻辑进行了修改。

> 注意：不同的OS内核实现机制是不同的，例如基于BSD包括Mac OS X 在内，最大的等待时间是75秒。

### read timeout

连接建立了，那么就要相互交换数据了。举个例子：你发出了一个请求，通过读取对端的响应数据来判断对端是否正确处理了请求，如果不设置读取超时时间，那么就只能死等，非阻塞读还好，如果是阻塞模式，那么就导致线程占用，整个机器的线程资源会很快耗尽，不能服务。

这里需要注意的是：写操作是写入socket写缓冲区就返回了(TCP会进行重试)，作为客户端你是不知道你的请求对端究竟有没有收到。只能通过对端对请求的响应来判断。

### 数据库应该开发中超时设置

目前开发的大多数应用都是基于数据库，数据库作为稀缺资源一定要谨慎使用。

举个例子：如果没有设置java.sql.Statement执行sql的超时时间，哪个不小心上线了一个慢查询SQL，很容导致数据库连接池打满，整个服务不可用。

#### MySQL 连接超时和读取超时

MySQL的超时时间可以通过连接MySQL的url参数来指定，具体如下：

```java
jdbc:mysql://127.0.0.1:3306/test?connectTimeout=10000&socketTimeout=3000
```

connectTimeout 连接超时时间，单位为毫秒，默认值为0，依赖OS设置。

socketTimeout 读写超时时间，单位为毫秒，默认值为0。

#### java.jdbc.Statement 超时时间


```java
void setQueryTimeout(int seconds) throws SQLException;
```

设置数据库驱动等待Statement执行的超时时间，默认是0，表示永不超时。如果超时会抛出`SQLTimeoutException`异常。

在MySQL的驱动实现中，通过一个定时器来检查SQL执行超时，如果超时则通过一个新的连接给MySQL发送`kill query`命令，并抛出一个异常告诉客户端SQL执行超时。


#### 事务超时时间

事务超时通常是基于我们使用的事物框架来设置的。我们通常使用的是Spring提供的事物管理器。

AbstractPlatformTransactionManager

```java
/**
	 * Specify the default timeout that this transaction manager should apply
	 * if there is no timeout specified at the transaction level, in seconds.
	 * <p>Default is the underlying transaction infrastructure's default timeout,
	 * e.g. typically 30 seconds in case of a JTA provider, indicated by the
	 * <code>TransactionDefinition.TIMEOUT_DEFAULT</code> value.
	 * @see org.springframework.transaction.TransactionDefinition#TIMEOUT_DEFAULT
	 */
	public final void setDefaultTimeout(int defaultTimeout) {
		if (defaultTimeout < TransactionDefinition.TIMEOUT_DEFAULT) {
			throw new InvalidTimeoutException("Invalid default timeout", defaultTimeout);
		}
		this.defaultTimeout = defaultTimeout;
	}
```

默认情况下，没有设置事务超时时间。 

##### spring实现超时

1. 根据timeout+当前时间点 赋值给一个deadLine。
2. 每一次执行sql，就会获取到一个statement时，计算liveTime =（deadline- 当前时间），分如下两种情况处理：
    1. 如果liveTime>0，此时就执行stam
    2. 如果liveTime < 0,此时就抛出异常


### 参考资料

[mac下的iptables---pfctl](https://www.jianshu.com/p/eefe3877650f)
[Using pf on OS X Mountain Lion](https://blog.scottlowe.org/2013/05/15/using-pf-on-os-x-mountain-lion/)
[A Cheat Sheet For Using pf in OS X Lion and Up](https://krypted.com/mac-security/a-cheat-sheet-for-using-pf-in-os-x-lion-and-up/)
[Overriding the default Linux kernel 20-second TCP socket connect timeout](http://willbryant.net/overriding_the_default_linux_kernel_20_second_tcp_socket_connect_timeout)
[聊一聊重传次数](http://perthcharles.github.io/2015/09/07/wiki-tcp-retries/)
[TCP系列12—重传—2、Linux超时重传引入示例](https://www.cnblogs.com/lshs/p/6038527.html)
[Configuration Properties for Connector/J](https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-reference-configuration-properties.html)

