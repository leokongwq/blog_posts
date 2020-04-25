---
layout: post
comments: true
title: spring处理分布式事务（使用或不使用XA）
date: 2017-01-02 13:08:06
tags:
- spring
categories:
- spring
- 分布式事务
---

原文地址[http://www.javaworld.com/article/2077963/open-source-tools/distributed-transactions-in-spring--with-and-without-xa.html](http://www.javaworld.com/article/2077963/open-source-tools/distributed-transactions-in-spring--with-and-without-xa.html)

在spring中处理分布式事务我们通常会使用JTA(Java Transaction API)和XA协议，当然你也可以使用其它的解决方案。不同的解决方案取决于你的系统需要处理的资源类型和你在性能，安全性，可靠性和数据一致性间做出的妥协。SpringSource's David Syer guides 给出了在spring环境中处理分布式的七个模式，其中三个是使用XA,剩余的四个没有使用。

<!-- more -->

Spring框架对JTA（Java Transaction API）的支持使应用程序能够[不依赖Java EE容器](http://www.javaworld.com/javaworld/jw-04-2007/ jw-04-xa.html)而使用分布式事务和XA协议。 然而，即使具有这种支持，但是XA的使用是昂贵的，并且可能是不可靠的或难以管理的。 它可能会作为一个受欢迎的惊喜，然而某一类的应用程序可以完全避免使用XA。

为了帮助你理解处理分布式事务的各种方法中涉及的注意事项，我将分析七个事务处理模式，并提供样例代码以来使其具体化。 我将以于安全或可靠性相反的顺序呈现模式，从最常见的具有最高数据完整性和原子性的保证模式开始介绍。 当您沿着列表向下移动时，更多注意事项和限制将被应用。 这些模式也大致与运行时间成想法顺序（从最耗时的开始）。 模式更多是和架构或技术相关的而不是业务模式，所以我不的关注点不在业务示例，只有足够精简代码才能看到每个模式是如何工作的。

需要注意的是只有前三个模式使用了XA协议，因此在关注性能的前提下它们可能不被接受或不可用。我不会像其他人那样过多地讨论XA模式，因为它们在别处已经有很多了，尽管我提供了第一个的简单示例。 通过阅读本文，您将了解到针对分布式事务你可以做什么和不能做什么，以及如何和何时避免使用XA，以及何时不使用。

### 分布式事务和原子性

分布式事务是涉及多个事务性资源的事务。 事务资源的示例是用于与关系型数据库和消息中间件进行通信的连接器。 通常这样的资源有一个看起来像begin（），rollback（），commit（）的API。 在Java的世界中，事务型资源通常表现为由底层平台提供的工厂的产品：对于数据库，它是一个Connection（由DataSource生成）或[Java Persistence API](http://www.javaworld.com/javaworld/jw-01-2008/jw-01-jpa1.html)（JPA）EntityManager; 对于[Java消息服务](http://www.javaworld.com/jw-01-1999/jw-01-jms.html)（JMS），它是一个会话。

在一个典型的示例中, 一个JMS消息会触发一个数据库更新。 如果进行时间线分解, 一个成功的交互可能有如下的步骤：

1. 开始消息事务
2. 接收消息
3. 开始数据库事务
4. 更新数据库数据
5. 提交数据库事务
6. 提交消息事务

如果在更新时发生数据库错误，例如违反数据库约束，则期望的序列将如下所示：

1. 开始消息事务
2. 接收消息
3. 开始数据库事务
4. 更新数据库数据失败
5. 回滚数据库事务
6. 回滚消息事务

在这种情况下，消息在最后一次回滚后返回到中间件，并在某个时间点返回，以在另一个事务中接收。 这通常是一件好事，因为否则您可能没有记录发生故障。 （处理自动重试和处理异常的机制不在本文的范围之内。）

上面两种场景的重要特征是它们是原子的，形成完全成功或完全失败的单个逻辑事务。

但，是什么来保证时间轴看起来像这些序列所表现的一样呢？ 必须在这些事务资源之间进行必要的同步，因此如果一个事务提交，其它的事务都必须提交，反之亦然。 否则，整个事务不是原子的。 事务是分布式的，因为涉及多个资源，并且没有同步它不会是原子的。 分布式事务的技术和概念中的困难都涉及资源的同步（或缺乏资源的同步）。

下面讨论的前三个模式基于XA协议。 因为这些模式已经被广泛使用，我不会在这里详细介绍他们。 熟悉XA模式的人可能想跳到[共享事务资源模式](http://www.javaworld.com/article/2077963/open-source-tools/distributed-transactions-in-spring--with-and-without-xa.html?page=3#strp
)。

### Full XA with 2PC

如果您需要接近于防弹级别的安全保证，以确保您的应用程序的事务在中断（包括服务器崩溃）后恢复，则Full XA是您唯一的选择。 在这种情况下用于同步事务的共享资源是使用XA协议协调有关进程的信息的特殊事务管理器。 在Java中，从开发人员的角度来看，该协议通过JTA UserTransaction暴露。

作为一个系统级接口，XA是一个授权技术，大多数开发人员从来没有看到。 他们需要知道它在那里，它能够实现什么，它的成本，以及它们如何使用事务资源的影响。 成本来自事务管理器使用的[两阶段提交](http://www.javaworld.com/jw-07-2000/jw-0714-transaction.html)（2PC）协议，以确保所有资源在交易结束前就交易的结果达成一致。

如果应用程序是基于Spring的，它使用Spring的`JtaTransactionManager`和Spring的声明式事务管理来隐藏基础同步的细节。开发人员在使用XA和不使用XA之间的区别是关于配置工厂资源：DataSource实例和应用程序的事务管理器。本文包括一个演示此配置的示例应用程序（atomikos-db项目）。 DataSource实例和事务管理器是应用程序的唯一XA或JTA特定元素。

要查看示例是如何工作的，请运行`com.springsource.open.db`下的单元测试。 一个简单的MulipleDataSourceTests类只是将数据插入两个数据源，然后使用Spring集成支持功能来回滚事务，如清单1所示：

#### Listing 1. Transaction rollback

```java
@Transactional
  @Test
  public void testInsertIntoTwoDataSources() throws Exception {

    int count = getJdbcTemplate().update(
        "INSERT into T_FOOS (id,name,foo_date) values (?,?,null)", 0,
        "foo");
    assertEquals(1, count);

    count = getOtherJdbcTemplate()
        .update(
            "INSERT into T_AUDITS (id,operation,name,audit_date) values (?,?,?,?)",
            0, "INSERT", "foo", new Date());
    assertEquals(1, count);
    // Changes will roll back after this method exits
}
```

下面演示`MulipleDataSourceTests`验证这两个操作都回滚了，如清单2所示：

#### Listing 2. Verifying rollback

```java
@AfterTransaction
  public void checkPostConditions() {

    int count = getJdbcTemplate().queryForInt("select count(*) from T_FOOS");
    // This change was rolled back by the test framework
    assertEquals(0, count);

    count = getOtherJdbcTemplate().queryForInt("select count(*) from T_AUDITS");
    // This rolled back as well because of the XA
    assertEquals(0, count);
  }
```

为了更好的理解spring的事务管理器是如何工作的和如何配置，可以参考 [Spring Reference Guide](http://static.springframework.org/spring/docs/2.5.x/reference/new-in-2.html#new-in-2-middle-tier).

### XA with 1PC Optimization

此模式是许多事务管理器在事务只包含单个资源的情况下为了避免2PC的开销而进行的优化。 你会期望你的应用服务器能够明白这一点。

### XA and the Last Resource Gambit

许多XA事务管理器的另一个特点是，当除了一个资源都具有XA能力之外，它们仍然可以提供相同的恢复保证，他们通过对资源排序并使用非XA资源作为决定性投票来做到这一点。 如果它无法提交，则所有其他资源可以回滚。 它接近100％防弹。 当它失败时不会因失败而留下大量的跟踪信息，除非采取了额外的步骤（如在一些顶层实现中所做的）。

### 共享事务资源模式

在一些系统中降低复杂性和增加吞吐量的非常重要的模式是通过确保系统中的所有的事务资源实际上由相同资源来支持来的，以此来完全消除对XA的需要。 这显然不可能适用于所有的处理用例中，但是它却像XA一样坚实，并且通常也快得多。 共享事务资源模式是防弹的，但特定于某些平台和处理场景。

此模式的一个简单而熟悉的示例是在对象关系映射（ORM）的组件与[JDBC](http://www.javaworld.com/javaworld/jw-05-2006/jw-0501-jdbc.html)组件之间共享数据库连接。 这就是你使用支持ORM工具（如[Hibernate](http://www.javaworld.com/javaworld/jw-10-2004/jw-1018-hibernate.html)，[EclipseLink](http://www.eclipse.org/eclipselink/)和[Java持久性API](http://www.javaworld.com/javaworld/jw-01-2008/jw-01-jpa1.html)（JPA））的Spring事务管理器所做的。 相同的事务可以安全地用于整个ORM和JDBC组件，通常由上面的服务级别方法执行，其中事务被控制。

此模式的另一个有效使用是消息驱动更新单个数据库的情况（如本文介绍中的简单示例所示）。消息中间件系统需要将其数据存储在某个地方，通常在关系数据库中。要实现这种模式，所需要的是将消息传递系统指向业务数据进入的同一个数据库。此模式依赖于消息传递中间件供应商公开其存储策略的详细信息，以便可以将其配置为指向同一数据库并挂接到同一事务中。

不是所有的消息中间件供应商都能这么容易实现此模式。另一种方法适用于几乎任何数据库，它使用[Apache ActiveMQ](http://activemq.apache.org/)进行消息传递，并将存储策略插入到消息代理中。这是相当容易配置（一旦你知道如何配置）。它在本文的shared-jms-db示例项目中进行了演示。应用程序代码（在这种情况下为单元测试）不需要知道这个模式正在使用，因为它在Spring配置中都是声明性地启用的。

样例中的单元测试称为`SynchronousMessageTriggerAndRollbackTests`，验证一切是否正在使用同步消息接收。 testReceiveMessageUpdateDatabase方法接收两个消息，并使用它们在数据库中插入两个记录。当此方法退出时，测试框架回滚事务，因此您可以验证消息和数据库更新是否都回滚，如清单3所示：

#### 清单3 验证消息回滚和数据库更新

```java
@AfterTransaction
public void checkPostConditions() {

  assertEquals(0, SimpleJdbcTestUtils.countRowsInTable(jdbcTemplate, "T_FOOS"));
  List<String> list = getMessages();
  assertEquals(2, list.size());

}
```

ActiveMQ配置中最重要的功能是持久性策略配置，将消息传递系统链接到与业务数据相同的DataSource，以及Spring JmsTemplate上用于接收消息的标志。 清单4显示了如何配置ActiveMQ持久性策略：

#### 清单4 配置 ActiveMQ 持久化策略

```java
<bean id="connectionFactory" class="org.apache.activemq.ActiveMQConnectionFactory"
  depends-on="brokerService">
  <property name="brokerURL" value="vm://localhost?async=false" />
</bean>
<bean id="brokerService" class="org.apache.activemq.broker.BrokerService" init-method="start"
  destroy-method="stop">
    ...
  <property name="persistenceAdapter">
    <bean class="org.apache.activemq.store.jdbc.JDBCPersistenceAdapter">
      <property name="dataSource">
        <bean class="com.springsource.open.jms.JmsTransactionAwareDataSourceProxy">
          <property name="targetDataSource" ref="dataSource"/>
          <property name="jmsTemplate" ref="jmsTemplate"/>
        </bean>
      </property>
      <property name="createTablesOnStartup" value="true" />
    </bean>
  </property>        
</bean>
```

清单5显示了Spring JmsTemplate上用于接收消息的标志：

#### 清单5 配置事务所使用的JmsTemplate

```java
<bean id="jmsTemplate" class="org.springframework.jms.core.JmsTemplate">
  ...
  <!-- This is important... -->
  <property name="sessionTransacted" value="true" />
</bean>
```

如果没有`sessionTransacted = true`，则永远不会进行JMS会话事务API调用，并且不能回滚消息接收。这里的重要成分是具有特殊`async = false`参数的嵌入式代理和用于DataSource的包装器，它们一起确保ActiveMQ使用与Spring相同的事务JDBC连接。

共享数据库资源有时可以从现有的单独资源合成，特别是如果它们都在同一个RDBMS平台中。企业级数据库供应商都支持同义词（或等价物）的概念，其中一个模式中的表（使用Oracle术语）在另一个模式中被声明为同义词。这样，在平台中物理分区的数据可以从JDBC客户端中的同一个连接事务处理。例如，在实际系统（与示例相反）中使用ActiveMQ实现共享资源模式通常涉及创建消息传递和业务数据的同义词。

### 性能和JDBCPersistenceAdapter

ActiveMQ社区中的一些人声称JDBCPersistenceAdapter会导致性能问题。然而，许多项目和线上系统使用ActiveMQ和关系数据库。在这些情况下，收到的智慧是，适配器的日志版本应该用于提高性能。这不适合共享资源模式（因为日志本身是一个新的事务资源）。但是，陪审团可能仍然在JDBCPersistenceAdapter上。确实有理由认为使用共享资源可能会提高性能超过日记的情况。这是Spring和ActiveMQ工程团队积极研究的领域。

非消息场景（多个数据库）中的另一种共享资源技术是使用Oracle数据库链接功能将两个数据库模式在RDBMS平台级别链接在一起（请参阅参考资料）。这可能需要更改应用程序代码或创建同义词，因为引用链接数据库的表名别名包括链接的名称。


### 最大努力 1PC pattern

最大努力1PC模式是相当常见的，但在某些情况下可能会失败，开发人员必须知道。这是一个非XA模式，涉及多个资源的同步单阶段提交。因为没有使用2PC，它永远不会像XA事务一样安全，但是如果参与者知道妥协，通常是足够好的。许多大容量，高吞吐量事务处理系统以这种方式设置以提高性能。

基本思想是在事务中尽可能晚地延迟所有资源的提交，以便唯一可能出错的是基础结构故障（而不是业务处理错误）。依赖Best Efforts的系统1PC的原因是基础设施故障很少，他们可以负担风险，以获得更高的吞吐量。如果业务处理服务也被设计为幂等的，那么在实践中很少会出错。

为了帮助您更好地理解模式并分析失败的后果，我将使用消息驱动的数据库更新作为示例。

此事务中的两个资源被计入和计数。 消息事务在数据库事务之前启动，它们以相反的顺序结束（提交或回滚）。 所以在成功的情况下的顺序可能与本文开头的一样：

1. 开始消息事务
2. **接收消息**
3. 开始数据库事务
4. **更新数据库**
5. 提交数据库事务
6. 提交消息事务

实际上，前四个步骤的顺序并不重要，除了必须在更新数据库之前接收到消息，并且每个事务必须在使用相应资源之前启动。 所以这个序列也是有效的：

1. 开始消息事务
2. 开始数据库事务
3. **接收消息**
4. **更新数据库**
5. 提交数据库事务
6. 提交消息事务

关键是，最后两个步骤很重要：他们必须按照这个最后这个顺序出现。 排序很重要的原因是因为技术，但顺序本身是由业务需求决定的。 顺序告诉你这种情况下的事务资源之一是特殊的; 它包含有关如何执行其他工作的说明。 这是一个业务排序：系统不能自动告诉它是哪个方式（虽然如果消息和数据库是两个资源，那么它通常在这个顺序）。 排序的重要性的原因与失败的情况有关。 最常见的故障情况（到目前为止）是业务处理的故障（坏数据，编程错误等）。 在这种情况下，两个事务都可以轻松地进行操作，以响应异常和回滚。 在这种情况下，保留业务数据的完整性，时间线类似于本文开头所述的理想故障情况。

触发回滚的精确机制是不重要的;几个是可用的。重要的一点是，提交或回滚按照资源中业务排序的相反顺序发生。在示例应用程序中，消息传递事务必须最后提交，因为业务流程的说明包含在该资源中。这是重要的，因为（罕见的）第一次提交成功，第二次失败的故障情况。因为通过设计，所有的业务处理在这一点上已经完成，所以这种部分故障的唯一原因将是消息中间件的基础设施问题。

注意，如果数据库资源的提交失败，那么净效果仍然是回滚。因此，唯一的非原子故障模式是第一个事务提交并且第二个回滚的模式。更一般地，如果事务中存在n个资源，则存在n-1个这样的故障模式，使得一些资源在回滚之后处于不一致（提交）状态。在消息数据库用例中，此故障模式的结果是消息被回滚并在另一个事务中返回，即使它已经被成功处理。所以你可以放心地假设可能发生的更糟糕的事情是可以传递重复的消息。在更一般的情况下，因为事务中较早的资源被认为潜在地携带关于如何对稍后的资源执行处理的信息，故障模式的净结果通常可以被称为重复消息。

有些人承担重复邮件发生的频率不够高的风险，他们不打算试图预测他们。 然而，为了更有信心地确定业务数据的正确性和一致性，您需要在业务逻辑中了解它们。 如果业务处理意识到重复消息可能到达，它所需要做的（通常是一些额外的成本，但不是2PC），检查它是否已经处理了该数据，如果它有什么，什么都不做。 这种专业化有时被称为幂等商业服务模式。

示例代码包括使用此模式同步事务资源的两个示例。 我将依次讨论每一个，然后检查一些其他选项。

### Spring和消息驱动的POJOs

在[样例代码](http://images.techhive.com/downloads/idge/imported/article/jvw/2009/01/springxa-src.zip)的best-jms-db项目中，参与者设置 使用主流配置选项，以便遵循Best Efforts 1PC模式。 想法是，发送到队列的消息由异步侦听器拾取，并用于将数据插入到数据库中的表中。

TransactionAwareConnectionFactoryProxy - Spring中的库存组件，设计用于此模式 - 是关键要素。 配置不使用原始供应商提供的ConnectionFactory，而是在处理事务同步的装饰器中封装ConnectionFactory。 这发生在jms-context.xml中，如清单6所示：

#### 清单6. 配置`TransactionAwareConnectionFactoryProxy`以包装供应商提供的JMS ConnectionFactory

```java
<bean id="connectionFactory" class="org.springframework.jms.connection.TransactionAwareConnectionFactoryProxy">
  <property name="targetConnectionFactory">
    <bean class="org.apache.activemq.ActiveMQConnectionFactory" depends-on="brokerService">
      <property name="brokerURL" value="vm://localhost"/>
    </bean>
  </property>
  <property name="synchedLocalTransactionAllowed" value="true" />
</bean>
```

没有必要让ConnectionFactory知道要同步哪个事务管理器，因为只有一个事务处于活动状态，在需要的时候，Spring可以在内部处理它。 驱动事务由在data-source-context.xml中配置的正常DataSourceTransactionManager处理。 需要了解事务管理器的组件是将轮询和接收消息的JMS侦听器容器：

```java
<jms:listener-container transaction-manager="transactionManager" >
  <jms:listener destination="async" ref="fooHandler" method="handle"/>
</jms:listener-container>
```

fooHandler和方法告诉侦听器容器当消息到达“异步”队列时，哪个组件调用哪个方法。 处理程序是这样实现的，接受一个String作为传入消息，并使用它插入一个记录：

```java
public void handle(String msg) {

  jdbcTemplate.update(
      "INSERT INTO T_FOOS (ID, name, foo_date) values (?, ?,?)", count.getAndIncrement(), msg, new Date());
  
}
```

为了模拟故障，代码使用FailureSimulator切面。 它检查消息内容，看它是否应该失败，以及以什么方式。 清单7中所示的maybeFail（）方法在FooHandler处理消息之后，但在事务结束之前调用，以便影响事务的结果：

#### 清单7.方法maybeFail()

```java
@AfterReturning("execution(* *..*Handler+.handle(String)) && args(msg)")
public void maybeFail(String msg) {
  if (msg.contains("fail")) {
    if (msg.contains("partial")) {
      simulateMessageSystemFailure();
    } else {
      simulateBusinessProcessingFailure();
    }
  }
}
```

simulateBusinessProcessingFailure（）方法只是抛出一个DataAccessException，就像数据库访问失败一样。触发此方法时，您期望完全回滚所有数据库和消息事务。此示例在示例项目的AsynchronousMessageTriggerAndRollbackTests单元测试中进行测试。

simulateMessageSystemFailure（）方法通过削弱底层JMS会话来模拟消息传递系统中的失败。这里的预期结果是部分提交：数据库工作保持提交，但消息回滚。这在AsynchronousMessageTriggerAndPartialRollbackTests单元测试中测试。

示例包还包括在AsynchronousMessageTriggerSunnyDayTests类中成功提交所有事务性工作的单元测试。

相同的JMS配置和相同的业务逻辑也可以在同步设置中使用，其中消息在业务逻辑内的阻塞调用中接收，而不是委派给侦听器容器。这种方法也在best-jms-db示例项目中演示。晴天情况和完全回滚分别在SynchronousMessageTriggerSunnyDayTests和SynchronousMessageTriggerAndRollbackTests中测试。

### 链式事务管理

在Best Efforts 1PC模式（best-db-db项目）的另一个示例中，事务管理器的粗略实现仅链接其他事务管理器的列表以实现事务同步。 如果业务处理成功，他们都提交，如果不是，他们都回滚。

实现在ChainedTransactionManager中，它接受其他事务管理器的列表作为注入属性，如清单8所示：

#### 清单8. 配置ChainedTransactionManager

```java
<bean id="transactionManager" class="com.springsource.open.db.ChainedTransactionManager">
  <property name="transactionManagers">
    <list>
      <bean
        class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource" />
      </bean>
      <bean
        class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="otherDataSource" />
      </bean>
    </list>
  </property>
</bean>
```

这个配置的最简单的测试只是在两个数据库中插入东西，回滚，并检查两个操作都没有留下痕迹。 这被实现为MulipleDataSourceTests中的单元测试，与XA示例的atomikos-db项目相同。 如果回滚未同步但提交正在运行，测试将失败。

记住资源的顺序是重要的。 它们是嵌套的，提交或回滚发生的顺序与它们登记的顺序相反（这是配置中的顺序）。 这使得其中一个资源特别：如果有问题，最外面的资源总是回滚，即使唯一的问题是该资源的故障。 此外，testInsertWithCheckForDuplicates（）测试方法显示一个幂等的业务流程，保护系统免受部分故障。 它被实现为内部资源（在这种情况下为otherDataSource）的业务操作中的防御性检查：

```java
int count = otherJdbcTemplate.update("UPDATE T_AUDITS ... WHERE id=, ...?");
if (count == 0) {
  count = otherJdbcTemplate.update("INSERT into T_AUDITS ...", ...);
}
```

首先使用where子句尝试更新。 如果没有任何反应，您希望在更新中找到的数据将被插入。 在这种情况下，来自幂等过程的额外保护的成本是在晴天情况下的一个额外查询（更新）。 在更复杂的业务流程中，这种成本相当低，其中每个事务执行许多查询。

### 其它建议

示例中的ChainedTransactionManager具有简单的优点;它不会打扰许多可用的扩展和优化。另一种方法是在第二个资源加入时，使用Spring中的TransactionSychronization API为当前事务注册一个回调。这是best-jms-db示例中的方法，其中关键特性是TransactionAwareConnectionFactory与DataSourceTransactionManager的组合。这种特殊情况可以扩展并通用包括使用TransactionSynchronizationManager的非JMS资源。优点是，原则上只有加入该交易的那些资源将被登记，而不是链中的所有资源。然而，配置仍然需要知道潜在事务中的哪些参与者对应于哪些资源。

此外，Spring工程团队正在为Spring Core考虑一个“Best Efforts 1PC事务管理器”功能。如果您喜欢该模式，并且希望在Spring中看到对它的明确和更透明的支持，您可以投票支持JIRA问题。

### 非事务访问模式

非事务访问模式需要一种特殊的业务流程才有意义。这个想法是，有时您需要访问的资源之一是边缘的，不需要在事务中。例如，您可能需要向审计表中插入一行，该行与业务事务是否成功无关;它只是记录了做某事的企图。更常见的是，人们高估了他们需要对一个资源进行读写更改的程度，而且往往只读访问是很好的。否则写操作可以被仔细地控制，使得如果任何错误，它可以被考虑或忽略。

在这些情况下，保持在事务之外的资源可能实际上具有它自己的事务，但是它不与正在发生的任何事物同步。如果您使用Spring，则主要事务由PlatformTransactionManager驱动，边缘资源可能是从不由事务管理器控制的DataSource获取的数据库连接。所有这一切都发生在每个访问边缘资源的默认设置autoCommit = true。读操作不会看到在另一个未提交的事务中同时发生的更新（假设合理的默认隔离级别），但是写操作的效果通常会被其他参与者立即看到。

这种模式需要更仔细的分析，更多的设计业务流程的信心，但它不是完全不同于Best Efforts 1PC。 当出现任何问题时，提供补偿事务的通用服务对于大多数项目来说太大目标。 但是简单的使用情况涉及服务是幂等的并且只执行一个写操作（并且可能多次读）并不罕见。 这些是非事务性游戏的理想情况。

### 翼和祷告：反模式

最后一个模式是一个反模式。 它往往发生在开发人员不了解分布式事务或不知道他们有一个。 如果没有显式调用底层资源的事务API，你不能只是假设所有的资源都加入一个事务。 如果你使用除JtaTransactionManager之外的Spring事务管理器，它将有一个事务资源附加到它。 该事务管理器将用于使用Spring声明性事务管理功能（如@Transactional）来拦截方法执行。 不能期望在同一事务中登记其他资源。 通常的结果是一切都在阳光明媚的日子很好，但只要有一个异常，用户发现其中一个资源没有回滚。 导致此问题的典型错误是使用DataSourceTransactionManager和使用Hibernate实现的存储库。

### 该使用哪个模式呢?

我将通过分析引入的模式的利弊得出结论，帮助您了解如何在它们之间做出决定。第一步是识别您有一个需要分布式事务的系统。一个必要的（但不是足够的）条件是存在具有多于一个事务资源的单个进程。一个足够的条件是这些资源在单个用例中一起使用，通常由对您的架构中的服务级别的调用驱动。

如果你没有认识到分布式交易，你可能已经实现了翼与祷告模式。迟早，你会看到应该已回滚但不是回滚的数据。可能当你看到效果，它将是一个很长的路下游实际失败，很难追溯。开发人员可能会不小心使用Wing-and-a-Prayer，他们认为他们受到XA的保护，但没有配置基础资源来参与事务。我在一个项目上工作，其中数据库已被另一个组安装，并且他们在安装过程中关闭了XA支持。一切都运行了好几个月，然后奇怪的失败开始渗透到业务流程。诊断问题需要很长时间。

如果你的混合资源的用例很简单，你可以做分析和一些重构，那么非事务资源模式可能是一个选项。当其中一个资源主要是读取时，这种方式最有效，写入操作可以通过对重复项的检查来防护。即使在失败之后，非事务性资源中的数据也必须在业务术语中有意义。审核，版本控制和日志记录信息通常适用于此类别。失败将是相对常见的（任何时候，任何事物在真正的事务回滚），但你可以相信没有副作用。

最佳努力1PC适用于需要更多保护以防止常见故障，但不希望2PC的开销的系统。性能改进可能很重要。设置比非事务性资源更为棘手，但它不需要那么多的分析，并且用于更通用的数据类型。完全确定数据一致性要求业务处理对“外部”资源（任何第一个提交）都是幂等的。消息驱动的数据库更新是一个完美的例子，在Spring已经有相当好的支持。更不寻常的情况需要一些额外的框架代码（它可能最终是Spring的一部分）。

共享资源模式适用于特殊情况，通常涉及特定类型和平台的两个资源（例如，ActiveMQ与任何RDBMS或Oracle AQ与Oracle数据库位于同一位置）。 其优点是极强的鲁棒性和卓越的性能。

### 样例代码和更新

本文提供的[示例代码]（http://images.techhive.com/downloads/idge/imported/article/jvw/2009/01/springxa-src.zip）将不可避免地显示其年龄作为Spring版本的新版本 并释放其他组件。 请参阅Spring [社区网站]（http://www.springframework.org/）以访问作者的最新代码，以及Spring Framework和相关组件的最新版本。

**具有2PC**的完整XA是通用的，并且将总是给出最高的置信度和最大的保护以防止在使用多个不同资源的情况下的故障。缺点是，它是昂贵的，因为协议规定的额外的I / O（但不要写，直到你尝试它），并需要特殊用途的平台。有开源JTA实现，可以提供一种方法来打破应用程序服务器，但许多开发人员认为他们第二好，仍然。当然，更多的人使用JTA和XA比需要更多的时间来思考他们的系统中的事务边界。至少如果他们使用Spring，它们的业务逻辑不需要知道事务是如何处理的，因此可以延迟平台选择。

Dr. [David Syer]（david.syer@springsource.com）是SpringSource的首席顾问，总部设在英国。他是Spring Batch项目的创始人和首席工程师，这是一个用于构建和配置离线和批处理应用程序的开源框架。他经常主持关于企业Java和行业评论员的会议。最近的出版物出现在服务器端，InfoQ和SpringSource博客。

### 更多参考资料

- [Download the source code](http://images.techhive.com/downloads/idge/imported/article/jvw/2009/01/springxa-src.zip) for this article. Also be sure to visit the [Spring Community Site](http://www.springframework.org/) for the most up-to-date sample code for this article.
- Learn more about javax.transaction from the Java docs for [JTA](http://java.sun.com/javaee/5/docs/api/javax/transaction/package-frame.html) and [XAResource](http://java.sun.com/javase/6/docs/api/javax/transaction/xa/XAResource.html).
- "[XA transactions using Spring](http://www.javaworld.com/javaworld/jw-04-2007/jw-04-xa.html)" (Murali Kosaraju, JavaWorld, April 2007) explains how to set up Spring with JTA outside a Java EE container.
- "[XA Exposed, Part I](http://jroller.com/pyrasun/category/XA)" (Mike Spille, Pyrasun, The Spille Blog, April 2004) is an excellent and entertaining resource for learning about 2PC in more detail.
- Learn more about how Spring transaction management works and how to configure it generally by reading the Spring Reference Guide, [Chapter 9. Transaction management](http://static.springframework.org/spring/docs/2.5.x/reference/transaction.html).
- "[Transaction management for J2EE 1.2](http://www.javaworld.com/jw-07-2000/jw-0714-transaction.html)" (Sanjay Mahapatra, JavaWorld, July 2000) defines the ACID properties of a transaction, including atomicity.
- In "[To XA or not to XA](http://guysblogspot.blogspot.com/2006/10/to-xa-or-not-to-xa.html)" (Guy's Blog, October 2006), Atomikos CTO Guy Pardon advocates for using XA.
- Check out the [Atomikos documentation](http://www.atomikos.com/Documentation/WebHome) to learn about this open source transaction manager.
- "[How to create a database link in Oracle](http://searchoracle.techtarget.com/tip/0,289483,sid41_gci1263933,00.html)" (Elisa Gabbert, SearchOracle.com, January 2004) explains how to create an Oracle database link.
- Weigh in on the [Provide a "best efforts" 1PC transaction manager out of the box](http://jira.springframework.org/browse/SPR-3844) proposal for the Spring Framework.


















