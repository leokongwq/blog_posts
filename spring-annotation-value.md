---
layout: post
comments: true
title: spring中注解@Value解密
date: 2016-12-28 17:58:01
tags:
- spring
categories:
- spring
---

### 背景

通常在使用Spring的项目中，总有将项目配置文件中的值作为Bean的属性进行注入的需要。此时我们就可以使用Spring3.0开始提供的一个注解`@Value`来实现。但这背后的实现究竟是如何实现的？这个问题值得我们了解一下。

<!-- more -->

### @Value实现解析

```java
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Value {

	/**
	 * The actual value expression: e.g. "#{systemProperties.myProp}".
	 */
	String value();

}
```

上面就是从Spring3.0开始提供的一个非常有用的注解。它的功能就是给字段，方法参数等设置默认值。通常是格式是：

```java
#{systemProperties.myProp}
```
    
我们也都知道注解只能提供元数据信息，并不能对代码产生任何实际的影响。在Spring中对该注解进行处理的Bean的类型是`BeanPostProcessor`。也就是说在`BeanPostProcessor`和`BeanFactoryPostProcessor`中是不能使用该注解的，这个应该很好理解。具体处理该注解的Bean是`AutowiredAnnotationBeanPostProcessor`
    
{% asset_img spring-annotation-value.png %}    
    
从上面的继承图我们可以明显看出`AutowiredAnnotationBeanPostProcessor`实现了接口`BeanPostProcessor`
    
`BeanPostProcessor`类有两个方法，分别提供了在Bean初始化前和初始化后对新创建的Bean进行处理的钩子。

```java
//在Bean初始化前被调用（例如在InitializingBean的afterPropertiesSet前）
Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
//在Bean初始化后被调用
Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
```

那具体是在初始化前还是初始化后进行调用的呢？我们继续看...

**第一个方法**
```java
@SuppressWarnings("unchecked")
public AutowiredAnnotationBeanPostProcessor() {
	this.autowiredAnnotationTypes.add(Autowired.class);
	this.autowiredAnnotationTypes.add(Value.class);
	try {
		this.autowiredAnnotationTypes.add((Class<? extends Annotation>)
				ClassUtils.forName("javax.inject.Inject", AutowiredAnnotationBeanPostProcessor.class.getClassLoader()));
		logger.info("JSR-330 'javax.inject.Inject' annotation found and supported for autowiring");
	}
	catch (ClassNotFoundException ex) {
		// JSR-330 API not available - simply skip.
	}
}
```

在上述的构造方法`AutowiredAnnotationBeanPostProcessor`中我们可以看到两行关键的代码

```java
this.autowiredAnnotationTypes.add(Autowired.class);
this.autowiredAnnotationTypes.add(Value.class);
```

这两行代码是将该类需要处理的注解类型放入到一个Set中，在后面后使用到。

**方法二**

```java
@Override
public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName)throws BeansException {
		return pvs;
}
```
该方法的作用是在Spring容器将参数`pvs`应用到指定Bean的属性前被调用的方法，在这个方法中我们可以对将要使用的数据值进行校验和转换。`AutowiredAnnotationBeanPostProcessor`覆盖了该方法。

```java
@Override
public PropertyValues postProcessPropertyValues(
        PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {

    InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
    try {
        metadata.inject(bean, beanName, pvs);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
    }
    return pvs;
}
```

从上面的代码逻辑可以简单看出，该方法首先根据给定Bean的class对象和声明的属性数组查找需要注入的元数据信息，然后进行了属性注入。

**方法三**

```java
private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, PropertyValues pvs) {
    // Fall back to class name as cache key, for backwards compatibility with custom callers.
    String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
    // Quick check on the concurrent map first, with minimal locking.
    InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
    if (InjectionMetadata.needsRefresh(metadata, clazz)) {
        synchronized (this.injectionMetadataCache) {
            metadata = this.injectionMetadataCache.get(cacheKey);
            if (InjectionMetadata.needsRefresh(metadata, clazz)) {
                if (metadata != null) {
                    metadata.clear(pvs);
                }
                try {
                    metadata = buildAutowiringMetadata(clazz);
                    this.injectionMetadataCache.put(cacheKey, metadata);
                }
                catch (NoClassDefFoundError err) {
                    throw new IllegalStateException("Failed to introspect bean class [" + clazz.getName() +
                            "] for autowiring metadata: could not find class that it depends on", err);
                }
            }
        }
    }
    return metadata;
}
```

