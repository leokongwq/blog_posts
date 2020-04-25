---
layout: post
comments: true
title: J2EE事务管理详解
date: 2017-01-03 14:15:25
tags:
- java
- spring
categories:
- 分布式事务
- spring
---

本文翻译自[http://www.javaworld.com/article/2076126/java-se/transaction-management-under-j2ee-1-2.html](http://www.javaworld.com/article/2076126/java-se/transaction-management-under-j2ee-1-2.html)

### 前言

事务可以被定义为由若干操作组成的不可分割的工作单元，为了保持数据完整性，必须执行所有或不必执行这些操作。 例如，将00从您的支票帐户转移到您的储蓄帐户将包括两个步骤：将您的支票帐户记入00，并将您的储蓄帐户记入00.为了保护数据完整性和一致性 - 以及银行和 客户 - 这两个操作必须一起应用或根本不应用。 因此，它们构成一项交易。

<!-- more -->

### Properties of a transaction

所有事务共享这些属性：原子性，一致性，隔离和持久性（由首字母缩写ACID表示）。

- **Atomicity**: 这意味着不可分割; 任何不可分割的操作（将完全完成或根本不完全）被称为是原子的。
- **Consistency**: 事务必须将持久性数据从一个一致状态转换到另一个状态。 如果在处理期间发生故障，则必须将数据恢复到事务之前的状态。
- **Isolation**: 交易不应互相影响。 正在进行，尚未提交或回滚的事务（这些术语在本节结尾处解释）必须与其他事务隔离。 虽然几个事务可以同时运行，但是它们应该在每个事务之前或之后完成。 所有这些并发事务必须有效地按顺序结束。
- **Durability**: 一旦事务成功提交，该事务提交的状态更改必须持久和持久，尽管之后发生任何故障。

事务因此可以以两种方式结束：提交，事务中的每个步骤的成功执行或回滚，这保证由于这些步骤之一中的错误而不执行任何步骤。

### 事务隔离级别

*隔离级别定义：*

衡量并发事务查看已被另一事务更新但尚未提交的数据的能力。 如果允许其他事务读取尚未提交的数据，那么当事务回滚时，这些事务可能以不一致的数据结束，或者在事务成功提交时不必要地结束等待。

更高的隔离级别意味着更低的并发性和遭遇更大的性能瓶颈的可能性，但也减少了读取不一致数据的机会。 一个好的经验法则是使用满足可接受的性能水平前提的最高隔离级别。 以下是常见的隔离级别，从最低到最高:

- **ReadUncommitted**: 已经更新但尚未由事务提交的数据可以被其他事务读取。
- **ReadCommitted**: 只有事务提交的数据才能被其他事务读取。
- **RepeatableRead**: 只有事务提交的数据才能被其他事务读取，并且只要数据未提交，多次读取将产生相同的结果。
- **Serializable**: 这是最高的隔离级别，确保事务对数据的独占读写访问。 它包括ReadCommitted和RepeatableRead的条件，并规定所有事务串行运行以实现最大的数据完整性。 这会导致最低的性能和最小的并发性。 在本上下文中，术语serializable与Java的对象序列化机制和`java.io.Serializable`接口绝对无关。

### J2EE对事务的支持

Java 2企业版（J2EE）平台包括规范，兼容性测试套件，应用程序开发蓝图和参考实现。 许多供应商提供基于相同规范的应用服务器/实现。 J2EE组件旨在以规范为中心，而不是以产品为中心（它们构建为规范，而不是特定的应用程序服务器产品）。 J2EE应用程序包括可以利用J2EE容器和服务器提供的基础设施服务的组件，因此只需要关注“业务逻辑”。 J2EE支持使用部署描述符提供的声明性属性在目标生产环境中灵活部署和定制。 J2EE旨在保护IT工作，降低应用程序开发成本。 J2EE组件可以在内部构建或从外部代理获得，这可以为您的IT部门带来灵活性和成本优势。

事务支持是J2EE平台提供的一个重要的基础设施服务。 该规范描述了Java事务API（JTA），其主要接口包括`javax.transaction.UserTransaction`和`javax.transaction.TransactionManager`。 UserTransaction暴露给应用程序组件，而J2EE服务器和`JTA TransactionManager`之间的底层交互对应用程序组件是透明的。 `TransactionManager`实现支持服务器对（容器划分）事务边界的控制。 JTA UserTransaction和JDBC的事务支持都可用于J2EE应用程序组件。

J2EE平台支持两个事务管理范例：声明式事务处理和编程式事务处理。

### 声明式事务

声明性事务管理是指通过在部署描述符中指定容器管理的EJB组件的各种方法的事务属性来实现的非程序化的事务边界划分。 这是一种灵活和优选的方法，有助于在不修改任何代码的情况下更改应用程序的事务特征。 实体EJB组件必须使用此容器管理的事务分界。

#### 什么是事务属性?

事务属性支持声明性事务划分，并向容器传达相关EJB组件方法的预期事务行为。 对于容器管理的事务划分，可能有六个事务属性：

- **Required**: 具有此事务属性的方法必须在JTA事务中执行; 根据情况，可以或可以不创建新的事务上下文。 如果调用组件已经与JTA事务相关联，则容器将在所述事务的上下文中调用该方法。 如果没有事务与调用组件相关联，容器将自动创建一个新的事务上下文，并在该方法完成时尝试提交事务。
- **RequiresNew**: 具有此事务属性的方法必须在新事务的上下文中执行。 如果调用组件已经与事务上下文相关联，则挂起该事务，创建新的事务上下文，并且在新事务的上下文中执行该方法，在该事务完成之后，恢复调用组件的事务。
- **NotSupported**: 具有此事务属性的方法不打算作为事务的一部分。 如果调用组件已经与事务上下文相关联，则容器挂起该事务，调用与事务无关的方法，并且在完成该方法后，恢复调用组件的事务。
- **Supports**: 具有此事务属性的方法支持调用组件的事务情境。 如果调用组件没有任何事务上下文，容器将执行该方法，如同其事务属性为NotSupported。 如果调用组件已经与事务上下文相关联，则容器将执行该方法，如同其事务属性为Required。
- **Mandatory**: 具有此事务属性的方法只能从调用组件的事务上下文中调用。 否则，容器将抛出javax.transaction.TransactionRequiredException异常.
- **Never**: 不应该从调用组件的事务上下文中调用具有此事务属性的方法。 否则，容器将抛出java.rmi.RemoteException异常。

同一EJB组件中的方法可能由于优化原因而具有不同的事务属性，因为所有方法可能不需要是事务性的。 具有容器管理的持久性的实体EJB组件的隔离级别是不变的，因为不能更改DBMS默认值。 大多数关系数据库系统的默认隔离级别通常为ReadCommitted。

### 编程式事务处理

编程式事务处理是指在应用程序代码中通过硬编码来处理事务的方式。编程式事务处理是会话EJB，servlet和JSP组件的可行选项。 编程式事务可以是JDBC或JTA事务。 对于容器管理的会话EJB，有可能（尽管不是最少建议）混合JDBC和JTA事务。

#### JDBC 事务

JDBC事务由DBMS的事务管理器控制，JDBC连接实现的。

**java.sql.Connection**

接口 - 支持事务划分。 JDBC连接默认情况下已启用其自动提交标志，导致在执行玩单个SQL语句立即提交事务。 但是，自动提交标志可以通过调用来以编程方式更改。

**setAutoCommit()**

调用该方法并传入false参数可以关闭事务自动提交，之后就可以进行编程式事务控制。

commit()

or

rollback()

因此，JDBC事务使用提交或回滚进行分隔。 特定DBMS的事务管理器可能无法工作在异构数据库中。支持分布式事务的JDBC驱动程序提供的实现

javax.transaction.xa.XAResource

和两个新的接口JDBC 2.0，

和

javax.sql.XAConnection

和

javax.sql.XADataSource

#### JTA事务

JTA事务由J2EE事务管理器控制和协调。 JTA事务可用于所有J2EE组件（servlet，JSP和EJB），用于编程事务划分。 与JDBC事务不同，在JTA事务中，事务上下文在不同组件之间传播，而不需要额外的编程工作。 在支持**分布式两阶段提交协议**的J2EE服务器产品中，JTA事务可以以最少的编码工作跨多个不同数据库的更新。 但是，JTA仅支持平面事务，它没有嵌套（子）事务。

`javax.transaction.UserTransaction`接口定义了允许应用程序定义事务边界并显式管理事务的方法。 UserTransaction实现还提供应用程序组件 - servlet，JSP，EJB（具有bean管理的事务） - 能够以编程方式控制事务边界。 EJB组件可以使用getUserTransaction（）方法通过EJBContext访问UserTransaction。 UserTransaction接口中指定的方法包括begin（），commit（），getStatus（），rollback（），setRollbackOnly（）和setTransactionTimeout（int seconds）。 J2EE服务器提供实现`javax.transaction.UserTransaction`接口的对象，并通过JNDI查找使其可用。使用bean管理的持久性的会话EJB组件和实体EJB组件的隔离级别可以使用setTransactionIsolation（）方法以编程方式更改;但是，不建议在中间事务中更改隔离级别。

### 可选的J2EE事务支持功能

J2EE平台的一些方面是可选的，这可能是由于逐渐演变的标准和引入新的概念（在互联网时间方面）。 例如，在EJB 1.0规范中，实体bean（和容器管理的持久性）是一个相对较新的概念和可选特性。 由于高的市场接受性和需求，对EJB 1.1规范的支持在一年后成为强制性的。 当产品成熟并支持更复杂的特征时，可以将非平凡特征作为规范的强制部分。 以下是一些可选的事务相关方面：

- **Multiple database connections within a transaction context and the two-phase commit protocol**: J2EE 1.2规范不需要J2EE服务器的实现来支持在一个事务上下文访问多个JDBC数据库（并支持两阶段提交协议）。 `javax.transaction.xa.XAResource`接口是基于X / Open CAE规范的行业标准XA接口的Java映射。 （参见[参考资料](http://www.javaworld.com/article/2076126/java-se/transaction-management-under-j2ee-1-2.html?page=2#resources)。）X / Open是一个供应商联盟，旨在定义支持应用程序可移植性的通用应用程序环境。 支持多个JDBC数据源，`javax.transaction.xa.XAResource`，两阶段提交等，在当前规范中是可选的，但下一个版本可能会强制这样的支持。 例如，Sun Microsystems的J2EE参考实现支持使用两阶段提交协议访问同一事务中的多个JDBC数据库。
- **Transactional support for application clients and applets**: J2EE 1.2规范不要求事务支持对应用程序客户端和applet可用。 一些J2EE服务器可能在其J2EE服务器产品中提供这样的支持。 作为设计实践，应该尽可能避免应用程序客户端中的事务管理，以符合瘦客户端和三层模型。 此外，作为珍贵资源的交易必须谨慎地分配。
- **Inter-Web-component transaction context propagation**: J2EE 1.2规范不强制要求在Web组件之间传播事务上下文。 通常，Web组件（如servlet和JSP）需要调用（会话）EJB组件，而不是其他Web组件。

### 事务支持和可移植性

为了组件可移植性，设计师和开发人员必须了解哪些方面的事务支持是强制性的，哪些是可选的。 在J2EE模型中，组件是针对规范编写的，并且意在部署在来自不同供应商的符合J2EE标准的应用程序服务器上。所有这些都是为了保护IT投资和跨J2EE服务器可移植性。 但是，如果一个关键的事务功能需要一个可选的事务功能，请充分注意清楚，明确地和尽早地声明，记录和突出显示依赖。

### 结论

J2EE的声明式事务管理方法比编程式事务管理更优雅。 同时，使用声明性事务管理意味着放弃对隔离级别的控制，因为受限于DBMS中提供的默认级别。 如果必须使用编程式事务划分，则JTA事务通常优先于JDBC事务。 但是，JTS事务不能嵌套。 为了便于移植，请注意J2EE平台中事务支持的可选和强制性方面。 在这种背景下，您的应用程序的特定事务需求自然会决定您选择的事务管理策略。

*Sanjay Mahapatra是Sun认证的Java程序员（JDK 1.1）和架构师（Java Platform 2）。 他目前为Cook Systems International工作，该公司是Java 2平台的咨询和系统集成供应商*

### 更多参考资料

- Designing Enterprise Applications with the Java 2 Platform, Enterprise Edition, Nicholas Kassem (Addison-Wesley, 2000)
[http://www.amazon.com/exec/obidos/ASIN/0201702770/qid%3D962058904/104-2583450-5558345](http://www.amazon.com/exec/obidos/ASIN/0201702770/qid%3D962058904/104-2583450-5558345)
- Java 2 Platform, Enterprise Edition specification, v1.2
[http://java.sun.com/j2ee/](http://java.sun.com/j2ee/)
- Enterprise JavaBeans specification, v1.1
[http://java.sun.com/products/ejb/](http://java.sun.com/products/ejb/)
- X/Open CAE specification -- Distributed Transaction Processing
[http://www.opengroup.org/publications/catalog/tp.htm](http://www.opengroup.org/publications/catalog/tp.htm)
- CORBAservices -- Common Object Services Specification, Object Management Group, Inc.
[http://sisyphus.omg.org/technology/documents/formal/corba_services_available_electro.htm](http://sisyphus.omg.org/technology/documents/formal/corba_services_available_electro.htm)

