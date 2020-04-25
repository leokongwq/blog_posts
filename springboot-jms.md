---
layout: post
comments: true
title: springboot配置activemq
date: 2019-08-28 20:09:19
tags:
- springboot
- activemq
categories:
---

### 前言

网上有好多介绍springboot集成activemq的文章，看了一些文章感觉比较零散，还是抽时间自己详细总结一个如何使用，需要注意哪些点。尤其是关于连接池的配置，需要重点关注，否则在消息量大的情况下会把服务器搞挂。

### 快速配置

如果你只是连接一个activemq集群或节点，那么配置非常简单(这也是springboot便捷的原因)。

如下:

```java
spring.activemq.broker-url=tcp://127.0.0.1:61616?connectionTimeout=3000&soTimeout=500&tcpNoDelay=true&jms.redeliveryPolicy.maximumRedeliveries=1&jms.redeliveryPolicy.initialRedeliveryDelay=10
spring.activemq.user=admin
spring.activemq.password=admin
```

就这么简单！有了上面的配置你就可以发送消息了(通过JmsTemplate)。这背后的原理是通过springboot提供的`ActiveMQAutoConfiguration`来实现的。

<!-- more -->

```java
@Configuration
@AutoConfigureBefore(JmsAutoConfiguration.class)
@AutoConfigureAfter({ JndiConnectionFactoryAutoConfiguration.class })
@ConditionalOnClass({ ConnectionFactory.class, ActiveMQConnectionFactory.class })
@ConditionalOnMissingBean(ConnectionFactory.class)
@EnableConfigurationProperties(ActiveMQProperties.class)
@Import({ ActiveMQXAConnectionFactoryConfiguration.class,
		ActiveMQConnectionFactoryConfiguration.class })
public class ActiveMQAutoConfiguration {

}
```

从`ActiveMQAutoConfiguration`的代码能得知，只要你的classpath里面存在`ConnectionFactory.class和ActiveMQConnectionFactory.class` 并且容器里面没有类型为`ConnectionFactory.class`的Bean，那么该自动配置组件就会生效。

通过`ActiveMQAutoConfiguration`，我们在spring容器中就能自动获取一个类型为`ConnectionFactory.class`的Bean 和 `JmsTemplate.class`的Bean。

#### 发送消息

```java
@RunWith(SpringRunner.class)
@SpringBootTest
@ActiveProfiles(profiles = {"dev"})
public class TestSendAmqMsg {

    @Resource
    private JmsTemplate jmsTemplate;
    
    @Test
    public void testSendMsgDefault() throws Exception {
        jmsTemplate.convertAndSend("java");
        //等待消费，因为是自发自消费
        Thread.sleep(1000 * 20);
    }
}
```

上面的代码其实是不能正常工作的。原因是`jmsTemplate.convertAndSend`没有指定`Destination`。

`Destination`的指定有两种方式，一种是通过方法参数指定。如下所示：

```java
jmsTemplate.convertAndSend("hello-jms-queue", "java");
```

一种是通过在application.properties文件中指定一个默认值:

```java
spring.jms.template.default-destination=hello-jms-default
```

还有一点需要注意的是`Destination`类型，是Topic还是Queue。默认是Queue。`Destination`类型也可以通过两种方式设置。

一种是通过在application.properties文件中指定一个默认值:

```java
# false 表示是Queue
spring.jms.pub-sub-domain=false
```

一种是通过API

```java
jmsTemplate.setPubSubDomain(false);
```

> 注意：上面的例子虽然能实现消息的发送和接收，但是非常有局限性。一个ActiveMQ上既有Topic也有Queue，我们通过`JmsTemplate`发送和消费消息时，最好是通过参数`Destination`来指定目的地，热不是一个字符串(不知道是具体是什么类型，只能通过全局配置)。


#### 消费消息

有了上面的配置，我们可以有两种消费消息的方式。

1. 通过`JmsTemplate`的API来主动消费。这个就不详细讲了。
2. 通过`@JmsListener`来被动消费

通过`@JmsListener`来实现消息消费，配置如下。

```java
@Configuration
@EnableJms
public class JmsConfig {

    @JmsListener(containerFactory = "jmsListenerContainerFactory", destination = "hello-jms")
    public void consumerMsg(String msg) {
        System.out.println("############# Received message is : [" + msg + "]*************");
    }
}
```

`@EnableJms`的作用是启用spring的Jms的注解驱动能力。注册了`JmsListenerAnnotationBeanPostProcessor`Bean。原理如下：


```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(JmsBootstrapConfiguration.class)
public @interface EnableJms {
}

@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class JmsBootstrapConfiguration {

@Bean(name = JmsListenerConfigUtils.JMS_LISTENER_ANNOTATION_PROCESSOR_BEAN_NAME)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public JmsListenerAnnotationBeanPostProcessor jmsListenerAnnotationProcessor() {
		return new JmsListenerAnnotationBeanPostProcessor();
	}

	@Bean(name = JmsListenerConfigUtils.JMS_LISTENER_ENDPOINT_REGISTRY_BEAN_NAME)
	public JmsListenerEndpointRegistry defaultJmsListenerEndpointRegistry() {
		return new JmsListenerEndpointRegistry();
	}

}
```

