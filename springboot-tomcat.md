---
layout: post
comments: true
title: springboot发布到tomcat
date: 2017-03-18 22:59:37
tags:
- springboot
categories:
- spring
---

### 前言

最近在学习springboot相关的东西，其中有一个点就是项目开发完成后的部署。我们都知道springboot项目可以打包为只执行jar的方式和war包的方式。可执行jar使用的是内嵌的servlet容器，war包的方式可以部署到外部的容器中。通常我们都使用外部容器的方式来部署应用。这篇文章就将我学习到的如何将springboot应用部署到tomcat中的方法记录下来。

<!-- more -->

### tomcat版本选择

springboot内嵌的tomcat版本是tomcat7或更高的版本（因为springboot依赖servlet3.0规范），所以我们外部的tomcat最低版本应该是tomcat7。如果必须使用tomcat6，则可以参考这篇文章[https://github.com/dsyer/spring-boot-legacy](https://github.com/dsyer/spring-boot-legacy)。

### 项目配置

第一步：修改项目打包方式

```xml
<packaging>war</packaging>
```

第二步：修改pom.xml配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>
</dependency>
```    

第三步：修改启动类

```java
@SpringBootApplication
public class Application extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(
            SpringApplicationBuilder builder) {
        return application.sources(Application.class);
    }

    public static void main(String[] args) {
        SpringApplication.run(ReadingListApplication.class, args);
    }
}
```
    
到此就可以完成将项目发布到外部tomcat中所需的改造操作。

### 原理分析

#### javax.servlet.ServletContainerInitializer

该类是servlet3.0规范中新增的类，该类的说明文档如下：

```
/**
 * ServletContainerInitializers (SCIs) are registered via an entry in the
 * file META-INF/services/javax.servlet.ServletContainerInitializer that must be
 * included in the JAR file that contains the SCI implementation.
 * <p>
 * SCI processing is performed regardless of the setting of metadata-complete.
 * SCI processing can be controlled per JAR file via fragment ordering. If an
 * absolute ordering is defined, the only those JARs included in the ordering
 * will be processed for SCIs. To disable SCI processing completely, an empty
 * absolute ordering may be defined.
 * <p>
 * SCIs register an interest in annotations (class, method or field) and/or
 * types via the {@link javax.servlet.annotation.HandlesTypes} annotation which
 * is added to the class.
 *
 * @since Servlet 3.0
 */
