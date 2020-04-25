---
layout: post
comments: true
title: spring扩展点及springboot自动配置总结
date: 2018-03-06 22:57:35
tags:
- spring
categories:
- web
---

## spring扩展点和自动配置

### BeanFactoryPostProcessor

```java
public interface BeanFactoryPostProcessor {  
  
    /** 
     * Modify the application context's internal bean factory after its standard 
     * initialization. All bean definitions will have been loaded, but no beans 
     * will have been instantiated yet. This allows for overriding or adding 
     * properties even to eager-initializing beans. 
     * @param beanFactory the bean factory used by the application context 
     * @throws org.springframework.beans.BeansException in case of errors 
     */  
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;  
  
} 
```

通过该扩展点，我们修改，新增`BeanDefinition`。因为此时所有的`BeanDefinition`已经加载，但是没有Bean被创建。一般用在需要覆盖或替换Bean的属性时。

<!-- more -->

spring中，有内置的一些BeanFactoryPostProcessor实现类，常用的有：

- org.springframework.beans.factory.config.PropertyPlaceholderConfigurer
- org.springframework.beans.factory.config.PropertyOverrideConfigurer
- org.springframework.beans.factory.config.CustomEditorConfigurer：用来注册自定义的属性编辑器


### BeanFactoryPostProcessor

```java
public interface BeanPostProcessor {

/**
 * Apply this BeanPostProcessor to the given new bean instance <i>before</i> any bean
 * initialization callbacks (like InitializingBean's {@code afterPropertiesSet}
 * or a custom init-method). The bean will already be populated with property values.
 * The returned bean instance may be a wrapper around the original.
 * @param bean the new bean instance
 * @param beanName the name of the bean
 * @return the bean instance to use, either the original or a wrapped one;
 * if {@code null}, no subsequent BeanPostProcessors will be invoked
 * @throws org.springframework.beans.BeansException in case of errors
 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
 */
Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

/**
 * Apply this BeanPostProcessor to the given new bean instance <i>after</i> any bean
 * initialization callbacks (like InitializingBean's {@code afterPropertiesSet}
 * or a custom init-method). The bean will already be populated with property values.
 * The returned bean instance may be a wrapper around the original.
 * <p>In case of a FactoryBean, this callback will be invoked for both the FactoryBean
 * instance and the objects created by the FactoryBean (as of Spring 2.0). The
 * post-processor can decide whether to apply to either the FactoryBean or created
 * objects or both through corresponding {@code bean instanceof FactoryBean} checks.
 * <p>This callback will also be invoked after a short-circuiting triggered by a
 * {@link InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation} method,
 * in contrast to all other BeanPostProcessor callbacks.
 * @param bean the new bean instance
 * @param beanName the name of the bean
 * @return the bean instance to use, either the original or a wrapped one;
 * if {@code null}, no subsequent BeanPostProcessors will be invoked
 * @throws org.springframework.beans.BeansException in case of errors
 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
 * @see org.springframework.beans.factory.FactoryBean
 */
Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

该扩展点提供了两个方法：

1. 是在Bean新创建后，未初始化前调用的。例如在`InitializingBean`的`afterPropertiesSet`前，或则自定义的`init-method`前。
2. 在Bean初始化后，调用方法`postProcessAfterInitialization`。


spring中，有内置的一些BeanPostProcessor实现类，例如：

- org.springframework.context.annotation.CommonAnnotationBeanPostProcessor：支持@Resource注解的注入
- org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor：支持@Required注解的注入
- org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor：支持@Autowired注解的注入
- org.springframework.orm.jpa.support.PersistenceAnnotationBeanPostProcessor：支持@PersistenceUnit和@PersistenceContext注解的注入
- org.springframework.context.support.ApplicationContextAwareProcessor：用来为bean注入ApplicationContext等容器对象

spring提供的这些Bean都可以通过`<context:annotation-config/>` 来自动进行配置。

不过我们一般都会配置包扫描：`<context:component-scan base-package=”xx.yy.zz”/> `，该配置已经包含了:`<context:annotation-config/>`。

### @Configuration

> Indicates that a class declares one or more {@link Bean @Bean} methods and
  may be processed by the Spring container to generate bean definitions and
  service requests for those beans at runtime, for example:
  
`@Configuration`是spring3.0引入的新注解。该注解主要用来为spring容器提供`BeanDefinition`和在运行时提供其它Bean所需要的Bean。

举个例子：

```java
@Configuration
public class AppConfig {
    
