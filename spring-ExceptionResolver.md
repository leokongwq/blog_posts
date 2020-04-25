---
layout: post
comments: true
title: spring错误处理之HandlerExceptionResolver
date: 2017-04-07 16:12:09
tags:
- spring
categories:
- java
- spring
---

### 错误处理接口

spring的错误处理主要是由接口`HandlerExceptionResolver`来定义的。不同的实现类有自己不同的错误处理机制。如果没有合适的Handler错误处理器，则最终会被容器处理，例如tomcat。好在spring本身提供了好多错误处理工我们使用，很少需要我们开发自己的错误处理器。

<!-- more -->

### AbstractHandlerExceptionResolver

`AbstractHandlerExceptionResolver`是一个抽象类，定义了错误处理的逻辑框架。

```java
public abstract class AbstractHandlerExceptionResolver implements HandlerExceptionResolver, Ordered {
    private Set<?> mappedHandlers;

    private Class<?>[] mappedHandlerClasses;

    /**
	 * Check whether this resolver is supposed to apply (i.e. if the supplied handler
	 * matches any of the configured {@linkplain #setMappedHandlers handlers} or
	 * {@linkplain #setMappedHandlerClasses handler classes}), and then delegate
	 * to the {@link #doResolveException} template method.
	 */       
    @Override
	public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse               response, Object handler, Exception ex) {
		if (shouldApplyTo(request, handler)) {
			if (this.logger.isDebugEnabled()) {
				this.logger.debug("Resolving exception from handler [" + handler + "]: " + ex);
			}
			prepareResponse(ex, response);
			//调用模板方法
			ModelAndView result = doResolveException(request, response, handler, ex);
			if (result != null) {
				logException(ex, request);
			}
			return result;
		} else {
			return null;
		}
	}
	/**
	 * Check whether this resolver is supposed to apply to the given handler.
	 * <p>The default implementation checks against the configured
	 * {@linkplain #setMappedHandlers handlers} and
	 * {@linkplain #setMappedHandlerClasses handler classes}, if any.
	 * @param request current HTTP request
	 * @param handler the executed handler, or {@code null} if none chosen
	 * at the time of the exception (for example, if multipart resolution failed)
	 * @return whether this resolved should proceed with resolving the exception
	 * for the given request and handler
	 * @see #setMappedHandlers
	 * @see #setMappedHandlerClasses
	 */
	protected boolean shouldApplyTo(HttpServletRequest request, Object handler) {
		if (handler != null) {
			if (this.mappedHandlers != null && this.mappedHandlers.contains(handler)) {
				return true;
			}
			if (this.mappedHandlerClasses != null) {
				for (Class<?> handlerClass : this.mappedHandlerClasses) {
					if (handlerClass.isInstance(handler)) {
						return true;
					}
				}
			}
		}
		// Else only apply if there are no explicit handler mappings.
		return (this.mappedHandlers == null && this.mappedHandlerClasses == null);
	}
	/**
	* 每个具体的子类来实现自己的异常处理逻辑
	*/
	protected abstract ModelAndView doResolveException(HttpServletRequest request,
			HttpServletResponse response, Object handler, Exception ex);
}
```

从上面的代码逻辑可以知道：所有`AbstractHandlerExceptionResolver`的子类必须实现自己的异常处理逻辑；可以选择覆盖`shouldApplyTo`方法来判断自己能否处理该Handler的异常。

### SimpleMappingExceptionResolver

#### SimpleMappingExceptionResolver的处理逻辑

`SimpleMappingExceptionResolver` 这个异常处理器是最早使用的。出现在好多springMVC异常处理blog中。
它的异常逻辑很简单：