```

其实就是servlet3.0规范定义了支持该规范的容器在启动加载项目时，需要通过ServiceLoader加载实现了接口`ServletContainerInitializer`的类，实例化并调用实例的`void onStartup(Set<Class<?>> c, ServletContext ctx) throws ServletException`方法。

需要注意的是：该方法的第一个参数是一个Set, 它的值的类型是由实现了接口`ServletContainerInitializer`的类上的注解`javax.servlet.annotation.HandlesTypes`指定的类型。

```java
/**
 * This annotation is used to declare an array of application classes which are passed to a {@link javax.servlet.ServletContainerInitializer}.
 *
 * @since Servlet 3.0
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface HandlesTypes {
    /**
     * @return array of classes
     */
    Class<?>[] value();
}
```

#### spring的支持

spring从3.1版本开始中提供了实现上述接口的类`SpringServletContainerInitializer`

```java
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {
    //....
}
```

Servlet容器在启动时就会调用`SpringServletContainerInitializer`的`onStartup`方法，并将实现了接口`WebApplicationInitializer`的所有类收集起来作为参数传递给该方法。

```java
@Override
public void onStartup(Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
		throws ServletException {

	List<WebApplicationInitializer> initializers = new LinkedList<WebApplicationInitializer>();

	if (webAppInitializerClasses != null) {
		for (Class<?> waiClass : webAppInitializerClasses) {
			// Be defensive: Some servlet containers provide us with invalid classes,
			// no matter what @HandlesTypes says...
			if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
					WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
				try {
					initializers.add((WebApplicationInitializer) waiClass.newInstance());
				}
				catch (Throwable ex) {
					throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
				}
			}
		}
	}

	if (initializers.isEmpty()) {
		servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
		return;
	}

	servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
	AnnotationAwareOrderComparator.sort(initializers);
	for (WebApplicationInitializer initializer : initializers) {
		initializer.onStartup(servletContext);
	}
}
```

在上面的方法中会调用每个实现了接口`WebApplicationInitializer`的`onStartup`并将`ServletContext`对象作为参数传入到该方法中，每个实例根据自己的策略来对ServletContext对象进行操作，例如添加Filer，添加Servlet和Listener。
	
抽象类`SpringBootServletInitializer`实现了接口`WebApplicationInitializer`。

```java
public abstract class SpringBootServletInitializer implements WebApplicationInitializer {
    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
    	// Logger initialization is deferred in case a ordered
    	// LogServletContextInitializer is being used
    	this.logger = LogFactory.getLog(getClass());
    	WebApplicationContext rootAppContext = createRootApplicationContext(
    			servletContext);
    	if (rootAppContext != null) {
    		servletContext.addListener(new ContextLoaderListener(rootAppContext) {
    			@Override
    			public void contextInitialized(ServletContextEvent event) {
    				// no-op because the application context is already initialized
    			}
    		});
    	}
    	else {
    		this.logger.debug("No ContextLoaderListener registered, as "
    				+ "createRootApplicationContext() did not "
    				+ "return an application context");
    	}
    }
    
    protected WebApplicationContext createRootApplicationContext(
			ServletContext servletContext) {
    		SpringApplicationBuilder builder = createSpringApplicationBuilder();
    		builder.main(getClass());
    		ApplicationContext parent = getExistingRootWebApplicationContext(servletContext);
    		if (parent != null) {
    			this.logger.info("Root context already created (using as parent).");
    			servletContext.setAttribute(
    					WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, null);
    			builder.initializers(new ParentContextApplicationContextInitializer(parent));
    		}
    		builder.initializers(
    				new ServletContextApplicationContextInitializer(servletContext));
    		builder.listeners(new ServletContextApplicationListener(servletContext));
    		builder.contextClass(AnnotationConfigEmbeddedWebApplicationContext.class);
    		builder = configure(builder);
    		SpringApplication application = builder.build();
    		if (application.getSources().isEmpty() && AnnotationUtils
    				.findAnnotation(getClass(), Configuration.class) != null) {
    			application.getSources().add(getClass());
    		}
    		Assert.state(!application.getSources().isEmpty(),
    				"No SpringApplication sources have been defined. Either override the "
    						+ "configure method or add an @Configuration annotation");
    		// Ensure error pages are registered
    		if (this.registerErrorPageFilter) {
    			application.getSources().add(ErrorPageFilterConfiguration.class);
    		}
    		return run(application);
    	}
    	protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
		return builder;
	}
	protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
		return builder;
	}
}    
```

上面的代码很清晰，就是为了创建Spring容器。在创建前通过Builder模式创建了`SpringApplication`对象，并通过`SpringApplication`对象的`run`方法来创建和启动整个spring应用。

在上面的代码中定义并调用了一个protected方法`configure()`，这其实就是spring提供的一个扩展点，让我们有机会来控制容器的启动。

我们应用中的类`ReadingListApplication`继承了类`SpringBootServletInitializer`并重写了方法`configure`。

```java
@Override
protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
    return builder.sources(ReadingListApplication.class);
}
```

在该方法中我们调用了`SpringApplicationBuilder`类的`sources`方法。该方法的代码如下：

```java
/**
* Add more sources (configuration classes and components) to this application.
* @param sources the sources to add
* @return the current builder
*/
public SpringApplicationBuilder sources(Class<?>... sources){ this.sources.addAll(new LinkedHashSet<Object>(Arrays.asList(sources)));
		return this;
}
```	

功能就是提供Spring容器所需的配置类和组件类。

我们的类`ReadingListApplication`就是一个集配置和组件为一体的类。这是因为我们的类加了注解`@SpringBootApplication`。该注解的功能相当于下面的三个注解：

@Configuration
@EnableAutoConfiguration
@ComponentScan

### 后记

关于SpringApplication如何创建Spring容器，需要在深入代码进行了解。



 

   


