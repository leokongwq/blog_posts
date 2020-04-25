---
layout: post
comments: true
title: SpringBoot异常处理之ErrorPageFilter
date: 2017-04-07 14:35:53
tags:
- spring
categories:
- java
- spring
---

### ErrorPageFilter 简介

ErrorPageFilter是SpringBoot在1.4.0版本提供的一个类，本质上是一个Filter。 它的作用主要有两方面：

1. 提供应用程序注册ErrorPage的接口，此时它的角色是：ErrorPageRegistry
2. 处理应用程序异常，根据异常的类型转发到对应的ErrorPage页, 从而不依赖部署的容器错误处理机制

<!-- more -->

### ErrorPageFilter 类的工作原理

#### 主要逻辑如下：

```java
@Order(Ordered.HIGHEST_PRECEDENCE)
public class ErrorPageFilter implements Filter, ErrorPageRegistr {
    private final OncePerRequestFilter delegate = new OncePerRequestFilter() {

		@Override
		protected void doFilterInternal(HttpServletRequest request,
				HttpServletResponse response, FilterChain chain)
						throws ServletException, IOException {
			ErrorPageFilter.this.doFilter(request, response, chain);
		}

		@Override
		protected boolean shouldNotFilterAsyncDispatch() {
			return false;
		}

	};

	@Override
	public void init(FilterConfig filterConfig) throws ServletException {
		this.delegate.init(filterConfig);
	}

	@Override
	public void doFilter(ServletRequest request, ServletResponse response,
			FilterChain chain) throws IOException, ServletException {
		this.delegate.doFilter(request, response, chain);
	}
	/**
	* 过滤每个请求，如果请求在处理过程中发生异常则处理
	*/
	private void doFilter(HttpServletRequest request, HttpServletResponse response,
			FilterChain chain) throws IOException, ServletException {
		ErrorWrapperResponse wrapped = new ErrorWrapperResponse(response);
		try {
			chain.doFilter(request, wrapped);
		}
		catch (Throwable ex) {
			Throwable exceptionToHandle = ex;
			if (ex instanceof NestedServletException) {
				exceptionToHandle = ((NestedServletException) ex).getRootCause();
			}
			handleException(request, response, wrapped, exceptionToHandle);
			response.flushBuffer();
		}
	}
	private void handleException(HttpServletRequest request, HttpServletResponse response,
			ErrorWrapperResponse wrapped, Throwable ex)
					throws IOException, ServletException {
		Class<?> type = ex.getClass();
		//根据异常类型查询对应错误页的地址
		String errorPath = getErrorPath(type);
		if (errorPath == null) {
			rethrow(ex);
			return;
		}
		if (response.isCommitted()) {
			handleCommittedResponse(request, ex);
			return;
		}
		//转发请求
		forwardToErrorPage(errorPath, request, wrapped, ex);
	}
	private void forwardToErrorPage(String path, HttpServletRequest request,
        		HttpServletResponse response, Throwable ex){
        setErrorAttributes(request, 500, ex.getMessage());
        request.setAttribute(ERROR_EXCEPTION, ex);
        request.setAttribute(ERROR_EXCEPTION_TYPE, ex.getClass());
        response.reset();
        response.sendError(500, ex.getMessage());
        //请求转发
        request.getRequestDispatcher(path).forward(request, response);
        request.removeAttribute(ERROR_EXCEPTION);
        request.removeAttribute(ERROR_EXCEPTION_TYPE);
	}
	/**
	* 该方法用来添加错误页信息， 是接口ErrorPageRegistry的方法
	*/
	@Override
	public void addErrorPages(ErrorPage... errorPages) {
		for (ErrorPage errorPage : errorPages) {
			if (errorPage.isGlobal()) {
				this.global = errorPage.getPath();
			}
			else if (errorPage.getStatus() != null) {
				this.statuses.put(errorPage.getStatus().value(), errorPage.getPath());
			}
			else {
				this.exceptions.put(errorPage.getException(), errorPage.getPath());
			}
		}
	}
}
```

### ErrorPageFilter 相关类

#### ErrorPageRegistry

```java
/**
 * Interface for a registry that holds {@link ErrorPage ErrorPages}.
 *
 * @author Phillip Webb
 * @since 1.4.0
 */
public interface ErrorPageRegistry {

	/**
	 * Adds error pages that will be used when handling exceptions.
	 * @param errorPages the error pages
	 */
	void addErrorPages(ErrorPage... errorPages);

}
```

