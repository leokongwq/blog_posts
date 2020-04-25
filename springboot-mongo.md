---
layout: post
comments: true
title: springboot-mongo
date: 2017-05-17 14:40:35
tags:
- springboot
categories:
- spring
---

### 背景

springboot对常见的各种SQL，nosql数据库提供了良好的支持。在springboot中配置mongodb的连接信息时遇到一些问题，主要原因是springboot对mongodb2.x和mongodb3.x的配置是不同的。本文记录如何在springboot中配置mongodb和mongodb的基本操作。

<!-- more -->

### springboot配置mongodb2.x

springboot 针对mongodb提供了如下的配置选项
```
# MONGODB (MongoProperties)
spring.data.mongodb.authentication-database= # 权限验证数据库名
spring.data.mongodb.database=test # 数据库名
spring.data.mongodb.field-naming-strategy= # 指定字端名称策略
spring.data.mongodb.grid-fs-database= # GridFS 数据库名
spring.data.mongodb.host=localhost # 要连接的mongodb的主机地址，可以通过mongodb格式的连接串来指定
spring.data.mongodb.port=27017 #要连接的mongodb的主机端口号。不可以通过连接串设置
spring.data.mongodb.username= # 登录验证的用户名， 不可以通过连接串设置
spring.data.mongodb.password= # 登录验证的密码，不可以通过连接串设置
spring.data.mongodb.repositories.enabled=true # Enable Mongo repositories.
spring.data.mongodb.uri=mongodb://localhost/test # Mongodb数据库的连接串。不能和 host，port，凭证信息信息一起使用
```

### springboot配置mongodb3.x

springboot针对mongodb3.x可以只通过连接串来配置：

```
spring.data.mongodb.uri=mongodb://name:pass@localhost:27017/test
```

### mongodb连接URI详解

mongodb标准的连接串格式如下：

```
mongodb://[username:password@]host1[:port1][,host2[:port2],...[,hostN[:portN]]][/[database][?options]]
```

每个部分的含义如下：

