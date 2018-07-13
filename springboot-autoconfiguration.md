---
layout: post
comments: true
title: consul之acl配置
date: 2017-02-28 23:08:31
tags:
- consul
---

## 前言

SpringBoot的使用无疑变得越来越广泛．好处是快速上手，但也带来了额外的复杂性．　了解SpringBoot自动配置的黑魔法对我们熟练使用，排查问题
是非常重要的，否在除了问题，你就会变得一头雾水，无处下手．下面以一个非常典型的SpringBoot应用来分析它的自动配置黑魔法．

## 应用入口

```java
@EnableDiscoveryClient
@EnableCircuitBreaker
@SpringBootApplication
public class Application {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

<!-- more -->

## @SpringBootApplication

`@SpringBootApplication` 注解的作用是为了简化配置．它的作用相当于给一个类同时打了`Configuration`,`ComponentScan`, `EnableAutoConfiguration`
三个注解．

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
}
```

## @Configuration

注解`Configuration`的作用是指定一个类，使得该类中所有加了`@Bean`注解的方法都会被Spring容器进行处理，
来生成一个`BeanDefinition`，最终会生成一个业务Bean。

```java
public class AppConfig {

    @Bean
    public MyBean myBean() {
        return new MyBean();
    }
}
```

## @SpringBootConfiguration

`@SpringBootConfiguration` 注解指示了一个类提供了SpringBoot应用的配置信息

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {
}
```

## @EnableAutoConfiguration

```java
@SuppressWarnings("deprecation")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
}
```

`@EnableAutoConfiguration`注解的作用启用Spring的自动配置功能．　

自动配置功能通常是基于classpath拥有哪些类或你配置了哪些来进行的．

自动配置通常是在用户自定义的Bean已经完成注册后才进行的．

## @AutoConfigurationPackage

`@AutoConfigurationPackage`注解的作用是：　将打了该注解的类所在的包注册到`AutoConfigurationPackages`

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {

}
```

## AutoConfigurationPackages

`AutoConfigurationPackages`类的作用是保存自动配置的包，通过该类可以访问自动配置的包．


## @Import

`＠Import`注解的作用是: 导入打了`@Configuration`注解的类, 或实现了接口`ImportSelector`,`@ImportBeanDefinitionRegistrar`的类．


## @ImportResource

`@ImportResource` 注解的作用是: 用来导入用来配置`Bean`的xml文件。

## ImportSelector

`ImportSelector`的作用是: 选择哪些加了注解`@Configuration`的类应该被导入．

一个`ImportSelector`的实现类应该同时实现一下接口：

- EnvironmentAware
- BeanFactoryAware
- BeanClassLoaderAware
- ResourceLoaderAware

```java
public interface ImportSelector {

	/**
	 * Select and return the names of which class(es) should be imported based on
	 * the {@link AnnotationMetadata} of the importing @{@link Configuration} class.
	 */
	String[] selectImports(AnnotationMetadata importingClassMetadata);

}
```

## DeferredImportSelector

`DeferredImportSelector`是`ImportSelector`的一个变种．　特点是在所有标记了注解`@Configuration`的类被处理以后才执行．
这类选择器非常有用，尤其是在导入的配置是基于条件的．


## AutoConfigurationImportSelector

`AutoConfigurationImportSelector`实现了`DeferredImportSelector`，用来处理`EnableAutoConfiguration`.