可以看出上的方法先是从缓存中查找是由对应Bean所需要的注入元数据信息，如果没有就构建

**方法四：**

```java
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
    LinkedList<InjectionMetadata.InjectedElement> elements = new LinkedList<InjectionMetadata.InjectedElement>();
    Class<?> targetClass = clazz;

    do {
        final LinkedList<InjectionMetadata.InjectedElement> currElements =
                new LinkedList<InjectionMetadata.InjectedElement>();

        ReflectionUtils.doWithLocalFields(targetClass, new ReflectionUtils.FieldCallback() {
            @Override
            public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
                AnnotationAttributes ann = findAutowiredAnnotation(field);
                if (ann != null) {
                    if (Modifier.isStatic(field.getModifiers())) {
                        if (logger.isWarnEnabled()) {
                            logger.warn("Autowired annotation is not supported on static fields: " + field);
                        }
                        return;
                    }
                    boolean required = determineRequiredStatus(ann);
                    currElements.add(new AutowiredFieldElement(field, required));
                }
            }
        });

        ReflectionUtils.doWithLocalMethods(targetClass, new ReflectionUtils.MethodCallback() {
            @Override
            public void doWith(Method method) throws IllegalArgumentException, IllegalAccessException {
                Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
                if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
                    return;
                }
                AnnotationAttributes ann = findAutowiredAnnotation(bridgedMethod);
                if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
                    if (Modifier.isStatic(method.getModifiers())) {
                        if (logger.isWarnEnabled()) {
                            logger.warn("Autowired annotation is not supported on static methods: " + method);
                        }
                        return;
                    }
                    if (method.getParameterTypes().length == 0) {
                        if (logger.isWarnEnabled()) {
                            logger.warn("Autowired annotation should be used on methods with parameters: " + method);
                        }
                    }
                    boolean required = determineRequiredStatus(ann);
                    PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
                    currElements.add(new AutowiredMethodElement(method, required, pd));
                }
            }
        });

        elements.addAll(0, currElements);
        targetClass = targetClass.getSuperclass();
    }
    while (targetClass != null && targetClass != Object.class);

    return new InjectionMetadata(clazz, elements);
}
```

该方法会从类的属性和所有方法上查找是否有构造方法中指定注解的属性和方法然后构造需要注入数据对象。

**方法五**

```java
public void inject(Object target, String beanName, PropertyValues pvs) throws Throwable {
    Collection<InjectedElement> elementsToIterate =
            (this.checkedElements != null ? this.checkedElements : this.injectedElements);
    if (!elementsToIterate.isEmpty()) {
        boolean debug = logger.isDebugEnabled();
        for (InjectedElement element : elementsToIterate) {
            if (debug) {
                logger.debug("Processing injected element of bean '" + beanName + "': " + element);
            }
            element.inject(target, beanName, pvs);
        }
    }
}
```

该方法就实现了对需要注入值的待处理属性进行了赋值的逻辑。

此时需要注意的是InjectedElement的类型是AutowiredAnnotationBeanPostProcessor的内部类`AutowiredMethodElement`或`AutowiredMethodElement`

```java
private class AutowiredMethodElement extends InjectionMetadata.InjectedElement

private class AutowiredMethodElement extends InjectionMetadata.InjectedElement {
}
```

**方法六**

