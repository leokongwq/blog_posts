---
layout: post
comments: true
title: spring对缓存的支持及相关注解说明
date: 2018-05-12 21:54:44
tags:
- spring
categories:
- web
---

Spring 3.1 引入了基于注释（annotation）的缓存（cache）技术，但是它本质上不是一个具体的缓存实现方案（例如 EHCache 或者 OSCache），而是一个对缓存使用的抽象，通过在既有代码中添加少量它定义的各种annotation，即能够达到缓存方法的返回对象的效果。

Spring 的缓存技术还具备相当的灵活性，不仅能够使用 `SpEL（Spring Expression Language）`来定义缓存的 `key`和各种`condition`，还提供开箱即用的缓存临时存储方案，也支持和主流的专业缓存例如`EHCache`集成。

<!-- more -->

### 一个例子

先从一个例子来直观感受一下spring Cache 如何使用。

#### 相关类

```java Product
@Data
public class Product {
    private Long id;
    private String name;
}
```

```java ProductDao
public interface ProductDao extends MongoRepository<Product, Long> {

    User findById(Long id);

    List<User> findByName(String name);
}
```

```java ProductService
@Service
public class ProductService {

    @Resource
    private ProductDao productDao;

    @Cacheable(cacheNames = {"products"}, key = "#id")
    public Product getProductById(Long id) {
        return productDao.findById(id);
    }
}
```

#### 说明

方法`getProductById`上加了注解`@Cacheable`，表示该方法的返回值是可以被缓存的。