```java
// 视图模型对象中存放异常对象的key
public static final String DEFAULT_EXCEPTION_ATTRIBUTE = "exception";
// 异常和异常视图的映射
private Properties exceptionMappings;
//不处理的异常类
private Class<?>[] excludedExceptions;

@Override
protected ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
		//检查该异常对应的错误视图名
		String viewName = determineViewName(ex, request);
		if (viewName != null) {
			// 检查该视图名是否配置了对应的http状态码
			Integer statusCode = determineStatusCode(request, viewName);
			if (statusCode != null) {
            //  设置http响应状态码
				applyStatusCodeIfPossible(request, response, statusCode);
			}
			//获取待渲染的ModelAndView对象
			return getModelAndView(viewName, ex, request);
		} else {
			return null;
		}
}
protected ModelAndView getModelAndView(String viewName, Exception ex) {
		ModelAndView mv = new ModelAndView(viewName);
		if (this.exceptionAttribute != null) {
			if (logger.isDebugEnabled()) {
				logger.debug("Exposing Exception as model attribute '" + this.exceptionAttribute + "'");
			}
			mv.addObject(this.exceptionAttribute, ex);
		}
		return mv;
}	
```

#### `SimpleMappingExceptionResolver`使用例子：

```java
<bean class="org.springframework.web.servlet.handler.SimpleMappingExceptionResolver">  
    <!-- 定义默认的异常处理页面 -->  
    <property name="defaultErrorView" value="error"/>
    <!-- 定义异常处理页面用来获取异常信息的变量名，默认名为exception -->  
    <property name="exceptionAttribute" value="ex"/>
    <property name="excludedExceptions">
        <list>
            <value>org.apache.shiro.authz.UnauthorizedException</value>
        </list>
    </property>
    <!-- 定义需要特殊处理的异常，用类名或完全路径名作为key，异常视图名作为值 -->  
    <property name="exceptionMappings">  
        <props>  
            <prop key="IOException">error/ioexp</prop>  
            <prop key="java.sql.SQLException">error/sqlexp</prop>  
        </props>  
    </property>
    <!-- 异常视图名 到 http状态码的映射 -->
    <property name="statusCodes">
        <map>
            <entry key="common/error/resourceNotFoundError" value="404" />
            <entry key="common/error/dataAccessError" value="500" />
        </map>
    </property>  
</bean>  
```
#### `SimpleMappingExceptionResolver`存在的问题

SimpleMappingExceptionResolver对于响应是页面的请求异常是很合适的，但是请求的是`REST`接口的话，该异常处理器就不合适了。

### DefaultHandlerExceptionResolver

DefaultHandlerExceptionResolver 是springMVC默认的异常处理器之一。这个可以从springMVC默认的配置文件dispatchServlet.properties文件可以知道：

```java
org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver
```

`DefaultHandlerExceptionResolver`的处理逻辑很简单，就是判断异常的类型然后返回对应的`ModelAndView`
如果异常不在`if else` 列表中则返回null

```java
@Override
	@SuppressWarnings("deprecation")
	protected ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
		try {
			if (ex instanceof org.springframework.web.servlet.mvc.multiaction.NoSuchRequestHandlingMethodException) {
				return handleNoSuchRequestHandlingMethod((org.springframework.web.servlet.mvc.multiaction.NoSuchRequestHandlingMethodException) ex,
						request, response, handler);
			}
			else if (ex instanceof HttpRequestMethodNotSupportedException) {
				return handleHttpRequestMethodNotSupported((HttpRequestMethodNotSupportedException) ex, request,
						response, handler);
			}   		
			//省略一大堆esle if判断
    } catch(Exception handlerException) {
        if (logger.isWarnEnabled()) {
				logger.warn("Handling of [" + ex.getClass().getName() + "] resulted in Exception", handlerException);
			}
    }		
    return null;	
```

### ResponseStatusExceptionResolver

该处理器用来哪些标注了`ResponseStatus`注解的异常