`JmsAnnotationDrivenConfiguration`该配置类非常关键：

```java
@Configuration
@ConditionalOnClass(EnableJms.class)
class JmsAnnotationDrivenConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public DefaultJmsListenerContainerFactoryConfigurer jmsListenerContainerFactoryConfigurer() {
        	DefaultJmsListenerContainerFactoryConfigurer configurer = new DefaultJmsListenerContainerFactoryConfigurer();
        	configurer.setDestinationResolver(this.destinationResolver.getIfUnique());
        	configurer.setTransactionManager(this.transactionManager.getIfUnique());
        	configurer.setMessageConverter(this.messageConverter.getIfUnique());
        	configurer.setJmsProperties(this.properties);
        	return configurer;
    }

	@Bean
	@ConditionalOnMissingBean(name = "jmsListenerContainerFactory")
	public DefaultJmsListenerContainerFactory jmsListenerContainerFactory(
			DefaultJmsListenerContainerFactoryConfigurer configurer,
			ConnectionFactory connectionFactory) {
		DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
		configurer.configure(factory, connectionFactory);
		return factory;
	}
}
```

### 配置JmsTemplate属性

```java
spring.jms.template.default-destination=hello-jms-default
spring.jms.template.delivery-mode=non_persistent
spring.jms.template.priority=100
spring.jms.template.qos-enabled=true
spring.jms.template.time-to-live=50
# 设置消息延迟投递时间 需要jms 2.0 支持
#spring.jms.template.delivery-delay=1
spring.jms.template.receive-timeout=100
```

### 配置消费属性

```java
# 消息消费
spring.jms.listener.acknowledge-mode=client
spring.jms.listener.auto-startup=true
spring.jms.listener.concurrency=10
spring.jms.listener.max-concurrency=20
```

### 连接池配置

通过上面的学习，我们已经能实现消息的发送和消费了。但是有一个问题就是，我们会和ActiveMQ Broker建立大量的短连接。在高并发下肯定是不可以的。通过在`application.properties`中简单配置，我们就能获得连接池能力。

#### 添加依赖

```xml
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-pool</artifactId>
    <version>5.14.3</version>
</dependency>
```

#### 修改配置

```java
# 只有配置了该选项，才会启用activemq连接池功能，参见：ActiveMQConnectionFactoryConfiguration
spring.activemq.pool.enabled=true
# 配置连接池参数
spring.activemq.pool.configuration.max-connections=10
spring.activemq.pool.configuration.idle-timeout=30000
spring.activemq.pool.configuration.expiry-timeout=0

spring.activemq.pool.configuration.create-connection-on-startup=false
spring.activemq.pool.configuration.time-between-expiration-check-millis=60000
spring.activemq.pool.configuration.maximum-active-session-per-connection=100
spring.activemq.pool.configuration.reconnect-on-exception=true
spring.activemq.pool.configuration.block-if-session-pool-is-full=true
spring.activemq.pool.configuration.block-if-session-pool-is-full-timeout=3000
```

#### 连接池自动配置实现原理

```java
@ConditionalOnClass(PooledConnectionFactory.class)
static class PooledConnectionFactoryConfiguration {

	@Bean(destroyMethod = "stop")
	@ConditionalOnProperty(prefix = "spring.activemq.pool", name = "enabled", havingValue = "true", matchIfMissing = false)
	@ConfigurationProperties(prefix = "spring.activemq.pool.configuration")
	public PooledConnectionFactory pooledJmsConnectionFactory(
			ActiveMQProperties properties) {
		PooledConnectionFactory pooledConnectionFactory = new PooledConnectionFactory(
				new ActiveMQConnectionFactoryFactory(properties)
						.createConnectionFactory(ActiveMQConnectionFactory.class));

		ActiveMQProperties.Pool pool = properties.getPool();
		pooledConnectionFactory.setMaxConnections(pool.getMaxConnections());
		pooledConnectionFactory.setIdleTimeout(pool.getIdleTimeout());
		pooledConnectionFactory.setExpiryTimeout(pool.getExpiryTimeout());
		return pooledConnectionFactory;
	}
}
```

有了这个实现作为参考，如果我们不想使用springboot提供的`ActiveMQ`自动配置功能，我们自己写代码配置，也能实现连接池的功能，无非就是普通的`ActiveMQConnectionFactoryFactory`进行包装而已。

有一个细节需要注意：前置为`spring.activemq.pool.configuration`的配置属性是如何设置到`PooledConnectionFactory`的呢？但是是通过`ConfigurationPropertiesBindingPostProcessor` 该类会处理注解`ConfigurationProperties` 指定的属性，通过反射设置到生成的Bean中（在Bean初始化前）。

### 总结

相关代码地址:[springboot-learn](https://github.com/leokongwq/springboot-learn)