    @Inject
    private DataSource dataSource;
    
    @Bean
    public MyBean myBean() {
        return new MyBean(dataSource);
    }
    
    @Configuration
    static class DatabaseConfig {
        @Bean
        DataSource dataSource() {
            return new EmbeddedDatabaseBuilder().build();
        }
    }
}
```

## springboot自动配置

### @EnableAutoConfiguration

要想使用springboot提供的自动配置功能，必须在你工程的启动主类上添加注解`@EnableAutoConfiguration`。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	/**
	 * Exclude specific auto-configuration classes such that they will never be applied.
	 * @return the classes to exclude
	 */
	Class<?>[] exclude() default {};

	/**
	 * Exclude specific auto-configuration class names such that they will never be
	 * applied.
	 * @return the class names to exclude
	 * @since 1.3.0
	 */
	String[] excludeName() default {};

}
```

这个注解的功能概况如下：

1. 自动配置基于应用的`classpath`以及你定义了什么Beans
2. 可以通过设置注解的`excludeName`属性或者通过`spring.autoconfigure.exclude`配置项来指定不需要自动配置的项目。
3. 自动配置的发生时机在用户定义的Beans被注册之后
4. 最好将`@EnableAutoConfiguration`注解放在`root package`的类上，这样就能够搜索到所有子`packages`中的类了
5. 自动配置类就是普通的Spring `@Configuration`类，通过`SpringFactoriesLoader`机制完成加载，实现上通常使用@Conditional(比如`@ConditionalOnClass`或者`@ConditionalOnMissingBean`)

不过在较新的springboot版本，可以直接使用注解`@SpringBootApplication`，原因如下：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class))
public @interface SpringBootApplication {
}
```

`@SpringBootApplication`已经包含了`@EnableAutoConfiguration`。


### @Import

```java
/**
 * @author Chris Beams
 * @author Juergen Hoeller
 * @since 3.0
 * @see Configuration
 * @see ImportSelector
 * @see ImportResource
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Import {

	/**
	 * {@link Configuration}, {@link ImportSelector}, {@link ImportBeanDefinitionRegistrar}
	 * or regular component classes to import.
	 */
	Class<?>[] value();

}
```

`@Import`注解的功能如下：

1. 该注解的功能和spring xml配置文件中的`<import/>`标签相同。
2. 可以用来导入的类有`@Configuration`,`@ImportSelector`,`@ImportBeanDefinitionRegistrar`
3. 要访问通过`@Import`导入的，定义在`@Configuration`中的类，应该通过`@Autowired`进行注入。

### @ImportResource

```java
/**
 * @author Chris Beams
 * @author Juergen Hoeller
 * @author Sam Brannen
 * @since 3.0
 * @see Configuration
 * @see Import
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
public @interface ImportResource {
    @AliasFor("locations")
    String[] value() default {};
    
    /**
	 * Resource locations from which to import.
	 * <p>Supports resource-loading prefixes such as {@code classpath:},
	 * {@code file:}, etc.
	 * <p>Consult the Javadoc for {@link #reader} for details on how resources
	 * will be processed.
	 * @since 4.2
	 * @see #value
	 * @see #reader
	 */
    @AliasFor("value")
 	 String[] locations() default {};
 	 
 	 Class<? extends BeanDefinitionReader> reader() default BeanDefinitionReader.class;
}
```

用来导入资源一个或多个包含`bean definitions`的资源文件。如果资源文件以`.groovy`结尾，那么就使用`GroovyBeanDefinitionReader`，否则使用`XmlBeanDefinitionReader`进行解析。

### @AutoConfigurationPackage

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {
    /**
	 * {@link Configuration}, {@link ImportSelector}, {@link ImportBeanDefinitionRegistrar}
	 * or regular component classes to import.
	 */
	Class<?>[] value();
}
```

该注解类的作用就是告诉spring:打了该注解类所在的包下面的被注解类，需要被`AutoConfigurationPackages`类进行注册。具体的实现原理可以参考第四小节提到的Bean动态注册，关于`ImportBeanDefinitionRegistrar`的解释。


#### @Import(EnableAutoConfigurationImportSelector.class)

