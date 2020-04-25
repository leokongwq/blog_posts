---
layout: post
comments: true
title: maven使用知识汇总
date: 2017-11-06 14:11:51
tags:
- maven
categories:
- devtools
---

### 背景

用了好多年maven了，在使用中遇到好多问题，虽然一一解决了，但是并没有详细总结归纳形成最佳实践。通过这篇文章将使用中遇到过的问题一一记录，以后再次遇到同类问题就不用再Google了。

### maven坐标

maven的世界里有海量的构件，如何查找需要的构件是非常重要的事情。maven通过称为坐标的概念来定位仓库中的构件。maven构件的坐标有一下几个元素：

- groupId 构件所属的项目
- artifactId 构件所在的模块。
- version 构件的版本号
- packaging 构件打包的类型，默认是`jar`
- classifier 构件使用的JDK版本

<!-- more -->

### maven构件依赖

#### maven构件依赖范围

首先需要知道的是：maven在编译项目主代码时使用一套classpath， 在编译运行测试代码时用一套classpath，在实际运行时又是一套classpath。 

依赖范围其实就是控制依赖于这三种classpath（编译， 测试，运行）的关系的。

- Compile: 编译依赖范围。如果没有指定，就会默认使用该依赖范围。使用此依赖范 围的 Maven 依赖，对于编译、测试、运行三种 classpath 都有效。典型的例子是 spring-core，*在编译、测试和运行*的时候都需要使用该依赖
- Test:测试依赖范围。使用此依赖范围的 Maven 依赖，只对于测试 classpath 有效，在 编译主代码或者运行项目的使用时将无法使用此类依赖。典型的例子是 JUnit，它只有 在编译测试代码及运行测试的时候才需要。
- Provided:已提供依赖范围。使用此依赖范围的 Maven 依赖，对于*编译和测试* classpath 有效，但在运行时无效。典型的例子是 `servlet-api`，编译和测试项目的时候 需要该依赖，但在运行项目的时候，由于容器已经提供，就不需要 Maven 重复的引入 一遍。
- Runtime:运行时依赖范围。使用此依赖范围的 Maven 依赖，对于*测试和运行* classpath 有效，但在编译主代码时无效。典型的例子是 JDBC 驱动实现，项目主代码的编译只需要 JDK 提供的 JDBC 接口，只有在执行测试或者运行项目的时候才需要实 现上述接口的具体 JDBC 驱动。
- System:系统依赖范围。该依赖与 3 种 classpath 的关系，和 Provided 依赖范围完全一致。但是，使用 System 范围的依赖时必须通过`systemPath`元素显式地指定依赖文件 的路径。由于此类依赖不是通过 Maven 仓库解析的，而且往往与本机系统绑定，可能 造成构建的不可移植，因此应该谨慎使用。systemPath 元素可以引用环境变量，如:
```xml
<dependency>    <groupId>javax.sql</groupId> 
    <artifactId>jdbc-stdext</artifactId> 
    <version>2.0</version>    <scope>system</scope> 
    <systemPath>${java.home}/lib/rt.jar</systemPath></dependency>
```
- Import(Maven 2.0.9 及以上):导入依赖范围。该依赖范围不会对 3 种 classpath 产生 实际的影响

#### 传递性依赖

传递性依赖，简单来说就是项目中直接依赖的构件本身依赖的其它构件。maven会自动处理这些间接的依赖构件。

#### 传递性依赖和依赖范围

假设 A 依赖于 B，B 依赖于 C，我们说 A 对于 B 是第一直接依赖， B 对于 C 是第二直接依赖，A 对于 C 是传递性依赖。第一直接依赖的范围和第二直接依赖 的范围决定了传递性依赖的范围，如下表所示，最左边一列表示第一直接依赖范围，最上面一行表示第二直接依赖范围，中间的交叉单元格则表示传递性依赖范围:

