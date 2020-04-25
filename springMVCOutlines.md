---
layout: post
title: Spring MVC HandlerMapping HandlerAdapter
date: 2015-04-17 
comments: true
categories:
- spring
- java
tags:
- springMVC
---

### Spring-MVC HandlerMapping HandlerAdapter简介

> 我们都知道spring-mvc的HandlerMapping是用来查找具体的Handler,HandlerAdapter是用来调用
查找到的Handler的具体方法的；下面我们就来分析一下HandlerMapping接口和HandlerAdapter接口
的具体实现：

<!-- more -->

#### HandlerAdapter
** 接口定义 **
	
	public interface HandlerAdapter {
		boolean supports(Object handler); 
		ModelAndView handle(HttpServletRequest request, HttpServletResponse 		response, Object handler) throws Exception;
		long getLastModified(HttpServletRequest request, Object handler);
	}
该接口的handle方法会执行参数handler的某个方法；不同的HandlerAdapter实现类实现逻辑各部相同；


#### HandlerAdapter子类及其实现

##### HttpRequestHandlerAdapter 和 HttpRequestHandler 

** HttpRequestHandlerAdapter **
	
	public class HttpRequestHandlerAdapter implements HandlerAdapter {

		public boolean supports(Object handler) {
    		return (handler instanceof HttpRequestHandler);
		}
		public ModelAndView handle(HttpServletRequest request, HttpServletResponse 			response, Object handler)
		throws Exception {
			((HttpRequestHandler) handler).handleRequest(request, response);
			return null;
		}
		public long getLastModified(HttpServletRequest request, Object handler) {
			if (handler instanceof LastModified) {
      			return ((LastModified) handler).getLastModified(request);
			}
			return -1L;
		}
	} 
由上面的代码可以看出：HttpRequestHandlerAdapter的handle方法的的handler参数必须是实现了HttpRequestHandler接口
的类；

**HttpRequestHandler**

	public interface HttpRequestHandler {
		void handleRequest(HttpServletRequest request, HttpServletResponse 		response) throws ServletException, IOException;
	}
	
我们可以实现该接口来根据我们自己的业务逻辑来处理http请求 ：例如HessianServiceExporter.java 就是实现了HttpRequestHandler
接口来处理Binary-RPC调用的；我们也可以实现该接口来完成基于ProtoBuffer的RPC和实现自己的http-json的RPC。

##### SimpleControllerHandlerAdapter 和 Controller

** SimpleControllerHandlerAdapter **

	public class SimpleControllerHandlerAdapter implements HandlerAdapter {
		public boolean supports(Object handler) {
			return (handler instanceof Controller);
		}
		public ModelAndView handle(HttpServletRequest request, HttpServletResponse 		response, Object handler)throws Exception {

	    	return ((Controller) handler).handleRequest(request, response);
		}
		public long getLastModified(HttpServletRequest request, Object handler) {
			if (handler instanceof LastModified) {
    			return ((LastModified) handler).getLastModified(request);
			}
			return -1L;
		}	
	}
由上面的代码可以看出：SimpleControllerHandlerAdapter 的handle方法的的handler参数必须是实现了Controller接口
的类；

** Controller接口 **

	public interface Controller {
    	ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse 		response) throws Exception;
	}
	
对于Controller接口的handleRequest方法我们应该比较亲切，相信大部分入门SpringMVC的人都用过@Controller注解和实现
过返回值是ModelAndView的handler方法；但在具体的开发中我们不需要实现Controller接口，这样就和springAPI耦合了，我们
只需要在我们的PoJo类上加上@ Controller注解，我们的Pojo就可以被特定HandlerAdapter来调用了。这是如何实现的呢，接着看。

#### AnnotationMethodHandlerAdapter