- mongodb:// 必须的连接串前缀，表示这个串是标准的连接串格式
- username:password@ 这部分是可选的，如果提供了，则客户端尝试使用该部分信息做认证。
- host1 这个是必须的，用来指定想要连接的主机地址
- :port1 这个是可选的，如果没有指定，默认是27017
- hostN 可选的。如果需要可以指定多个主机地址。例如连接`replica set`或分片后的`mongos`
- :portX 可选的。如果没有指定，默认是27017
- /database 可选的。用来指定登录的数据库名称。 如果指定了用户名和密码，但是没有指定认证数据库名，则客户端使用名为`admin`的数据库来认证。
- ?options 连接选项。可以参考:[Connection String Options](https://docs.mongodb.com/manual/reference/connection-string/#connections-connection-options)。 如果连接串没有指定`/database`部分，则必须在`?options`前添加一个`/`。

#### options详解

##### 副本集选项

- replicaSet 指定副本集的名称

##### socket连接选项

- ssl 该选项的可选值是`true`和`false`，默认是`false`。 指定连接使用启用SSL。
- connectTimeoutMS 连接超时时间，单位是毫秒。默认永不超时，但也和具体的驱动相关。
- socketTimeoutMS 读取或发生数据包的超时时间，单位是毫秒。默认永不超时，但也和具体的驱动相关。

##### 连接池选项

这些连接池选项只针对带有连接池功能的驱动有效

- maxPoolSize 连接池的最大连接数， 默认是100
- minPoolSize 连接池的最小连接数， 默认是0
- maxIdleTimeMS 连接最大空闲时间，该选项不是所有的驱动都支持
- waitQueueMultiple 等待获取连接池中连接的线程最大数
- waitQueueTimeoutMS 等待获取连接的线程的最大等待时间

##### Write Concern Options

- w 和`w`选项相对应，指定一个写操作最少写入多少个结点才正确返回。可以是一个数字，也可以是字符串`majority`或一个`tag set`。详细参考[w Option](https://docs.mongodb.com/manual/reference/write-concern/#wc-w)
- wtimeoutMS 和`wtimeout`选项相对应，指定超时时间。参考[wtimeout](https://docs.mongodb.com/manual/reference/write-concern/#wc-wtimeout)
- journal 和 `j`选项对应。指定mongodb的`w`写操作需要写`journal`日志。参考[j Option](https://docs.mongodb.com/manual/reference/write-concern/#wc-j)

##### readConcern Options

- readConcernLevel 指定读取副本集的隔离级别。 可选值：`local`,`majority`

##### read 操作首选项

指定读写副本集数据的读操作选项

- readPreference 可选值
    - primary
    - primaryPreferred
    - secondary
    - secondaryPreferred
    - nearest
    
- maxStalenessSeconds 3.4 版本新增，指定客户端停止读取该节点前， 该节点复制数据落后的秒数。
- readPreferenceTags 指定优先读取的`tag set`。 格式为`,`分割的`key:value`。

#### 认证选项

- authSource 指定认证凭证的来源，通常是创建凭证的数据库，默认值是连接串中指定的数据库。 该选项只有在认证机制是`MONGODB-CR`时使用。
- authMechanism 认证机制，2.6 版本添加`PLAIN` 和 `MONGODB-X509`； 3.0版本添加了`SCRAM-SHA-1`。可选的值为：
    - SCRAM-SHA-1
    - MONGODB-CR
    - MONGODB-X509 该选项必须使用SSL或TLS
    - GSSAPI (Kerberos) 企业级mongod和mongos才支持
    - PLAIN (LDAP SASL) 企业级mongod和mongos才支持
- gssapiServiceName

#### 服务器选择和发现选项

- localThresholdMS
- serverSelectionTimeoutMS
- serverSelectionTryOnce
- heartbeatFrequencyMS

#### Miscellaneous

- uuidRepresentation 取值：
    - standard The standard binary representation.
    - csharpLegacy The default representation for the C# driver.
    - javaLegacy The default representation for the Java driver.
    - pythonLegacy The default representation for the Python driver.    

#### 连接串例子

连接副本集：

```
mongodb://db1.example.net,db2.example.net:2500/?replicaSet=test
```

连接分片mongos

```
mongodb://r1.example.net:27017,r2.example.net:27017/
```

Unix Domian sockect 

```
mongodb://%2Ftmp%2Fmongodb-27017.sock
```

### springboot Mongodb自动配置原理

自动配置的核心类是：`MongoAutoConfiguration`

```
@Configuration
@ConditionalOnClass(MongoClient.class)
@EnableConfigurationProperties(MongoProperties.class)
@ConditionalOnMissingBean(type = "org.springframework.data.mongodb.MongoDbFactory")
public class MongoAutoConfiguration {

	private final MongoProperties properties;

	private final MongoClientOptions options;

	private final Environment environment;

	private MongoClient mongo;

	public MongoAutoConfiguration(MongoProperties properties,
			ObjectProvider<MongoClientOptions> optionsProvider, Environment environment) {
		this.properties = properties;
		this.options = optionsProvider.getIfAvailable();
		this.environment = environment;
	}

	@PreDestroy
	public void close() {
		if (this.mongo != null) {
			this.mongo.close();
		}
	}

	@Bean
	@ConditionalOnMissingBean
	public MongoClient mongo() throws UnknownHostException {
		this.mongo = this.properties.createMongoClient(this.options, this.environment);
		return this.mongo;
	}

}
```

#### 关键注解解释

**@ConditionalOnClass(MongoClient.class)**： 

表示在`classpath`中存在`MongoClient`类，该自动配置类才会被spring作为配置来源使用。

**@EnableConfigurationProperties(MongoProperties.class)**：

该注解告诉spring扫描类型为`MongoProperties.class`并且标有注解`@ConfigurationProperties`的配置项类。

**@ConditionalOnMissingBean(type = "org.springframework.data.mongodb.MongoDbFactory")**：

如果容器中没有类型为`org.springframework.data.mongodb.MongoDbFactory`的Bean, 者该配置类会被启用。

#### 原生操作

从上面的代码可以知道：方法`mongo`会创建一个`MongoClient`对象。可以直接通过该对象对mongo进行操作。

```java
@Resource
private MongoClient mongoClient;

//原生的mongo客户端操作
public void operateMongo(){
   MongoDatabase mongoDatabase = mongoClient.getDatabase("test");
   mongoDatabase.drop();
   mongoDatabase.createCollection("users");
   MongoCollection<Document> colUser = mongoDatabase.getCollection("users");
   colUser.insertOne(new Document());
}
```

#### MongoTemplate 操作

springboot除了生成`MongoClient`这个原生操作的Bean以外，还我们提供了非常好用的`MongoTemplate`。 该Bean是在类`MongoDataAutoConfiguration`中创建的，逻辑如下：

```java
@Configuration
@ConditionalOnClass({ Mongo.class, MongoTemplate.class })
@EnableConfigurationProperties(MongoProperties.class)
@AutoConfigureAfter(MongoAutoConfiguration.class)
public class MongoDataAutoConfiguration {

	private final ApplicationContext applicationContext;

	private final MongoProperties properties;

	public MongoDataAutoConfiguration(ApplicationContext applicationContext,
			MongoProperties properties) {
		this.applicationContext = applicationContext;
		this.properties = properties;
	}

	@Bean
	@ConditionalOnMissingBean(MongoDbFactory.class)
	public SimpleMongoDbFactory mongoDbFactory(MongoClient mongo) throws Exception {
		String database = this.properties.getMongoClientDatabase();
		return new SimpleMongoDbFactory(mongo, database);
	}

	@Bean
	@ConditionalOnMissingBean
	public MongoTemplate mongoTemplate(MongoDbFactory mongoDbFactory,
			MongoConverter converter) throws UnknownHostException {
		return new MongoTemplate(mongoDbFactory, converter);
	}

	@Bean
	@ConditionalOnMissingBean(MongoConverter.class)
	public MappingMongoConverter mappingMongoConverter(MongoDbFactory factory,
			MongoMappingContext context, BeanFactory beanFactory,
			CustomConversions conversions) {
		DbRefResolver dbRefResolver = new DefaultDbRefResolver(factory);
		MappingMongoConverter mappingConverter = new MappingMongoConverter(dbRefResolver,
				context);
		mappingConverter.setCustomConversions(conversions);
		return mappingConverter;
	}

	@Bean
	@ConditionalOnMissingBean
	public MongoMappingContext mongoMappingContext(BeanFactory beanFactory,
			CustomConversions conversions) throws ClassNotFoundException {
		MongoMappingContext context = new MongoMappingContext();
		context.setInitialEntitySet(new EntityScanner(this.applicationContext)
				.scan(Document.class, Persistent.class));
		Class<?> strategyClass = this.properties.getFieldNamingStrategy();
		if (strategyClass != null) {
			context.setFieldNamingStrategy(
					(FieldNamingStrategy) BeanUtils.instantiate(strategyClass));
		}
		context.setSimpleTypeHolder(conversions.getSimpleTypeHolder());
		return context;
	}

	@Bean
	@ConditionalOnMissingBean
	public GridFsTemplate gridFsTemplate(MongoDbFactory mongoDbFactory,
			MongoTemplate mongoTemplate) {
		return new GridFsTemplate(
				new GridFsMongoDbFactory(mongoDbFactory, this.properties),
				mongoTemplate.getConverter());
	}

	@Bean
	@ConditionalOnMissingBean
	public CustomConversions customConversions() {
		return new CustomConversions(Collections.emptyList());
	}

	/**
	 * {@link MongoDbFactory} decorator to respect
	 * {@link MongoProperties#getGridFsDatabase()} if set.
	 */
	private static class GridFsMongoDbFactory implements MongoDbFactory {

		private final MongoDbFactory mongoDbFactory;

		private final MongoProperties properties;

		GridFsMongoDbFactory(MongoDbFactory mongoDbFactory, MongoProperties properties) {
			Assert.notNull(mongoDbFactory, "MongoDbFactory must not be null");
			Assert.notNull(properties, "Properties must not be null");
			this.mongoDbFactory = mongoDbFactory;
			this.properties = properties;
		}

		@Override
		public DB getDb() throws DataAccessException {
			String gridFsDatabase = this.properties.getGridFsDatabase();
			if (StringUtils.hasText(gridFsDatabase)) {
				return this.mongoDbFactory.getDb(gridFsDatabase);
			}
			return this.mongoDbFactory.getDb();
		}

		@Override
		public DB getDb(String dbName) throws DataAccessException {
			return this.mongoDbFactory.getDb(dbName);
		}

		@Override
		public PersistenceExceptionTranslator getExceptionTranslator() {
			return this.mongoDbFactory.getExceptionTranslator();
		}
	}
}
```

关键注解解释：
**@ConditionalOnClass({ Mongo.class, MongoTemplate.class })**：

该注解签名已经提到了， 如果`classpath`里有`MongoTemplate.class`该配置类才会被启用。

**@EnableConfigurationProperties(MongoProperties.class)**

前面已经提到过。

**@AutoConfigureAfter(MongoAutoConfiguration.class)**

该注解非常重要，表示在该配置类必须在`MongoAutoConfiguration.class`后使用。

从上面的代码可以看出，该类创建了好多Bean，其中就包括了`mongoTemplate`和`gridFsTemplate`

`mongoTemplate`例子：

```java
@Resource
private MongoTemplate mongoTemplate;
public void testMongoTemplate(String orderCode){
   DB db = mongoTemplate.getDb();
   mongoTemplate.getCollection("");
   mongoTemplate.createCollection("");
   mongoTemplate.find(Query.query(Criteria.where("id").is("1")), User.class, "users");
}
```

### MongoRepository介绍

提到`MongoRepository`就不能不完整的说明spring-data中的一些核心接口和类

#### Repository

```java
public interface Repository<T, ID extends Serializable> {

}
```

`Repository`接口是spring-data里最核心的仓库`标记`接口。用来获取域对象的类型和它的ID类型，核心的目的是让开发者扩展该接口，spring可以在`classpath`中扫描到该类型的接口并创建对应的Bean。

#### CrudRepository

`CrudRepository`继承自接口`Repository`,并提供了一些基本的`CRUD`方法来操作特定仓库上实体对象。

#### PagingAndSortingRepository

`PagingAndSortingRepository`继承自接口`CrudRepository`,比提供了基本的排序分页功能。

`MongoRepository`接口就继承自该接口，并提供了特定于mongo的一个操作方法。

### MongoRepository操作mongo

我们可以定义自己mongo DAO接口来实现上述接口没有提供的功能，例如：

```java
public interface UserMongoDao extends MongoRepository<User, Long> {

    public User findById(Long id);

    public List<User> findByName(String name);
}
// 测试代码
@Resource
private UserMongoDao userMongoDao;

@Test
public void testUserMongoDao(){
   User user = userMongoDao.findById(Long.valueOf(1));
   Assert.assertTrue(null != user);
}
```

### 配置

上面的代码是如何生效的呢？ 这个其实是利用的spring和springboot的约定大于配置的机制，显示的配置如下：

```java
@EnableJpaRepositories(basePackages = "com.leokongwq.springcloud.bookservice.dal.mysql.dao")
@EnableMongoRepositories(basePackages = "com.leokongwq.springcloud.bookservice.dal.mongo.dao")
interface Configuration { }
```

或

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans:beans xmlns:beans="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns="http://www.springframework.org/schema/data/jpa"
  xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/data/jpa
    http://www.springframework.org/schema/data/jpa/spring-jpa.xsd">

    <repositories base-package="com.leokongwq.springcloud.bookservice.dal.mongo.dao" />
    
    <!-- 过滤机制 -->
    <repositories base-package="com.acme.repositories">
        <context:exclude-filter type="regex" expression=".*SomeRepository" />
    </repositories>

</beans:beans>
```

### 参考

[https://docs.mongodb.com/manual/reference/connection-string/](https://docs.mongodb.com/manual/reference/connection-string/)

[https://docs.spring.io/spring-data/jpa/docs/current/reference/html/](https://docs.spring.io/spring-data/jpa/docs/current/reference/html)