`@EnableAutoConfiguration`注解的另外一个作用就是引入了`EnableAutoConfigurationImportSelector`

它的类图如下所示：

{% asset_img EnableAutoConfigurationImportSelector.png %}

可以发现它除了实现几个Aware类接口外，最关键的就是实现了`DeferredImportSelector`(继承自`ImportSelector`)接口。

所以我们先来看看`ImportSelector`以及`DeferredImportSelector`接口的定义：

```java
public interface ImportSelector {
    /**
     * 基于被引入的Configuration类的AnnotationMetadata信息选择并返回需要引入的类名列表
     */
    String[] selectImports(AnnotationMetadata importingClassMetadata);
}
```

这个接口的Javadoc比较长，还是捡重点说明一下：

1. 功能通过`selectImports`方法实现，用于筛选可以`@Import`进的`@Configuration`配置类。
2. 实现了`ImportSelector`接口的类也可以实现一系列`Aware`接口，这些`Aware`接口中的相应方法会在`selectImports`方法之前被调用(这一点通过上面的类图也可以佐证，`EnableAutoConfigurationImportSelector`确实实现了四个Aware类型的接口)
3. `ImportSelector`的实现和通常的`@Import`在处理方式上是一致的，然而还是可以在所有`@Configuration`类都被处理后再进行引入筛选(具体看下面即将介绍的`@DeferredImportSelector`)

```java
/**
* A variation of {@link ImportSelector} that runs after all {@code @Configuration} beans
* have been processed. This type of selector can be particularly useful when the selected
* imports are {@code @Conditional}.
/
public interface DeferredImportSelector extends ImportSelector {
}
```

这个接口是一个标记接口，它本身没有定义任何方法。那么这个接口的含义是什么呢：

它是`ImportSelector`接口的一个变体，在所有的`@Configuration`被处理之后才会执行。在需要筛选的引入类型具备`@Conditional`注解的时候非常有用实现类同样也可以实现`Ordered`接口，来定义多个`DeferredImportSelector`的优先级别(当然了，也可以使用注解`@Order`)

下面来看看是如何实现的：

```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    //可以通过配置`spring.boot.enableautoconfiguration`来关闭该功能，默认是true
    if (!isEnabled(annotationMetadata)) {
        return NO_IMPORTS;
    }
    try {
      // Step1: 得到注解信息
        AutoConfigurationMetadata autoConfigurationMetadata =   AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);
        // Step2: 得到注解中的所有属性信息
        AnnotationAttributes attributes = getAttributes(annotationMetadata);
        // Step3: 得到候选配置列表
        List<String> configurations = getCandidateConfigurations(annotationMetadata,
                attributes);
        // Step4: 去重
        configurations = removeDuplicates(configurations);
        // Step5: 排序
        configurations = sort(configurations, autoConfigurationMetadata);
        // Step6: 根据注解中的exclude信息去除不需要的
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        checkExcludedClasses(configurations, exclusions);
        configurations.removeAll(exclusions);
        configurations = filter(configurations, autoConfigurationMetadata);
        // Step7: 派发事件
        fireAutoConfigurationImportEvents(configurations, exclusions);
        return configurations.toArray(new String[configurations.size()]);
    }
    catch (IOException ex) {
        throw new IllegalStateException(ex);
    }
}
```

注解`@EnableAutoConfiguration`一般加在应用的启动主类上，并且提供了两个配置属性:`exclude`和`excludeName`来指定排除那些配置类。

所以上面代码中的`attributes`变量的值其实就是注解`@EnableAutoConfiguration`的配置属性

核心就在于上面的步骤3：

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
        AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
            getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
    Assert.notEmpty(configurations,
            "No auto configuration classes found in META-INF/spring.factories. If you "
                    + "are using a custom packaging, make sure that file is correct.");
    return configurations;
}
```

它将实现委托给了`SpringFactoriesLoader`的`loadFactoryNames`方法:

```java
// 传入的factoryClass：org.springframework.boot.autoconfigure.EnableAutoConfiguration
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
    String factoryClassName = factoryClass.getName();
    try {
        Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        List<String> result = new ArrayList<String>();
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
            String factoryClassNames = properties.getProperty(factoryClassName);
            result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
        }
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() +
                "] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}