属性`cacheNames`或`value`作用相同，都是用来指定返回结果保存到哪个缓存中的，属性`key`用来指定缓存结果对应的`key`(缓存通常都是key-value结构）。在改例中，key是由参数`id`进行计算的。


### Cacheable 详解

`@Cacheable`注解是Spring Cache中最重要的注解，它的配置属性如下：

#### value 

指定方法返回结果保存到的缓存名称。作用和`cacheNames`相同。支持`数组`配置

#### cacheNames

参考`value`属性解释

#### key

属性`key`的类型是字符串，格式为`SpEL`表达式，用来动态计算缓存结果的`键`值。默认值为空，意味着所有的方法参数都会参与计算。


#### keyGenerator

该属性指定自定义`KeyGenerator`的Bean的名称。默认值为空，意味着使用`SimpleKeyGenerator`

#### cacheManager 

类型为：`org.springframework.cache.CacheManager` 的bean的名称。该bean需要我们在spring配置文件中配置。在SpringBoot中，如果检测到有对应的缓存框架，则会自动配置一个名称为`cacheManager`的bean。

#### cacheResolver

类型为：`org.springframework.cache.interceptor.CacheResolver` 的bean的名称。

#### condition

类型为字符串，结果是SpEL表达式，指定缓存的条件。默认值为空字符串，意味着方法的返回值总是被缓存。

#### unless

类型为字符串，结果是SpEL表达式，和`condition`不同的是，该表示式是在方法调用会进行计算的，用来否决方法的缓存。默认是空字符串，意味着从来不否决缓存结果。

#### sync

类型是boolean，4.3版本开始支持。表示在多个线程同时`加载`同一个缓存key对应的资源时，是否进行同步。默认值为false。

如果设置为true, 则能避免重复加载。该属性的功能能否起作用取决于如下条件：

1. 不支持unless 
2. 只能指定一个缓存（cacheName取值唯一）
3. 不能和其它缓存相关的操作进行组合

### 缓存key的计算规则

上面提到缓存`key`的格式为SpEL表达式。默认的计算规则如下：

- 如果方法没有参数，则返回0
- 如果方法只有一个参数，则返回该参数。
- 如果方法有多余一个参数，则key的值由所有参数的hashcode计算而来。具体可以参考`SimpleKeyGenerator`

### KeyGenerator

`KeyGenerator` 接口用来计算方法返回值对应的缓存键的值。

```java
public interface KeyGenerator {
    Object generate(Object target, Method method, Object... params);
}
```

spring默认提供了两个实现类：`DefaultKeyGenerator`（4.0版本开始不推荐使用） 和 `SimpleKeyGenerator`（默认使用）。

我们可以通过实现该接口自定义实现。

### key SpEL 举例

```java
@Cacheable(value="books", key="#isbn"
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)


@Cacheable(value="books", key="#isbn.rawNumber")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)


@Cacheable(value="books", key="T(someType).hash(#isbn)")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)
```

SpEL语法可以参考：[Chapter 7, Spring Expression Language (SpEL)](https://docs.spring.io/spring/docs/3.1.x/spring-framework-reference/html/expressions.html)

### 条件缓存

```java
@Cacheable(value="book", condition="#name.length < 32")
public Book findBook(String name)
```

### SpEL计算上下文

{% asset_img SpEL-contect.jpg %}

### @CachePut 

如果缓存需要更新，且不干扰方法的执行,可以使用注解`@CachePut`。`@CachePut`标注的方法在执行前不会去检查缓存中是否存在之前执行过的结果，而是每次都会执行该方法，并将执行结果以键值对的形式存入指定的缓存中。

```java
@CachePut(cacheNames="book", key="#isbn")
public Book updateBook(ISBN isbn, BookDescriptor descriptor)
```

**注意**：应该避免@CachePut 和 @Cacheable同时使用的情况。

### @CacheEvict

spring cache不仅支持将数据缓存，还支持将缓存数据删除。此过程经常用于从缓存中清除过期或未使用的数据。 
`@CacheEvict`要求指定一个或多个缓存，使之都受影响。此外，还提供了一个额外的参数`allEntries` 。表示是否需要清除缓存中的所有元素。默认为`false`，表示不需要。当指定了`allEntries`为`true`时，Spring Cache将忽略指定的key。有的时候我们需要Cache一下清除所有的元素。

```java
@CacheEvict(cacheNames="books", allEntries=true)
public void loadBooks(InputStream batch)
```

清除操作默认是在对应**方法成功执行之后**触发的，即方法如果因为抛出异常而未能成功返回时也不会触发清除操作。使用`beforeInvocation`可以改变触发清除操作的时间，当我们指定该属性值为true时，Spring会在调用该方法之前清除缓存中的指定元素。

```java
@CacheEvict(cacheNames="books", beforeInvocation=true)
public void loadBooks(InputStream batch)
```

### @CacheConfig

有时候一个类中可能会有多个缓存操作，而这些缓存操作可能是重复的。这个时候可以使用@CacheConfig

```java
@CacheConfig("books")
public class BookRepositoryImpl implements BookRepository {

    @Cacheable
    public Book findBook(ISBN isbn) {...}
}
```

`@CacheConfig`是一个类级别的注解，允许共享缓存的名称、KeyGenerator、CacheManager 和CacheResolver。 
该操作会被覆盖。

### 开启缓存注解 

java类配置：

```java
@Configuration
@EnableCaching
public class AppConfig {
}
```

XML 配置

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:cache="http://www.springframework.org/schema/cache"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/cache http://www.springframework.org/schema/cache/spring-cache.xsd">

        <cache:annotation-driven />

</beans>
```

### 配置CacheManager

CacheManager是Spring定义的一个用来管理Cache的接口。Spring自身已经为我们提供了两种CacheManager的实现，一种是基于Java API的ConcurrentMap，另一种是基于第三方Cache实现——Ehcache，如果我们需要使用其它类型的缓存时，我们可以自己来实现Spring的CacheManager接口或AbstractCacheManager抽象类。下面分别来看看Spring已经为我们实现好了的两种CacheManager的配置示例。

#### 基于ConcurrentMap的配置
  
```xml
 <bean id="cacheManager" class="org.springframework.cache.support.SimpleCacheManager">
     <property name="caches">
        <set>
        <bean class="org.springframework.cache.concurrent.ConcurrentMapCacheFactoryBean"    p:name="xxx"/>
        </set>
    </property>
</bean>
```  

上面的配置使用的是一个SimpleCacheManager，其中包含一个名为“xxx”的ConcurrentMapCache。

#### 基于Ehcache的配置
 
 ```xml
 <bean id="cacheManager" class="org.springframework.cache.ehcache.EhCacheCacheManager" p:cache-manager-ref="ehcacheManager"/>

   <bean id="ehcacheManager" class="org.springframework.cache.ehcache.EhCacheManagerFactoryBean" p:config-location="ehcache-spring.xml"/>
```

上面的配置使用了一个Spring提供的EhCacheCacheManager来生成一个Spring的CacheManager，其接收一个Ehcache的CacheManager，因为真正用来存入缓存数据的还是Ehcache。Ehcache的CacheManager是通过Spring提供的EhCacheManagerFactoryBean来生成的，其可以通过指定ehcache的配置文件位置来生成一个Ehcache的CacheManager。若未指定则将按照Ehcache的默认规则取classpath根路径下的ehcache.xml文件，若该文件也不存在，则获取Ehcache对应jar包中的ehcache-failsafe.xml文件作为配置文件。更多关于Ehcache的内容这里就不多说了，它不属于本文讨论的内容，欲了解更多关于Ehcache的内容可以参考我之前发布的Ehcache系列文章，也可以参考官方文档等。


### 总结

- 通过少量的配置 annotation 注释即可使得既有代码支持缓存
- 支持开箱即用 Out-Of-The-Box，即不用安装和部署额外第三方组件即可使用缓存
- 支持 Spring Express Language，能使用对象的任何属性或者方法来定义缓存的 key 和 condition
- 支持 AspectJ，并通过其实现任何方法的缓存支持
- 支持自定义 key 和自定义缓存管理者，具有相当的灵活性和扩展性

### 资料

[https://www.ibm.com/developerworks/cn/opensource/os-cn-spring-cache/](https://www.ibm.com/developerworks/cn/opensource/os-cn-spring-cache/)



