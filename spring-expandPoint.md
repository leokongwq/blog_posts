---
layout: post
comments: true
title: spring扩展点整理
date: 2017-04-02 09:09:48
tags:
- spring
categories:
- java
- spring
---

### 背景

Spring的强大和灵活性不用再强调了。而灵活性就是通过一系列的扩展点来实现的，这些扩展点给应用程序提供了参与Spring容器创建的过程，好多定制化的东西都需要扩展点的支持。尤其在使用SpringBoot的过程中。

<!-- more -->

### BeanFactoryPostProcessor

这个扩展点是以一个接口定义的：

```java
/**
 * Allows for custom modification of an application context's bean definitions,
 * adapting the bean property values of the context's underlying bean factory.
 *
 * <p>Application contexts can auto-detect BeanFactoryPostProcessor beans in
 * their bean definitions and apply them before any other beans get created.
 *
 * <p>Useful for custom config files targeted at system administrators that
 * override bean properties configured in the application context.
 *
 * <p>See PropertyResourceConfigurer and its concrete implementations
 * for out-of-the-box solutions that address such configuration needs.
 *
 * <p>A BeanFactoryPostProcessor may interact with and modify bean
 * definitions, but never bean instances. Doing so may cause premature bean
 * instantiation, violating the container and causing unintended side-effects.
 * If bean instance interaction is required, consider implementing
 * {@link BeanPostProcessor} instead.
 *
 * @author Juergen Hoeller
 * @since 06.07.2003
 * @see BeanPostProcessor
 * @see PropertyResourceConfigurer
 */
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

BeanFactoryPostProcessor 这个扩展点的功能是：让应用程序在Spring创建Bean对象前修改BeanDefinition。其中一个例子就是Bean属性配置的类型转换，占位符替换。

### BeanDefinitionRegistryPostProcessor

该扩展点的定义如下：

```java
/**
 * Extension to the standard {@link BeanFactoryPostProcessor} SPI, allowing for
 * the registration of further bean definitions <i>before</i> regular
 * BeanFactoryPostProcessor detection kicks in. In particular,
 * BeanDefinitionRegistryPostProcessor may register further bean definitions
 * which in turn define BeanFactoryPostProcessor instances.
 *
 * @author Juergen Hoeller
 * @since 3.0.1
 * @see org.springframework.context.annotation.ConfigurationClassPostProcessor
 */
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

如上面的注释说明：该扩展点可以让应用程序注册自定义的BeanDefinition，并且该扩展点在 BeanFactoryPostProcessor 前执行。

### Aware

```java
/**
 * Marker superinterface indicating that a bean is eligible to be
 * notified by the Spring container of a particular framework object
 * through a callback-style method. Actual method signature is
 * determined by individual subinterfaces, but should typically
 * consist of just one void-returning method that accepts a single
 * argument.
 *
 * <p>Note that merely implementing {@link Aware} provides no default
 * functionality. Rather, processing must be done explicitly, for example
 * in a {@link org.springframework.beans.factory.config.BeanPostProcessor BeanPostProcessor}.
 * Refer to {@link org.springframework.context.support.ApplicationContextAwareProcessor}
 * and {@link org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory}
 * for examples of processing {@code *Aware} interface callbacks.
 *
 * @author Chris Beams
 * @since 3.1
 */
public interface Aware {

}
```

Aware 接口是一个标记接口，表示所有实现该接口的类是会被Spring容器选中，并得到某种通知。所有该接口的子接口提供固定的接收通知的方法。这样的接口有很多，最常用的如下：

```java
public interface ApplicationContextAware extends Aware {
	/**
	 * Set the ApplicationContext that this object runs in.
	 * Normally this call will be used to initialize the object.
	 * <p>Invoked after population of normal bean properties but before an init callback such
	 * as {@link org.springframework.beans.factory.InitializingBean#afterPropertiesSet()}
	 * or a custom init-method. Invoked after {@link ResourceLoaderAware#setResourceLoader},
	 * {@link ApplicationEventPublisherAware#setApplicationEventPublisher} and
	 * {@link MessageSourceAware}, if applicable.
	 * @param applicationContext the ApplicationContext object to be used by this object
	 * @throws ApplicationContextException in case of context initialization errors
	 * @throws BeansException if thrown by application context methods
	 * @see org.springframework.beans.factory.BeanInitializationException
	 */
	void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}
```

`ApplicationContextAware`的回调方法是在一般的属性注入后，但没有执行Bean的初始化方法前被执行的。初始化方法包括`init-method`和`InitializingBean`的`afterPropertiesSet`方法。

```java
/**
 * Interface to be implemented by any bean that wishes to be notified
 * of the {@link Environment} that it runs in.
 *
 * @author Chris Beams
 * @since 3.1
 */
public interface EnvironmentAware extends Aware {
	/**
	 * Set the {@code Environment} that this object runs in.
	 */
	void setEnvironment(Environment environment);
}
```

该接口用来接收应用程序的运行环境对象。

### ApplicationListener 

```java
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

	/**
	 * Handle an application event.
	 * @param event the event to respond to
	 */
	void onApplicationEvent(E event);

}
```

ApplicationListener 接口主要用来监听应用程序上下文的事件，不同的实现子类注册自己感兴趣的事件。

### InitializingBean

```java
public interface InitializingBean {

	/**
	 * Invoked by a BeanFactory after it has set all bean properties supplied
	 * (and satisfied BeanFactoryAware and ApplicationContextAware).
	 * <p>This method allows the bean instance to perform initialization only
	 * possible when all bean properties have been set and to throw an
	 * exception in the event of misconfiguration.
	 * @throws Exception in the event of misconfiguration (such
	 * as failure to set an essential property) or if initialization fails.
	 */
	void afterPropertiesSet() throws Exception;

}
```

InitializingBean 主要用来实现自定义的Bean初始化逻辑。 InitializingBean接口的方法会被BeanFactory在BeanFactoryAware或ApplicationContextAware接口方法调用后进行调用。

InitializingBean 接口的功能还有一个替代方式，就是在xml配置bean结点属性的`init-method`属性，指定初始化方法。需要注意的时候如同同时实现了InitializingBean接口并且配置了`init-method`属性，则InitializingBean接口的方法会被先执行。

### BeanPostProcessor

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

BeanPostProcessor 接口定义了在bean初始化前和初始化后执行的方法。应用程序自己可以实现该接口定义自己特殊的逻辑。

参考：[http://jinnianshilongnian.iteye.com/blog/1489787](http://jinnianshilongnian.iteye.com/blog/1489787) 和 [http://jinnianshilongnian.iteye.com/blog/1492424](http://jinnianshilongnian.iteye.com/blog/1492424)

### FactoryBean

```java
public interface FactoryBean<T> {
    T getObject() throws Exception;
    Class<?> getObjectType();
    boolean isSingleton();
}
```

通过实现接口`FactoryBean`来自定义Bean的创建逻辑。Spring容器对待实现了接口`FactoryBean`的类是特殊处理的。当你需要向容器请求一个真实的`FactoryBean`实例，而不是它生产的bean，例如调用`ApplicationContext`的`getBean()`方法时，在`bean`的`id`之前要有连字符（&）。对于一个给定 `id`为`myBean`的`FactoryBean`，调用容器的`getBean("myBean")`方法返回的`FactoryBean`产品。