// 相关常量
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
```

这段代码的意图很明确，它会从类路径中拿到所有名为`META-INF/spring.factories`的配置文件，然后按照`factoryClass`的名称取到对应的值。那么我们就来找一个`META-INF/spring.factories`配置文件看看.

#### META-INF/spring.factories

比如spring-boot-autoconfigure包：

```java
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.cloud.CloudAutoConfiguration,\
# 省略了很多
```

> 如果需要禁用某些自动配置功能， 可以通过配置`@EnableAutoConfiguration`的`exclude`属性。


上面的文件列举了非常多的自动配置候选项，挑一个AOP相关的`AopAutoConfiguration`看看究竟：

```java
// 如果设置了spring.aop.auto=false，那么AOP不会被配置
// 需要检测到@EnableAspectJAutoProxy注解存在才会生效
// 默认使用JdkDynamicAutoProxyConfiguration，如果设置了spring.aop.proxy-target-class=true，那么使用CglibAutoProxyConfiguration
@Configuration
@ConditionalOnClass({ EnableAspectJAutoProxy.class, Aspect.class, Advice.class })
@ConditionalOnProperty(prefix = "spring.aop", name = "auto", havingValue = "true", matchIfMissing = true)
public class AopAutoConfiguration {

    @Configuration
    @EnableAspectJAutoProxy(proxyTargetClass = false)
    @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false", matchIfMissing = true)
    public static class JdkDynamicAutoProxyConfiguration {

    }

    @Configuration
    @EnableAspectJAutoProxy(proxyTargetClass = true)
    @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true", matchIfMissing = false)
    public static class CglibAutoProxyConfiguration {

    }
}
```

这个自动配置类的作用是判断是否存在配置项：

```
spring.aop.proxy-target-class=true
```

如果存在并且值为`true`的话使用基于`CGLIB`字节码操作的动态代理方案，否则使用JDK自带的动态代理机制。

在这个配置类中，使用到了两个全新的注解：

- @ConditionalOnClass
- @ConditionalOnProperty

从这两个注解的名称，就大概能够猜出它们的功能了：

### @ConditionalOnClass

当类路径上存在指定的类时，满足条件。

### @ConditionalOnProperty

当配置中存在指定的属性时，满足条件。

其实除了这两个注解之外，还有几个类似的，它们都在`org.springframework.boot.autoconfigure.condition`这个包下，在具体介绍实现之前，下面先来看看Spring Boot对于`@Conditional`的扩展。

### Spring Boot对于@Conditional的扩展

Spring Boot提供了一个实现了`Condition`接口的抽象类`SpringBootCondition`。

这个类的主要作用是打印一些用于诊断的日志，告诉用户哪些类型被自动配置了。

它实现`Condition`接口的方法：

```java
@Override
public final boolean matches(ConditionContext context,
        AnnotatedTypeMetadata metadata) {
    String classOrMethodName = getClassOrMethodName(metadata);
    try {
        ConditionOutcome outcome = getMatchOutcome(context, metadata);
        logOutcome(classOrMethodName, outcome);
        recordEvaluation(context, classOrMethodName, outcome);
        return outcome.isMatch();
    }
    catch (NoClassDefFoundError ex) {
        throw new IllegalStateException(
                "Could not evaluate condition on " + classOrMethodName + " due to "
                        + ex.getMessage() + " not "
                        + "found. Make sure your own configuration does not rely on "
                        + "that class. This can also happen if you are "
                        + "@ComponentScanning a springframework package (e.g. if you "
                        + "put a @ComponentScan in the default package by mistake)",
                ex);
    }
    catch (RuntimeException ex) {
        throw new IllegalStateException(
                "Error processing condition on " + getName(metadata), ex);
    }
}
/**
 * Determine the outcome of the match along with suitable log output.
 * @param context the condition context
 * @param metadata the annotation metadata
 * @return the condition outcome
 */
public abstract ConditionOutcome getMatchOutcome(ConditionContext context,
        AnnotatedTypeMetadata metadata);
```

SpringBootCondition已经提供了基本的实现，将内部的匹配细节定义成抽象方法getMatchOutcome，交给其子类去完成。

另外，还提供了两个可能会被子类使用到的方法：

```java
/**
 * 如果指定的conditions中有任意一个匹配，那么就返回true
 * @param context the context
 * @param metadata the annotation meta-data
 * @param conditions conditions to test
 * @return {@code true} if any condition matches.
 */
