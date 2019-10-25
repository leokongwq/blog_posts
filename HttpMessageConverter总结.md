### HttpMessageConverter 是什么？

简单来说`HttpMessageConverter`是spring-web提供的一个接口，用来将HTTP请求转为应用使用的数据，或将HTTP接口的返回数据转为客户端需要的数据。

接口定义如下：

```java
public interface HttpMessageConverter<T> {

	/**
	 * 根据参数判断该Converter是否能对请求进行转换
	 */
	boolean canRead(Class<?> clazz, MediaType mediaType);

	/**
	 * 根据参数判断是否可以转换，生成指定的HTTP响应
	 */
	boolean canWrite(Class<?> clazz, MediaType mediaType);

	/**
	 * Return the list of {@link MediaType} objects supported by this converter.
	 * @return the list of supported media types
	 */
	List<MediaType> getSupportedMediaTypes();

	/**
	 * 从 inputMessage 读取数据，转为class指定的对象
	 */
	T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException;

	/**
	 * 将参数t指定的数据，转换为 contentType 指定的媒体类型，写入到HTTP响应。
	 */
	void write(T t, MediaType contentType, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException;

}
```



### FastJsonHttpMessageConverter

`FastJsonHttpMessageConverter` 是fastjson提供的一个 `HttpMessageConverter`实现，

在实践中，用来将HTTP请求(请求体是json格式)， 或将响应对象转为 json响应(application/json)。

```java
public class FastJsonHttpMessageConverter extends AbstractHttpMessageConverter<Object>//
        implements GenericHttpMessageConverter<Object> {
    
     /**
     * with fastJson config
     * 可以通过 FastJsonConfig 设置 json 序列化的配置， 例如：日期格式，编码等
     */
    private FastJsonConfig fastJsonConfig = new FastJsonConfig();
    
     /**
     * Can serialize/deserialize all types.
     * MediaType.ALL 表示可以转换所有的媒体类型
     */
    public FastJsonHttpMessageConverter() {
        super(MediaType.ALL);
    }
    
    /**
     * @param fastJsonConfig the fastJsonConfig to set.
     * @since 1.2.11
     */
    public void setFastJsonConfig(FastJsonConfig fastJsonConfig) {
        this.fastJsonConfig = fastJsonConfig;
    }
}
```

一个例子：

```java
@Override
public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
    FastJsonHttpMessageConverter jsonHttpMessageConverter = new FastJsonHttpMessageConverter();
    FastJsonConfig fastJsonConfig = new FastJsonConfig();
    fastJsonConfig.setCharset(Charsets.UTF_8);
    fastJsonConfig.setDateFormat("yyyy-MM-dd HH:mm:ss");
    jsonHttpMessageConverter.setFastJsonConfig(fastJsonConfig);
    super.extendMessageConverters(converters);
}
```



### spring-mvc 如何获取支持的媒体类型 (MediaType) ?

答：通过获取所有`HttpMessageConverter`支持的媒体类型。

```java
AbstractMessageConverterMethodArgumentResolver 类：

private static List<MediaType> getAllSupportedMediaTypes(List<HttpMessageConverter<?>> messageConverters) {
		Set<MediaType> allSupportedMediaTypes = new LinkedHashSet<MediaType>();
		for (HttpMessageConverter<?> messageConverter : messageConverters) {
			allSupportedMediaTypes.addAll(messageConverter.getSupportedMediaTypes());
		}
		List<MediaType> result = new ArrayList<MediaType>(allSupportedMediaTypes);
		MediaType.sortBySpecificity(result);
		return Collections.unmodifiableList(result);
	}
```

### spring-mvc 针对Response对象如何获取可以生成的媒体类(MediaType) ?

答：通过判断所有`HttpMessageConverter`是否可以转换(`canWrite`)Response对象。

```java
AbstractMessageConverterMethodProcessor 类：

protected List<MediaType> getProducibleMediaTypes(HttpServletRequest request, Class<?> valueClass, Type declaredType) {
		Set<MediaType> mediaTypes = (Set<MediaType>) request.getAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
		if (!CollectionUtils.isEmpty(mediaTypes)) {
			return new ArrayList<MediaType>(mediaTypes);
		}
		else if (!this.allSupportedMediaTypes.isEmpty()) {
			List<MediaType> result = new ArrayList<MediaType>();
			for (HttpMessageConverter<?> converter : this.messageConverters) {
				if (converter instanceof GenericHttpMessageConverter && declaredType != null) {
					if (((GenericHttpMessageConverter<?>) converter).canWrite(declaredType, valueClass, null)) {
						result.addAll(converter.getSupportedMediaTypes());
					}
				}
				else if (converter.canWrite(valueClass, null)) {
					result.addAll(converter.getSupportedMediaTypes());
				}
			}
			return result;
		}
		else {
			return Collections.singletonList(MediaType.ALL);
		}
}
```

### spring-mvc 如何判断客户端请求的媒体类(MediaType) ?

答：通过`ContentNegotiationStrategy` 策略接口的实现类来判断。

- HeaderContentNegotiationStrategy 通过请求头`Accept`
- FixedContentNegotiationStrategy  固定类型
- PathExtensionContentNegotiationStrategy  通过扩
- ServletPathExtensionContentNegotiationStrategy 通过请求路径扩展名称 eg. user.json / user.xml
- ParameterContentNegotiationStrategy 通过请求参数。user?format**=**json
- OptionalPathExtensionContentNegotiationStrategy



