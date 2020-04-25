---
layout: post
comments: true
title: rocketmq快速入门
date: 2017-01-12 17:18:23
tags:
- MQ
categories:
- MQ
---

### rocketmq 介绍

rocketmq 是阿里开源的一款MQ产品。关于它的前世今生可以参考[官方文档](https://github.com/alibaba/RocketMQ/wiki)

### 单机部署

#### 构建

```
1. git clone https://github.com/alibaba/RocketMQ.git
2. cd RocketMQ
3. bash install.sh
```

<!-- more -->

**注意** 该脚本需要我们本机安装了`git`,`jdk1.6+`,`maven3.x`才可以。一般我喜欢将软件安装在`/usr/local`目录下。 因此在构建完成后可以通过如下的命令创建自己的RocketMQ安装目录：

    sudo mkdir -p /usr/local/rocketmq
    sudo cp -rf target/alibaba-rocketmq-broker/alibaba-rocketmq/* /usr/local/rocketmq
    

#### 配置环境变量

rocketmq需要我们配置两个环境变量`JAVA_HOME`,`ROCKETMQ_HOME`

    echo "ROCKETMQ_HOME=`pwd`" >> ~/.bash_profile
    source ~/.bash_profile

#### 启动RocketMQ 命名服务和Broker服务

进入安装目录的`bin`子目录，启动命名服务

    screen bash mqnamesrv
    
当你看到信息"The Name Server boot success. serializeType=JSON"时这表明命名服务已经成功启动。然后`Ctrl + A`,然后`D`断开screen回话。

启动broker服务

    screen bash mqbroker -n localhost:9876
    
如果看到下面的输出信息：

    The broker[lizhanhui-Lenovo, 172.30.30.233:10911] boot success.     serializeType=JSON and name server is localhost:9876
    
则表示broker已经成功启动。

你也可以通过下面的命令查看日志文件来确人broker是否启动成功:

    tail -f ~/logs/rocketmqlogs/broker.log
    
检查是否会输出如下的心态信息：

    2016-07-29 12:19:11 INFO BrokerControllerScheduledThread1 - register broker to name server localhost:9876 OK
    
#### 发送和接收消息

在发送或接收消息前我们必须告诉客户端命名服务的位置。RocketMQ提供了多种方式来实现，为了简单我们使用环境变量:`NAMESRV_ADDR`

    export NAMESRV_ADDR=localhost:9876
    
现在我们就可以发送和接收消息了：

    bash tools.sh com.alibaba.rocketmq.example.quickstart.Producer
    
你可以在控制台看到好多表示消息已经发送到broker的日志

可以通过下面的命令来进行消息的消费测试：

    bash tools.sh com.alibaba.rocketmq.example.quickstart.Consumer    
    
### 代码示例

#### Producer    

```java
public class Producer { 
    public static void main(String[] args) throws MQClientException, InterruptedException {
        DefaultMQProducer producer = new DefaultMQProducer("YOUR_PRODUCER_GROUP"); // (1)
        producer.start(); // (2)
        for (int i = 0; i < 1000; i++) {
            try {
                Message msg = new Message("TopicTest",// topic // (3)
                        "TagA",// tag (4)
                        ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET)// body (5)
                );
                SendResult sendResult = producer.send(msg); // (6)
                System.out.println(sendResult);
            } catch (Exception e) {
                e.printStackTrace();
                Thread.sleep(1000);
            }
        }
        producer.shutdown();
    }
}
```

第一行代码实例化了一个`DefaultMQProducer`实例

在第二行代码中通过调用`start()`方法， `DefaultMQProducer`实例对自己做一些初始化操作并准备好发送消息。在这期间它会执行尝试获取命名服务的服务器地址列表（在我们的代码中是通过环境变量获取的），创建通信组件等操作。

3-5行代码我们创建了一个消息实例。可以看到我们指定了消息发送的目标topic,消息标签，并设置了消息体数据。

在第六行我们通过调用`DefaultMQProducer`的`send()`方法来将消息发送到`broker`。如果消息发送成功，则返回值`SendResult`的实例中的字段`msgId`将包含标识该消息在`RocketMQ`中的唯一性值。

`DefaultMQProducer` 提供了一些重载方法来满足不同的发送需求，同步的，异步的类似UDP的单向方式。

虽然所有的发送方法都很简单，易于使用，但内部却相当复杂。一般来说，生产者将做以下事情：

1. 检查指定主题是否存在现有路由数据;
2. 如果在上一步中没有，则连接到一个名称服务器以查询主题的路由信息​​;一旦获取了主题路由数据，更新主题路由表;如果从名称服务器获取主题路由仍然失败，则抛出一个未找到的异常。
3. 根据默认或自定义消息队列选择器从路由表中选择消息队列。
4. 检查是否存在与所选消息队列的代理的连接;如果没有，创建一个。
5. 编码并将消息传递给所选的代理。
6. 如果send（）使用同步/异步的范例，则等待来自代理的响应。如果在TIMEOUT间隔内未收到响应，则引发超时异常。
7. 一旦收到来自代理的响应，解码并换行成SendResult。恢复同步发送的方法执行或调用异步发送的回调。

#### Consumer

```java
 DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name_4");

consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

consumer.subscribe("TopicTest", "*");

consumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt>    msgs,                                                               ConsumeConcurrentlyContext context) {
                System.out.println(Thread.currentThread().getName() + " Receive New Messages: " + msgs);
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});

consumer.start();
```

代码比较简单就不用解释了。

### 参考资料

https://github.com/alibaba/RocketMQ/wiki/quick-start

    



    