protected final boolean anyMatches(ConditionContext context,
        AnnotatedTypeMetadata metadata, Condition... conditions) {
    for (Condition condition : conditions) {
        if (matches(context, metadata, condition)) {
            return true;
        }
    }
    return false;
}

/**
 * 检查指定的condition是否匹配
 * @param context the context
 * @param metadata the annotation meta-data
 * @param condition condition to test
 * @return {@code true} if the condition matches.
 */
protected final boolean matches(ConditionContext context,
        AnnotatedTypeMetadata metadata, Condition condition) {
    if (condition instanceof SpringBootCondition) {
        return ((SpringBootCondition) condition).getMatchOutcome(context, metadata)
                .isMatch();
    }
    return condition.matches(context, metadata);
}
```

### org.springframework.boot.autoconfigure.condition包

除了上面已经遇到的`@ConditionalOnClass`和`@ConditionalOnProperty`，这个包中还定义了很多条件实现类，下面简单列举几个：

### @ConditionalOnExpression - 基于SpEL的条件判断

```java
/**
 * Configuration annotation for a conditional element that depends on the value of a SpEL
 * expression.
 *
 * @author Dave Syer
 */
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.TYPE, ElementType.METHOD })
@Documented
@Conditional(OnExpressionCondition.class)
public @interface ConditionalOnExpression {

    /**
     * The SpEL expression to evaluate. Expression should return {@code true} if the
     * condition passes or {@code false} if it fails.
     * @return the SpEL expression
     */
    String value() default "true";
```

然后相应的实现类是`OnExpressionCondition`，它继承自`SpringBootCondition`。

### @ConditionalOnMissingClass - 基于类不存在与classpath的条件判断

这一个条件实现正好和`@ConditionalOnClass`条件相反

下面列举所有由Spring Boot提供的条件注解：

* @ConditionalOnBean
* @ConditionalOnClass
* @ConditionalOnCloudPlatform
* @ConditionalOnExpression
* @ConditionalOnJava
* @ConditionalOnJndi
* @ConditionalOnMissingBean
* @ConditionalOnMissingClass
* @ConditionalOnNotWebApplication
* @ConditionalOnProperty
* @ConditionalOnResource
* @ConditionalOnSingleCandidate
* @ConditionalOnWebApplication


一般的模式，就是一个条件注解对应一个继承自SpringBootCondition的具体实现类。
 
## 动态注册Bean

在某些特殊的场景下，我们需要动态的向spring的容器注册Bean。spring框架本身提供了一些机制来帮助我们来实现这些特殊的需求。

### BeanDefinitionRegistry

```java
public interface BeanDefinitionRegistry extends AliasRegistry {
    
    /**
	 * Register a new bean definition with this registry.
	 * Must support RootBeanDefinition and ChildBeanDefinition.
	 * @param beanName the name of the bean instance to register
	 * @param beanDefinition definition of the bean instance to register
	 * @throws BeanDefinitionStoreException if the BeanDefinition is invalid
	 * or if there is already a BeanDefinition for the specified bean name
	 * (and we are not allowed to override it)
	 * @see RootBeanDefinition
	 * @see ChildBeanDefinition
	 */
    void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException;
}
```

BeanDefinitionRegistry 扩展点允许我们来动态的想spring容器注册`BeanDefinition`。

### SingletonBeanRegistry 

```java
public interface SingletonBeanRegistry {
    /**
	 * Register the given existing object as singleton in the bean registry,
	 * under the given bean name.
	 * <p>The given instance is supposed to be fully initialized; the registry
	 * will not perform any initialization callbacks (in particular, it won't
	 * call InitializingBean's {@code afterPropertiesSet} method).
	 * The given instance will not receive any destruction callbacks
	 * (like DisposableBean's {@code destroy} method) either.
	 * <p>When running within a full BeanFactory: <b>Register a bean definition
	 * instead of an existing instance if your bean is supposed to receive
	 * initialization and/or destruction callbacks.</b>
	 * <p>Typically invoked during registry configuration, but can also be used
	 * for runtime registration of singletons. As a consequence, a registry
	 * implementation should synchronize singleton access; it will have to do
	 * this anyway if it supports a BeanFactory's lazy initialization of singletons.
	 * @param beanName the name of the bean
	 * @param singletonObject the existing singleton object
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 * @see org.springframework.beans.factory.DisposableBean#destroy
	 * @see org.springframework.beans.factory.support.BeanDefinitionRegistry#registerBeanDefinition
	 */
    void registerSingleton(String beanName, Object singletonObject);
}
```

`SingletonBeanRegistry`允许我们来直接向Spring容器注册Singleton的Bean实例。

两者区别在于使用前者时，Spring容器会根据BeanDefinition实例化bean实例，而使用后者时，bean实例就是传递给registerSingleton方法的对象。

DefaultListableBeanFactory接口同时实现了这两个接口，在实践中通常会使用这个接口。

#### 在普通bean中进行动态注册

可以在任何获得了`BeanDefinitionRegistry`或者`SingletonBeanRegistry`实例的地方进行动态注册。

但是如果bean不是在`BeanFactoryPostProcessor`中被注册，那么该bean则无法被**BeanPostProcessor**处理，即无法对其应用aop、Bean Validation等功能。

```java
RestController
@Slf4j
public class PersonManagerRegisterController {