相信接触springMVC的人对这个类再熟悉不过，该类功能强大，不需要我们的Handler实现哪个Spring接口，我们的Handler仅仅是一个Pojo就行，配合spring提供的丰富注解（@Controller, @RequestMapping）就可以被AnnotationMethodHandlerAdapter调用。该类有2个重要的方法如下：

	public ModelAndView handle(HttpServletRequest request, HttpServletResponse 	response, Object handler)throws Exception {

		if (handler.getClass().getAnnotation(SessionAttributes.class) != null) {
			// Always prevent caching in case of session attribute management.
			checkAndPrepare(request, 		response,this.cacheSecondsForSessionAttributeHandlers, true);
		// Prepare cached set of session attributes names.
		}else {
			// Uses configured default cacheSeconds setting.
			checkAndPrepare(request, response, true);
		}

	// Execute invokeHandlerMethod in synchronized block if required.
		if (this.synchronizeOnSession) {
			HttpSession session = request.getSession(false);
			if (session != null) {
    			Object mutex = WebUtils.getSessionMutex(session);
				synchronized (mutex) {
    				return invokeHandlerMethod(request, response, handler);
				}
			}
		}
			return invokeHandlerMethod(request, response, handler);
		}
	
	protected ModelAndView invokeHandlerMethod( HttpServletRequest request, 		HttpServletResponse response, Object handler) throws Exception {
	try {
		ServletHandlerMethodResolver methodResolver = getMethodResolver(handler);
		Method handlerMethod = methodResolver.resolveHandlerMethod(request);
		ServletHandlerMethodInvoker methodInvoker = new 		ServletHandlerMethodInvoker(methodResolver);
		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		ExtendedModelMap implicitModel = new BindingAwareModelMap();

		Object result = methodInvoker.invokeHandlerMethod(handlerMethod, handler, webRequest, implicitModel);
		ModelAndView mav =
		methodInvoker.getModelAndView(handlerMethod, handler.getClass(), result, 		implicitModel, webRequest);
		methodInvoker.updateModelAttributes(
		handler, (mav != null ? mav.getModel() : null), implicitModel, webRequest);
		return mav;
	}
	catch (NoSuchRequestHandlingMethodException ex) {
    	return handleNoSuchRequestHandlingMethod(ex, request, response);
	}
	}
invokeHandlerMethod 方法具体来完成Handler方法的调用和返回值的处理；这里面逻辑就是：

1.根据请求url和每个方法上的@RequestMapping注解来解析出具体的方法

2.准备方法入参

3.调用方法

由上面的分析得出：我们在使用不同的Handler来处理http请求时需要配置不同的HandlerAdapter，否则可能出现ClassCastException

上面我们分析了常用的HandlerAdapter,下面我们就看看HandlerAdapter使用的Handler是如何查找的。

### HandlerMapping

#### HandlerMapping 接口定义

	public interface HandlerMapping {
		String PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE = HandlerMapping.class.getName() + 		".pathWithinHandlerMapping";
		HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
	}
	
#### HandleMapping 子类实现分析	
	
##### 1.BeanNameUrlHandlerMapping

当我们的Handler Bean 的id是以“/”开头的时候我们需要配置该HandlerMapping

##### 2.SimpleUrlHandlerMapping

SimpleUrlHandlerMapping 的 父类 AbstractUrlHandlerMapping 有个字段：
private final Map handlerMap = new LinkedHashMap(); 
该字段用来保存所有的请求路径到具体Handler的映射

	SimpleUrlHandlerMapping配置
    <bean id="handlerMapping" class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
     <property name="urlMap">
         <map>
            <entry key="/user/login.do" value-ref="loginController"/>
         </map>
     </property>
    // 配置方式二  
    <property name="mappings">
       <bean class="org.springframework.beans.factory.config.PropertiesFactoryBean">
           <property name="location">
              <value>urlMap.properties</value>  <!-- 此时urlMap.properties文件应放在WebRoot
           </property>
       </bean>
    </property>
	<property name="mappings">
        <props>
           <prop key="/user/login.do">loginController</prop>
        </props>
     </property>
     </bean>

### DispatcherServlet 分析

#### 继承关系