| 第一直接依赖/第二直接依赖 | compile | test | provided | runtime |
| --- | --- | --- | --- | --- |
| compile | compile | - | - | runtime |
| test | test | - | - | test |
| provided | provided | - | provided | provided |
| runtime | runtime | - | - | runtime |

仔细观察一下表格，可以发现这样的规律:当第二直接依赖的范围是`compile`的时候，传递性依赖的范围与第一直接依赖的范围一致;当第二直接依赖的范围是`test`的时候，依赖不会得以传递;当第二直接依赖的范围是`provided`的时候，只传递第一直接依赖范围也为`provided`的依赖，且传递性依赖的范围同样为`provided`;当第二直接依赖的范围是`runtime`的时候，传递性依赖的范围与第一直接依赖的范围一致，但 compile 例外，此时 传递性依赖的范围为`runtime`。

#### 依赖调解

所谓依赖调解指的是当项目中的直接依赖构件依赖了相同的简介依赖而造成的重复依赖问题，例如：`A->B->C->X(1.0)`、`A->D->X(2.0)`，`X` 是`A` 的传递性依赖，但是两条依赖路径上有两个版本的`X`。 

Maven依赖调解的第一原则是: `路径最近者优先`。该例中 `X(1.0)`的路径长度为`3`，而`X(2.0)`的路径长度为 `2`，因此 `X(2.0)`会被解析使用。

依赖调解第一原则不能解决所有问题，比如这样的依赖关系:`A->B->Y(1.0)`、`A->C->Y (2.0)`，`Y(1.0)`和 `Y(2.0)`的依赖路径长度是一样的，都为`2`。那么到底谁会被解析使 用呢?在 Maven 2.0.8 及之前的版本中，这是不确定的，但是从Maven 2.0.9 开始，为了 尽可能避免构建的不确定性，Maven 定义了依赖调解的第二原则:`第一声明者优先`。在依赖路径长度相等的前提下，在POM中依赖声明的顺序决定了谁会被解析使用，顺序最靠前的那个依赖优胜。该例中，如果B的依赖声明在C之前，那么`Y(1.0)`就会被解析使用。

#### 可选依赖

假设有这样一个依赖关系，项目 A 依赖于项目 B，B 依赖于项目 X 和 Y，B 对于 X 和 Y 的 依赖都是可选依赖:A->B、B->X(可选)、B->Y(可选)。根据传递性依赖的定义，如果 所有这三个依赖的范围都是 compile，那么 X、Y 就是 A 的 compile 范围传递性依赖。然 而，由于这里 X、Y 是可选依赖，依赖将不会得以传递，换句话说，X、Y 将不会对A有任何影响

{% asset_img maven-optional.png %}

为什么要使用可选依赖这一特性呢?可能项目 B 实现了两个特性，其中的特性一依赖于 X，特性二依赖于 Y，而且这两个特性是互斥的，用户不可能同时使用两个特性。比如 B 是一个持久层隔离工具包，它支持多种数据库，包括 MySQL，PostgreSQL 等等，在构建这 个工具包的时候，需要这两种数据库的驱动程序，但在使用这个工具包的时候，只会依赖 一种数据库。项目 B 的依赖声明见代码清单 3-7。

```xml
<project> 
    <modelVersion>4.0.0</modelVersion> 
    <groupId>com.juvenxu.mvnbook</groupId> 
    <artifactId>project-b</artifactId> 
    <version>1.0.0</version>    <dependencies>        <dependency>            <groupId>mysql</groupId> 
            <artifactId>mysql-connector-java</artifactId> 
            <version>5.1.10</version> 
            <optional>true</optional>        </dependency> 
        <dependency>            <groupId>postgresql</groupId> 
            <artifactId>postgresql</artifactId>       
            <version>8.4-701.jdbc3</version> 
            <optional>true</optional>        </dependency> 
    </dependencies></project>
```

