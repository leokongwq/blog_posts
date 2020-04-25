---
layout: post
comments: true
title: spring扩展点之PropertyPlaceholderConfigurer
date: 2016-12-28 16:06:00
tags:
- java
- spring
categories:
- spring
---

### 背景

在使用Spring框架的开发过程中肯定少不了项目的配置文件，通常是Java的Properties文件。并且会通过类`PropertyPlaceholderConfigurer`在运行时动态替换xml配置文件中的Bean配置项。

<!-- more -->

### 先看例子

```java
<bean id="propertiesConfig" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
   <property name="ignoreUnresolvablePlaceholders" value="true"/>
   <property name="locations">
       <list>
           <value>classpath*:config.properties</value>
       </list>
   </property>
</bean>
```

### 原理解析

`PropertyPlaceholderConfigurer`类是如何做到xml配置的bean的配置熟悉的运行时替换的呢？这个离不开Spring框架提供的许多扩展点。这个类依赖的扩展点就是`BeanFactoryPostProcessors`见下面的类继承图：

{% asset_img PropertyPlaceholderConfigurer.png %}

`BeanFactoryPostProcessors`类有一个方法，提供了在Bean创建前对Bean的Definition进行处理的钩子机制。

```java
//在Bean初始化前被调用（例如在InitializingBean的afterPropertiesSet前）
void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
```

从上面的继承图我们知道`PropertyPlaceholderConfigurer`继承自类`PropertyResourceConfigurer`, 而该类实现了接口`BeanFactoryPostProcessors`

```java
public abstract class PropertyResourceConfigurer extends PropertiesLoaderSupport
		implements BeanFactoryPostProcessor, PriorityOrdered {
@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		try {
			Properties mergedProps = mergeProperties();

			// Convert the merged properties, if necessary.
			convertProperties(mergedProps);

			// Let the subclass process the properties.
			processProperties(beanFactory, mergedProps);
		}
		catch (IOException ex) {
			throw new BeanInitializationException("Could not load properties", ex);
		}
	}
	protected abstract void processProperties(ConfigurableListableBeanFactory beanFactory, Properties props)
			throws BeansException;
}
```

我们的主角`PropertyPlaceholderConfigurer`实现了父类中的方法`processProperties`, 在该类中对xml配置文件中的展位符进行替换处理

```java
/**
 * Visit each bean definition in the given bean factory and attempt to replace ${...} property
 * placeholders with values from the given properties.
 */
@Override
protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess, Properties props)
        throws BeansException {

    StringValueResolver valueResolver = new PlaceholderResolvingStringValueResolver(props);
    doProcessProperties(beanFactoryToProcess, valueResolver);
}

protected void doProcessProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
		StringValueResolver valueResolver) {

	BeanDefinitionVisitor visitor = new BeanDefinitionVisitor(valueResolver);

	String[] beanNames = beanFactoryToProcess.getBeanDefinitionNames();
	for (String curName : beanNames) {
		// Check that we're not parsing our own bean definition,
		// to avoid failing on unresolvable placeholders in properties file locations.
		if (!(curName.equals(this.beanName) && beanFactoryToProcess.equals(this.beanFactory))) {
			BeanDefinition bd = beanFactoryToProcess.getBeanDefinition(curName);
			try {
				visitor.visitBeanDefinition(bd);
			}
			catch (Exception ex) {
				throw new BeanDefinitionStoreException(bd.getResourceDescription(), curName, ex.getMessage(), ex);
			}
		}
	}

	// New in Spring 2.5: resolve placeholders in alias target names and aliases as well.
	beanFactoryToProcess.resolveAliases(valueResolver);

	// New in Spring 3.0: resolve placeholders in embedded values such as annotation attributes.
	beanFactoryToProcess.addEmbeddedValueResolver(valueResolver);
}
```

这样就解开了Spring中xml配置文件中占位符的替换魔法功能。
	
	