    /**
     * The Application context.
     */
    @Autowired
    GenericApplicationContext applicationContext;

    /**
     * The Bean factory.
     */
    @Autowired
    ConfigurableBeanFactory beanFactory;

    /**
     * 动态注册bean，此处注册的bean没有AOP的支持
     * curl http://localhost:8080/registerPersonManager
     */
    @GetMapping("/registerPersonManager")
    public void registerPersonManager() {
        PersonDao personDao = applicationContext.getBean(PersonDao.class);
        PersonManager personManager = new PersonManager();
        personManager.setPersonDao(personDao);
        beanFactory.registerSingleton("personManager3", personManager);

    }
}
```

#### 在`BeanFactoryPostProcessor`中进行动态注册

在Spring容器的启动过程中，`BeanFactory`载入bean的定义后会立刻执行`BeanFactoryPostProcessor`，此时动态注册bean，则可以保证动态注册的bean被`BeanPostProcessor`处理，并且可以保证其的实例化和初始化总是先于依赖它的bean。

```java
@Component
@Slf4j
public class PersonBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        DefaultListableBeanFactory defaultListableBeanFactory
                = (DefaultListableBeanFactory) beanFactory;

        //注册Bean定义，容器根据定义返回bean
        log.info("register personManager1>>>>>>>>>>>>>>>>");
        BeanDefinitionBuilder beanDefinitionBuilder =
                BeanDefinitionBuilder.genericBeanDefinition(PersonManager.class);
        beanDefinitionBuilder.addPropertyReference("personDao", "personDao");
        BeanDefinition personManagerBeanDefinition = beanDefinitionBuilder.getRawBeanDefinition();
        defaultListableBeanFactory.registerBeanDefinition("personManager1", personManagerBeanDefinition);