上述 XML 代码片段中，使用<optional>元素表示 mysql-connector-java 和 postgresql 这两个 依赖为可选依赖，它们只会对当前项目 B 产生影响，当其他项目依赖于 B 的时候，这两个依赖不会被传递。因此，当项目 A 依赖于项目 B 的时候，如果其实际使用基于 MySQL 数 据库，那么在项目 A 中就需要显式地声明 mysql-connector-java 这一依赖。

```xml
<project> 
    <modelVersion>4.0.0</modelVersion> 
    <groupId>com.juvenxu.mvnbook</groupId> 
    <artifactId>project-a</artifactId> <version>1.0.0</version>    <dependencies>        <dependency> 
            <groupId>com.juvenxu.mvnbook</groupId> 
            <artifactId>project-b</artifactId> 
            <version>1.0.0</version>        </dependency> 
        <dependency>        <groupId>mysql</groupId>
             <artifactId>mysql-connector-java</artifactId> 
             <version>5.1.10</version>        </dependency> 
    </dependencies></project>
```

最后，关于可选依赖需要说明的一点是，在理想的情况下，是不应该使用可选依赖的。前 面我们可以看到，使用可选依赖的原因是某一个项目实现了多个特性，在面向对象设计 中，有个单一职责性原则，意指一个类应该只有一项职责，而不是糅合太多的功能。这个 原则在规划 Maven 项目的时候也同样适用，在上面的例子中，更好的做法是为 MySQL 和 PostgreSQL 分别创建一个 Maven 项目，基于同样的 groupId 分配不同的 artifactId，如 `com.juvenxu.mvnbook:project-b-mysql` 和 `com.juvenxu.mvnbook:project-b-postgresql`，在各 自的 POM 中声明对应的 JDBC 驱动依赖，而且不使用可选依赖，用户则根据需要选择使用`project-b-mysql`或者 `project-b-postgresql`，由于传递性依赖的作用，就不用再声明 JDBC 驱 动依赖。

### 依赖最佳实践

#### 排除依赖

传递性依赖会给项目隐式的引入很多依赖，这极大地简化了项目依赖的管理，但是有些时 候这种特性也会带来问题。例如，当前项目有一个第三方依赖，而这个第三方依赖由于某 些原因依赖了另外一个类库的 SNAPSHOT 版本，那么这个 SNAPSHOT 就会成为当前项目的 传递性依赖，而 SNAPSHOT 的不稳定性会直接影响到当前的项目，这时就需要排除掉该 SNAPSHOT，并且在当前项目中声明该类库的某个正式发布的版本。还有一些情况，你可 能也想要替换某个传递性依赖，比如 Sun JTA API，Hibernate 依赖于这个 JAR，但是由于版 权的因素，该类库不在中央仓库中，而Apache Geronimo项目有一个对应的实现，这时你 就可以排除 Sun JAT API，再声明 Geronimo 的 JTA API 实现

```xml
<project> 
    <modelVersion>4.0.0</modelVersion> 
    <groupId>com.juvenxu.mvnbook</groupId> 
    <artifactId>project-a</artifactId> 
    <version>1.0.0</version><dependencies>    <dependency> 
            <groupId>com.juvenxu.mvnbook</groupId> 
            <artifactId>project-b</artifactId> 
            <version>1.0.0</version>        <exclusions>            <exclusion> 
              <groupId>com.juvenxu.mvnbook</groupId> 
              <artifactId>project-c</artifactId>            </exclusion> 
        </exclusions>    </dependency> 
    <dependency>        <groupId>com.juvenxu.mvnbook</groupId> 
        <artifactId>project-c</artifactId> 
        <version>1.1.0</version>    </dependency> 
</dependencies></project>
```

上述代码中，项目 A 依赖于项目 B，但是由于一些原因，不想引入传递性依赖 C，而是自 己显式地声明对于项目 C 1.1.0 版本的依赖。代码中使用 exclusions 元素声明排除依赖， exclusions 可以包含一个或者多个 exclusion 子元素，因此可以排除一个或者多个传递性依 赖。需要注意的是，声明 exclusion 的时候只需要 groupId 和 artifactId，而不需要 version 元素，这是因为只需要 groupId 和 artifactId 就能唯一定位依赖图中的某个依赖，换句话说，Maven 解析后的依赖中，不可能出现 groupId 和 artifactId 相同，但是 version 不同的两个依赖。

