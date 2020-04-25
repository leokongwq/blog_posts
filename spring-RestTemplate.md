---
layout: post
comments: true
title: spring中RestTemplate简介
date: 2018-05-30 16:44:51
tags:
- spring
categories:
- web
---

### RestTemplate 是什么？

RestTemplate是Spring提供的一个访问Http服务的客户端类，非常类似JdbcTemplate, JmsTemplate。它是线程安全的（一旦创建完成）。从名称上来看，该类更多是针对RESTFUL风格API设计的。当然如果你想通过它调用普通的Http接口也是可以的。

### RestTemplate 的方法

RestTemplate提供的方法都是以Http协议中的6个动词开头的：

{% asset_img spring-template.png %}

这些方法的名称清楚地表明它们调用的是哪个HTTP方法，而名称的第二部分表示返回的内容。 例如，`getForObject（）`将执行GET，将HTTP响应转换为你选择的对象类型，并返回该对象。`postForLocation`将执行POST，将给定对象转换为HTTP请求，并返回可以找到新创建对象的响应`HTTP Location`标头。 如你所见，这些方法试图强制执行REST最佳实践。

<!-- more -->

### URI 模板

这些方法中的每一个都将URI作为第一个参数。 该URI可以是URI模板，可以使用变量将模板扩展为正常的URI。 模板变量可以以两种形式传递：`作为String可变参数数组`，或作为`Map <String，String>`。 字符串可变数组按顺序展开复制给模板变量，例如：

```java
String result = restTemplate.getForObject("http://example.com/hotels/{hotel}/bookings/{booking}", String.class, "42", "21");
```

上面的代码最终会请求：[http://example.com/hotels/42/bookings/21](http://example.com/hotels/42/bookings/21)。Map类型的模板数据，则会以模板变量的名称为key, 查询对应的值来做替换。例如：

```java
Map<String, String> vars = new HashMap<String, String>();
vars.put("hotel", "42");
vars.put("booking", "21");
String result = restTemplate.getForObject("http://example.com/hotels/{hotel}/bookings/{booking}", String.class, vars);
```

最终同样会发送请求：[http://example.com/hotels/42/rooms/42](http://example.com/hotels/42/rooms/42)。

### HttpMessageConverters

传递给方法`getForObject()`,`postForLocation()`,`put()`的数据对象，或是这些方法的返回数据对象都是通过`HttpMessageConverter`进行转换的。在发送请求时，将数据对象转为Http请求数据， 接收响应时，将Http响应数据转为对应的数据对象。具体参考下面的例子

#### 搜索图片

Flickr公开了各种API来操纵庞大的照片库。`flickr.photos.search`方法允许你通过使用`GET`方法请求地址：[http://www.flickr.com/services/rest?method=flickr.photos.search&api+key=xxx&tags=penguins](http://www.flickr.com/services/rest?method=flickr.photos.search&api+key=xxx&tags=penguins)来搜索照片，结果会以xml的形式展示:

```xml
<photos page="2" pages="89" perpage="10" total="881">
	<photo id="2636" owner="47058503995@N01" 
		secret="a123456" server="2" title="test_04"
		ispublic="1" isfriend="0" isfamily="0" />
	<photo id="2635" owner="47058503995@N01"
		secret="b123456" server="2" title="test_03"
		ispublic="0" isfriend="1" isfamily="1" />
	<photo id="2633" owner="47058503995@N01"
		secret="c123456" server="2" title="test_01"
		ispublic="1" isfriend="0" isfamily="0" />
	<photo id="2610" owner="12037949754@N01"
		secret="d123456" server="2" title="00_tall"
		ispublic="1" isfriend="0" isfamily="0" />
</photos>
```

Java代码：

```java
final String photoSearchUrl =
   "http://www.flickr.com/services/rest?method=flickr.photos.search&api+key={api-key}&tags={tag}&per_page=10";
Source photos = restTemplate.getForObject(photoSearchUrl, Source.class, apiKey, searchTerm);
```

### 代码实例

#### getXXX

GET请求的方法如下：

```get 
public <T> T getForObject(String url, Class<T> responseType, Object... urlVariables) throws RestClientException 

public <T> T getForObject(String url, Class<T> responseType, Map<String, ?> urlVariables) throws RestClientException

public <T> T getForObject(URI url, Class<T> responseType) throws RestClientException
```

```java
@RequestMapping("/hello")
public String helloGet() {
    ResponseEntity<String> responseEntity = restTemplate.getForEntity("http://abc.com", String.class);
    String body = responseEntity.getBody();
    HttpStatus statusCode = responseEntity.getStatusCode();
    int statusCodeValue = responseEntity.getStatusCodeValue();
    HttpHeaders headers = responseEntity.getHeaders();
    return body;
}
```

### postXXX

POST 请求的方法如下：

```java
public <T> T postForObject(String url, Object request, Class<T> responseType, Object... uriVariables) throws RestClientException

public <T> T postForObject(String url, Object request, Class<T> responseType, Map<String, ?> uriVariables) throws RestClientException

public <T> T postForObject(URI url, Object request, Class<T> responseType) throws RestClientException
```

例子：

```java
HttpHeaders headers = new HttpHeaders();
headers.add("X-Auth-Token", "e348bc22-5efa-4299-9142-529f07a18ac9");

MultiValueMap<String, String> postParameters = new LinkedMultiValueMap<String, String>();
postParameters.add("owner", "11");
postParameters.add("subdomain", "aoa");
postParameters.add("comment", "");

HttpEntity<MultiValueMap<String, String>> requestEntity  = new HttpEntity<MultiValueMap<String, String>>(postParameters, headers);

WebResult<Person> result = null;
try {
  result = restTemplate.postForObject("请求地址",  requestEntity, WebResult.class);
  logger.info(result);
} catch (RestClientException e) {
  logger.info("error", e);
}
```

### RestTemplate配置

```java
@Configuration
public class RestTemplateConfig {

    @Bean
    @ConditionalOnMissingBean({ RestOperations.class, RestTemplate.class })
    public RestTemplate restTemplate(ClientHttpRequestFactory factory) {
        RestTemplate restTemplate = new RestTemplate(factory);

        List<HttpMessageConverter<?>> messageConverters = restTemplate.getMessageConverters();
        Iterator<HttpMessageConverter<?>> iterator = messageConverters.iterator();
        while (iterator.hasNext()) {
            HttpMessageConverter<?> converter = iterator.next();
            if (converter instanceof StringHttpMessageConverter) {
                iterator.remove();
            }
        }
        messageConverters.add(new StringHttpMessageConverter(Charset.forName("UTF-8")));
        restTemplate.getMessageConverters().add(new MappingJackson2HttpMessageConverter());
        return restTemplate;
    }

    @Bean
    @ConditionalOnMissingBean({ClientHttpRequestFactory.class})
    public ClientHttpRequestFactory simpleClientHttpRequestFactory() {
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        factory.setReadTimeout(15000);// ms
        return factory;
    }
}
```

### 参考

[https://spring.io/blog/2009/03/27/rest-in-spring-3-resttemplate](https://spring.io/blog/2009/03/27/rest-in-spring-3-resttemplate)