```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return NO_IMPORTS;
    }
    try {
        AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
                .loadMetadata(this.beanClassLoader);
        AnnotationAttributes attributes = getAttributes(annotationMetadata);
        //　通过SpringFactoriesLoader加载spring.factories文件中EnableAutoConfiguration对应的＠Configuration类
        List<String> configurations = getCandidateConfigurations(annotationMetadata,
                attributes);
        configurations = removeDuplicates(configurations);
        configurations = sort(configurations, autoConfigurationMetadata);
        //　剔除不需要的
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        checkExcludedClasses(configurations, exclusions);
        configurations.removeAll(exclusions);
        configurations = filter(configurations, autoConfigurationMetadata);
        // 处罚自动配置导入事件
        fireAutoConfigurationImportEvents(configurations, exclusions);
        return configurations.toArray(new String[configurations.size()]);
    }
    catch (IOException ex) {
        throw new IllegalStateException(ex);
    }
}
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

## BeanFactoryPostProcessor

`BeanFactoryPostProcessor` 是Spring提供的SPI．允许我们在Bean实例化前注册新的BeanDefination或修改BeanDefinination.

## BeanDefinitionRegistryPostProcessor

`BeanDefinitionRegistryPostProcessor`继承自`BeanFactoryPostProcessor`.　可以在Bean初始化前，注册自定义的Bean.

## ConfigurationClassPostProcessor

`ConfigurationClassPostProcessor`继承自`BeanDefinitionRegistryPostProcessor`．　专门用来处理标记了注解`@Configuration`的类．

```java
/**
* Derive further bean definitions from the configuration classes in the registry.
* 解析标注了＠Configuration注解的类，生成更多的BeanDefination．
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
    //
    processConfigBeanDefinitions(registry);
}

/**
* Prepare the Configuration classes for servicing bean requests at runtime
* by replacing them with CGLIB-enhanced subclasses.
* 通过 CGLIB　生成配置类的增强子类，以此来满足运行时的bean请求服务
*/
@Override
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    int factoryId = System.identityHashCode(beanFactory);
    if (this.factoriesPostProcessed.contains(factoryId)) {
        throw new IllegalStateException(
                "postProcessBeanFactory already called on this post-processor against " + beanFactory);
    }
    this.factoriesPostProcessed.add(factoryId);
    if (!this.registriesPostProcessed.contains(factoryId)) {
        // BeanDefinitionRegistryPostProcessor hook apparently not supported...
        // Simply call processConfigurationClasses lazily at this point then.
        processConfigBeanDefinitions((BeanDefinitionRegistry) beanFactory);
    }
    //　获取是通过@Configuration生成的BeanDefinition.
    enhanceConfigurationClasses(beanFactory);
    beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
}

/**
* Build and validate a configuration model based on the registry of
* {@link Configuration} classes.
*/
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    List<BeanDefinitionHolder> configCandidates = new ArrayList<BeanDefinitionHolder>();
    String[] candidateNames = registry.getBeanDefinitionNames();

    for (String beanName : candidateNames) {
        BeanDefinition beanDef = registry.getBeanDefinition(beanName);
        if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
                ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
            if (logger.isDebugEnabled()) {
                logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
            }
        }
        else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
            configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
        }
    }
    // 省略一部分代码

    // 解析打了注解 @Configuration 的类
    ConfigurationClassParser parser = new ConfigurationClassParser(
            this.metadataReaderFactory, this.problemReporter, this.environment,
            this.resourceLoader, this.componentScanBeanNameGenerator, registry);

    Set<BeanDefinitionHolder> candidates = new LinkedHashSet<BeanDefinitionHolder>(configCandidates);
    Set<ConfigurationClass> alreadyParsed = new HashSet<ConfigurationClass>(configCandidates.size());

    do {
        parser.parse(candidates);
        parser.validate();

        Set<ConfigurationClass> configClasses = new LinkedHashSet<ConfigurationClass>(parser.getConfigurationClasses());
        configClasses.removeAll(alreadyParsed);

        // Read the model and create bean definitions based on its content
        if (this.reader == null) {
            this.reader = new ConfigurationClassBeanDefinitionReader(
                    registry, this.sourceExtractor, this.resourceLoader, this.environment,
                    this.importBeanNameGenerator, parser.getImportRegistry());
        }
        //解析出BeanDefinition并注册
        this.reader.loadBeanDefinitions(configClasses);
        alreadyParsed.addAll(configClasses);

        candidates.clear();
        if (registry.getBeanDefinitionCount() > candidateNames.length) {
            String[] newCandidateNames = registry.getBeanDefinitionNames();
            Set<String> oldCandidateNames = new HashSet<String>(Arrays.asList(candidateNames));
            Set<String> alreadyParsedClasses = new HashSet<String>();
            for (ConfigurationClass configurationClass : alreadyParsed) {
                alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
            }
            for (String candidateName : newCandidateNames) {
                if (!oldCandidateNames.contains(candidateName)) {
                    BeanDefinition bd = registry.getBeanDefinition(candidateName);
                    if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
                            !alreadyParsedClasses.contains(bd.getBeanClassName())) {
                        candidates.add(new BeanDefinitionHolder(bd, candidateName));
                    }
                }
            }
            candidateNames = newCandidateNames;
        }
    }
    while (!candidates.isEmpty());

    // Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
    if (sbr != null) {
        if (!sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
            sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
        }
    }

    if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
        ((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
    }
}
```    

## ConfigurationClassParser

`ConfigurationClassParser`的作用是用来解析打了注解`@Configuration`的类的．

`@SpringBootConfiguration`标记的类相当于由`@Configuration`标记类．可以由`ConfigurationClassParser`进行处理．

下面的方法会被`ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry`调用．这对我们理解SpringBoot的
自动配置功能非常重要．


```java
// 解析打了注解`@Configuration`的类的．
public void parse(Set<BeanDefinitionHolder> configCandidates) {
		this.deferredImportSelectors = new LinkedList<DeferredImportSelectorHolder>();

    for (BeanDefinitionHolder holder : configCandidates) {
        BeanDefinition bd = holder.getBeanDefinition();
        try {
            if (bd instanceof AnnotatedBeanDefinition) {
                // 执行真正的解析逻辑，其中就处理了@Configuration上的注解
                parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
            }
            else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
            }
            else {
                parse(bd.getBeanClassName(), holder.getBeanName());
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
        }
    }
    // 处理 DeferredImportSelector 在前面解析的过程中已经将解析到的 
    // DeferredImportSelector 放入　this.deferredImportSelectors 这样就实现了延迟导入的功能
    processDeferredImportSelectors();
}

protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
    // 条件判断，不满足直接返回，不会被注册
    if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
        return;
    }

    ConfigurationClass existingClass = this.configurationClasses.get(configClass);
    if (existingClass != null) {
        if (configClass.isImported()) {
            if (existingClass.isImported()) {
                existingClass.mergeImportedBy(configClass);
            }
            // Otherwise ignore new imported config class; existing non-imported class overrides it.
            return;
        }
        else {
            // Explicit bean definition found, probably replacing an import.
            // Let's remove the old one and go with the new one.
            this.configurationClasses.remove(configClass);
            for (Iterator<ConfigurationClass> it = this.knownSuperclasses.values().iterator(); it.hasNext();) {
                if (configClass.equals(it.next())) {
                    it.remove();
                }
            }
        }
    }

    // 递归处理配置类和其子类
    SourceClass sourceClass = asSourceClass(configClass);
    do {
        sourceClass = doProcessConfigurationClass(configClass, sourceClass);
    }
    while (sourceClass != null);

    this.configurationClasses.put(configClass, configClass);
}

/**
* Apply processing and build a complete {@link ConfigurationClass} by reading the
* annotations, members and methods from the source class. This method can be called
* multiple times as relevant sources are discovered.
* @param configClass the configuration class being build
* @param sourceClass a source class
* @return the superclass, or {@code null} if none found or previously processed
*/
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
        throws IOException {

    // Recursively process any member (nested) classes first
    processMemberClasses(configClass, sourceClass);

    // Process any @PropertySource annotations
    for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), PropertySources.class,
            org.springframework.context.annotation.PropertySource.class)) {
        if (this.environment instanceof ConfigurableEnvironment) {
            processPropertySource(propertySource);
        }
        else {
            logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                    "]. Reason: Environment must implement ConfigurableEnvironment");
        }
    }

    // 处理 @ComponentScan 注解
    Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
    if (!componentScans.isEmpty() &&
            !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
        for (AnnotationAttributes componentScan : componentScans) {
            // The config class is annotated with @ComponentScan -> perform the scan immediately
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                    this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
            // Check the set of scanned definitions for any further config classes and parse recursively if needed
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                if (bdCand == null) {
                    bdCand = holder.getBeanDefinition();
                }
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                    parse(bdCand.getBeanClassName(), holder.getBeanName());
                }
            }
        }
    }

    // 处理 @Import 注解
    processImports(configClass, sourceClass, getImports(sourceClass), true);

    // 处理 @ImportResource 注解
    if (sourceClass.getMetadata().isAnnotated(ImportResource.class.getName())) {
        AnnotationAttributes importResource =
                AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
        String[] resources = importResource.getStringArray("locations");
        Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
        for (String resource : resources) {
            String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
            configClass.addImportedResource(resolvedResource, readerClass);
        }
    }

    // 处理 打了注解@Bean的方法
    Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
    for (MethodMetadata methodMetadata : beanMethods) {
        configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
    }

    // 处理接口的默认方法
    processInterfaces(configClass, sourceClass);

    // 处理子类
    if (sourceClass.getMetadata().hasSuperClass()) {
        String superclass = sourceClass.getMetadata().getSuperClassName();
        if (!superclass.startsWith("java") && !this.knownSuperclasses.containsKey(superclass)) {
            this.knownSuperclasses.put(superclass, configClass);
            // Superclass found, return its annotation metadata and recurse
            return sourceClass.getSuperClass();
        }
    }

    // No superclass -> processing is complete
    return null;
}

