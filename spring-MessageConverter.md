---
layout: post
comments: true
title: spring中HttpMessageConverter详解
date: 2017-06-14 17:06:50
tags:
- spring
categories:
- spring
---

### HttpMessageConverter 介绍

> Strategy interface that specifies a converter that can convert from and to HTTP requests and responses.

简单来说，`HttpMessageConverter`就是一个策略接口， 用来将http请求转为某个对象， 或许将http响应转为指定类型的对象。 
<!-- more -->
`HttpMessageConverter`接口声明如下：

```java
public interface HttpMessageConverter<T> {

   //检测参数clazz指定的类型是否可以被该转换器读取 
	boolean canRead(Class<?> clazz, MediaType mediaType);

   //检测参数clazz指定的类型是否可以被该转换器写入
	boolean canWrite(Class<?> clazz, MediaType mediaType);

   //获取该转换器支持的媒体类型列表
	List<MediaType> getSupportedMediaTypes();

   //从输入信息中读取指定类型对象 
	T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException;

   // 将指定的对象写入到输出
	void write(T t, MediaType contentType, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException;

}
```

### 什么时候使用？

通常，我们在`Controller`里带有返回值得请求处理方法上加上注解`@ResponseBody`注解后，springMVC内部就会使用到`HttpMessageConverter`。 原理如下：

`AnnotationMethodHandlerAdapter`接口是专门用来处理加了`@Controller`注解的`Controller`的。

```java
@Override
public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)throws Exception {
    //省略
    return invokeHandlerMethod(request, response, handler);
}			
protected ModelAndView invokeHandlerMethod(HttpServletRequest request, HttpServletResponse response, Object handler)throws Exception {
		// 省略
    Object result = methodInvoker.invokeHandlerMethod(handlerMethod, handler, webRequest, implicitModel);
		ModelAndView mav = methodInvoker.getModelAndView(handlerMethod, handler.getClass(), result, implicitModel, webRequest);
		methodInvoker.updateModelAttributes(handler, (mav != null ? mav.getModel() : null), implicitModel, webRequest);
    return mav;
}
@SuppressWarnings("unchecked")
		public ModelAndView getModelAndView(Method handlerMethod, Class<?> handlerType, Object returnValue,ExtendedModelMap implicitModel, ServletWebRequest webRequest) throws Exception {
    // 省略
    if (returnValue instanceof HttpEntity) {
    	handleHttpEntityResponse((HttpEntity<?>) returnValue, webRequest);
    	return null;
    }
    else if (AnnotationUtils.findAnnotation(handlerMethod, ResponseBody.class) != null) {
    	handleResponseBody(returnValue, webRequest);
    	return null;
    }			
    // 省略	
}
private void handleResponseBody(Object returnValue, ServletWebRequest webRequest) throws Exception {
	if (returnValue == null) {
		return;
	}
	HttpInputMessage inputMessage = createHttpInputMessage(webRequest);
	HttpOutputMessage outputMessage = createHttpOutputMessage(webRequest);
	writeWithMessageConverters(returnValue, inputMessage, outputMessage);
}

@SuppressWarnings({ "unchecked", "rawtypes" })
private void writeWithMessageConverters(Object returnValue,HttpInputMessage inputMessage, HttpOutputMessage outputMessage) throws IOException, HttpMediaTypeNotAcceptableException {
    List<MediaType> acceptedMediaTypes = inputMessage.getHeaders().getAccept();
		if (acceptedMediaTypes.isEmpty()) {
			acceptedMediaTypes = Collections.singletonList(MediaType.ALL);
		}
		MediaType.sortByQualityValue(acceptedMediaTypes);
		Class<?> returnValueType = returnValue.getClass();
		List<MediaType> allSupportedMediaTypes = new ArrayList<MediaType>();
		if (getMessageConverters() != null) {
			for (MediaType acceptedMediaType : acceptedMediaTypes) {
				for (HttpMessageConverter messageConverter : getMessageConverters()) {
				    // 判断是否支持 
					if (messageConverter.canWrite(returnValueType, acceptedMediaType)) {
					   // 写入输出
						messageConverter.write(returnValue, acceptedMediaType, outputMessage);
						if (logger.isDebugEnabled()) {
							MediaType contentType = outputMessage.getHeaders().getContentType();
							if (contentType == null) {
								contentType = acceptedMediaType;
							}
							logger.debug("Written [" + returnValue + "] as \"" + contentType +
									"\" using [" + messageConverter + "]");
						}
						this.responseArgumentUsed = true;
						return;
					}
				}
			}
			for (HttpMessageConverter messageConverter : messageConverters) {
				allSupportedMediaTypes.addAll(messageConverter.getSupportedMediaTypes());
			}
		}
		throw new HttpMediaTypeNotAcceptableException(allSupportedMediaTypes);
}
```

### 输出json

SpringMVC默认是使用Jackson（`MappingJackson2HttpMessageConverter`）来将返回对象转为json的。当然了，你也可以配置其它的类来实现Json的输出。

### 输出XML

具体的类是：MappingJackson2XmlHttpMessageConverter。 请求头`Accept`是"application/*+xml" 或 "application/xml"

### 输出feed

具体的类是：RssChannelHttpMessageConverter。 请求头`Accept`是："application/rss+xml"








