---
layout: post
comments: true
title: RestTemplate超时引发的血案
date: 2018-11-21 23:01:05
tags:
- RestTemplate
- springboot
categories:
- springboot
---

### 前言

最近线上出了一次故障，收银台系统所有服务全部假死。订单量瞬时下降，造成很大损失。

故障总结，导致问题的原因有两方面：

1. 数据库慢查询
2. RestTemplate超时时间设置不生效。
3. spring-web不同版本设置RestTemplate方式不完全一样。

<!-- more -->

### 默认超时设置

默认情况下是没有超时设置的，此时超时依赖两方面：

1. 依赖TCP连接本身的超时时间（tcp空闲连接，超过一定时间，连接会被关闭）。
2. 请求所经过的网络节点的超时时间。e.g. 中间经过nginx, nginx默认读取后端服务的超时时间是60s，所以超时时间在60s左右（日志显示稍微大一点，不会大很多）。

#### 代码分析

```java 例子
long start = System.currentTimeMillis();
try {
    RestTemplate restTemplate = new RestTemplate();
    Map responseObject = restTemplate.getForObject(url, Map.class);
    System.out.println(responseObject);
} catch (Exception e) {
    Assert.assertNotNull(e);
    System.out.println("timeout = " + (System.currentTimeMillis() - start));
}
```

原因：

`RestTemplate` 继承自 `HttpAccessor`, 默认使用的`ClientHttpRequestFactory`是`SimpleClientHttpRequestFactory`

```java
public abstract class HttpAccessor {

	/**
	 * Logger available to subclasses.
	 */
	protected final Log logger = LogFactory.getLog(getClass());

	private ClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();
}

public class SimpleClientHttpRequestFactory implements ClientHttpRequestFactory {

	private static final int DEFAULT_CHUNK_SIZE = 4096;


	private Proxy proxy;

	private boolean bufferRequestBody = true;

	private int chunkSize = DEFAULT_CHUNK_SIZE;

  // 连接和读取超时都是 -1, 也就是没有超时设置。
	private int connectTimeout = -1;
	private int readTimeout = -1;
}
```

那么我们使用`RestTemplate`该如何设置超时时间呢？ 

### RestTemplate超时设置

由上面的代码我们了解到，超时设置其实应该通过内部的`ClientHttpRequestFactory`来设置的。

所以就可以通过给`RestTemplate`设置一个我们自己创建的，设置了超时时间的`ClientHttpRequestFactory`来实现。

```java
SimpleClientHttpRequestFactory clientHttpRequestFactory = new SimpleClientHttpRequestFactory();
clientHttpRequestFactory.setConnectTimeout(1000);
clientHttpRequestFactory.setReadTimeout(50);
RestTemplate restTemplate = new RestTemplate();
restTemplate.setRequestFactory(clientHttpRequestFactory);
```

或者

```java
HttpComponentsClientHttpRequestFactory clientHttpRequestFactory = new HttpComponentsClientHttpRequestFactory();
clientHttpRequestFactory.setConnectTimeout(1000);
clientHttpRequestFactory.setReadTimeout(50);
RestTemplate restTemplate = new RestTemplate();
restTemplate.setRequestFactory(clientHttpRequestFactory);
```

但是要注意的是: `HttpComponentsClientHttpRequestFactory` 底层使用了apache的`HttpClient`，超时时间的设置其实是针对它进行设置的。

```java HttpComponentsClientHttpRequestFactory

private static final int DEFAULT_MAX_TOTAL_CONNECTIONS = 100;

private static final int DEFAULT_MAX_CONNECTIONS_PER_ROUTE = 5;
//默认读取超时 60s
private static final int DEFAULT_READ_TIMEOUT_MILLISECONDS = (60 * 1000);

private HttpClient httpClient;

/**
 * Set the connection timeout for the underlying HttpClient.
 * A timeout value of 0 specifies an infinite timeout.
 * @param timeout the timeout value in milliseconds
 */
public void setConnectTimeout(int timeout) {
	Assert.isTrue(timeout >= 0, "Timeout must be a non-negative value");
	getHttpClient().getParams().setIntParameter(CoreConnectionPNames.CONNECTION_TIMEOUT, timeout);
}
```

到此，如果就通过上面提到的方式设置超时时间，那么我们的应用就不用有超时问题，也不会发生故障了。

但问题就发生在，公司内部使用的组件，不是通过`HttpComponentsClientHttpRequestFactory`设置超时时间，而是通过设置`HttpComponentsClientHttpRequestFactory`内部的`HttpClient`设置的超时时间，并且设置了`HttpClient` 使用的 `HttpClientConnectionManager`，从而导致了问题的发生。

### 问题代码&测试

```java
@Test
public void testRestTemplateWithRequestFactoryWithoutTimeOut() {
    long start = System.currentTimeMillis();
    try {
        HttpComponentsClientHttpRequestFactory requestFactory = new HttpComponentsClientHttpRequestFactory();

        //2.设置超时时间， 设置/不设置ConnectionManager
        HttpClient httpClient = HttpClientBuilder.create()
                .setDefaultRequestConfig(getRequestConfig())
                .setDefaultSocketConfig(getSocketConfig())
                .setConnectionManager(new PoolingHttpClientConnectionManager(3, TimeUnit.MINUTES))
                .build();

        requestFactory.setHttpClient(httpClient);
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.setRequestFactory(requestFactory);

        Map responseObject = restTemplate.getForObject(QUERY_USER_RENEW_URL, Map.class);
        System.out.println(responseObject);
    } catch (Exception e) {
        Assert.assertNotNull(e);
        System.out.println("timeout = " + (System.currentTimeMillis() - start));
    }
}
```

### 结论

#### spring-web 版本 3.2.0

1. 默认超时 60s, 因为nginx默认的proxy_read_timeout 是60s
2. 设置了 HttpClient的超时时间， 不设置 ConnectionManager 超时生效
3. 设置了 HttpClient的超时时间， 设置 ConnectionManager 超时生效
 
#### spring-web 版本 4.0.9.RELEASE

1. 默认超时 60s, 因为nginx默认的proxy_read_timeout 是60s
2. 设置了 HttpClient的超时时间， 不设置 ConnectionManager 超时生效
3. 设置了 HttpClient的超时时间， 设置 ConnectionManager 超时不生效 （qiyue-store 就是这样问题）
 
#### spring-web 版本 4.3.0.RELEASE

1. 默认超时 60s, 因为nginx默认的proxy_read_timeout 是60s
2. 设置了 HttpClient的超时时间， 不设置 ConnectionManager 超时生效
3. 设置了 HttpClient的超时时间， 设置 ConnectionManager 超时生效
 
 
#### spring-web 版本 4.3.11.RELEASE

1. 默认超时 60s, 因为nginx默认的 proxy_read_timeout 是60s
2. 设置了 HttpClient的超时时间， 不设置 ConnectionManager 超时生效
3. 设置了 HttpClient的超时时间， 设置 ConnectionManager 超时生效


其实问题就在与不同的版本中`HttpComponentsClientHttpRequestFactory.createRequest`方法的实现逻辑不同。如何不同，自己查看。😁

### 总结

1. 超时设置至关重要。外部依赖接口调用可以通过Hystrix进行包装。
2. 任何参数的设置都需要验证是否可以正常工作，可以加入到测试环节中，方便在不同的依赖版本中进行验证。