/**
* 收集所有　@Import　的配置类　
* Returns {@code @Import} class, considering all meta-annotations.
*/
private Set<SourceClass> getImports(SourceClass sourceClass) throws IOException {
    Set<SourceClass> imports = new LinkedHashSet<SourceClass>();
    Set<SourceClass> visited = new LinkedHashSet<SourceClass>();
    collectImports(sourceClass, imports, visited);
    return imports;
}

/**
* 递归收集所有　@Import注解
* Recursively collect all declared {@code @Import} values. Unlike most
* meta-annotations it is valid to have several {@code @Import}s declared with
* different values; the usual process of returning values from the first
* meta-annotation on a class is not sufficient.
* <p>For example, it is common for a {@code @Configuration} class to declare direct
* {@code @Import}s in addition to meta-imports originating from an {@code @Enable}
* annotation.
* @param sourceClass the class to search
* @param imports the imports collected so far
* @param visited used to track visited classes to prevent infinite recursion
* @throws IOException if there is any problem reading metadata from the named class
*/
private void collectImports(SourceClass sourceClass, Set<SourceClass> imports, Set<SourceClass> visited)
        throws IOException {

    if (visited.add(sourceClass)) {
        for (SourceClass annotation : sourceClass.getAnnotations()) {
            String annName = annotation.getMetadata().getClassName();
            if (!annName.startsWith("java") && !annName.equals(Import.class.getName())) {
                collectImports(annotation, imports, visited);
            }
        }
        imports.addAll(sourceClass.getAnnotationAttributes(Import.class.getName(), "value"));
    }
}

private void processDeferredImportSelectors() {
    List<DeferredImportSelectorHolder> deferredImports = this.deferredImportSelectors;
    this.deferredImportSelectors = null;
    Collections.sort(deferredImports, DEFERRED_IMPORT_COMPARATOR);

    // 在我们文章开头的代码中出现了３个注解:
    //@EnableDiscoveryClient
    //@EnableCircuitBreaker
    //@SpringBootApplication
    //这三个注解都是: DeferredImportSelector
    for (DeferredImportSelectorHolder deferredImport : deferredImports) {
        ConfigurationClass configClass = deferredImport.getConfigurationClass();
        try {
            //　判断哪个@Configuration类因该被导入
            String[] imports = deferredImport.getImportSelector().selectImports(configClass.getMetadata());
            //　处理由 DeferredImportSelector 选择导入的配置类
            processImports(configClass, asSourceClass(configClass), asSourceClasses(imports), false);
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to process import candidates for configuration class [" +
                    configClass.getMetadata().getClassName() + "]", ex);
        }
    }
}

