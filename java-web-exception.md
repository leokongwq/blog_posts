---
layout: post
comments: true
title: java web 开发异常处理总结
date: 2017-03-25 08:05:19
tags:
- spring
categories:
- java
---

## 背景

在Java web开发中异常处理是非常重要的事。主要有这几方面的考虑：

1. 用户友好
2. 影藏敏感信息
3. 接口响应数据友好

下面就是我在工作中对使用tomcat部署springmvc应用时各种处理异常的总结

<!-- more -->

## web.xml 异常处理

web.xml文件是Java web应用的部署描述符，在serlet 3.0规范之前是必须的。关于web.xml文件的详细描述可以参考[https://docs.oracle.com/cd/E24329_01/web.1211/e21049/web_xml.htm](https://docs.oracle.com/cd/E24329_01/web.1211/e21049/web_xml.htm)

在web.xml配置文件中我们可以通过如下的配置来实现异常的处理

java代码

```java
@WebServlet(name = "exceptionTestServlet", urlPatterns = {"/exception", "/"})
public class ExceptionServlet extends HttpServlet {
    @Override
    public void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        if ("/myException".equals(request.getRequestURI())){
            throw new MyException("自定义异常 MyException");
        }
        throw new IOException("File Not Found");
    }
}
public class MyException extends RuntimeException {

    public MyException(String message) {
        super(message);
    }
}
```

### web.xml 配置

```xml
<error-page>
    <error-code>404</error-code>
    <location>/404.html</location>
</error-page>
<error-page>
   <error-code>500</error-code>
   <location>/500.html</location>
</error-page>
<error-page>
   <exception-type>com.leokongwq.springlearn.web.exception.MyException</exception-type>
   <location>/exception.html</location>
</error-page>
```

当我们访问`http://localhost:7001/notfound`时，tomcat就会返回`404.html`文件的内容。

当我们访问地址：`http://localhost:7001/exception`时，tomcat就会返回`500.html`文件的内容。

当我们访问地址：`http://localhost:7001/myException`时，tomcat就会返回`myException.html`文件的内容。

### 原理分析

tomcat或者说servlet引擎在收到客户端的请求后，会根据请求的URL来查找对应的HOST和HOST下对应的Context进行处理，具体的入口就是

servlet配置中的`url-pattern`查找对应的servlet进行请求处理，`CoyoteAdapter`类的`postParseRequest`中，其中的核心语句如下：

```java
connector.getService().getMapper().map(serverName, decodedURI,version, request.getMappingData());
// If there is no context at this point, it is likely no ROOT context
// has been deployed
 if (request.getContext() == null) {
      res.setStatus(404);
      res.setMessage("Not found");
      // No context, so use host
      Host host = request.getHost();
      // Make sure there is a host (might not be during shutdown)
      if (host != null) {
          host.logAccess(request, response, 0, true);
      }
      return false;
  }
```

在该行代码中会设置`request`对应的`Context`，以及由`Context`中的哪个`servletClass`来处理该请求。如果没有找到对应的Context，那说明tomcat就没有部署任何应用，包括`ROOT`。

在请求的后续处理流程流转到`StandardContextValve`的`invoke`方法中，下面的代码就可以获取前面处理流程中设置的Wrapper对象（Wrapper就是servlet的表示）。如果没有查找到对应的Wrapper对象，则会设置`org.apache.catalina.connector.Response`响应对象的的错误标志位：

```java
if (wrapper == null || wrapper.isUnavailable()) {
  //注意此时并没有向客户端发现数据
  response.sendError(HttpServletResponse.SC_NOT_FOUND);
  return;
}
```

但是我们不要忘记tomcat的`DefaultServlet`，该servlet本身的配置如下：

```xml
<servlet>
   <servlet-name>default</servlet-name>
   <servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>
</servlet>
<servlet-mapping>
   <servlet-name>default</servlet-name>
   <url-pattern>/</url-pattern>
</servlet-mapping>
```

所以我们可以看到请求如果没有匹配的Servlet进行处理，最终会由DefaultServlet来处理。其中的核心代码如下：

```java
WebResource resource = resources.getResource(path);
if (!resource.exists()) {
  // Check if we're included so we can return the appropriate
  // missing resource name in the error
  String requestUri = (String) request.getAttribute(
          RequestDispatcher.INCLUDE_REQUEST_URI);
  if (requestUri == null) {
      requestUri = request.getRequestURI();
  } else {
      // We're included
      // SRV.9.3 says we must throw a FNFE
      throw new FileNotFoundException(sm.getString(
              "defaultServlet.missingResource", requestUri));
  }
  response.sendError(HttpServletResponse.SC_NOT_FOUND, requestUri);
  return;
}
```

可以看到如果查找不到请求URL指定的资源，则它会设置响应对象`org.apache.catalina.connector.ResponseFacade`的错误标志位。然后处理流程就流转到`StandardHostValve`，具体处理逻辑如下：

```java
@Override
public final void invoke(Request request, Response response)
   throws IOException, ServletException {
    Context context = request.getContext();
    if (context == null) {
    response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR,
            sm.getString("standardHost.noContext"));
        return;
    }
    try {
        try {
            //执行后续的Valve， 
            if (!asyncAtStart || asyncDispatching) {
                context.getPipeline().getFirst().invoke(request, response);
            } else {
                if (!response.isErrorReportRequired()) {
                    throw new IllegalStateException(sm.getString("standardHost.asyncStateError"));
                }
            }
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            container.getLogger().error("Exception Processing " + request.getRequestURI(), t);
            if (!response.isErrorReportRequired()) {
                request.setAttribute(RequestDispatcher.ERROR_EXCEPTION, t);
                throwable(request, response, t);
            }
        }

        response.setSuspended(false);

        Throwable t = (Throwable)request.getAttribute(RequestDispatcher.ERROR_EXCEPTION);
        
        if (!context.getState().isAvailable()) {
            return;
        }
        //判断response是否设置错误标志位
        if (response.isErrorReportRequired()) {
            //判断是否发生了异常
            if (t != null) {
                throwable(request, response, t);
            } else {
                //处理http状态码， 例如404
                status(request, response);
            }
        }
    } finally {
        if (ACCESS_SESSION) {
            request.getSession(false);
        }
        context.unbind(Globals.IS_SECURITY_ENABLED, MY_CLASSLOADER);
    }
}
/**
     * Handle the specified Throwable encountered while processing
     * the specified Request to produce the specified Response.  Any
     * exceptions that occur during generation of the exception report are
     * logged and swallowed.
     *
     * @param request The request being processed
     * @param response The response being generated
     * @param throwable The exception that occurred (which possibly wraps
     *  a root cause exception
     */
    protected void throwable(Request request, Response response,
                             Throwable throwable) {
        Context context = request.getContext();
        if (context == null) {
            return;
        }

        Throwable realError = throwable;

        if (realError instanceof ServletException) {
            realError = ((ServletException) realError).getRootCause();
            if (realError == null) {
                realError = throwable;
            }
        }

        // If this is an aborted request from a client just log it and return
        if (realError instanceof ClientAbortException ) {
            if (log.isDebugEnabled()) {
                log.debug
                    (sm.getString("standardHost.clientAbort",
                        realError.getCause().getMessage()));
            }
            return;
        }

        ErrorPage errorPage = findErrorPage(context, throwable);
        if ((errorPage == null) && (realError != throwable)) {
            errorPage = findErrorPage(context, realError);
        }

        if (errorPage != null) {
            if (response.setErrorReported()) {
                response.setAppCommitted(false);
                request.setAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR,
                        errorPage.getLocation());
                request.setAttribute(Globals.DISPATCHER_TYPE_ATTR,
                        DispatcherType.ERROR);
                request.setAttribute(RequestDispatcher.ERROR_STATUS_CODE,
                        Integer.valueOf(HttpServletResponse.SC_INTERNAL_SERVER_ERROR));
                request.setAttribute(RequestDispatcher.ERROR_MESSAGE,
                                  throwable.getMessage());
                request.setAttribute(RequestDispatcher.ERROR_EXCEPTION,
                                  realError);
                Wrapper wrapper = request.getWrapper();
                if (wrapper != null) {
                    request.setAttribute(RequestDispatcher.ERROR_SERVLET_NAME,
                                      wrapper.getName());
                }
                request.setAttribute(RequestDispatcher.ERROR_REQUEST_URI,
                                     request.getRequestURI());
                request.setAttribute(RequestDispatcher.ERROR_EXCEPTION_TYPE,
                                  realError.getClass());
                if (custom(request, response, errorPage)) {
                    try {
                        response.finishResponse();
                    } catch (IOException e) {
                        container.getLogger().warn("Exception Processing " + errorPage, e);
                    }
                }
            }
        } else {
            // A custom error-page has not been defined for the exception
            // that was thrown during request processing. Check if an
            // error-page for error code 500 was specified and if so,
            // send that page back as the response.
            response.setStatus(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
            // The response is an error
            response.setError();

            status(request, response);
        }
    }
```

```java
/**
* Handle the specified Throwable encountered while processing
* the specified Request to produce the specified Response.  Any
* exceptions that occur during generation of the exception report are
* logged and swallowed.
*
* @param request The request being processed
* @param response The response being generated
* @param throwable The exception that occurred (which possibly wraps
*  a root cause exception
*/
protected void throwable(Request request, Response response,
                        Throwable throwable) {
   Context context = request.getContext();
   if (context == null) {
       return;
   }

   Throwable realError = throwable;

   if (realError instanceof ServletException) {
       realError = ((ServletException) realError).getRootCause();
       if (realError == null) {
           realError = throwable;
       }
   }

   // If this is an aborted request from a client just log it and return
   if (realError instanceof ClientAbortException ) {
       if (log.isDebugEnabled()) {
           log.debug
               (sm.getString("standardHost.clientAbort",
                   realError.getCause().getMessage()));
       }
       return;
   }
   // 查询异常页面 404, 500 web.xml 中配置的
   ErrorPage errorPage = findErrorPage(context, throwable);
   if ((errorPage == null) && (realError != throwable)) {
       errorPage = findErrorPage(context, realError);
   }

   if (errorPage != null) {
       if (response.setErrorReported()) {
           response.setAppCommitted(false);
           request.setAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR,
                   errorPage.getLocation());
           request.setAttribute(Globals.DISPATCHER_TYPE_ATTR,
                   DispatcherType.ERROR);
           request.setAttribute(RequestDispatcher.ERROR_STATUS_CODE,
                   Integer.valueOf(HttpServletResponse.SC_INTERNAL_SERVER_ERROR));
           request.setAttribute(RequestDispatcher.ERROR_MESSAGE,
                             throwable.getMessage());
           request.setAttribute(RequestDispatcher.ERROR_EXCEPTION,
                             realError);
           Wrapper wrapper = request.getWrapper();
           if (wrapper != null) {
               request.setAttribute(RequestDispatcher.ERROR_SERVLET_NAME,
                                 wrapper.getName());
           }
           request.setAttribute(RequestDispatcher.ERROR_REQUEST_URI,
                                request.getRequestURI());
           request.setAttribute(RequestDispatcher.ERROR_EXCEPTION_TYPE,
                             realError.getClass());
           if (custom(request, response, errorPage)) {
               try {
                   response.finishResponse();
               } catch (IOException e) {
                   container.getLogger().warn("Exception Processing " + errorPage, e);
               }
           }
       }
   } else {
       // 么有找到自定义配置的错误页面，由tomcat自己处理
       // A custom error-page has not been defined for the exception
       // that was thrown during request processing. Check if an
       // error-page for error code 500 was specified and if so,
       // send that page back as the response.
       response.setStatus(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
       // The response is an error
       response.setError();
    
       status(request, response);
   }
}
```    

```java
//处理 http 状态码
private void status(Request request, Response response) {
    int statusCode = response.getStatus();
    // Handle a custom error page for this status code
    Context context = request.getContext();
    if (context == null) {
        return;
    }
    if (!response.isError()) {
        return;
    }
    //这个就是我们分析的最终点：根据http状态码查找web.xml配置的error-page信息
    ErrorPage errorPage = context.findErrorPage(statusCode);
    if (errorPage == null) {
        // 查找默认的错误页
        errorPage = context.findErrorPage(0);
    }
    // 这里的错误页就是tomcat提供的： org.apache.tomcat.util.descriptor.web.ErrorPage
    if (errorPage != null && response.isErrorReportRequired()) {
        response.setAppCommitted(false);
        request.setAttribute(RequestDispatcher.ERROR_STATUS_CODE, Integer.valueOf(statusCode));

        String message = response.getMessage();
        if (message == null) {
            message = "";
        }
        request.setAttribute(RequestDispatcher.ERROR_MESSAGE, message);
        request.setAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR, errorPage.getLocation());
        request.setAttribute(Globals.DISPATCHER_TYPE_ATTR, DispatcherType.ERROR);

        Wrapper wrapper = request.getWrapper();
        if (wrapper != null) {
            request.setAttribute(RequestDispatcher.ERROR_SERVLET_NAME, wrapper.getName());
        }
        request.setAttribute(RequestDispatcher.ERROR_REQUEST_URI, request.getRequestURI());
        // 这个逻辑很重要，这对我们后面分析springMVC异常处理很关键
        // custom的逻辑是将error-page配置中的location的值
        // 作为请求的URI转发到合适的servlet中进行处理，
        // 如果没有合适的servlet进行处理，则由DefaultServlet进行处理。
        // 这个逻辑就给我们自定义异常处理逻辑提供了入口。
        // 如果DefaultServlet没有查找到location指定的资源，
        // 则由 ErrorReportValve 的report方法进行处理
        if (custom(request, response, errorPage)) {
            //设置标志位，表示异常已经处理过了。
            response.setErrorReported();
            try {
                // 完成响应， 输出结果到浏览器。
                response.finishResponse();
            } catch (ClientAbortException e) {
                // Ignore
            } catch (IOException e) {
                container.getLogger().warn("Exception Processing " + errorPage, e);
            }
        }
    }
}   
```

#### StandardWrapperValve

`StandardWrapperValve`是pipeline的最后一个Valve

```java
/**
* Invoke the servlet we are managing, respecting the rules regarding
* servlet lifecycle and SingleThreadModel support.
*
* @param request Request to be processed
* @param response Response to be produced
*
* @exception IOException if an input/output error occurred
* @exception ServletException if a servlet error occurred
*/
@Override
public final void invoke(Request request, Response response)
   throws IOException, ServletException {

   //省略代码    

   // Allocate a servlet instance to process this request
   try {
       if (!unavailable) {
           servlet = wrapper.allocate();
       }
   } catch (UnavailableException e) {
      
   } catch (ServletException e) {

   } catch (Throwable e) {
   
   } 
    try {
    if ((servlet != null) && (filterChain != null)) {
        // Swallow output if needed
        if (context.getSwallowOutput()) {
            try {
                SystemLogHandler.startCapture();
                if (request.isAsyncDispatching()) {
                    request.getAsyncContextInternal().doInternalDispatch();
                } else {
                    filterChain.doFilter(request.getRequest(),
                            response.getResponse());
                }
            } finally {
                String log = SystemLogHandler.stopCapture();
                if (log != null && log.length() > 0) {
                    context.getLogger().info(log);
                }
            }
        } else {
            if (request.isAsyncDispatching()) {
                request.getAsyncContextInternal().doInternalDispatch();
            } else {
                filterChain.doFilter
                    (request.getRequest(), response.getResponse());
            }
        }

    }
} catch (ClientAbortException e) {
    throwable = e;
    exception(request, response, e);
} catch (IOException e) {
    container.getLogger().error(sm.getString("standardWrapper.serviceException", wrapper.getName(),context.getName()), e);
    throwable = e;
    exception(request, response, e);
} catch (UnavailableException e) {
    container.getLogger().error(sm.getString(
            "standardWrapper.serviceException", wrapper.getName(),
            context.getName()), e);
    //            throwable = e;
    //            exception(request, response, e);
    wrapper.unavailable(e);
    long available = wrapper.getAvailable();
    if ((available > 0L) && (available < Long.MAX_VALUE)) {
        response.setDateHeader("Retry-After", available);
        response.sendError(HttpServletResponse.SC_SERVICE_UNAVAILABLE,
                   sm.getString("standardWrapper.isUnavailable",
                                wrapper.getName()));
    } else if (available == Long.MAX_VALUE) {
        response.sendError(HttpServletResponse.SC_NOT_FOUND,
                    sm.getString("standardWrapper.notFound",
                                wrapper.getName()));
    }
    // Do not save exception in 'throwable', because we
    // do not want to do exception(request, response, e) processing
    // 异常处理
} catch (ServletException e) {
    Throwable rootCause = StandardWrapper.getRootCause(e);
    if (!(rootCause instanceof ClientAbortException)) {
        container.getLogger().error(sm.getString(
                "standardWrapper.serviceExceptionRoot",
                wrapper.getName(), context.getName(), e.getMessage()),
                rootCause);
    }
    throwable = e;
    exception(request, response, e);
} catch (Throwable e) {
    ExceptionUtils.handleThrowable(e);
    container.getLogger().error(sm.getString(
            "standardWrapper.serviceException", wrapper.getName(),
            context.getName()), e);
    throwable = e;
    exception(request, response, e);
}
/**
* Handle the specified ServletException encountered while processing
* the specified Request to produce the specified Response.  Any
* exceptions that occur during generation of the exception report are
* logged and swallowed.
*
* @param request The request being processed
* @param response The response being generated
* @param exception The exception that occurred (which possibly wraps
*  a root cause exception
*/
private void exception(Request request, Response response,
                      Throwable exception) {
   // 设置异常熟悉，后续流程会检查                       
   request.setAttribute(RequestDispatcher.ERROR_EXCEPTION, exception);
   // 如果是302跳转后，有抛出异常，此时 状态码由 302 变为500
   // 后续的异常处理交由 StandardHostValve处理
   response.setStatus(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
   response.setError();
}
```

#### ErrorReportValve
      
ErrorReportValve 的 report 方法逻辑如下：

```java
protected void report(Request request, Response response, Throwable throwable) {
    int statusCode = response.getStatus();

    if (statusCode < 400 || response.getContentWritten() > 0 || !response.setErrorReported()) {
        return;
    }
    String message = RequestUtil.filter(response.getMessage());
    if (message == null) {
        if (throwable != null) {
            String exceptionMessage = throwable.getMessage();
            if (exceptionMessage != null && exceptionMessage.length() > 0) {
            message = RequestUtil.filter((new Scanner(exceptionMessage)).nextLine());
            }
        }
        if (message == null) {
            message = "";
        }
    }
   String report = null;
   StringManager smClient = StringManager.getManager(
           Constants.Package, request.getLocales());
   response.setLocale(smClient.getLocale());
   try {
       report = smClient.getString("http." + statusCode);
   } catch (Throwable t) {
       ExceptionUtils.handleThrowable(t);
   }
   if (report == null) {
       if (message.length() == 0) {
           return;
       } else {
           report = smClient.getString("errorReportValve.noDescription");
       }
   }
   // 拼接响应字符串， 省略了
    try {
       try {
           response.setContentType("text/html");
           response.setCharacterEncoding("utf-8");
       } catch (Throwable t) {
           ExceptionUtils.handleThrowable(t);
           if (container.getLogger().isDebugEnabled()) {
               container.getLogger().debug("status.setContentType", t);
           }
       }
       // 发送响应， 这个就是我们通常看到的tomcat的错误响应页面，比较难看。
       Writer writer = response.getReporter();
       if (writer != null) {
           writer.write(sb.toString());
           response.finishResponse();
       }
   } catch (IOException e) {
       // Ignore
   } catch (IllegalStateException e) {
       // Ignore
   }
}
```

至此关于web.xml中error-page的配置，和tomcat如何处理的过程就算分析完毕了。结论就是：

*tomcat可以帮你处理错误，前提是error-page配置的location地址指向的请求应用层不拦截处理并且存在指定的资源，否则tomcat默认的处理机制就被启用，对用户是不友好的；如果引用层拦截了错误处理location指定的URI，则应用层必须处理，否则客户端只能收到404或500的响应头信息，同样对用户是不友好的。*

## springMVC 配置式异常处理

在分析springMVC异常处理流程时，我们先分析一下springMVC的请求处理过程。

springMVC的一个请求处理的大致流程

```java
HandlerExecutionChain mappedHandler = null;
try {
    // 查找 HandlerMapper
    mappedHandler = getHandler(processedRequest);
    if (mappedHandler == null || mappedHandler.getHandler() == null) {
			noHandlerFound(processedRequest, response);
			return;
	  }
    // 查找 HandlerAdapter
    HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
	  //拦截器-前置处理
    if (!mappedHandler.applyPreHandle(processedRequest, response)) {
        return;
    }
    //进行handler方法调用
    mv=ha.handle(processedRequest,response,mappedHandler.getHandler());
    //默认视图处理
    applyDefaultViewName(processedRequest, mv);
    //拦截器-后置处理
    mappedHandler.applyPostHandle(processedRequest, response, mv);     
} 
catch(Exception ex) {
    dispatchException = ex;
}
catch (Throwable err) {
    // As of 4.3, we're processing Errors thrown from handler methods as well,
    // making them available for @ExceptionHandler methods and other scenarios.
    dispatchException = new NestedServletException("Handler dispatch    failed", err);
}
//Handler 调用结果处理
processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
```

### 404 异常处理

在请求处理的过程中如果没有查找到对应的Hanlder，则由DispatchServlet的`noHandlerFound`来处理。

```java
protected void noHandlerFound(HttpServletRequest request, HttpServletResponse response) throws Exception {
   if(pageNotFoundLogger.isWarnEnabled()) {
       pageNotFoundLogger.warn("No mapping found for HTTP request with URI [" + getRequestUri(request) + "] in DispatcherServlet with name \'" + this.getServletName() + "\'");
   }
   response.sendError(404);
}
```

可以看出springMVC没有找到请求处理的Handler时，会设置响应对象response的错误标志位，后续的流程交由tomcat来处理，这就回到了上面我们分析的逻辑里面了。在这里有个小技巧可以使用：

如果web.xml配置的error-page的location是一个静态页面，则可以使用springMVC的静态资源处理servlet来处理, 完整配置如下：

```xml
//springMVC 配置文件中启用静态资源处理
<mvc:default-servlet-handler />

// web.xml 配置如下
<servlet>
   <servlet-name>dispatchServlet</servlet-name>
   <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
   <init-param>
       <param-name>contextConfigLocation</param-name>
       <param-value>classpath:dispatchServlet.xml</param-value>
   </init-param>
</servlet>
<servlet-mapping>
   <servlet-name>dispatchServlet</servlet-name>
   <url-pattern>/</url-pattern>
</servlet-mapping>

<error-page>
   <error-code>404</error-code>
   <location>/404.html</location>
</error-page>
<error-page>
   <error-code>500</error-code>
   <location>/500.html</location>
</error-page>
<error-page>
   <exception-type>com.leokongwq.springlearn.web.exception.MyException</exception-type>
   <location>/exception.html</location>
</error-page>
```

### 500 异常处理

上面分析了404异常处理，下面就分析500异常如何处理。

在springMVC的请求处理流程中有一行代码比较关键：

```java
processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
```

这行代码就是用来处理Hanlder的执行结果的，可能是通过ViewReslover解析视图，也可能是进行JSON格式的数据响应，当然也会处理Handler执行时发生的异常。

*这里有个点需要注意， 如果请求是被`DefaultServletHttpRequestHandler`处理的话，那么发生错误后就会由servlet容器进行错误处理了，如果是tomcat的话，处理流程就和上面分析404大致一样。*

通常我们的Handler都是通过`@Controller`注解进行配置的，下面就主要分析这样的异常处理机制。

DispatchServlet的方法processDispatchResult是在请求被Handler处理以后进行调研的，其中就包含了异常的处理，具体如下：

```java
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
			HandlerExecutionChain mappedHandler, ModelAndView mv, Exception exception) throws Exception {
    if (exception != null) {
    		if (exception instanceof ModelAndViewDefiningException) {
    			logger.debug("ModelAndViewDefiningException encountered", exception);
    			mv = ((ModelAndViewDefiningException) exception).getModelAndView();
    		}
    		else {
    			Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
    			// 处理异常
    			mv = processHandlerException(request, response, handler, exception);
    			errorView = (mv != null);
    		}
	}
}	
```

如果有异常，则由方法来处理

```java
protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
			Object handler, Exception ex) throws Exception {
	// 遍历所有的异常解析器来处理Handler抛出的异常
	ModelAndView exMv = null;
	for (HandlerExceptionResolver handlerExceptionResolver : this.handlerExceptionResolvers) {
		exMv = handlerExceptionResolver.resolveException(request, response, handler, ex);
		if (exMv != null) {
			break;
		}
	}
	if (exMv != null) {
		if (exMv.isEmpty()) {
			return null;
		}
		// We might still need view name translation for a plain error model...
		if (!exMv.hasView()) {
			exMv.setViewName(getDefaultViewName(request));
		}
		if (logger.isDebugEnabled()) {
			logger.debug("Handler execution resulted in exception - forwarding to resolved error view: " + exMv, ex);
		}
		WebUtils.exposeErrorRequestAttributes(request, ex, getServletName());
		return exMv;
	}
	// 如果没有合适的异常处理器，则将异常许往上抛, 由servlet容器进行处理
	throw ex;
}
```

当我们启用了springMVC注解驱动后， 默认有下面三个异常解析器

```java
org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver

org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver

org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver
```

springMVC异常处理的核心是接口`HandlerExceptionResolver`，该接口的签名如下：

```java
/**
 * 实现该接口的对象可以解析在`handler mapping`和`execution`过程中发生的异常。
 * 通常情况下是返回一个错误的视图。实现类通常作为一个Bean注册到Spring应用上下文
 * 中。 
*/ 
public interface HandlerExceptionResolver {
ModelAndView resolveException(
    HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex);
}
```

上面三个类都最终实现了该接口，如果我们要实现自定义的异常处理器，则我们可以直接实现接口`HandlerExceptionResolver`， 或者继承自`AbstractHandlerExceptionResolver`这个异常处理抽象类。通常情况下继承该类，并实现方法`doResolveException`就可以了。

springMVC本身提供了一些异常处理器供我们使用。最常用的是如下配置的类

```xml
<bean class="org.springframework.web.servlet.handler.SimpleMappingExceptionResolver">  
    <property name="exceptionMappings">  
        <props>  
            <prop key="NumberFormatException">number</prop><!-- 表示当抛出NumberFormatException的时候就返回名叫number的视图 -->  
            <prop key="NullPointerException">null</prop>  
        </props>  
    </property>  
    <property name="defaultErrorView" value="exception"/><!-- 表示当抛出异常但没有在exceptionMappings里面找到对应的异常时 返回名叫exception的视图-->  
    <property name="statusCodes"><!-- 定义在发生异常时视图跟返回码的对应关系 -->  
        <props>  
            <prop key="number">500</prop><!-- 表示在发生NumberFormatException时返回视图number，然后这里定义发生异常时视图number对应的HttpServletResponse的返回码是500 -->  
            <prop key="null">503</prop>  
        </props>  
    </property>  
    <property name="defaultStatusCode" value="404"/><!-- 表示在发生异常时默认的HttpServletResponse的返回码是多少，默认是200 -->  
</bean>  
```

大多数情况下，我们通过SimpleMappingExceptionResolver就可以实现Handler的异常处理。但是在前后端分离，或使用springMVC实现REST接口的应用中，当发生异常后需要给客户端一个JSON格式的响应，而不是一个错误的页面。这种情况下就需要自定义我们的异常处理器了。

这个自定义异常处理器的实现可以参考：[http://blog.csdn.net/linwei_1029/article/details/41674767](http://blog.csdn.net/linwei_1029/article/details/41674767)

*注意点：* 在spring的高版本中， DispatchServlet提供了一个开关属性`throwExceptionIfNoHandlerFound`， 该属性用在当查找不好对应的Handler时是否抛出异常，默认是false。该属性会在下面的方法使用到。

```java
protected void noHandlerFound(HttpServletRequest request, HttpServletResponse response) throws Exception {
    if (this.throwExceptionIfNoHandlerFound) {
			throw new NoHandlerFoundException(request.getMethod(), getRequestUri(request),
					new ServletServerHttpRequest(request).getHeaders());
		}
		else {
			response.sendError(HttpServletResponse.SC_NOT_FOUND);
		}
}
```

## 总结

1. 在不使用类似springMVC这样的框架时，可以通过web.xml配置error-page元素来实现异常的处理，404， 500等，如果需要处理json等格式接口错误，可以通过Filter来实现。
2. 在使用springMVC时，也可以在web.xml中配置error-page元素来处理异常，前题是springMVC没有拦击error-page中的location指定的请求。如果拦截了，则springMVC需要提供相应的Handler来处理异常。
3. 在使用高版本的springMVC时，可以通过自定义异常处理器，实现404， 500， http接口异常友好响应等这种复杂的异常处理。



