        //注册bean实例
        log.info("register personManager2>>>>>>>>>>>>>>>>");
        PersonDao personDao = beanFactory.getBean(PersonDao.class);
        PersonManager personManager = new PersonManager();
        personManager.setPersonDao(personDao);
        beanFactory.registerSingleton("personManager2", personManager);

    }
}
```

### BeanDefinitionRegistryPostProcessor

```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {

	/**
	 * Modify the application context's internal bean definition registry after its
	 * standard initialization. All regular bean definitions will have been loaded,
	 * but no beans will have been instantiated yet. This allows for adding further
	 * bean definitions before the next post-processing phase kicks in.
	 * @param registry the bean definition registry used by the application context
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;

}
```

`BeanDefinitionRegistryPostProcessor` 这个类也可以提供`BeanDefinition`注册的扩展点。

### ConfigurationClassPostProcessor

```java
/**
 * {@link BeanFactoryPostProcessor} used for bootstrapping processing of
 * {@link Configuration @Configuration} classes.
 *
 * <p>Registered by default when using {@code <context:annotation-config/>} or
 * {@code <context:component-scan/>}. Otherwise, may be declared manually as
 * with any other BeanFactoryPostProcessor.
 *
 * <p>This post processor is {@link Ordered#HIGHEST_PRECEDENCE} as it is important
 * that any {@link Bean} methods declared in Configuration classes have their
 * respective bean definitions registered before any other BeanFactoryPostProcessor
 * executes.
 *
 * @author Chris Beams
 * @author Juergen Hoeller
 * @author Phillip Webb
 * @since 3.0
 */
public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor,
		PriorityOrdered, ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware {
```

这个类的主要功能就是用来处理打了`@Configuration`注解的Java配置类。解析配置类，并注册`BeanDefinition`到`BeanDefinitionRegistry`

最重要的方法如下：

```java
/**
* Derive further bean definitions from the configuration classes in the registry.
*/
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    int registryId = System.identityHashCode(registry);
    if (this.registriesPostProcessed.contains(registryId)) {
    	throw new IllegalStateException(
    			"postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
    }
    if (this.factoriesPostProcessed.contains(registryId)) {
    	throw new IllegalStateException(
    			"postProcessBeanFactory already called on this post-processor against " + registry);
    }
    this.registriesPostProcessed.add(registryId);
    
    processConfigBeanDefinitions(registry);
}

/**
 * Build and validate a configuration model based on the registry of
 * {@link Configuration} classes.
 */
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    //代码太多了。
}
```

### ImportBeanDefinitionRegistrar

```java
/**
* Interface to be implemented by types that register additional bean definitions when
* processing @{@link Configuration} classes. Useful when operating at the bean definition
* level (as opposed to {@code @Bean} method/instance level) is desired or necessary.
*/
public interface ImportBeanDefinitionRegistrar {

	/**
	 * Register bean definitions as necessary based on the given annotation metadata of
	 * the importing {@code @Configuration} class.
	 * <p>Note that {@link BeanDefinitionRegistryPostProcessor} types may <em>not</em> be
	 * registered here, due to lifecycle constraints related to {@code @Configuration}
	 * class processing.
	 * @param importingClassMetadata annotation metadata of the importing class
	 * @param registry current bean definition registry
	 */
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,     BeanDefinitionRegistry registry);
}
```

Spring官方在动态注册bean时，大部分套路其实是使用ImportBeanDefinitionRegistrar接口。

所有实现了该接口的类的都会被`ConfigurationClassPostProcessor`处理，`ConfigurationClassPostProcessor`实现了`BeanFactoryPostProcessor`接口，所以`ImportBeanDefinitionRegistrar`中动态注册的bean是优先与依赖其的bean初始化的，也能被aop、validator等机制处理。

使用方法

`ImportBeanDefinitionRegistrar`需要配合`@Configuration`和`@Import`注解，`@Configuration`定义Java格式的Spring配置文件，`@Import`注解导入实现了`ImportBeanDefinitionRegistrar`接口的类。

例子：

首先编写一个实现了接口`ImportBeanDefinitionRegistrar`的类，来实现我们自定义的Bean注册功能

```java
@Component
public class UranusConfigRegistrar implements ImportBeanDefinitionRegistrar {

  @Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
      System.out.println(registry.getClass());
      System.out.println(importingClassMetadata.getClassName());
  }
}
```

随后我们定义两个配置类：

```java
@Configuration
@Import(UranusConfigRegistrar.class)
public class DatabaseConfig {

}

@Configuration
@Import(UranusConfigRegistrar.class)
public class WebConfig {
}
```

启动springboot应用，我们能再控制台看到如下的输出：

```java
class org.springframework.beans.factory.support.DefaultListableBeanFactory
com.leokongwq.springcloud.bookservice.config.DatabaseConfig

class org.springframework.beans.factory.support.DefaultListableBeanFactory
com.leokongwq.springcloud.bookservice.config.WebConfig
```

从输出我们可以知道，方法`registerBeanDefinitions`的参数`AnnotationMetadata`里面包含`@Configuration`配置类的注解原信息。`registry`参数其实就是spring容器。

通过`AnnotationMetadata`我们能获取到`@Configuration`配置类上的所有注解信息，包括配置类里面加了注解的方法配置信息。

通过该机制，我们可以在`@Configuration`配置类上添加配置原元数据注解，以此来实现动态注册Bean的功能。

例如:

配置注解类：`EnableRedisCache`

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Import(UranusConfigRegistrar.class)
public @interface EnableRedisCache {

    String host() default "127.0.0.1";

    int port() default 9527;
}
```

启用注解`EnableRedisCache`