#### ErrorPage

该类相当于应用部署描述符`web.xml`中的`error-page`结点的Java类表示

```java
/**
 * Simple container-independent abstraction for servlet error pages. Roughly equivalent to
 * the {@literal &lt;error-page&gt;} element traditionally found in web.xml.
 *
 * @author Dave Syer
 * @since 1.4.0
 */
public class ErrorPage {

	private final HttpStatus status;

	private final Class<? extends Throwable> exception;

	private final String path;

	public ErrorPage(String path) {
		this.status = null;
		this.exception = null;
		this.path = path;
	}

	public ErrorPage(HttpStatus status, String path) {
		this.status = status;
		this.exception = null;
		this.path = path;
	}

	public ErrorPage(Class<? extends Throwable> exception, String path) {
		this.status = null;
		this.exception = exception;
		this.path = path;
	}
}
```

#### ErrorPageRegistrar

```java
/**
 * Interface to be implemented by types that register {@link ErrorPage ErrorPages}.
 *
 * @author Phillip Webb
 * @since 1.4.0
 */
public interface ErrorPageRegistrar {

	/**
	 * Register pages as required with the given registry.
	 * @param registry the error page registry
	 */
	void registerErrorPages(ErrorPageRegistry registry);

}
```

容器中实现了`ErrorPageRegistrar`接口的类会被自动注册到`ErrorPageFilter`中。

### SpringBoot 错误页处理原理

有了上面的原理基础，我们就可以实现自己的错误页处理类（实现接口`ErrorPageRegistrar`并注册到spring容器中）。熟悉spring的肯定能猜到spring已经提供了这样的类，该类就是：`ErrorPageCustomizer`

#### ErrorPageCustomizer

`ErrorPageCustomizer` 实现了接口 `ErrorPageRegistrar`，该类会在SpringBoot启动过程中被注册到`ErrorPageFilter`。 它是如何注册的呢？

```java
@Configuration
@ConditionalOnWebApplication
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class })
// Load before the main WebMvcAutoConfiguration so that the error View is available
@AutoConfigureBefore(WebMvcAutoConfiguration.class)
@EnableConfigurationProperties(ResourceProperties.class)
public class ErrorMvcAutoConfiguration {

    /**
    * 这里就实现了Bean的注册
    */
    @Bean
    public ErrorPageCustomizer errorPageCustomizer() {
        return new ErrorPageCustomizer(this.serverProperties);
    }
    private static class ErrorPageCustomizer implements ErrorPageRegistrar, Ordered {
    		private final ServerProperties properties;
    
    		protected ErrorPageCustomizer(ServerProperties properties) {
    			this.properties = properties;
    		}
    
    		@Override
    		public void registerErrorPages(ErrorPageRegistry errorPageRegistry) {
    			ErrorPage errorPage = new ErrorPage(this.properties.getServletPrefix()
    					+ this.properties.getError().getPath());
    			errorPageRegistry.addErrorPages(errorPage);
    		}
    
    		@Override
    		public int getOrder() {
    			return 0;
    		}
    }
}
```

`ErrorPageCustomizer` 默认会转发错误请求到`/error`， 这个可以在配置文件`application.properties`中进行设置：`server.error.path`

#### 错误处理

当错误请求被转发到`/error`时， 该请求将会由SpringBoot提供的`BasicErrorController`进行处理：

```java
//${server.error.path:${error.path:/error}}
@Controller
@RequestMapping("${server.error.path:${error.path:/error}}")
public class BasicErrorController extends AbstractErrorController {
    @RequestMapping(produces = "text/html")
	public ModelAndView errorHtml(HttpServletRequest request,
			HttpServletResponse response) {
		HttpStatus status = getStatus(request);
		Map<String, Object> model = Collections.unmodifiableMap(getErrorAttributes(
				request, isIncludeStackTrace(request, MediaType.TEXT_HTML)));
		response.setStatus(status.value());
		ModelAndView modelAndView = resolveErrorView(request, response, status, model);
		return (modelAndView == null ? new ModelAndView("error", model) : modelAndView);
	}
}
```

最后就是渲染出SpringBoot默认的错误页面。一般来说我们需要自己处理/error请求来渲染我们自己提供的错误页。





















