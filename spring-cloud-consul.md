---
layout: post
comments: true
title: springcloud服务注册之consul
date: 2018-07-08 13:28:16
tags:
- spring
- springcloud
categories:
- web
---

### 前言

使用springcloud构建微服务，除了可以使用eureka来实现服务注册和发现外，springcloud对consul也提供了良好的支持。本文就简单介绍下使用springcloud和consul实现服务注册和发现。

### consul安装

consul的安装非常简单，只需要在官网[下载](https://www.consul.io/downloads.html)对应平台的安装文件即可。

具体安装步骤参考[install](https://www.consul.io/intro/getting-started/install.html)

<!-- more -->

### 服务注册

#### maven 配置

```xml
<dependencyManagement>
   <dependencies>
       <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-dependencies</artifactId>
           <version>Camden.SR6</version>
           <type>pom</type>
           <scope>import</scope>
       </dependency>
   </dependencies>
</dependencyManagement>

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```

#### 相关注解配置

```java

@EnableDiscoveryClient
@SpringBootApplication
public class Application {
	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}
```

#### application.properties配置

```java
#默认就是true
spring.cloud.consul.enabled=true
spring.cloud.consul.host=127.0.0.1
spring.cloud.consul.port=8500
#默认就是true
spring.cloud.consul.discovery.enabled=true
#服务名
spring.cloud.consul.discovery.serviceName=${spring.application.name}
#服务注册实例名
spring.cloud.consul.discovery.instanceId=${spring.application.name}-${spring.cloud.client.ipAddress}-${server.port}
#服务所在的主机名
spring.cloud.consul.discovery.hostname=${spring.cloud.client.ipAddress}
#服务所在的端口
spring.cloud.consul.discovery.port=${server.port}
#服务的健康检查url
spring.cloud.consul.discovery.healthCheckUrl=http://localhost:${server.port}/
spring.cloud.consul.discovery.healthCheckInterval=10s
spring.cloud.consul.discovery.healthCheckTimeout=10s
spring.cloud.consul.discovery.healthCheckPath=/
spring.cloud.consul.discovery.tags=dev,dev1
```

> healthCheckUrl 是consul检查应用健康状态的地址，可以选择一个自定义地址， 也可以使用`actuator`提供的`/health`地址。需要注意的是：如果你使用`actuator`提供的`/health`地址，则需要确保返回的json注解中的`status`字段为`UP`。也就是说`actuator`实施健康检查的所有组件的健康状态都必须是`UP`状态，有一个不是，则整个返回界都就是`DOWN`状态。

### 服务调用

properties文件配置和服务注册端都是一致的。Spring的java Bean配置也基本相同。

#### maven 配置

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
```
        
#### java Bean 配置
        
```java
@EnableDiscoveryClient
@EnableCircuitBreaker
@SpringBootApplication
public class Application {
    
    //启动负载均衡功能
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

服务调用示例：

```java
@Service
public class BookService {

    @Resource
    private RestTemplate restTemplate;

    @HystrixCommand(fallbackMethod = "addServiceFallback")
    public String add(){
        //url格式为服务名 + 请求的URI
        return restTemplate.getForEntity("http://bookservice/add?a=10&b=20", String.class).getBody();
    }

    public String addServiceFallback() {
        return "error";
    }
}
```