```java
@SpringBootApplication
@EnableRedisCache
public class ApplicationBootstrap {
	public static void main(String[] args) {
        SpringApplication.run(ApplicationBootstrap.class, args); 
	}
}
```

启动应用，可以在控制台看到如下的输出：

```java
class org.springframework.beans.factory.support.DefaultListableBeanFactory
com.leokongwq.springcloud.bookservice.config.WebConfig
host=127.0.0.1
port=9527
```

有了配置的元数据，我们就可以自由发挥，动态的生成一些Bean。


举一个springboot中的例子：

```java
/**
 * {@link ImportBeanDefinitionRegistrar} used by {@link ServletComponentScan}.
 *
 * @author Andy Wilkinson
 * @author Stephane Nicoll
 */
class ServletComponentScanRegistrar implements ImportBeanDefinitionRegistrar {
    private static final String BEAN_NAME = "servletComponentRegisteringPostProcessor";

	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
			BeanDefinitionRegistry registry) {
		Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);
		if (registry.containsBeanDefinition(BEAN_NAME)) {
			updatePostProcessor(registry, packagesToScan);
		}
		else {
			addPostProcessor(registry, packagesToScan);
		}
	}
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(ServletComponentScanRegistrar.class)
public @interface ServletComponentScan {
    /**
	 * Alias for the {@link #basePackages()} attribute. Allows for more concise annotation
	 * declarations e.g.: {@code @ServletComponentScan("org.my.pkg")} instead of
	 * {@code @ServletComponentScan(basePackages="org.my.pkg")}.
	 * @return the base packages to scan
	 */
	@AliasFor("basePackages")
	String[] value() default {};

	/**
	 * Base packages to scan for annotated servlet components. {@link #value()} is an
	 * alias for (and mutually exclusive with) this attribute.
	 * <p>
	 * Use {@link #basePackageClasses()} for a type-safe alternative to String-based
	 * package names.
	 * @return the base packages to scan
	 */
	@AliasFor("value")
	String[] basePackages() default {};

	/**
	 * Type-safe alternative to {@link #basePackages()} for specifying the packages to
	 * scan for annotated servlet components. The package of each class specified will be
	 * scanned.
	 * @return classes from the base packages to scan
	 */
	Class<?>[] basePackageClasses() default {};
}
```

`ServletComponentScanRegistrar`实现了接口：`ImportBeanDefinitionRegistrar`。 同时注解`@ServletComponentScan`上也打了注解`@Import(ServletComponentScanRegistrar.class)`。

这样我们就可以在使用注解`ServletComponentScan`时，指定`basePackages`和`basePackageClasses`属性来配置Servlet规范里面的组件。Filter，Listener， Servlet等。

例如：

```java
@SpringBootApplication
@ServletComponentScan(basePackages = "com.leokongwq.springcloud.bookservice.web")
public class BookServiceApplication {

	public static void main(String[] args) {
		SpringApplication.run(BookServiceApplication.class, args);
	}
}
```

## SpringFactoriesLoader 机制

SpringFactoriesLoader 的主要目的是在Spring框架内部完成工厂的加载机制。

主要的方法有2个：

```java
public static <T> List<T> loadFactories(Class<T> factoryClass, ClassLoader classLoader) {
		Assert.notNull(factoryClass, "'factoryClass' must not be null");
		ClassLoader classLoaderToUse = classLoader;
		if (classLoaderToUse == null) {
			classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
		}
		List<String> factoryNames = loadFactoryNames(factoryClass, classLoaderToUse);
		if (logger.isTraceEnabled()) {
			logger.trace("Loaded [" + factoryClass.getName() + "] names: " + factoryNames);
		}
		List<T> result = new ArrayList<T>(factoryNames.size());
		for (String factoryName : factoryNames) {
			result.add(instantiateFactory(factoryName, factoryClass, classLoaderToUse));
		}
		AnnotationAwareOrderComparator.sort(result);
		return result;
}

public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
		try {
			Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			List<String> result = new ArrayList<String>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
				String factoryClassNames = properties.getProperty(factoryClassName);
				result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
			}
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() +
					"] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
}
```

这两个方法都是从classpath的指定文件中获取需要加载的类。这个文件就是classpath下面的：`META-INF/spring.factories`。在签名的小节已经看到过这个文件。