#### 归类依赖

归类依赖指的是依赖了同一个项目的多个模块，这些模块统一称为归类依赖（典型的就是spring）。在使用时我们可以通过maven属性来为这些归类依赖的定义统一的版本号，方便以后统一升级。

#### 优化依赖

maven提供了许多功能来分析当前项目的依赖情况，具体如下：

```shell
mvn dependency:list 
```

通过上面的命令我们能得知maven解析后，项目所有依赖的构建和其依赖范围。

```shell
mvn dependency:tree
```

通过该命令我们能清晰的知道项目依赖的关系，直接依赖和间接依赖等信息。

```shell
dependency:analyze 
```

该命令的输出需要注意两点：

`Used undeclared dependencies`：指的是项目中用到的依赖但是并没有显示声明依赖。这种依赖是通过直接依赖传递进来的，当升级直接依赖的时候，相关传递性依赖的版本也可能发生变化，这种变化不易察觉，但是有可能导致当前项目出错，例如由于接口的改变，当前项目中的相关代 码无法编译。这种隐藏的、潜在的威胁一旦出现，就往往需要耗费大量的时间来查明真 相。因此，显式声明任何项目中直接用到的依赖。

`Unused declared dependencies`：指定是项目中没有用到但已经声明的依赖。需要注意的是，对于这样一类依赖， 我们不应该简单地直接删除其声明，而是应该仔细分析。由于 dependency:analyze 只会分 析编译主代码和测试代码需要用到的依赖，一些执行测试和运行时需要的依赖它就发现不了，很显然，该例中的 spring-core 和 spring-beans 是运行 Spring Framework 项目必要的类 库，因此不应该删除依赖声明。当然，有时候确实能通过该信息找到一些没用的依赖，但一定要小心测试。

### Repositories