private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
			Collection<SourceClass> importCandidates, boolean checkForCircularImports) {

    if (importCandidates.isEmpty()) {
        return;
    }

    if (checkForCircularImports && isChainedImportOnStack(configClass)) {
        this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
    }
    else {
        this.importStack.push(configClass);
        try {
            for (SourceClass candidate : importCandidates) {
                // 如果　@Import注解的Value属性指定的值的类型是 ImportSelector
                if (candidate.isAssignable(ImportSelector.class)) {
                    // Candidate class is an ImportSelector -> delegate to it to determine imports
                    Class<?> candidateClass = candidate.loadClass();
                    ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                    ParserStrategyUtils.invokeAwareMethods(
                            selector, this.environment, this.resourceLoader, this.registry);
                    //  如果 ImportSelector 的类型是  DeferredImportSelector
                    if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {
                        this.deferredImportSelectors.add(
                                new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));
                    }
                    // 普通的 ImportSelector, 递归处理
                    else {
                        String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                        Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
                        processImports(configClass, currentSourceClass, importSourceClasses, false);
                    }
                }
                // 如果　@Import注解的Value属性指定的值的类型是 ImportBeanDefinitionRegistrar
                else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                    // Candidate class is an ImportBeanDefinitionRegistrar ->
                    // delegate to it to register additional bean definitions
                    Class<?> candidateClass = candidate.loadClass();
                    ImportBeanDefinitionRegistrar registrar =
                            BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
                    ParserStrategyUtils.invokeAwareMethods(
                            registrar, this.environment, this.resourceLoader, this.registry);
                    configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                }
                else {
                    // 处理打了注解：@Configuration　的类
                    this.importStack.registerImport(
                            currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                    processConfigurationClass(candidate.asConfigClass(configClass));
                }
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to process import candidates for configuration class [" +
                    configClass.getMetadata().getClassName() + "]", ex);
        }
        finally {
            this.importStack.pop();
        }
    }
}
```

## SpringFactoriesLoader

`SpringFactoriesLoader`是Spring框架内部使用的通用工厂加载机制。从`META-INF/spring.factories`文件中加载工厂类的实现类

```java
/**
* 从`META-INF/spring.factories`加载实现工厂类的完整类名
* @param factoryClass 代表工厂的接口或抽象类
* @param classLoader 加载资源的类加载器
*/
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
}
/**
*　加载并实例化
*/
public static <T> List<T> loadFactories(Class<T> factoryClass, ClassLoader classLoader) {
｝
```

## Condition

`Condition`定义了一个组件被注册所必须满足的条件. 并且该条件是在BeanDefinination注册前被验证是否匹配的．

```java
public interface Condition {
	/**
	 * Determine if the condition matches.
	 * @param context the condition context
	 * @param metadata metadata of the {@link org.springframework.core.type.AnnotationMetadata class}
	 * or {@link org.springframework.core.type.MethodMetadata method} being checked.
	 * @return {@code true} if the condition matches and the component can be registered
	 * or {@code false} to veto registration.
	 */
	boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```

## ConfigurationCondition

`ConfigurationCondition` 是对`Condition`的增强，在和`@Configuration`配合使用时，提供了更细粒度的控制．

```java
public interface ConfigurationCondition extends Condition {
	/**
     * 返回该条件被使用时所属的配置阶段
	 */
	ConfigurationPhase getConfigurationPhase();

    //　定义了配置被使用的阶段
	enum ConfigurationPhase {
		// 解析
		PARSE_CONFIGURATION,
        // 注册
		REGISTER_BEAN
	}
}
```

## Conditional

`Conditional`　用来指定一个组件被注册是所必须满足的条件集合.

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {
	/**
	 * 组件在被注册时所满足的条件集合
	 */
	Class<? extends Condition>[] value();
}
```

## AllNestedConditions

`AllNestedConditions`是一种特殊的条件．打了该注解的组件想要被注册，则必须满足该条件类所有内部类定义的条件才可以．

举个例子：

```java
@Configuration
@EnableConfigurationProperties
@ConditionalOnConsulEnabled
public class ConsulAutoConfiguration {
}

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@Conditional(ConditionalOnConsulEnabled.OnConsulEnabledCondition.class)
public @interface ConditionalOnConsulEnabled {

	class OnConsulEnabledCondition extends AllNestedConditions {

		public OnConsulEnabledCondition() {
			super(ConfigurationPhase.REGISTER_BEAN);
		}

		@ConditionalOnProperty(value = "spring.cloud.consul.enabled", matchIfMissing = true)
		static class FoundProperty {}

		@ConditionalOnClass(ConsulClient.class)
		static class FoundClass {}
	}
}
```

`ConsulAutoConfiguration`该配置类想要被使用，则必须满足`OnConsulEnabledCondition`指定的条件.

`OnConsulEnabledCondition`是一个嵌套条件．　它内部定义了２个条件.

1. 配置属性`spring.cloud.consul.enabled`的值是true．
2. classpath路经上能找到类`ConsulClient.class`

