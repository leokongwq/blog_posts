---
layout: post
comments: true
title: 使用spring处理分布式事务
date: 2017-01-09 17:28:41
tags:
- xa
- spring
categories:
- 分布式事务
---

本文翻译自[http://www.javaworld.com/article/2077714/java-web-development/xa-transactions-using-spring.html](http://www.javaworld.com/article/2077714/java-web-development/xa-transactions-using-spring.html)

### 前言

当我们需要一些高级特性如事务，安全性，可用性和可扩展性时使用J2EE应用服务器已成为一种必然选择。只有很少的java应用程序才需要这些企业功能，并且组织通常需要一个功能完整的J2EE服务器。 本文主要关注使用JTA（Java事务API）的分布式事务，并将详细阐述如何使用广泛流行的Spring框架和开源代码，在没有J2EE服务器的独立java应用程序中使用分布式事务（也称为XA） JTA实现JBossTS，Atomikos和Bitronix。

<!-- more -->

分布式事务处理系统被设计用来简化在分布式环境中处理异构的，事务可感知资源的事务。 使用分布式事务，应用可以完成如从消息队列检索消息并且在遵循ACID（原子性，一致性，隔离和持久性）标准的单个事务单元中更新一个或多个数据库。 本文概述了可以使用分布式事务（XA）的一些用例，以及应用程序如何使用JTA最佳技术实践实现分布式事务处理。 主要关注的是使用Spring作为服务器框架，以及如何将各种JTA实现无缝集成到企业级分布式事务中。

### 分布式事务和JTA API

由于本文的范围限于使用Spring框架来集成JTA实现，我们将简要介绍分布式事务处理的架构概念。

#### 分布式事务

由Open Group（供应商联盟）设计的X/A OPEN 分布式事务处理规范定义了一种标准通信架构，它允许多个应用共享由多个资源管理器提供的资源，并允许它们的工作被协调到全局事务中。 XA接口使资源管理器能够连接事务，执行2PC（两阶段提交）并在故障后恢复不确定事务。

图 1: DTP环境的概念模型.

{% asset_img xaconceptualview-100159019-orig.gif %}

如图1所示，该模型具有以下接口：

1. 应用程序和资源管理器之间的接口允许应用程序使用资源管理器的本地API或本地XA API直接调用资源管理器，这取决于事务是否需要由事务监视器管理。
2. 应用程序和事务监视器（TX接口）之间的接口允许应用程序为所有事务需要调用事务监视器，例如启动事务，结束事务，回滚事务等。
3. 事务监视器和资源管理器之间的接口是XA接口。 这是接口，它有助于两阶段提交协议在一个全局事务下实现分布式事务.

#### JTA API

由Sun Microsystems定义的JTA API是一种高级API，其定义了事务管理器与分布式事务系统中涉及的各方之间的接口。 JTA主要由三部分组成:

- 一个应用程序用来指定事务边界的高级API。 `UserTransaction`接口封装了这个。
- 一个行业标准的X / Open XA协议的Java映射。 这包括在`javax.transaction.xa`包中定义的接口，它由`XAResource`，`Xid`和`XAException`接口.
- 一个高级事务管理器接口，允许应用程序服务器管理用户应用程序的事务。 `TransactionManager`，`Transaction`，`Status`和`Synchronization`接口几乎定义了应用程序服务器如何管理事务.

现在我们对JTA和XA标准的简要总结，让我们通过一些用例来演示使用Spring的不同JTA实现的集成，用于我们的假设Java应用程序。

### 我们的用例

为了演示不同JTA实现与Spring的集成，我们将使用以下用例：

1. 在全局事务中更新两个数据库资源 - 我们将使用JBossTS作为JTA实现。 在这个过程中，我们将看到我们如何以声明的方式将分布式事务语义应用于简单的POJO
2. 更新数据库并将JMS消息发送到全局事务中的队列 - 我们将演示与Atomikos和Bitronix JTA实现的集成。
3. 在全局事务中使用JMS消息并更新数据库 - 我们将使用Atomikos和Bitronix JTA实现。 在这个过程中，我们将看到我们如何可以模拟事务MDP（消息驱动POJO的）

我们将使用MySQL作为数据库，使用Apache ActiveMQ作为我们的JMS消息提供程序。 在讨论用例之前，让我们简要地看一下我们将要使用的技术栈。

### Spring framework

Spring框架已经成为Java世界中最有用和最有效的框架之一。 在它提供的众多功能中，它还提供了运行具有任何JTA实现的应用程序所需的管道。 这使得它的独特之处在于应用程序不需要在J2EE容器中运行也可以获得JTA事务的好处。 请注意，Spring不提供任何JTA实现。 从用户角度来看，唯一的任务是确保JTA实现可以集成在Spring框架中。 这是我们将在下面几节中关注的.

#### spring中的事务处理

Spring提供了轻量级的事务处理框架并可以使用编程式和声明式事务管理。这使得独立的Java应用程序能够以编程方式或声明方式包含事务（JTA或非JTA）。程序化事务划分可以通过使用`PlatformTransactionManager`接口及其子类公开的API来实现。另一方面，声明性事务划分使用基于AOP（面向方面​​编程）的解决方案。对于本文，我们将探讨声明式事务划分，因为它使用`TransactionProxyFactoryBean`类较少干扰和易于理解。在我们的例子中，事务管理策略是使用JtaTransactionManager，因为我们有多个资源来处理。如果只有一个资源，根据底层技术有几个选择，并且它们都实现PlatformTransactionManager接口。例如，对于Hibernate，可以选择使用HibernateTransactionManager和基于JDO的持久性，可以使用JdoTransactionManager。还有一个JmsTransactionManager，它只用于本地事务。

Spring的事务框架还为应用程序提供了必要的工具来定义事务传播行为，事务隔离等。 对于声明式事务管理，TransactionDefinition接口指定传播行为，这与EJB CMT属性非常相似。 TransactionAttribute接口允许应用程序指定哪些异常将导致回滚，哪些异常将被提交。 这两个关键的接口，使声明式事务管理非常容易使用和配置，我们将看看，当我们通过我们的用例。

#### 使用spring处理异步消息

Spring一直支持通过其JMS抽象层使用JMS API发送消息。 它采用回调机制，由消息创建者和一个真正发送创建者创建的消息的JMS模板组成。

自从Spring 2.0发布以来，使用JMS API可以实现消息的异步消费。 虽然Spring针对消息消费提供了不同的消息监听器容器，但在J2EE和J2SE环境中最适合的消息侦听器容器是`DefaultMessageListenerContainer`（DMLC）。 `DefaultMessageListenerContainer`扩展了`AbstractPollingMessageListenerContainer`类，并对JMS 1.1 API提供了完整的支持。 它主要在循环中使用了JMS的同步消息接收调用（MessageConsumer.receive（）），并允许事务性的接收消息。 对于J2SE环境，独立的JTA实现可以连接到使用Spring的JtaTransactionManager，这将在以下部分展示。

### JTA的实现者

#### JBossTS

JBossTS，以前称为Arjuna Transaction Service，具有非常强大的实现，支持JTA和JTS API。 JBossTS附带了一个恢复服务，它可以作为一个单独的进程从您的应用程序进程运行。 不幸的是，它不支持与Spring的开箱即用的集成，但它很容易集成，我们将在我们的练习中看到。 也不支持JMS资源，只支持数据库资源.

#### Atomikos Transaction Essentials

Atomikos的JTA实现最近才开源。 互联网上的文档和文献表明，这是一个生产质量的实施，这也支持恢复和一些异乎寻常的特点，超越JTA API。 Atomikos提供了开箱即用的Spring集成以及一些不错的示例。 Atomikos支持数据库和JMS资源。 它还为数据库和JMS资源的池连接提供支持。

#### Bitronix JTA

Bitronix的JTA实现是相当新的，仍在测试。 它还声称支持交易恢复与一些商业产品一样好或甚至更好。 Bitronix为数据库和JMS资源提供支持。 Bitronix还提供了开箱即用的连接池和会话池。

### 分布式资源

#### JMS 资源

JMS API规范不要求实现者支持分布式事务，但如果实现者提供分布式事务功能，则应通过JTA XAResource API完成。 因此，实现者应该使用`XAConnectionFactory`，`XAConnection`和`XASession`接口公开其JTA支持。 幸运的是Apache的`[ActiveMQ](http://activemq.apache.org/)`提供了处理XA事务所需的实现。 我们的项目（参见参考资料部分）还包括使用TIBCO EMS（来自TIBCO的JMS服务器）的配置文件，可以注意到，当提供商切换时，配置文件需要最少的更改。

#### 数据库资源

MySQL数据库提供了一个XA实现，但只适用于它的InnoDB引擎。 它还提供了一个好用的支持JTA / XA协议的JDBC驱动程序。 虽然对某些XA功能的使用有一些限制，但为了本文的目的，它是足够好的。

### 用例环境

#### 准备数据库环境

第一个数据库`mydb1`将用于用例1和2，并将具有下表：

```
mysql> use mydb1;
Database changed
mysql> select * from msgseq;
+---------+-----------+-------+
| APPNAME | APPKEY    | VALUE |
+---------+-----------+-------+
| spring  | execution |    13 |
+---------+-----------+-------+
1 row in set (0.00 sec)
```

第二个数据库`mydb2`将用于用例3，并将具有下表：   

```
mysql> use mydb2;
Database changed
mysql> select * from msgseq;
+---------+------------+-------+
| APPNAME | APPKEY     | VALUE |
+---------+------------+-------+
| spring  | aaaaa      | 15    |
| spring  | allocation | 13    |
+---------+------------+-------+
2 rows in set (0.00 sec)
```

### 配置 JMS 提供者（针对用例2，3）

要在ActiveMQ中创建物理消息目的地，请执行以下操作：

1. 在ActiveMQ安装目录下的配置文件目录中的配置文件`activmq.xml`添加如下的配置：
    ```
      <destinations>
        <queue physicalName="test.q1" />
      </destinations>
    ```
2. 在classpath路径下的文件`jndi.properties`中添加如下行：

        queue.test.q1=test.q1
        
### 用例1 - 使用JBossTS在一个全局事务中更新两个数据库

*图 2: 在一个全局事务中更新两个数据库*

{% asset_img usecase1-100159020-orig.png%}

Let us assume that our application has a requirement where it needs to persist a sequence number, associated with an event, in two different databases(mydb1 and mydb2), within the same transactional unit of work as shown in Figure 2 above. To achieve this, let us write a simple method in our POJO class, which updates the two databases.

让我们假设我们的应用程序需要在两个不同的数据库（mydb1和mydb2）中，在如上图所示的相同的事务中持久化与事件相关联的序列号。 为了实现这一点，让我们在我们的POJO类中写一个简单的方法，它更新两个数据库

我们的EventHandler POJO类的代码如下：

```java
public void handleEvent(boolean fail) throws Exception {
     MessageSequenceDAO dao = (MessageSequenceDAO) springContext.getBean("sequenceDAO");
     int value = 13;
     String app = "spring";
     String appKey = "execution";
     int upCnt = dao.updateSequence(value, app, appKey);
     log.debug(" sql updCnt->" + upCnt);
     
     if (springContext.containsBean("sequenceDAO2")) {
      // this is for use case 1 with JBossTS
      MessageSequenceDAO dao2 = (MessageSequenceDAO) springContext.getBean("sequenceDAO2");
      appKey = "allocation";
      upCnt = dao2.updateSequence(value, app, appKey);
      log.debug(" sql updCnt2->" + upCnt);
     }

        ...
     
     if (fail) {
      throw new RuntimeException("Simulating Rollback by throwing Exception !!");
     }
    }
```

正如你所看到的，我们在代码的第一段中所做的是获取一个对第一个数据库进行操作的MessageSequenceDAO对象引用，并更新序列的值。

你当然可以猜到，第二段的代码的作用是更新第二个数据库。

最后一个if语句当条件成立时引发运行时异常。 这是用于模拟运行时异常，以测试事务管理器是否已成功回滚全局事务。

让我们看一下spring的配合文件，看看是如何配置我们的DAO类和数据源的：

```xml
<bean id="dsProps" class="java.util.Properties">
     <constructor-arg>
       <props>
        <prop key="user">root</prop>
        <prop key="password">murali</prop>
        <prop key="DYNAMIC_CLASS">com.findonnet.service.transaction.jboss.jdbc.Mysql</prop>
       </props>
     </constructor-arg>
 </bean>
 
 <bean id="dataSource1" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
  <property name="driverClassName">
   <value>com.arjuna.ats.jdbc.TransactionalDriver</value>
  </property>
  <property name="url" value="jdbc:arjuna:mysql://127.0.0.1:3306/mydb1"/>
  <property name="connectionProperties">
    <ref bean="dsProps"/>
  </property>
 </bean>
 <bean id="dataSource2" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
  <property name="driverClassName">
   <value>com.arjuna.ats.jdbc.TransactionalDriver</value>
  </property>
  <property name="url" value="jdbc:arjuna:mysql://127.0.0.1:3306/mydb2"/>
  <property name="connectionProperties">
    <ref bean="dsProps"/>
  </property>
 </bean>
 
 <bean id="sequenceDAO" class="com.findonnet.persistence.MessageSequenceDAO">
  <property name="dataSource">
   <ref bean="dataSource1"/>
  </property>
 </bean>
 <bean id="sequenceDAO2" class="com.findonnet.persistence.MessageSequenceDAO">
  <property name="dataSource">
   <ref bean="dataSource2"/>
  </property>
 </bean>
```

上面的bean定义了两个数据库的两个数据源。 要使用JBossTS的TransactionalDriver，我们需要使用JNDI绑定或动态类实例来注册数据库。 我们将使用后者，这需要我们实现DynamicClass接口。 dsProps的bean定义显示，我们将使用我们编写的com.findonnet.service.transaction.jboss.jdbc.Mysql类，它实现了DynamicClass接口。 由于JBossTS不提供任何开箱的MySQL数据库包装器，我们不得不这样做。 除此之外，代码依赖于以`jdbc：arjuna`开头的`jdbc URL`，否则JBossTS代码会抛出错误。

DynamicClass接口的实现非常简单，我们所要做的就是实现方法`getDataSource（）`和`shutdownDataSource（）`方法。 getDataSource（）方法返回一个合适的XADataSource对象，在我们的例子中是`com.mysql.jdbc.jdbc2.optional.MysqlXADataSource`。

请注意，这两个数据源的配置使用了来自Spring的DriverManagerDataSource，它不使用任何连接池，不推荐用于生产环境。 另一种方法是使用一个支持XA数据源池化的数据源实现。

现在是我们在EventHandler类中为我们的handleEvent（）方法提供事务语义的时候了:

```xml
<bean id="eventHandlerTarget" class="com.findonnet.messaging.EventHandler"></bean>
 <bean id="eventHandler" class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">
   <property name="transactionManager"><ref bean="transactionManager" /></property>
   <property name="target"><ref bean="eventHandlerTarget"/></property>
   <property name="transactionAttributes">
     <props>
      <prop key="handle*">PROPAGATION_REQUIRED,-Exception</prop>
     </props>
   </property>
 </bean>
 ```
 
在这里我们定义TransactionProxyFactoryBean并传入transactionManager引用，我们将在后面看到，事务属性为“PROPAGATION_REQUIRED，-Exception”。 这将应用于目标bean eventHandlerTarget，并且只应用于以“handle”（注意句柄*）开头的方法。 简单来说，我们要求Spring框架做的是： 对于目标对象上的方法名称以“handle”开头的所有方法调用，请应用事务属性“PROPAGATION_REQUIRED，-Exception”。 在后台，Spring框架将创建一个基于CGLIB的代理，它拦截EventHandler上以“handle”开头的方法名称的所有调用。 在任何异常的情况下，在方法调用内，当前事务将被回滚，这是“-Exception”意味着什么。 这表明它使用Spring以声明方式提供事务支持有多么容易。 

现在让我们看看如何让Spring的JtaTransactionManager来使用我们选择的JTA实现。 上面定义的eventHandler bean使用transactionManager属性，它将引用Spring JtaTransactionManager，如下所示:

```xml
<bean id="jbossTransactionManager"
  class="com.arjuna.ats.internal.jta.transaction.arjunacore.TransactionManagerImple">
 </bean>
 
 <bean id="jbossUserTransaction"
  class="com.arjuna.ats.internal.jta.transaction.arjunacore.UserTransactionImple"/>
 
 <bean id="transactionManager"  class="org.springframework.transaction.jta.JtaTransactionManager">
  <property name="transactionManager">
   <ref bean="jbossTransactionManager" />
  </property>
  <property name="userTransaction">
   <ref bean="jbossUserTransaction" />
  </property>
 </bean>
```

如上面在配置中所示，我们让spring提供的`JtaTransactionManager`类使用`com.arjuna.ats.internal.jta.transaction.arjunacore.TransactionManagerImple`作为`TransactionManager`实现和`com.arjuna.ats.internal.jta.transaction。 arjunacore.UserTransactionImple`作为`UserTransaction`实现.

这就是该示例全部。 我们刚刚使用Spring和JbossTS启用了XA事务，MySQL数据源作为XA资源。 请注意，在撰写本文时，MySQL XA实现存在一些已知问题（请参阅资源部分）。

要运行此用例，请查看Ant构建文件中的JbossSender任务。 项目下载在参考资料部分提供.

### 用例2 - 在全局事务中更新数据库并发送JMS消息

*图3: 在全局事务中更新数据库并发送JMS消息*

{% asset_img usecase2-100159021-orig.png %}

此用例使用与用例1相同的代码。代码更新数据库mydb1，然后在全局事务中向消息队列发送消息，如上图3所示。 Atomikos和Bitronix配置都将按顺序处理。

POJO与我们用于用例-1的一样，EventHandler类和相关代码看起来如下:

```java
public void handleEvent(boolean fail) throws Exception {
 
     MessageSequenceDAO dao = (MessageSequenceDAO) springContext.getBean("sequenceDAO");
     int value = 13;
     String app = "spring";
     String appKey = "execution";
     int upCnt = dao.updateSequence(value, app, appKey);
     log.debug(" sql updCnt->" + upCnt);
     
        ...
     
     if (springContext.containsBean("appSenderTemplate")) {
      this.setJmsTemplate((JmsTemplate) springContext.getBean("appSenderTemplate"));
      this.getJmsTemplate().convertAndSend("Testing 123456");
      log.debug("Sent message succesfully");
     }
     if (fail) {
      throw new RuntimeException("Simulating Rollback by throwing Exception !!");
     }
  }
```

上面代码中的第一个代码片段用序列号更新数据库mydb1中的msgseq表.

下一段代码使用appSenderTemplate，它被配置为使用Spring中的JmsTemplate类。 此模板类用于将JMS消息发送到消息提供者。

外部事件（在这种情况下是MainApp类）将调用上面图3所示的方法handleEvent.

最后一个if语句用于模拟运行时异常，以测试事务管理器是否已成功回滚全局事务。

让我们看看spring配置文件中我们的消息传递需要的JNDI bean定义:

```xml
 <bean id="jndiTemplate" class="org.springframework.jndi.JndiTemplate">
  <property name="environment">
   <props>
    <prop key="java.naming.factory.initial">
     org.apache.activemq.jndi.ActiveMQInitialContextFactory
    </prop>
   </props>
  </property>
 </bean>
 
 <bean id="appJmsDestination"
  class="org.springframework.jndi.JndiObjectFactoryBean">
  <property name="jndiTemplate">
   <ref bean="jndiTemplate"/>
  </property>
  <property name="jndiName" value="test.q1"/>
 </bean>
 <bean id="appSenderTemplate" 
    class="org.springframework.jms.core.JmsTemplate">
  <property name="connectionFactory">
    <ref bean="queueConnectionFactoryBean"/>
  </property>
  <property name="defaultDestination">
    <ref bean="appJmsDestination"/>
  </property>
  <property name="messageTimestampEnabled" value="false"/>
  <property name="messageIdEnabled" value="false"/>
  <!-- sessionTransacted should be true only for Atomikos -->
  <property name="sessionTransacted" value="true"/>
 </bean>
 ```
 
这里我们为Spring指定一个JndiTemplate，在appJmsDestination bean指定的目的地上进行JNDI查找。 appJmsDestination bean已与如上所示的appSenderTemplate（JmsTemplate）bean连接。 bean定义还显示appSenderTemplate有线连接以使用queueConnectionFactoryBean，稍后我们将看到。对于Atomikos，sessionTransacted属性应该设置为“true”，这是Spring框架不建议的，JMS的文献似乎支持这个观点。这是一个黑客我们只需要做的Atomikos实现。如果这被设置为“false”，你会注意到在2PC协议期间抛出一些启发式异常。这主要归因于prepare（）调用在JMS资源上没有响应，最终，Atomikos决定回滚导致启发式异常。 messageTimestampEnabled和messageIdEnabled属性设置为“false”，因此不会生成这些属性，因为我们不会使用它们。这将减少JMS提供程序的开销，并提高性能。

让我们看看如何在spring中配置Atomikos

```xml
<bean id="xaFactory" class="org.apache.activemq.ActiveMQXAConnectionFactory">
        <constructor-arg>
           <value>tcp://localhost:61616</value>
        </constructor-arg>
 </bean>
 <bean id="queueConnectionFactoryBean"
    class="com.atomikos.jms.QueueConnectionFactoryBean" init-method="init">
    <property name="resourceName">
     <value>Execution_Q</value>
    </property>
    <property name="xaQueueConnectionFactory">
     <ref bean="xaFactory" />
    </property>
 </bean>
 <bean id="atomikosTransactionManager"
  class="com.atomikos.icatch.jta.UserTransactionManager" init-method="init" destroy-method="close">
  <property name="forceShutdown"><value>true</value></property>
 </bean>
 <bean id="atomikosUserTransaction" class="com.atomikos.icatch.jta.UserTransactionImp"/>
 
 <bean id="transactionManager"
  class="org.springframework.transaction.jta.JtaTransactionManager">
  <property name="transactionManager">
   <ref bean="atomikosTransactionManager" />
  </property>
  <property name="userTransaction">
   <ref bean="atomikosUserTransaction" />
  </property>
 </bean>
```
bean `xaFactory` 定义显示正在使用的xa连接的底层工厂是`org.apache.activemq.ActiveMQXAConnectionFactory`类。 `transactionManager` bean包装了`atomikosTransactionManager` bean和`atomikosUserTransaction` bean。

让我们看看针对Atomikos如何配置数据库：

```xml
 <bean id="dataSource" class="com.atomikos.jdbc.SimpleDataSourceBean" init-method="init"  destroy-method="close">
  <property name="uniqueResourceName"><value>Mysql</value></property>
  <property name="xaDataSourceClassName">
   <value>com.mysql.jdbc.jdbc2.optional.MysqlXADataSource</value>
  </property>
  <property name="xaDataSourceProperties">
   <value>URL=jdbc:mysql://127.0.0.1:3306/mydb1?user=root&password=murali</value>
  </property>
  <property name="exclusiveConnectionMode"><value>true</value></property>
 </bean>
```

Atomikos提供了一个通用的包装类，这使得很容易传递`xaDataSourceClassName`，在我们的例子中是`com.mysql.jdbc.jdbc2.optional.MysqlXADataSource`。 对于JDBC url的情况也是如此。 `exclusiveConnectionMode`设置为“true”，以确保当前事务中的连接不共享。 Atomikos提供了开箱即用的连接池，并且可以使用`connectionPoolSize` `attributexaDataSourceClassName`设置池大小。

现在让我们看看Bitronix的相关bean定义：

```java
<bean id="ConnectionFactory" factory-bean="ConnectionFactoryBean" factory-method="createResource" />
 <bean id="dataSourceBean1" class="bitronix.tm.resource.jdbc.DataSourceBean">
  <property name="className" value="com.mysql.jdbc.jdbc2.optional.MysqlXADataSource" />
  <property name="uniqueName" value="mysql" />
  <property name="poolSize" value="2" />
  <property name="driverProperties">
   <props>
    <prop key="user">root</prop>
    <prop key="password">murali</prop>
    <prop key="databaseName">mydb1</prop>
   </props>
  </property>
 </bean>
<bean id="Db1DataSource" factory-bean="dataSourceBean1" factory-method="createResource" />
 <bean id="BitronixTransactionManager" factory-method="getTransactionManager"
     class="bitronix.tm.TransactionManagerServices" depends-on="btmConfig,ConnectionFactory" destroy-method="shutdown" />
 <bean id="transactionManager" class="org.springframework.transaction.jta.JtaTransactionManager">
  <property name="transactionManager" ref="BitronixTransactionManager" />
  <property name="userTransaction" ref="BitronixTransactionManager" />
 </bean>
<bean id="btmConfig" factory-method="getConfiguration" class="bitronix.tm.TransactionManagerServices">
     <property name="serverId" value="spring-btm-sender" />
</bean> 
```
为Atomikos定义的appSenderTemplate bean可以重复使用，唯一的例外是sessionTransacted值。 这被设置为“false”，Spring中的JmsTemplate的默认值为false，所以甚至可以忽略此属性。

上面显示的数据源bean定义看起来类似于Atomikos，但主要的区别是bean的创建是使用实例工厂方法而不是静态工厂方法完成的。 在这种情况下，使用工厂bean dataSourceBean1创建Db1DataSource bean。 指定的工厂方法是createResource。

transactionManager bean引用BitonixTransactionManager，用于transactionManager属性和userTransaction属性。

要运行此用例，请查看Antik文件中的AtomikosSender任务和BitronixSender，作为项目下载的一部分提供。

此用例的顺序图（不一定是全面的）如下所示：

*图5: 时序图-说明了用例2的处理流程.*

{% asset_img usecase2-seqdiagram-100159022-orig.png %}

*图 4: 用例3中使用JMS消息并更新全局事务中的数据库.*

{% asset_img usecase3-100159023-orig.png %}

此用例与先前的用例不同，它所做的是定义一个POJO以处理从消息中间件接收到的消息，并在同一事务中更新数据库，如上图4所示.

我们的MessageHandler类的相关代码如下：

```java
public void handleOrder(String msg) {
  log.debug("Receieved message->: " + msg);
  MessageSequenceDAO dao = (MessageSequenceDAO) MainApp.springCtx.getBean("sequenceDAO");
  String app = "spring";
  String appKey = "allocation";
  int upCnt = dao.updateSequence(value++, app, appKey);
  log.debug("Update SUCCESS!! Val: " + value + " updateCnt->"+ upCnt);
  if (fail)
   throw new RuntimeException("Rollback TESTING!!");
 }
```

正如你所看到的，代码只是更新数据库mydb2。 MessageHandler只是一个POJO，它有一个handleOrder（）方法。 通过使用Spring我们将把它转换成消息驱动的POJO（类似于J2EE服务器中的MDB）。 要完成这个，我们将使用MessageListenerAdapter类，它通过反射将消息处理委托给目标侦听方法。 这是一个非常方便的功能，它使简单的POJO可以转换为消息驱动的POJO（beans）。 我们的MDP现在支持分布式事务。

现在是我们查看更加清晰的配置的时候了：

```xml
<bean id="msgHandler" class="com.findonnet.messaging.MessageHandler"/>
<bean id="msgListener" class="org.springframework.jms.listener.adapter.MessageListenerAdapter">
        <property name="delegate" ref="msgHandler"/>
        <property name="defaultListenerMethod" value="handleOrder"/>
</bean>
```

上面的配置表明，`msgListener` bean将调用委托给由`msgHandler`定义的bean。 此外，我们已经指定了`handleOrder()`，当消息从消息中间件到达时该方法会被调用。 这是使用`defaultListenerMethod`属性完成的。 现在让我们看看消息侦听器，它侦听消息提供器上的目的地：

```xml
<bean id="listenerContainer" class="org.springframework.jms.listener.DefaultMessageListenerContainer" 
  destroy-method="close">
    <property name="concurrentConsumers" value="1"/>
    <property name="connectionFactory" ref="queueConnectionFactoryBean" />
    <property name="destination" ref="appJmsDestination"/>
    <property name="messageListener" ref="msgListener"/>
    <property name="transactionManager" ref="transactionManager"/>
    <property name="sessionTransacted" value="false"/>
    <property name="receiveTimeout" value="5000"/>
    <property name="recoveryInterval" value="6000"/>
    <property name="autoStartup" value="false"/>
</bean>
```

在这种情况下，listenerContainer使用Spring提供的DefaultMessageListenerContainer。 concurrentConsumers属性设置为“1”，这表示我们将只有一个消费者在队列上。此属性主要用于通过产生多个侦听器（线程）并发排队，并且在您具有快速生产者和慢消费者且消息的排序不重要的情况下非常有用。对于Spring 2.0.3及更高版本，支持使用maxConcurrentConsumers属性动态调整基于负载的侦听器数量。 recoveryInterval属性用于恢复目的，并且在消息传递提供程序关闭时非常有用，我们希望在不使应用程序关闭的情况下重新连接。但是，此功能运行在无限循环中，并且只要应用程序正在运行，就可能不再需要重试。此外，必须小心正确处置DMLC，因为有后台线程，即使在JVM关闭后，仍然可能尝试从消息提供程序接收消息。从Spring 2.0.4，这个问题已经修复。如前所述，只有Atomikos的sessionTransacted属性应设置为“true”。 listenerContainer的相同bean定义适用于Atomikos和Bitronix。

请注意，transactionManger属性指向上面定义的bean定义（对于usecase2），我们只是重复使用相同的bean定义。

这就是所有的，我们只是实现了我们的MDP，它接收消息并在单个全局事务中更新数据库。

要运行此用例，请查看Ant构建文件中的AtomikosConsumer任务和BitronixConsumer，作为项目下载的一部分提供。

### 一些切面

为了拦截调用，在Spring框架和JTA代码之间，已经使用了一个Interceptor类，并编织到JTA实现jar文件的运行时库中。 它使用AspectJ，代码片段如下所示：

```java
pointcut xaCalls() : call(*  XAResource.*(..)) 
                   || call(*  TransactionManager.*(..))
                   || call(*  UserTransaction.*(..))
                   ;
 
  Object around() : xaCalls() {
   log.debug("XA CALL -> This: " + thisJoinPoint.getThis());
   log.debug("       -> Target: " + thisJoinPoint.getTarget());
   log.debug("       -> Signature: " + thisJoinPoint.getSignature());
   Object[] args = thisJoinPoint.getArgs();
   StringBuffer str = new StringBuffer(" ");
   for(int i=0; i< args.length; i++) {
    str.append(" [" + i + "] = " + args[i]);
   }
   log.debug(str);
   Object obj = proceed();
   log.debug("XA CALL RETURNS-> " + obj);
   return obj;
}   
```   

上面的代码定义了对所有JTA相关代码的所有调用的切入点，它还定义了一个around通知，它记录正在传递的参数和方法的返回值。 当我们尝试跟踪和调试JTA实现的问题时，这将派上用场。 项目中的ant build.xml文件（参见参考资料）包含了针对JTA实现编织这些方面的任务。 另一个选择是使用eclipse的MaintainJ插件，它从IDE（Eclipse）的舒适性提供相同的，甚至生成流程图的顺序图。

### 一些问题

分布式事务是一个非常复杂的主题，应该注意事务恢复足够强大并提供用户或应用程序所需的所有ACID（原子性，一致性，隔离和持久性）标准的实现。 我们在本文中测试的是针对pre-2PC异常（记住我们抛出的RuntimeException测试回滚）。 应用程序应该在两阶段提交过程中彻底测试JTA实现的故障，因为它们是最关键和麻烦的。

我们查看的所有JTA实现提供恢复测试用例，这使得它易于根据实现本身和参与的XA资源运行。请注意，使用XA可能会成为一个巨大的性能问题，特别是当事务变大时。还应该看到支持2PC优化，如“最后一次资源提交”，这可能适合一些应用程序需要，其中只有一个参与资源不能或不需要支持2PC。应注意数据库或消息中间件支持的XA功能和施加的限制（如果有的话）。例如，MySQL不支持suspend()和resume()操作，并且似乎对使用XA有一些限制，在某些情况下甚至可能保持数据处于不一致的状态。要了解有关XA的更多信息，*事务处理原则*是一本非常好的书，其中详细介绍了2PC故障条件和优化。此外，Mike Spille的博客（参见参考资料部分）是另一个很好的资源，它专注于JTA上下文中的XA，并提供丰富的信息，尤其是在2PC期间的故障，并有助于了解有关XA事务的更多信息。

当使用Spring框架发送和接收JMS消息时，在非J2EE环境中使用`JmsTemplate`和`DefualtMessageListenerContainer`应当谨慎。在JmsTemplate的情况下，对于发送的每个消息，将创建一个新的JMS连接。在接收消息时使用`DefaultMessageListenerContainer`的情况也是如此。创建重型JMS连接非常耗费资源，并且应用程序在重负载下可能无法良好扩展。一个办法是从JMS消息中间件提供方或第三方库中寻找支持连接/会话池的客户端库。另一个选项是使用Spring中的SingleConnnectionFactory，这使得配置单个连接变得容易，可以重新使用。然而，为每个正在发送的消息创建会话，并且这可能不是真正的开销，因为JMS会话是轻量级的。相同的情况是当消息被接收时，不管它们是否是事务性的。

### 结论

在本文中，我们看到Spring框架如何与JTA实现集成以提供分布式事务，以及如何满足需要分布式事务而不需要完整J2EE服务器的应用程序的需求。 文章还展示了Spring框架如何提供基于POJO的解决方案和声明式事务管理，同时最小化入侵，同时促进最佳设计实践。 我们看到的用例也演示了Spring如何为我们提供丰富的事务管理抽象，使我们能够轻松地在不同的JTA提供程序之间无缝切换。

### 关于作者

Murali Kosaraju在北卡罗来纳州夏洛特的Wachovia银行担任技术架构师。 他拥有系统和信息工程硕士学位，他的兴趣包括消息传递，面向服务的体系结构（SOA），Web服务，J2EE和.NET为中心的应用程序。 他目前住在他的妻子Vidya和儿子Vishal在南卡罗来纳州。

### 更多参考资料

[Download the source code (8M Zip file)](http://images.techhive.com/downloads/idge/imported/article/jvw/2007/04/xaspring.zip).

[X/Open XA](http://www.opengroup.org/onlinepubs/009680699/toc.pdf)

[JTA API](http://java.sun.com/products/jta/)

[Spring Framework](http://www.springframework.org/)

[ActiveMQ](http://activemq.apache.org/)

[Mike Spille's Blog](http://jroller.com/page/pyrasun?catname=%2FXA)

[MySQL XA issues](http://dev.mysql.com/doc/refman/5.0/en/xa-restrictions.html)

JTA Implementations:

1. [JBossTS](http://www.jboss.com/products/transactions)

2. [Atomikos](http://www.atomikos.com/)

3. [Bitronix](http://www.bitronix.be/Btm/Overview)

[Tibco Software](http://www.tibco.com/)

[Principles of Transaction Processing](http://books.google.com/books?id=G-e7tvWJxZoC&dq=principles+of+transaction+processing&pg=PP1)