```java
@Override
protected ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
//判断异常类型是否含有 ResponseStatus 注解
ResponseStatus responseStatus = AnnotatedElementUtils.findMergedAnnotation(ex.getClass(), ResponseStatus.class);
    if (responseStatus != null) {
    	try {
    		return resolveResponseStatus(responseStatus, request, response, handler, ex);
    	}
    	catch (Exception resolveEx) {
    		logger.warn("Handling of @ResponseStatus resulted in Exception", resolveEx);
    	}
    }
    else if (ex.getCause() instanceof Exception) {
    	ex = (Exception) ex.getCause();
    	return doResolveException(request, response, handler, ex);
    }
    return null;
}
/**
* 根据 ResponseStatus 注解的属性值设置response的响应码和对应的解释消息
*/
protected ModelAndView resolveResponseStatus(ResponseStatus responseStatus, HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
	int statusCode = responseStatus.code().value();
	String reason = responseStatus.reason();
	if (this.messageSource != null) {
		reason = this.messageSource.getMessage(reason, null, reason, LocaleContextHolder.getLocale());
	}
	if (!StringUtils.hasLength(reason)) {
		response.sendError(statusCode);
	}
	else {
		response.sendError(statusCode, reason);
	}
	return new ModelAndView();
}
```		

### ExceptionHandlerExceptionResolver

该处理器在Handler的请求处理方法发生异常后，调用该Handler上有注解：`@ExceptionHandler`的方法来处理异常。

`ExceptionHandlerExceptionResolver`继承自`AbstractHandlerMethodExceptionResolver`

#### AbstractHandlerMethodExceptionResolver

```java
public abstract class AbstractHandlerMethodExceptionResolver extends AbstractHandlerExceptionResolver {
    @Override
    protected boolean shouldApplyTo(HttpServletRequest request, Object handler) {
        if (handler == null) {
        	return super.shouldApplyTo(request, handler);
        }
        // 只处理 HandlerMethod 调用中发生的异常。
        else if (handler instanceof HandlerMethod) {
        	HandlerMethod handlerMethod = (HandlerMethod) handler;
        	handler = handlerMethod.getBean();
        	return super.shouldApplyTo(request, handler);
        }
        else {
        	return false;
        }
	 }
	 @Override
	protected final ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
		return doResolveHandlerMethodException(request, response, (HandlerMethod) handler, ex);
	}
	/**
	* 模板方法，子类来实现
	*/
	protected abstract ModelAndView doResolveHandlerMethodException(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod, Exception ex);
}
```

#### ExceptionHandlerExceptionResolver的异常处理逻辑

```java
/**
 * 查找一个标有注解@ExceptionHandler 的方法，调用该方法来处理异常。
 */
@Override
protected ModelAndView doResolveHandlerMethodException(HttpServletRequest request,
		HttpServletResponse response, HandlerMethod handlerMethod, Exception exception) {
		//省略代码
}		
```

#### 使用示例

```java
@Controller
public class IndexController {
    @RequestMapping("/")
    public ModelAndView index(){
        throw new RuntimeException("error occur");
    }
    
    @ExceptionHandler(RuntimeException.class)
    public ModelAndView error(RuntimeException error, HttpServletRequest request) {
        ModelAndView mav = new ModelAndView();
        mav.setViewName("error");
        mav.addObject("param", "Runtime error");
        return mav;
    }  
            
    @ExceptionHandler()
    public ModelAndView error(Exception error, HttpServletRequest request, HttpServletResponse response) {
        ModelAndView mav = new ModelAndView();
        mav.setViewName("error");
        mav.addObject("param", "Exception error");
        return mav;
    }  
}
```

### ControllerAdvice

如果在每个Controller里面都写一个或多个标注了注解`ExceptionHandler`的异常处理方法是非常繁琐的，而且代码是重复的，当然不能这么做。 可以将这些公有的方法抽离到父类中。还有一种方案就是将这些方法抽离到标注了注解`ControllerAdvice`的类中，并将该类注册到容器中。

使用示例：

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    /**
    * 页面
    */
    @ExceptionHandler(value = {BusinessException.class})
    public ModelAndView errorHandler(HttpServletRequest servletRequest,    BusinessException e) {
        //自定义的处理逻辑
    }
    /**
    * 接口
    */
    @ExceptionHandler(value = {BusinessException.class})
    @ResponseBody
    public WebResult<String> jsonErrorHandler(HttpServletRequest servletRequest, BusinessException e) {
        //自定义的处理逻辑
    }
}
```



 
 