```java 字段值注入
protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {
    Field field = (Field) this.member;
    try {
        Object value;
        if (this.cached) {
            value = resolvedCachedArgument(beanName, this.cachedFieldValue);
        }
        else {
            DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
            desc.setContainingClass(bean.getClass());
            Set<String> autowiredBeanNames = new LinkedHashSet<String>(1);
            TypeConverter typeConverter = beanFactory.getTypeConverter();
            value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
            synchronized (this) {
                if (!this.cached) {
                    if (value != null || this.required) {
                        this.cachedFieldValue = desc;
                        registerDependentBeans(beanName, autowiredBeanNames);
                        if (autowiredBeanNames.size() == 1) {
                            String autowiredBeanName = autowiredBeanNames.iterator().next();
                            if (beanFactory.containsBean(autowiredBeanName)) {
                                if (beanFactory.isTypeMatch(autowiredBeanName, field.getType())) {
                                    this.cachedFieldValue = new RuntimeBeanReference(autowiredBeanName);
                                }
                            }
                        }
                    }
                    else {
                        this.cachedFieldValue = null;
                    }
                    this.cached = true;
                }
            }
        }
        if (value != null) {
            ReflectionUtils.makeAccessible(field);
            field.set(bean, value);
        }
    }
    catch (Throwable ex) {
        throw new BeanCreationException("Could not autowire field: " + field, ex);
    }
}
```

**方法七** 方法注入
```java
protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {
    if (checkPropertySkipping(pvs)) {
        return;
    }
    Method method = (Method) this.member;
    try {
        Object[] arguments;
        if (this.cached) {
            // Shortcut for avoiding synchronization...
            arguments = resolveCachedArguments(beanName);
        }
        else {
            Class<?>[] paramTypes = method.getParameterTypes();
            arguments = new Object[paramTypes.length];
            DependencyDescriptor[] descriptors = new DependencyDescriptor[paramTypes.length];
            Set<String> autowiredBeanNames = new LinkedHashSet<String>(paramTypes.length);
            TypeConverter typeConverter = beanFactory.getTypeConverter();
            for (int i = 0; i < arguments.length; i++) {
                MethodParameter methodParam = new MethodParameter(method, i);
                DependencyDescriptor desc = new DependencyDescriptor(methodParam, this.required);
                desc.setContainingClass(bean.getClass());
                descriptors[i] = desc;
                Object arg = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
                if (arg == null && !this.required) {
                    arguments = null;
                    break;
                }
                arguments[i] = arg;
            }
            synchronized (this) {
                if (!this.cached) {
                    if (arguments != null) {
                        this.cachedMethodArguments = new Object[arguments.length];
                        for (int i = 0; i < arguments.length; i++) {
                            this.cachedMethodArguments[i] = descriptors[i];
                        }
                        registerDependentBeans(beanName, autowiredBeanNames);
                        if (autowiredBeanNames.size() == paramTypes.length) {
                            Iterator<String> it = autowiredBeanNames.iterator();
                            for (int i = 0; i < paramTypes.length; i++) {
                                String autowiredBeanName = it.next();
                                if (beanFactory.containsBean(autowiredBeanName)) {
                                    if (beanFactory.isTypeMatch(autowiredBeanName, paramTypes[i])) {
                                        this.cachedMethodArguments[i] = new RuntimeBeanReference(autowiredBeanName);
                                    }
                                }
                            }
                        }
                    }
                    else {
                        this.cachedMethodArguments = null;
                    }
                    this.cached = true;
                }
            }
        }
        if (arguments != null) {
            ReflectionUtils.makeAccessible(method);
            method.invoke(bean, arguments);
        }
    }
    catch (InvocationTargetException ex) {
        throw ex.getTargetException();
    }
    catch (Throwable ex) {
        throw new BeanCreationException("Could not autowire method: " + method, ex);
    }
}
```

### 相关知识点
BeanExpressionContext
SpelExpression 表示我们上面提到的@Value注解值："#{config['selection.sellerUserId']}"

Spring 会根据@Value注解的值解析里面的spel表达式生产一个SpelExpression对象。然后调用BeanExpressionContext的getObject方法从容器中获取一个name为config的bean,再从该Bean中获取键为`selection.sellerUserId`对应的值






    


    

