---
layout: post
comments: true
title: 深入理解JTA事务处理
date: 2017-01-03 17:39:08
tags:
- java
- jta
categories:
- 分布式事务
---

### 前言

Java事务API（JTA）允许应用程序执行分布式事务，也就是说事务可以访问或更新两个或更多网络上的计算机资源。 JTA指定事务管理器和分布式事务系统中涉及的各方之间的标准Java接口：应用程序，应用程序服务器和控制对受事务影响的共享资源的访问的资源管理器。 本文档概述了该处理过程以及DataDirectConnect®for JDBC™驱动程序与其相关的方式

一个事务定义了完全成功或根本不产生结果的逻辑工作单元。 分布式事务仅仅是访问和更新两个或更多网络资源上的数据的事务，因此必须在这些资源之间进行协调。 在本文中，我们主要关注涉及关系数据库系统的事务。
<!-- more -->
在分布式事务处理（DTP）模型中相关的组件有如下几个：

- 应用程序
- 应用程序服务器
- 事务管理器
- 资源适配器
- 资源管理器

在以下部分中，我们描述这些组件及其与JTA和数据库访问的关系。

### 访问数据库

最好将分布式事务中涉及的各个组件视为独立的进程，而不是在特定计算机上的位置。 几个组件可以驻留在一个机器上，或者它们可以分布在多个机器中。 以下示例中的图表可能显示特定计算机上的组件，但进程之间的关系是主要考虑因素。

#### 最简单的例子：应用程序和数据库本地事务

关系数据库访问的最简单形式仅涉及应用程序，资源管理器和资源适配器。 应用程序只是向数据库发送请求和从数据库获取数据的最终用户访问点。

在我们的讨论中资源管理器是关系型数据库管理系统（RDBMS），例如Oracle或SQL Server。 所有实际的数据库管理由此组件处理。

资源适配器是在“外部世界”（在这种情况下是应用程序）和资源管理器之间的通信信道或请求转换器的组件。 对我们的讨论，这是一个JDBC驱动程序。

以下描述是资源管理器本地事务，即限于单个特定企业数据库的一个事务。

应用程序向JDBC驱动程序发送数据请求，然后JDBC驱动程序会转换请求并将其通过网络发送到数据库。 数据库将数据返回给驱动程序，然后将结果转换为应用程序，如下图所示：

{% asset_img jta1.gif %}

本示例说明了简化系统中的信息的基本流程; 然而，今天的企业使用应用服务器，这增加了另一个组件到过程。

#### 应用程序服务器

应用程序服务器是JTA分布式事务处理的另一个组件。 应用程序服务器负责处理大量的应用程序操作，并取消终端用户应用程序的一些负载。 基于上述示例，我们看到应用程序服务器在分布式事务中添加了另一个进程层：

{% asset_img jta2.gif %}

到目前为止，我们的例子说明了一个单一的本地事务，并描述了分布式事务模型的五个组件中的四个。 第五个组件，事务管理器，只有当事务是分布式时才会考虑引入。

### 分布式事务和事务管理器

如前所述，分布式事务是访问和更新两个或多个网络资源上的数据的事务。 这些资源可以由单个服务器（例如Oracle，SQL Server和Sybase）上的几个不同的RDBMS组成; 或者它们可以包括驻留在多个不同服务器上的单个类型的数据库的若干实例。 在任何情况下，分布式事务涉及各种资源管理器之间的协调。 这种协调是事务管理器的功能。

事务管理器负责做出提交或回滚任何分布式事务的最终决定。 提交决定应导致成功的事务; 回滚使数据库中的数据保持不变。 JTA在事务管理器和分布式事务中的其他组件之间指定标准Java接口：应用程序，应用程序服务器和资源管理器。 这种关系如下图所示：

{% asset_img jta3.gif %}

事务管理器周围的编号框对应于JTA的三个接口部分：

1. `javax.transaction.UserTransaction`接口为应用程序提供了以编程方式控制事务边界的能力。 `javax.transaction.UserTransaction`方法启动一个全局事务，并将事务与调用线程相关联。
2. `javax.transaction.TransactionManager`接口允许应用服务器代表被管理的应用来控制事务边界。
3. `javax.transaction.xa.XAResource` 接口是基于X / Open CAE规范（分布式事务处理：XA规范）的工业标准XA接口的Java映射。