DispatcherServlet 继承自FrameworkServlet, FrameworkServlet 继承自HttpServletBean, HttpServletBean继承自HttpServlet；
    
#### DispatcherServlet 工作分析
    
#### 1.初始化

DispatcherServlet 类的初始化可以分为2部分：

1.static块的初始化

在DispatcherServlet 类中有一段静态初始化块

	static {
		// Load default strategy implementations from properties file.
		// This is currently strictly internal and not meant to be customized
		// by application developers.
		try {
			ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, DispatcherServlet.class);
			defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
		}
		catch (IOException ex) {
			throw new IllegalStateException("Could not load 'DispatcherServlet.properties': " + ex.getMessage());
		}
	}
        该初始化块会加载和DispatcherServlet 同包下的DispatcherServlet.properties配置文件，该配置文件中包含了springMVC默认的一些配置bean
        
	Default implementation classes for DispatcherServlet's strategy interfaces.
	Used as fallback when no matching beans are found in the DispatcherServlet context.
	Not meant to be customized by application developers.

	org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

	org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

	org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping

	org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.throwaway.ThrowawayControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter

	org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

	org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

2.servlet本身初始化机制的初始化

    GenericServlet 实现了servlet接口并实现了init方法
	public void init(ServletConfig config) throws ServletException {
        this.config = config;
        this.init(); //该方法留给子类来覆盖，实现自己特点的初始化逻辑；
}
我们的HttpServletBean覆盖了init()方法实现了自己的初始化逻辑，并留给子类初始化接口方法：initServletBean
	public final void init() throws ServletException {
		if (logger.isDebugEnabled()) {
			logger.debug("Initializing servlet '" + getServletName() + "'");
		}
		// Set bean properties from init parameters.
		try {
			PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
			BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
			ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
			bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader));
			initBeanWrapper(bw);
			bw.setPropertyValues(pvs, true);
		}
		catch (BeansException ex) {
			logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
			throw ex;
		}
		// 让子类实现自己喜欢的初始化逻辑，在HttpServletBean中initServletBean方法时空实现
		initServletBean();
		if (logger.isDebugEnabled()) {
			logger.debug("Servlet '" + getServletName() + "' configured successfully");
		}
	}
FrameworkServlet 实现的initServletBean方法

protected final void initServletBean() throws ServletException, BeansException {
		try {
                        // 初始化spring MVC的子上下文，包括框架需要的各种bean
			this.webApplicationContext = initWebApplicationContext();
                          //此处调用的方法是空实现，留给DispatcherServlet实现
			initFrameworkServlet();
		}
}

	protected WebApplicationContext initWebApplicationContext() throws BeansException {
		WebApplicationContext wac = findWebApplicationContext();
		if (wac == null) {
			// No fixed context defined for this servlet - create a local one.
			WebApplicationContext parent =
					WebApplicationContextUtils.getWebApplicationContext(getServletContext());
			wac = createWebApplicationContext(parent);
		}

		if (!this.refreshEventReceived) {
			// Apparently not a ConfigurableApplicationContext with refresh support:
			// triggering initial onRefresh manually here.
			onRefresh(wac);
		}

		if (this.publishContext) {
			// Publish the context as a servlet context attribute.
			String attrName = getServletContextAttributeName();
			getServletContext().setAttribute(attrName, wac);
			if (this.logger.isDebugEnabled()) {
				this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
						"' as ServletContext attribute with name [" + attrName + "]");
			}
		}

		return wac;
	}
        /** template method which can be overridden to add servlet-specific refresh work.
	 * Called after successful context refresh.
        */
        protected void onRefresh(ApplicationContext context) throws BeansException {
		// For subclasses: do nothing by default.
	}

//DispatcherServlet 覆盖了FrameworkServlet的onRefresh方法
protected void onRefresh(ApplicationContext context) throws BeansException {
		initStrategies(context); //初始化默认的配置信息
}
//该方法初始化了框架工作必须的基础组件
protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
}

