---
layout: post
comments: true
title: springboot 之 ConfigurationProperties
date: 2018-11-16 22:49:59
tags:
- spring
- springboot
- 译 
categories:
- springboot
---

### 前言

`ConfigurationProperties` 是SpringBoot引入的一个和外部配置文件相关的注解类。它可以帮助我们更好的使用外置的配置文件属性。

### 源码解析

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ConfigurationProperties {
    @AliasFor("prefix")
    String value() default "";
    @AliasFor("value")
    String prefix() default "";
    boolean ignoreInvalidFields() default false;
    boolean ignoreNestedProperties() default false;
    boolean ignoreUnknownFields() default true;
    @Deprecated
    boolean exceptionIfInvalid() default true;
}
```


<!-- more -->

#### prefix & value

prefix 属性可以指定配置文件中配置项的前缀，如此，相同前缀的配置项就可以统一解析。

#### ignoreInvalidFields

是否忽略不可用的字段，默认为`false`， 当配置项不能被正确转化为Java类的字段值时，会抛出异常。

#### ignoreNestedProperties 

是否忽略嵌套属性，默认为`false`， 

#### ignoreUnknownFields

是否忽略Java类不存在的字段，默认值为`true`。

#### exceptionIfInvalid

如果Java类加了注解`@Validated`，并且校验失败了，是否抛出异常。默认`true`


### 属性注入到Java类

```java
@Configuration
@ConfigurationProperties
@PropertySource("classpath:configprops.properties")
public class ConfigProperties {
 
    public static class Credentials {
        @Length(max = 4, min = 1)
        private String authMethod;
        private String username;
        private String password;
 
       // standard getters and setters
    }
    @NotBlank
    private String host;
    @Min(1025)
    @Max(65536)
    private int port;
    @Pattern(regexp = "^[a-z0-9._%+-]+@[a-z0-9.-]+\\.[a-z]{2,6}$")
    private String from;
    private Credentials credentials;
    private List<String> defaultRecipients;
    private Map<String, String> additionalHeaders;
  
    // standard getters and setters
}
```

默认属性配置从`application.properties`文件中获取，也可以通过`@PropertySource`指定。

`@Configuration`注解不可缺少。

资源文件内容如下：

```java
#Simple properties
mail.host=mailer@mail.com
mail.port=9000
mail.from=mailer@mail.com
 
#List properties
mail.defaultRecipients[0]=admin@mail.com
mail.defaultRecipients[1]=owner@mail.com
 
#Map Properties
mail.additionalHeaders.redelivery=true
mail.additionalHeaders.secure=true
 
#Object properties
mail.credentials.username=john
mail.credentials.password=password
mail.credentials.authMethod=SHA1
```

#### 内嵌类属性

`mail.credentials.username`可以注入到内嵌属性`credentials`中

#### 列表字段

`mail.defaultRecipients[0]` 可以注入到类的列表(数组页可以)属性中。

#### map字段

`mail.additionalHeaders.redelivery` 格式的配置项也可以注入到`Map`类型的属性中。


### 通过@ConfigurationProperties + @Bean注解在配置类的bean定义方法上

```java
@Bean
@ConfigurationProperties(prefix = "mail")
public ConfigProperties mailConfig() {
    return new ConfigProperties();
}
```

### @ConfigurationProperties + @EnableConfigurationProperties

```java
@SpringBootApplication
@EnableConfigurationProperties({ConfigProperties.class})
public class Application {
    public static void main(String[] args) throws Exception {
        SpringApplication.run(Application.class, args);
    }
}
```

### 属性校验

可以给属性类上加入`javax.validation.constraints.*`中的注解，来对配置项进行校验。配合`exceptionIfInvalid`可以更早的发现问题。