Repository 是maven构件的集合（或叫做仓库），默认值是：[https://repo.maven.apache.org/maven2/](https://repo.maven.apache.org/maven2/)。我们在项目中开发中也可以指定自己的私服地址。

配置示例如下：

```xml
<repositories>
    <repository>
      <releases>
        <enabled>false</enabled>
        <updatePolicy>always</updatePolicy>
        <checksumPolicy>warn</checksumPolicy>
      </releases>
      <snapshots>
        <enabled>true</enabled>
        <updatePolicy>never</updatePolicy>
        <checksumPolicy>fail</checksumPolicy>
      </snapshots>
      <id>codehausSnapshots</id>
      <name>Codehaus Snapshots</name>
      <url>http://snapshots.maven.codehaus.org/maven2</url>
      <layout>default</layout>
    </repository>
  </repositories>
  <pluginRepositories>
    ...
  </pluginRepositories>
```

- id 仓库的唯一标志
- name 仓库的描述性名称，以易读为准。
- url 仓库的地址
- layout 仓库的存储布局格式。默认是default（maven2 和 maven3）。 取值范围：default 或 legacy(maven1)。
- releases, snapshots: 针对不同的构件版本类型设置不同的服务策略。
- enabled: 取值为 true 或 false，表明该仓库是否提供该构件发布类型的服务。
- updatePolicy: 仓库依赖的更新策略。取值为：always, daily (默认), interval:X (X表示分钟数) 或 never。
- checksumPolicy: 当maven部署构件到仓库时，也会部署对应的校验和文件，你可以设置：ignore，fail或者warn用于当校验和文件不存在或者检验失败时的处理策略。

### pluginRepositories

插件仓库。仓库是两种主要构件的家。第一种构件被用作其它构件的依赖。这是中央仓库中存储的大部分构件类型。另外一种构件类型是插件。Maven插件是一种特殊类型的构件。由于这个原因，插件仓库独立于其它仓库。pluginRepositories元素的结构和repositories元素的结构类似。每个pluginRepository元素指定一个Maven可以用来寻找新插件的远程地址。

pluginRepositories的配置和repositories基本类似。

### 在settings.xml中配置远程仓库

为什么需要在maven的配置文件`settings.xml`中配置repository呢？答案就是为了避免重复。在`settings.xml`中配置repository对所有的项目都是起作用的。

```xml
<settings>  
  ...  
  <profiles>  
    <profile>  
      <id>dev</id>  
      <!-- repositories and pluginRepositories here-->  
    </profile>  
  </profiles>  
  <activeProfiles>  
    <activeProfile>dev</activeProfile>  
  </activeProfiles>  
  ...  
</settings>  
```

这里我们定义一个id为dev的profile，将所有repositories以及pluginRepositories元素放到这个profile中，然后，使用<activeProfiles>元素自动激活该profile。这样，你就不用再为每个POM重复配置仓库。
使用profile为settings.xml添加仓库提供了一种用户全局范围的仓库配置。

### maven使用repository的顺序

1. settings.xml - Auto-activated profiles (<activation/>), reverse definition order
2. settings.xml - CLI activated profiles, reverse definition order
3. pom.xml repos
4. parent pom repos

### 镜像

如果你的地理位置附近有一个速度更快的central镜像，或者你想覆盖central仓库配置，或者你想为所有POM使用唯一的一个远程仓库（这个远程仓库代理的所有必要的其它仓库），你可以使用settings.xml中的mirror配置。
以下的mirror配置用maven.net.cn覆盖了Maven自带的central：

```xml
<settings>  
...  
  <mirrors>  
    <mirror>  
      <id>maven-net-cn</id>  
      <name>Maven China Mirror</name>  
      <url>http://maven.net.cn/content/groups/public/</url>  
      <mirrorOf>central</mirrorOf>  
    </mirror>  
  </mirrors>  
...  
</settings> 
```

这里唯一需要解释的是`<mirrorOf>`，这里我们配置`central`的镜像，我们也可以配置一个所有仓库的镜像，以保证该镜像是Maven唯一使用的仓库：

```xml
<settings>  
...  
  <mirrors>  
    <mirror>  
      <id>my-org-repo</id>  
      <name>Repository in My Orgnization</name>  
      <url>http://192.168.1.100/maven2</url>  
      <mirrorOf>*</mirrorOf>  
    </mirror>  
  </mirrors>  
...  
</settings>  
```

mirrorOf 可以有如下的使用方式：

1. `*` 代理所有仓库
2. repo3 只针对repo3
3. *, !repo3  所有仓库，除了repo3 
4. external:* 所有的外部仓库

### mirror和repository 区别

internal repository是指在局域网内部搭建的repository，它跟central repository, jboss repository等的区别仅仅在于其URL是一个内部网址。 mirror则相当于一个代理，它会拦截去指定的远程repository下载构件的请求，然后从自己这里找出构件回送给客户端。配置mirror的目的一般是出于网速考虑。 可以看出，internal repository和mirror是两码事。前者本身是一个repository，可以和其它repository一起提供服务，比如它可以用来提供公司内部的maven构件；而后者本身并不是repository，它只是远程repository的网络加速器。 

不过，很多internal repository搭建工具往往也提供mirror服务，比如Nexus就可以让同一个URL,既用作internal repository，又使它成为所有repository的mirror。

如果仓库X可以提供仓库Y存储的所有内容，那么就可以认为X是Y的一个镜像。换句话说，任何一个可以从仓库Y获得的构件，都胡够从它的镜像中获取。举个例 子，http://maven.net.cn/content/groups/public/ 是中央仓库http://repo1.maven.org/maven2/ 在中国的镜像，由于地理位置的因素，该镜像往往能够提供比中央仓库更快的务。因此，可以配置Maven使用该镜像来替代中央仓库。编辑 settings.xml，代码如下：

```xml
<settings>
  ...
  <mirrors>
    <mirror>
      <id>maven.net.cn</id>
      <name>one of the central mirrors in china</name>
      <url>http://maven.net.cn/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
  </mirrors>
  ...
</settings>
```

该例中，<mirrorOf>的值为central，表示该配置为中央仓库的镜像，任何对于中央仓库的请求都会转至该镜像，用户也可以使用同 样的方法配置其他仓库的镜像。另外三个元素id,name,url与一般仓库配置无异，表示该镜像仓库的唯一标识符、名称以及地址。类似地，如果该镜像需 认证，也可以基于该id配置仓库认证。 
**关于镜像的一个更为常见的用法是结合私服**。由于私服可以代理任何外部的公共仓库(包括中央仓库)，因此，对于组织内部的Maven用户来说，使用一个私服地址就等于使用了所有需要的外部仓库，这可以将配置集中到私服，从而简化Maven本身的配置。在这种情况下，任何需要的构件都可以从私服获得，私服就是所有仓库的镜像。这时，可以配置这样的一个镜像，如例：

```xml
<settings>
  ...
  <mirrors>
    <mirror>
      <id>internal-repository</id>
      <name>Internal Repository Manager</name>
      <url>http://192.168.1.100/maven2</url>
      <mirrorOf>*</mirrorOf>
    </mirror>
  </mirrors>
  ...
</settings>
```

### 构件发布-distributionManagement

当我们完成一些公共组件的开发时，就需要将这些组件发布到公司的私服中去，这样需要使用该组件的项目就可以依赖该组件了。在maven中发布开发好的构件时需要配置`distributionManagement`元素。`distributionManagement`元素的基本构成如下：

```xml
<distributionManagement>
    <downloadUrl>http://mojo.codehaus.org/my-project</downloadUrl>
    <status>deployed</status>
</distributionManagement>
```

- downloadUrl，一个URL，其它Maven项目可以通过该URL下载并引用当前Maven项目的构件。
- status，当前Maven项目的状态，可用的状态如下所示。注意，该值是由Maven自动设置，永远不要人工设置。
    - none，未指明状态，默认值
    - converted，该Maven项目的构件已经被转换为兼容Maven 2
    - partner，该Maven项目的构件保持与另一个库的Maven版本一致
    - deployed，该Maven项目的构件是通过Maven 2或Maven 3发布的，最常用的值
    - verified，该Maven项目的构件已经被验证过

#### <distributionManagement>的<repository>配置

Maven部署当前构件到远程库时，关于远程库的配置。示例如下：

```xml
  <distributionManagement>
    <repository>
      <uniqueVersion>false</uniqueVersion>
      <id>corp1</id>
      <name>Corporate Repository</name>
      <url>scp://repo/maven2</url>
      <layout>default</layout>
    </repository>
    <snapshotRepository>
      <uniqueVersion>true</uniqueVersion>
      <id>propSnap</id>
      <name>Propellors Snapshots</name>
      <url>sftp://propellers.net/maven</url>
      <layout>legacy</layout>
    </snapshotRepository>
    ...
  </distributionManagement>
```

- id, name：id用来在多个仓库中标志一个仓库的唯一性，name则是一个易于人理解的名称。
- uniqueVersion：该属性的取值是true/false，用来表示发布到该仓库的构件是否需要生成一个唯一的版本号，还是使用address里的其中version部分
- url: 指定仓库的地址和传输使用的协议。
- layout: default或者legacy

#### <distributionManagement>的<site>配置

除了部署当前Maven项目的构件，还可以部署当前Maven项目的网站和文档。示例如下：

```xml
<distributionManagement>
    ...
    <site>
      <id>mojo.website</id>
      <name>Mojo Website</name>
      <url>scp://beaver.codehaus.org/home/projects/mojo/public_html/</url>
    </site>
    ...
  </distributionManagement>
```

#### <distributionManagement>的<relocation>配置


```xml
  <distributionManagement>
    ...
    <relocation>
      <groupId>org.apache</groupId>
      <artifactId>my-project</artifactId>
      <version>1.0</version>
      <message>We have moved the Project under Apache</message>
    </relocation>
    ...
  </distributionManagement>
```

#### distributionManagement对应的settings.xml中的配置

```xml
<settings>    
  ...    
  <servers>    
    <server>    
      <id>nexus-releases</id>    
      <username>admin</username>    
      <password>admin123</password>    
    </server>    
    <server>    
      <id>nexus-snapshots</id>    
      <username>admin</username>    
      <password>admin123</password>    
    </server>      
  </servers>    
  ...    
</settings>  
```

需要注意的是，settings.xml中server元素下id的值必须与POM中repository或snapshotRepository下id的值完全一致。将认证信息放到settings下而非POM中，是因为POM往往是它人可见的，而settings.xml是本地的。

### 反应堆

反应堆指的是在一个多模块项目中，所有模块组成的一个构建结构。

#### 反应堆构建顺序

反应堆的构建顺序是指：maven根据模块间的依赖和继承顺序来决定整个项目的构建顺序。

#### 剪裁反应堆

在一个多模块项目中如果修改了某个模块的代码其实是不需要构建整个项目的，尤其是大项目。否则会大大减慢开发效率。maven提供了剪裁构建反应堆的能力，让你有能力之构建需要构建的模块。maven提供多个命令行参数来支持反应堆的剪裁。具体如下：

- -am, --also-make 同时构建所列模块的依赖模块
- -amd --also-make-dependents 同时构建依赖于所列模块的模块
- pl --projects <arg> 构建指定的模块，模块间用逗号分割
- rf --resume-from <arg> 从指定的模块回复反应堆

### 测试集成

maven 本身不提供代码测试功能 ，它只是在构建执行到特定生命周期阶段的时候，通过插件来执行`JUnit`或者`TestNG`的测试用例。`maven-surefire-plugin`就是这样一个提供运行测试的插件。

#### maven-surefire-plugin

`maven-surefire-plugin`可以称为测试运行器(Test Runner)，它能兼容JUnit 3、JUnit 4以及TestNG。

在默认情况下，maven-surefire-plugin的test目标会自动执行测试源码路径（默认为src/test/java/）下所有符合一组命名模式的测试类。这组模式为：
- **/Test*.java：任何子目录下所有命名以Test开关的Java类。
- **/*Test.java：任何子目录下所有命名以Test结尾的Java类。
- **/*TestCase.java：任何子目录下所有命名以TestCase结尾的Java类。

##### 跳过测试

要想跳过测试，在命令行加入参数`skipTests`就可以了。如：

```
mvn package -DskipTests  
```
又或者
```xml
<plugin>  
    <groupId>org.apache.maven.plugins</groupId>  
    <artifactId>maven-surefire-plugin</artifactId>  
    <version>2.5</version>  
    <configuration>  
        <skipTests>true</skipTests>  
    </configuration>  
</plugin> 
```

有时候可能不仅仅需要跳过测试运行，还要跳过测试代码的编译：

```
mvn package -Dmaven.test.skip=true
```

也可以在pom中配置maven.test.skip:

```xml
<plugin>  
    <groupId>org.apache.maven.plugin</groupId>  
    <artifactId>maven-compiler-plugin</artifactId>  
    <version>2.1</version>  
    <configuration>  
        <skip>true</skip>  
    </configuration>  
</plugin>  
<plugin>  
    <groupId>org.apache.maven.plugins</groupId>  
    <artifactId>maven-surefire-plugin</artifactId>  
    <version>2.5</version>  
    <configuration>  
        <skip>true</skip>  
    </configuration>  
</plugin>
```

#### 动态指定要运行的测试用例

`maven-surefire-plugin`提供了一个test参数让Maven用户能够在命令行指定要运行的测试用例。如：

```
mvn test -Dtest=RandomGeneratorTest  
```

也可以使用通配符

```
mvn test -Dtest=Random*Test
```

或者也可以使用“，”号指定多个测试类:

```
mvn test -Dtest=Random*Test,AccountCaptchaServiceTest 
```

如果没有指定测试类，那么会报错并导致构建失败。

```
mvn test -Dtest  
```

这时候可以添加`-DfailIfNoTests=false`参数告诉`maven-surefire-plugin`即使没有任何测试也不要报错。

```
mvn test -Dtest -DfailIfNoTests=false  
```

#### 包含与排除测试用例

如果由于历史原因，测试类不符合默认的三种命名模式，可以通过pom.xml设置maven-surefire-plugin插件添加命名模式或排除一些命名模式。

```xml
<plugin>  
    <groupId>org.apache.maven.plugins</groupId>  
    <artifactId>maven-surefire-plugin</artifactId>  
    <version>2.5</version>  
    <configuration>  
        <includes>  
            <include>**/*Tests.java</include>  
        </includes>  
        <excludes>  
            <exclude>**/*ServiceTest.java</exclude>  
            <exclude>**/TempDaoTest.java</exclude>  
        </excludes>  
    </configuration>  
</plugin> 
```

#### 生成测试报告

默认情况下，`maven-surefire-plugin`会在项目的`target/surefire-reports`目录下生成两种格式的错误报告。

1. 简单文本格式——内容十分简单，可以看出哪个测试项出错。
2. 与JUnit兼容的XML格式——XML格式已经成为了Java单元测试报告的事实标准，这个文件可以用其他的工具如IDE来查看。

#### 测试覆盖率报告

测试覆盖率是衡量项目代码质量的一个重要的参考指标。[Cobertura](http://cobertura.sourceforge.net/)是一个优秀的开源测试覆盖率统计工具，Maven通过`cobertura-maven-plugin`与之集成，用户可以使用简单的命令为Maven项目生成测试覆盖率报告。运行下面命令生成报告：

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>cobertura-maven-plugin</artifactId>
    <version>2.7</version>
</plugin>
```

命令行生成报告：

```shell
mvn cobertura:cobertura
```

执行完后会在target目录里找到site目录,用浏览器打开里面的index.html,这就是测试用例执行完后`cobertura-maven-plugin`得出的覆盖率报告。

#### 重用测试代码

当命令行运行mvn package的时候，Maven只会打包主代码及资源文件，并不会对测试代码打包。如果测试代码中有需要重用的代码，这时候就需要对测试代码打包了。
这时候需要配置maven-jar-plugin将测试类打包，如：

```xml
<plugin>  
    <groupId>org.apache.maven.plugins</groupId>  
    <artifactId>maven-jar-plugin</artifactId>  
    <version>2.2</version>  
    <executions>  
        <execution>  
            <goals>  
                <goal>test-jar</goal>  
            </goals>  
        </execution>  
    </executions>  
</plugin> 
```
maven-jar-plugin有两个目标，分别为jar和test-jar。这两个目标都默认绑定到default生命周期的package阶段运行，只是test-jar并没有在超级POM中配置，因此需要我们另外在pom中配置。

现在如要引用test-jar生成的测试代码包，可以如下配置：

```xml
<dependency>  
    <groupId>com.juvenxu.mvnbook.account</groupId>  
    <artifactId>account-captcha</artifactId>  
    <version>1.0.0-SNAPSHOT</version>  
    <type>test-jar</type>  
    <scope>test</scope>  
</dependency>  
```



### 参考资料

[https://developer.jboss.org/thread/160185](https://developer.jboss.org/thread/160185)
[https://maven.apache.org/settings.html](https://maven.apache.org/settings.html)
[https://my.oschina.net/vshcxl/blog/649775](https://my.oschina.net/vshcxl/blog/649775)
[https://www.cnblogs.com/qyf404/archive/2015/12/12/5040593.html](https://www.cnblogs.com/qyf404/archive/2015/12/12/5040593.html)