一般我们都比较关心HandlerMapping 和HandlerAdapter，下面我们就来分析一下
/**
* 初始化HandlerMappings ，是一个List
*/
private void initHandlerMappings(ApplicationContext context) {
		this.handlerMappings = null;
                //是否检测所有的HandlerMapping, 默认是true
		if (this.detectAllHandlerMappings) {
			// Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
                        //查找上下文中所有的HandlerMapping,包括父容器中的，
			Map matchingBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(
					context, HandlerMapping.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerMappings = new ArrayList(matchingBeans.values());
				// We keep HandlerMappings in sorted order.
				Collections.sort(this.handlerMappings, new OrderComparator());
			}
		}
		else {
                        // 如果检测所有的HandlerMapping ，则默认使用BeanNameUrlHandlerMapping
                        //HANDLER_MAPPING_BEAN_NAME  = "handlerMapping"
			try {
                                //从mvc上下文查找name 为handlerMapping的Bean
				Object hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
				this.handlerMappings = Collections.singletonList(hm);
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Ignore, we'll add a default HandlerMapping later.
			}
		}

		// Ensure we have at least one HandlerMapping, by registering
		// a default HandlerMapping if no other mappings are found.
		if (this.handlerMappings == null) {
                        //使用默认的配置文件（DispatcherServlet.properties）中的配置，我们可以获得多个HandlerMapping,
                        // BeanNameUrlHandlerMapping， DefaultAnnotationHandlerMapping， 
			this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
			if (logger.isDebugEnabled()) {
				logger.debug("No HandlerMappings found in servlet '" + getServletName() + "': using default");
			}
		}
}

初始化handlerAdapters 的逻辑和上面相同

4.2.2  DispatcherServlet 如何处理请求
我们都知道HttpServlet 是通过service方法根据Http Method 的不同调用不同的处理方法来处理http请求的，我们开发自己的Servlet，一般都是覆盖HttpServlet类的不同请求处理方法来处理我们的业务。Spring-MVC也是这样，FrameworkServlet覆盖了所有的Http方法，让后把具体处理
请求的逻辑派发给processRequest() , 该方法调用抽象的doService()方法来处理请求，这样就把处理请求的真实逻辑延迟到子类（DispatcherServlet）中去实现；
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
			doService(request, response);
          }
protected abstract void doService(HttpServletRequest request, HttpServletResponse response)
	    throws Exception;

DispatcherServlet的doService实现
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
		// Make framework objects available to handlers and view objects.
                //此处把一些框架对象放到request作用域中，我们可以在自己的Handler中获取这些对象
                //常用的就是通过WebApplicationContextUtils方法获得spirng上下文，就可以获得我们配置的所有bean
		request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
		request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
		request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
		request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());
                // 此处真正完成请求的分发和处理
	        doDispatch(request, response);
}

protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		int interceptorIndex = -1;

		// Expose current LocaleResolver and request as LocaleContext.
		LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
		LocaleContextHolder.setLocaleContext(buildLocaleContext(request), this.threadContextInheritable);

		// Expose current RequestAttributes to current thread.
		RequestAttributes previousRequestAttributes = RequestContextHolder.getRequestAttributes();
		ServletRequestAttributes requestAttributes = new ServletRequestAttributes(request);
		RequestContextHolder.setRequestAttributes(requestAttributes, this.threadContextInheritable);

		if (logger.isTraceEnabled()) {
			logger.trace("Bound request context to thread: " + request);
		}
		
		try {
			ModelAndView mv = null;
			boolean errorView = false;

			try {
				processedRequest = checkMultipart(request);

				// Determine handler for the current request.
                                //按顺序查找，只要找到一个就停止
				mappedHandler = getHandler(processedRequest, false);
				if (mappedHandler == null || mappedHandler.getHandler() == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Apply preHandle methods of registered interceptors.
				HandlerInterceptor[] interceptors = mappedHandler.getInterceptors();
				if (interceptors != null) {
					for (int i = 0; i < interceptors.length; i++) {// 调用拦截器栈中的每个拦截器的预拦截方法
						HandlerInterceptor interceptor = interceptors[i];
                                                //如果某个拦截器校验失败，则逆序调用每个拦截器的完成拦截方法
						if (!interceptor.preHandle(processedRequest, response, mappedHandler.getHandler())) {
							triggerAfterCompletion(mappedHandler, interceptorIndex, processedRequest, response, null);
							return;
						}
						interceptorIndex = i;
					}
				}

				// Actually invoke the handler.
                                //按顺序查找，只要找到一个就停止
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				// Do we need view name translation?
				if (mv != null && !mv.hasView()) {
					mv.setViewName(getDefaultViewName(request));
				}

				// Apply postHandle methods of registered interceptors.
				if (interceptors != null) {
					for (int i = interceptors.length - 1; i >= 0; i--) {
						HandlerInterceptor interceptor = interceptors[i];
						interceptor.postHandle(processedRequest, response, mappedHandler.getHandler(), mv);
					}
				}
			}
			catch (ModelAndViewDefiningException ex) {
				logger.debug("ModelAndViewDefiningException encountered", ex);
				mv = ex.getModelAndView();
			}
			catch (Exception ex) {
				Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
				mv = processHandlerException(processedRequest, response, handler, ex);
				errorView = (mv != null);
			}

			// Did the handler return a view to render?
			if (mv != null && !mv.wasCleared()) {
				render(mv, processedRequest, response);
				if (errorView) {
					WebUtils.clearErrorRequestAttributes(request);
				}
			}
			else {
				if (logger.isDebugEnabled()) {
					logger.debug("Null ModelAndView returned to DispatcherServlet with name '" +
							getServletName() + "': assuming HandlerAdapter completed request handling");
				}
			}

			// Trigger after-completion for successful outcome.
			triggerAfterCompletion(mappedHandler, interceptorIndex, processedRequest, response, null);
		}

		catch (Exception ex) {
			// Trigger after-completion for thrown exception.
			triggerAfterCompletion(mappedHandler, interceptorIndex, processedRequest, response, ex);
			throw ex;
		}
		catch (Error err) {
			ServletException ex = new NestedServletException("Handler processing failed", err);
			// Trigger after-completion for thrown exception.
			triggerAfterCompletion(mappedHandler, interceptorIndex, processedRequest, response, ex);
			throw ex;
		}

		finally {
			// Clean up any resources used by a multipart request.
			if (processedRequest != request) {
				cleanupMultipart(processedRequest);
			}

			// Reset thread-bound context.
			RequestContextHolder.setRequestAttributes(previousRequestAttributes, this.threadContextInheritable);
			LocaleContextHolder.setLocaleContext(previousLocaleContext, this.threadContextInheritable);

			// Clear request attributes.
			requestAttributes.requestCompleted();
			if (logger.isTraceEnabled()) {
				logger.trace("Cleared thread-bound request context: " + request);
			}
		}
	}
查找处理请求的Handler
protected HandlerExecutionChain getHandler(HttpServletRequest request, boolean cache) throws Exception {
		HandlerExecutionChain handler =
				(HandlerExecutionChain) request.getAttribute(HANDLER_EXECUTION_CHAIN_ATTRIBUTE);
		if (handler != null) {
			if (!cache) {
				request.removeAttribute(HANDLER_EXECUTION_CHAIN_ATTRIBUTE);
			}
			return handler;
		}

		Iterator it = this.handlerMappings.iterator();
		while (it.hasNext()) {
			HandlerMapping hm = (HandlerMapping) it.next();
			if (logger.isTraceEnabled()) {
				logger.trace("Testing handler map [" + hm  + "] in DispatcherServlet with name '" +
						getServletName() + "'");
			}
			handler = hm.getHandler(request);
			if (handler != null) {
				if (cache) {
					request.setAttribute(HANDLER_EXECUTION_CHAIN_ATTRIBUTE, handler);
				}
				return handler;
			}
		}
		return null;
	}



 

 




