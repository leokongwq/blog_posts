## Linux TCP连接超时

建立TCP连接时，需要三次握手。

```client -&gt; server  发送 sync 分节
第一次：client -> server  发送 sync 分节
```

如果 client 迟迟不能收到 server返回的ack就会重试。默认重试6次。每次重试的时间间隔都是2的幂。

具体是：

1. 第 1 次发送 SYN 报文后等待 1s（2 的 0 次幂），如果超时，则重试
2. 第 2 次发送后等待 2s（2 的 1 次幂），如果超时，则重试
3. 第 3 次发送后等待 4s（2 的 2 次幂），如果超时，则重试
4. 第 4 次发送后等待 8s（2 的 3 次幂），如果超时，则重试
5. 第 5 次发送后等待 16s（2 的 4 次幂），如果超时，则重试
6. 第 6 次发送后等待 32s（2 的 5 次幂），如果超时，则重试
7. 第 7 次发送后等待 64s（2 的 6 次幂），如果超时，则超时失败

总计127秒。也就是说连接超时的默认时间是127秒。

我们的应用通常需要的是快速失败策略，显然这个超时时间太长了。可以通过减少重试的次数来缩短超时时间。

具体如下：

```shell
修改/etc/sysctl.conf文件添加如下配置：
tcp_syn_retries = 1 //超时时间为3s
或 直接通过命令：sysctl net.ipv4.tcp_syn_retries=1
```

顺便提一下：`net.ipv4.tcp_synack_retries`这个参数

这个参数用来控制 server 端发生第二次握手分节的重试次数。



## Socket 连接超时

Java 中的Socket API 可以指定连接超时时间：

```java
/**
* timeout  the timeout value to be used in milliseconds.
*/
public void connect(SocketAddress endpoint, int timeout) throws IOException {
}
```

内核在放弃建立连接前会重试指定的次数（可以计算出最大时间），Socket API也可以指定超时时间，这2个超时时间是有关联的。

1. 如果内核还在重试，但是API指定的超时时间已经到了，那么应用也会收到超时异常。
2. 如果内核不再重试，API还在等待建立连接，那么在指定的超时时间后收到超时。

### Socket 读取超时

读超时可以通过下面的API进行设置

```java
/**
* Enable/disable {@link SocketOptions#SO_TIMEOUT SO_TIMEOUT}
* with the specified timeout, in milliseconds. With this option set
* to a non-zero timeout, a read() call on the InputStream associated with
* this Socket will block for only this amount of time. 
* timeout the specified timeout, in milliseconds
*/
public synchronized void setSoTimeout(int timeout) throws SocketException {
}
```

通过上面的知识，我们可以知道， 虽然可能读取超时了，但是正常的数据还是能被内核接收到。 在下次读取的时候依然可以读取。



### MySQL连接超时和读取超时

我们现在使用MySQL驱动都是type4型驱动，通过TCP和MySQL Server进行通信。既然是基于TCP，那么一定会受到TCP超时设置的影响。 

在建立MySQL连接时，可以通过在连接串添加超时参数来控制连接超时时间。

例如：jdbc:mysql://127.0.0.1:3306/test?connectTimeout=2000&socketTime=2000



## JDBC Statement超时时间

Statement执行的超时时间可以通过下面的API进行设置。

```java
/**
* Sets the number of seconds the driver will wait for a
* <code>Statement</code> object to execute to the given number of seconds.
*/
void setQueryTimeout(int seconds) throws SQLException;
```

注意：JDBC 底层TCP的读取超时时间 必须大于 Statement的超时时间，否则Statement的超时设置没有任何意义，包括上面的事务超时设置。

### MySQL如何处理Statement超时

1. 调用 Connection 的 createStatement() 方法创建一个 Statement 对象

2. 调用 Statement 的 executeQuery() 方法

3. 如果启用了查询超时功能(通过连接串参数enableQueryTimeouts指定，默认为true)，并且设置了Statement 的超时时间，Statement 创建一个新的延时任务提交到所属Connection的Timer中
4. Timer到期后会创建一个线程，然后创建一个和当前连接配置相同的新连接，用新连接向MySQL发送一条取消查询的命令：`KILL QUERY connectionId`
5. 通过内部的 Connection 将查询命令传输到  MySQL数据库

7. 获取sql执行的返回结果，判断取消查询的任务是否已经执行

8. 如果已经执行，则抛出超时异常。

![img](https://upload-images.jianshu.io/upload_images/5015984-f32a72cdff5de604.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/614/format/webp)



### 连接池获取连接超时

不同的连接池配置属性不同，这点就不展开了。

DruidDataSource通过下面的属性进行配置。默认值为：0，表示不进行超时限制。

```xml
<property name="queryTimeout" value="100"/>
```



## 网络相关知识

### RTT

定义：RTT(Round-Trip Time) 往返时延 

计算公式：RTT=2*Tp+Tb; Tp为传播时延；Tb为确认信号时间。

### RTO

RTO = Retransmission TimeOut 指连接的超时重传时间，根据RTT不断的进行调整，防止重传时间太短导致发出太多包，防止重传时间太长使得应用层反应缓慢。



## 参考

<http://weakyon.com/2015/07/30/the-impact-fo-rto-to-tcp-timeout.html>

[http://xiaorui.cc/2016/06/05/%E7%90%86%E8%A7%A3linux%E7%BD%91%E7%BB%9C%E7%9A%84tcp%E8%B6%85%E6%97%B6%E5%92%8C%E9%87%8D%E4%BC%A0/](http://xiaorui.cc/2016/06/05/理解linux网络的tcp超时和重传/)

<http://www.chengweiyang.cn/2017/02/18/linux-connect-timeout/>

<https://xwl-note.readthedocs.io/en/latest/linux/tuning.html>

<http://perthcharles.github.io/2015/09/07/wiki-tcp-retries/>

<https://www.jianshu.com/p/2deaf51bf715>