需要注意的是，我们需要使用支持XAResource接口的JDBC驱动程序。除了正常的JDBC交互还需要支持JTA的XAResource部分。 DataDirect Connect for JDBC驱动程序提供此支持。

应用程序代码的开发人员不应该关心分布式事务管理的细节。 这是分布式事务基础结构的工作，包括应用程序服务器，事务管理器和JDBC驱动程序。 应用程序代码的唯一注意事项是，当连接在分布式事务的范围内时，它不应调用会影响事务边界的方法。 具体来说，应用程序不应调用Connection方法commit，rollback和setAutoCommit（true），因为它们会干扰基础架构对分布式事务的管理。

### 分布式事务处理过程

事务管理器是分布式事务基础设施的主要组件; 但是，JDBC驱动程序和应用程序服务器组件应具有以下特性：

- 驱动程序应该实现JDBC 2.0 API（包括可选软件包接口XADataSource和XAConnection）或更高版本和JTA接口XAResource.
- 应用服务器应提供一个为了与分布式事务基础结构进行交互并用于提高性能的具有连接池功能的实现了接口DataSource的类。

分布式事务处理的第一步是应用程序向事务管理器发送对事务的请求。 虽然最终事务的提交或回滚决定将事务视为单个逻辑单元，但这其中可能包含许多事务分支。 事务分支与分布式事务中涉及的每个资源管理器的请求相关联。 因此，对三个不同RDBMS的请求需要三个事务分支。 每个事务分支必须由本地资源管理器提交或回滚。 事务管理器控制事务的边界，并且负责关于总事务是否应该提交或回滚的最终决定。 该决策分为两个阶段，称为两阶段提交协议。

在第一阶段，事务管理器轮询分布式事务中涉及的所有资源管理器（RDBMS），以查看每个资源管理器是否准备好提交。 如果资源管理器无法提交，它会做出响应并回滚其事务的特定部分，以便不会更改数据。

在第二阶段中，事务管理器确定是否有资源管理器否定响应，并且如果是，则回滚整个事务。 如果没有否定响应，则管理器提交整个事务，并将结果返回给应用程序。