### spring-mvc 如何进行请求和响应内容的匹配和选择呢？

答：根据请求的媒体类型和框架组件能生成的媒体类型进行匹配。

```java
protected <T> void writeWithMessageConverters(T value, MethodParameter returnType,
			ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
			throws IOException, HttpMediaTypeNotAcceptableException, 
HttpMessageNotWritableException {
	//省略
    Set<MediaType> compatibleMediaTypes = new LinkedHashSet<MediaType>();
    for (MediaType requestedType : requestedMediaTypes) {
        for (MediaType producibleType : producibleMediaTypes) {
            if (requestedType.isCompatibleWith(producibleType)) {
                compatibleMediaTypes.add(getMostSpecificMediaType(requestedType, producibleType));
            }
        }
    }
    //省略
    
    List<MediaType> mediaTypes = new ArrayList<MediaType>(compatibleMediaTypes);
	//排序， 越是具体的媒体类型越靠前
    MediaType.sortBySpecificityAndQuality(mediaTypes);
    MediaType selectedMediaType = null;
    for (MediaType mediaType : mediaTypes) {
        // 选择第一个具体的媒体类型
        if (mediaType.isConcrete()) {
            selectedMediaType = mediaType;
            break;
        }
        else if (mediaType.equals(MediaType.ALL) || mediaType.equals(MEDIA_TYPE_APPLICATION)) {
            selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
            break;
        }
    }
}
```

### spring-mvc HttpMessageConverter 写入Response

```java
if (selectedMediaType != null) {
    selectedMediaType = selectedMediaType.removeQualityValue();
    for (HttpMessageConverter<?> messageConverter : this.messageConverters) {
        //泛型实现
        if (messageConverter instanceof GenericHttpMessageConverter) {
            if (((GenericHttpMessageConverter) messageConverter).canWrite(
                declaredType, valueType, selectedMediaType)) {
                // 可以对outputValue进行进一步处理，例如jsonp接口
                outputValue = (T) getAdvice().beforeBodyWrite(outputValue, returnType, selectedMediaType,
                                                              (Class<? extends HttpMessageConverter<?>>) messageConverter.getClass(),
                                                              inputMessage, outputMessage);
                if (outputValue != null) {
                    addContentDispositionHeader(inputMessage, outputMessage);
                    ((GenericHttpMessageConverter) messageConverter).write(
                        outputValue, declaredType, selectedMediaType, outputMessage);
                    if (logger.isDebugEnabled()) {
                        logger.debug("Written [" + outputValue + "] as \"" + selectedMediaType +
                                     "\" using [" + messageConverter + "]");
                    }
                }
                return;
            }
        }
        else if (messageConverter.canWrite(valueType, selectedMediaType)) {
            outputValue = (T) getAdvice().beforeBodyWrite(outputValue, returnType, selectedMediaType,
                                                          (Class<? extends HttpMessageConverter<?>>) messageConverter.getClass(),
                                                          inputMessage, outputMessage);
            if (outputValue != null) {
                addContentDispositionHeader(inputMessage, outputMessage);
                ((HttpMessageConverter) messageConverter).write(outputValue, selectedMediaType, outputMessage);
                if (logger.isDebugEnabled()) {
                    logger.debug("Written [" + outputValue + "] as \"" + selectedMediaType +
                                 "\" using [" + messageConverter + "]");
                }
            }
            return;
        }
    }
}
```

### spring-mvc 返回jsonp

```java
@Bean
public FastJsonpResponseBodyAdvice fastJsonpResponseBodyAdvice() {
    //当请求参数有 callback 或 jsonp时，次Advice生效
    return new FastJsonpResponseBodyAdvice("callback", "jsonp") {
        @Override
        public MediaType getContentType(MediaType contentType, ServerHttpRequest request, ServerHttpResponse response) {
            return new MediaType("application", "javascript", Charsets.UTF_8);
        }
    };
}
```

>  需要注意的是，如果是jsonp响应， `FastJsonHttpMessageConverter`会重新设置`Content-Type`响应头， 并且不带`charset`属性。1.2.28 没有问题。 为了修复安全漏洞，1.2.58有问题了。

```java
boolean isJsonp = false;

//不知道为什么会有这行代码， 但是为了保持和原来的行为一致，还是保留下来
Object value = strangeCodeForJackson(object);

if (value instanceof FastJsonContainer) {
    FastJsonContainer fastJsonContainer = (FastJsonContainer) value;
    PropertyPreFilters filters = fastJsonContainer.getFilters();
    allFilters.addAll(filters.getFilters());
    value = fastJsonContainer.getValue();
}

//revise 2017-10-23 ,
// 保持原有的MappingFastJsonValue对象的contentType不做修改 保持旧版兼容。
// 但是新的JSONPObject将返回标准的contentType：application/javascript ，不对是否有function进行判断
if (value instanceof MappingFastJsonValue) {
    if (!StringUtils.isEmpty(((MappingFastJsonValue) value).getJsonpFunction())) {
        isJsonp = true;
    }
} else if (value instanceof JSONPObject) {
    isJsonp = true;
}
if (isJsonp) {
     // 重新设置了Content-Type的值
     headers.setContentType(APPLICATION_JAVASCRIPT);
}
```





### spring-mvc 内容协商

参考：[https://junq.io/spring-mvc%E5%AE%9E%E7%8E%B0http%E5%86%85%E5%AE%B9%E5%8D%8F%E5%95%86-content-negotiation.html](https://junq.io/spring-mvc实现http内容协商-content-negotiation.html)