事务管理器代码的开发人员必须熟悉JTA的所有三个接口：`UserTransaction`，`TransactionManager`和`XAResource`，这些接口在[Sun Java Transaction API（JTA）](http://java.sun.com/products/jta/)规范。 [JDBC API教程和参考，第三版](http://java.sun.com/docs/books/jdbc/)也是一个有用的参考。 JDBC驱动程序开发人员只需要关心`XAResource`接口。 此接口是允许资源管理器参与事务的工业标准X / Open XA协议的Java映射。 与`XAResource`接口连接的驱动程序的组件负责在事务管理器和资源管理器之间“转换”。 以下部分提供XAResource调用的示例。

### JDBC驱动和XAResource

为了简化XAResource的解释，这些示例说明了当没有应用程序服务器和事务管理器时，应用程序如何使用JTA。 事实上，这些示例中的应用程序也充当应用程序服务器和事务管理器。 大多数企业使用事务管理器和应用程序服务器，因为它们比起应用程序可以更高效地管理分布式事务。 然而，通过遵循这些示例，应用程序开发人员可以测试JDBC驱动程序中JTA支持的鲁棒性。 某些示例可能不适用于特定数据库，因为是这些数据库固有问题导致的。

在使用JTA之前，你首先必须实现一个Xid类来识别事务（这通常由事务管理器完成）。 Xid包含三个元素：formatID，gtrid（全局事务ID）和bqual（分支限定符ID）。

formatID通常为零，这意味着您正在使用OSI CCR（开放系统互连承诺，并发和恢复标准）进行命名。 如果使用其他格式，formatID应大于零值，-1表示Xid为空。

gtrid和bqual可以各自包含高达64字节的二进制代码，以分别标识全局事务和分支事务。 唯一的要求是，gtrid和bqual一起必须是全局唯一的。 同样，这可以通过使用在OSI CCR中指定的命名规则来实现。

以下示例说明了Xid的实现：

```java
import javax.transaction.xa.*;

public class MyXid implements Xid {
    protected int formatId;
    protected byte gtrid[];
    protected byte bqual[];

    public MyXid() {
    }

    public MyXid(int formatId, byte gtrid[], byte bqual[]) {
        this.formatId = formatId;
        this.gtrid = gtrid;
        this.bqual = bqual;
    }

    public int getFormatId() {
        return formatId;
    }

    public byte[] getBranchQualifier() {
        return bqual;
    }

    public byte[] getGlobalTransactionId() {
        return gtrid;
    }
}
```

其次，您需要为您使用的数据库创建一个数据源：

```java
public DataSource getDataSource() throws SQLException {
    SQLServerDataSource xaDS = new com.ddtek.jdbc.sqlserver.SQLServerDriver.SQLServerDataSource();
    xaDS.setDataSourceName("SQLServer");
    xaDS.setServerName("server");
    xaDS.setPortNumber(1433);
    xaDS.setSelectMethod("cursor");
    return xaDS;
}
```

**Example 1**—此示例使用两阶段提交协议提交一个事务分支：

```java
XADataSource xaDS;
XAConnection xaCon;
XAResource xaRes;
Xid xid;
Connection con;
Statement stmt;
int ret;
xaDS=getDataSource();
xaCon=xaDS.getXAConnection("jdbc_user","jdbc_password");
xaRes=xaCon.getXAResource();
con=xaCon.getConnection();
stmt=con.createStatement();
xid=new MyXid(100,new byte[]{0x01},new byte[]{0x02});
try{
    xaRes.start(xid,XAResource.TMNOFLAGS);
    stmt.executeUpdate("insert into test_table values (100)");
    xaRes.end(xid,XAResource.TMSUCCESS);

    ret=xaRes.prepare(xid);
    if(ret==XAResource.XA_OK){
        xaRes.commit(xid,false);
    }
} catch(XAException e){
    e.printStackTrace();
} finally{
    stmt.close();
    con.close();
    xaCon.close();
}
```

因为对于所有示例，初始化代码是相同的或类似的，所以就省去了。

**Example 2**-示例，类似于示例1，说明回滚：

```java
xaRes.start(xid, XAResource.TMNOFLAGS);
stmt.executeUpdate("insert into test_table values (100)");
xaRes.end(xid, XAResource.TMSUCCESS);
ret = xaRes.prepare(xid);
if (ret == XAResource.XA_OK) {
    xaRes.rollback(xid);
}
```

**Example 3**—此示例显示分布式事务分支如何挂起，让同一连接执行本地事务，并稍后恢复分支。 分布式事务的两阶段提交操作不会影响本地事务。

```java
xid = new MyXid(100, new byte[]{0x01}, new byte[]{0x02});
xaRes.start(xid, XAResource.TMNOFLAGS);
stmt.executeUpdate("insert into test_table values (100)");
xaRes.end(xid, XAResource.TMSUSPEND);
// This update is done outside of transaction scope, so it
// is not affected by the XA rollback.
stmt.executeUpdate("insert into test_table2 values (111)");
xaRes.start(xid, XAResource.TMRESUME);
stmt.executeUpdate("insert into test_table values (200)");
xaRes.end(xid, XAResource.TMSUCCESS);
ret = xaRes.prepare(xid);
if (ret == XAResource.XA_OK) {
    xaRes.rollback(xid);
}
```

**Example 4**—此示例说明如何在不同事务之间共享一个XA资源。 创建了两个事务分支，但它们不属于同一个分布式事务。 JTA允许XA资源在第一个分支上执行两阶段提交，即使资源仍然与第二个分支相关联。

```java
xid1 = new MyXid(100, new byte[]{0x01}, new byte[]{0x02});
xid2 = new MyXid(100, new byte[]{0x11}, new byte[]{0x22});
xaRes.start(xid1, XAResource.TMNOFLAGS);
stmt.executeUpdate("insert into test_table1 values (100)");
xaRes.end(xid1, XAResource.TMSUCCESS);
xaRes.start(xid2, XAResource.TMNOFLAGS);
// Should allow XA resource to do two-phase commit on
// transaction 1 while associated to transaction 2
ret = xaRes.prepare(xid1);
if (ret == XAResource.XA_OK) {
    xaRes.commit(xid1, false);
}
stmt.executeUpdate("insert into test_table2 values (200)");
xaRes.end(xid2, XAResource.TMSUCCESS);
ret = xaRes.prepare(xid2);
if (ret == XAResource.XA_OK) {
    xaRes.rollback(xid2);
}
```

**Example 5**—此示例说明如果不同连接上的事务分支连接到同一资源管理器，则它们可以作为单个分支连接。 此功能提高了分布式事务的效率，因为它减少了两阶段提交进程的数量。 将创建到同一数据库服务器的两个XA连接。 每个连接都创建自己的XA资源，常规JDBC连接和语句。 在第二个XA资源启动事务分支之前，它会检查它是否使用与第一个XA资源使用相同的资源管理器。 如果是这种情况，如在此示例中，它加入在第一个XA连接上创建的第一个分支，而不是创建一个新的分支。 稍后，可以使用XA资源准备和提交事务分支。

```java
xaDS = getDataSource();
xaCon1 = xaDS.getXAConnection("jdbc_user", "jdbc_password");
xaRes1 = xaCon1.getXAResource();
con1 = xaCon1.getConnection();
stmt1 = con1.createStatement();
xid1 = new MyXid(100, new byte[]{0x01}, new byte[]{0x02});
xaRes1.start(xid1, XAResource.TMNOFLAGS);
stmt1.executeUpdate("insert into test_table1 values (100)");
xaRes1.end(xid, XAResource.TMSUCCESS);
xaCon2 = xaDS.getXAConnection("jdbc_user", "jdbc_password");
xaRes2 = xaCon2.getXAResource();
con2 = xaCon2.getConnection();
stmt2 = con2.createStatement();
if (xaRes2.isSameRM(xaRes1)) {
    xaRes2.start(xid1, XAResource.TMJOIN);
    stmt2.executeUpdate("insert into test_table2 values (100)");
    xaRes2.end(xid1, XAResource.TMSUCCESS);
} else {
    xid2 = new MyXid(100, new byte[]{0x01}, new byte[]{0x03});
    xaRes2.start(xid2, XAResource.TMNOFLAGS);
    stmt2.executeUpdate("insert into test_table2 values (100)");
    xaRes2.end(xid2, XAResource.TMSUCCESS);
    ret = xaRes2.prepare(xid2);
    if (ret == XAResource.XA_OK) {
         xaRes2.commit(xid2, false);
    }
}
ret = xaRes1.prepare(xid1);
if (ret == XAResource.XA_OK) {
    xaRes1.commit(xid1, false);
}
```

**Example 6**—此示例显示如何在故障恢复期间恢复已准备或启发式完成的事务分支。 它首先尝试回滚每个分支; 如果失败，它会尝试告诉资源管理器丢弃有关事务的知识。

```java
MyXid[] xids;
xids = xaRes.recover(XAResource.TMSTARTRSCAN | XAResource.TMENDRSCAN);
for (int i=0; xids!=null && i<xids.length; i++) {
    try {
        xaRes.rollback(xids[i]);
    }
    catch (XAException ex) {
        try {
            xaRes.forget(xids[i]);
        }
        catch (XAException ex1) {
            System.out.println("rollback/forget failed: " +
            ex1.errorCode);
        }
    }
}
```

### 结论

在JDBC驱动程序中提供JTA支持大大提高了数据访问能力。 DataDirect Connect for JDBC驱动程序提供此支持。 结合分布式事务处理的其他组件，DataDirect驱动程序提高了现代企业的能力，速度和效率。

### 参考文档

Cheung & Matena, Java Transaction API (JTA), 1999, Sun Microsystems, Inc.

Maydene Fisher, Jon Ellis, and Jonathan Bruce, JDBC API Tutorial and Reference, Third Edition, 2003, Addison-Wesley.

X/Open CAE Specification, Distributed Transaction Processing: The XA Specification, 1991, The X/Open Company.

https://www.progress.com/tutorials/jdbc

### 自测完整例子

```java
import com.mysql.cj.jdbc.MysqlXADataSource;
import javax.sql.XAConnection;
import javax.sql.XADataSource;
import javax.transaction.xa.XAException;
import javax.transaction.xa.XAResource;
import java.sql.Connection;
import java.sql.SQLException;
import java.sql.Statement;

/**
 * Created with IntelliJ IDEA.
 * User: jiexiu
 * Date: 17/1/6
 * Time: 下午10:30
 * Email:jiexiu@mogujie.com
 */
public class MultiDataSourceTest {

    private static final String dataSourceUrlA = "jdbc:mysql://127.0.0.1:3306/leoNode?useUnicode=true&characterEncoding=utf-8&autoReconnectForPools=true&autoReconnect=true&connectTimeout=0";

    private static final String dataSourceUrlB = "jdbc:mysql://127.0.0.1:3306/leoblog?useUnicode=true&characterEncoding=utf-8&autoReconnectForPools=true&autoReconnect=true&connectTimeout=0";

    private static final String userName = "";
    private static final String passWord = "";

    public static void main(String[] args) {
        testMultiDataSource();
    }

    /**
     * 测试多数据源正常提交
     */
    private static void testMultiDataSource() {
        XADataSource xaDataSourceA = getMysqlDataSource(dataSourceUrlA);
        XADataSource xaDataSourceB = getMysqlDataSource(dataSourceUrlB);

        XAConnection xaConnectionA = null;
        XAConnection xaConnectionB = null;

        Connection connA = null;
        Connection connB = null;
        Statement stmtA = null;
        Statement stmtB = null;

        XAResource xaResourceA = null;
        XAResource xaResourceB = null;
        try {
            xaConnectionA = xaDataSourceA.getXAConnection();
            xaConnectionB = xaDataSourceB.getXAConnection();

            connA = xaConnectionA.getConnection();
            connB = xaConnectionB.getConnection();
            stmtA = connA.createStatement();
            stmtB = connB.createStatement();

            xaResourceA = xaConnectionA.getXAResource();
            xaResourceB = xaConnectionB.getXAResource();
        } catch (SQLException e) {
            e.printStackTrace();
        }


        MyXid xidA = new MyXid(0, new byte[] { 0x01 }, new byte[] { 0x02 });
        MyXid xidB = new MyXid(0, new byte[] { 0x01 }, new byte[] { 0x03 });

        try {
            xaResourceA.start(xidA, XAResource.TMNOFLAGS);
            stmtA.execute("insert into test(name) values ('test123')");
            xaResourceA.end(xidA, XAResource.TMSUCCESS);

            xaResourceB.start(xidB, XAResource.TMNOFLAGS);
            stmtB.execute("insert into test(name) values ('test123')");
            xaResourceB.end(xidB, XAResource.TMSUCCESS);
            //prepare 阶段
            int retA = xaResourceA.prepare(xidA);
            int retB = xaResourceB.prepare(xidB);
            //都准备好了
            if (XAResource.XA_OK == retA && XAResource.XA_OK == retB){
                // 提交阶段
                xaResourceA.commit(xidA, false);
                xaResourceB.commit(xidB, false);
            }else {
                //回滚
                xaResourceA.rollback(xidA);
                xaResourceB.rollback(xidB);
            }
        } catch (XAException e) {
            e.printStackTrace();
        } catch (SQLException e){
            e.printStackTrace();
        }finally {
            try {
                if (stmtA != null){
                    stmtA.close();
                }
                if (connA != null){
                    connA.close();
                }
                if (xaConnectionA != null){
                    xaConnectionA.close();
                }
                if (stmtB != null){
                    stmtB.close();
                }
                if (connB != null) {
                    connB.close();
                }
                if (xaConnectionB != null){
                    xaConnectionB.close();
                }
            }catch (SQLException e){
                e.printStackTrace();
            }
        }
    }

    private static XADataSource getMysqlDataSource(String dataSourceUrl){
        MysqlXADataSource dataSource = new MysqlXADataSource();
        dataSource.setURL(dataSourceUrl);
        dataSource.setUser(userName);
        dataSource.setPassword(passWord);
        return dataSource;
    }
}
```

