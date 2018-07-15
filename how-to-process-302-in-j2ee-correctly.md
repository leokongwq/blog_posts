---
layout: post
comments: true
title: servlet如何正确处理302跳转
date: 2018-07-15 10:08:31
tags:
- tomcat
- resin
- web
---

## 背景

某天，将线上的resin容器替换为tomcat．　过了一段时间发现有个接口处理失败，提示异常．查看应用日志发现如下的日志：

```java
Caused by: javax.servlet.ServletException: java.lang.IllegalStateException: Cannot create a session after the response has been committed
        at org.apache.jsp.WEB_002dINF.content.order.page.error_jsp._jspService(error_jsp.java:293) ~[na:na]
        at org.apache.jasper.runtime.HttpJspBase.service(HttpJspBase.java:70) ~[jasper.jar:8.5.12]
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:742) ~[servlet-api.jar:na]
        at org.apache.jasper.servlet.JspServletWrapper.service(JspServletWrapper.java:443) ~[jasper.jar:8.5.12]
        ... 38 common frames omitted
```

查询相关接口的代码发现，代码对`302`跳转的逻辑处理有问题，具体如下：

<!-- more -->

```java
@Actions(value = {
            @Action(value = "dopay", results = {@Result(name = ERROR, location = "/WEB-INF/content/order/page/error.jsp")}),
    })
    @ActionMonitor(value = "pay.doPay")
    public String doPay() {
        // 省略代码
        String _result = dealWapClient(params);
        // 问题之所在，　当dealWapClient处理成功时，返回值就是null
        // 此时，返回ERROR， Struts2会继续执行，渲染错误页面(客户端就能看到错误页面了)
        // tomcat 能看到，　resin下看不到，原因下面分析
        if (_result == null) {
            return ERROR;
        }
        // 省略代码
    }

    public String dealWapClient(Map<String, String> params) {
        // 省略代码
        redirect(returnParams, returnUrl);
        return null;
        // 省略代码
    }
}
```

### 302跳转解释

关于302临时跳转的详细解释可以参考[HTTP_302](https://zh.wikipedia.org/wiki/HTTP_302). 
也可以参考RFC规范[http://www.ietf.org/rfc/rfc3986.txt](http://www.ietf.org/rfc/rfc3986.txt)
再次就不再赘述．

### servlet　api对302处理的规定

```java
/**
* Sends a temporary redirect response to the client using the
* specified redirect location URL and clears the buffer. The buffer will
* be replaced with the data set by this method. Calling this method sets the
* status code to {@link #SC_FOUND} 302 (Found).
* This method can accept relative URLs;the servlet container must convert
* the relative URL to an absolute URL
* before sending the response to the client. If the location is relative 
* without a leading '/' the container interprets it as relative to
* the current request URI. If the location is relative with a leading
* '/' the container interprets it as relative to the servlet container root.
* If the location is relative with two leading '/' the container interprets
* it as a network-path reference (see
* <a href="http://www.ietf.org/rfc/rfc3986.txt">
* RFC 3986: Uniform Resource Identifier (URI): Generic Syntax</a>, section 4.2
* &quot;Relative Reference&quot;).
*
* <p>If the response has already been committed, this method throws 
* an IllegalStateException.
* After using this method, the response should be considered
* to be committed and should not be written to.
*
* @param		location	the redirect location URL
* @exception	IOException	If an input or output exception occurs
* @exception	IllegalStateException	If the response was committed or
*              if a partial URL is given and cannot be converted into a valid URL
*/
public void sendRedirect(String location) throws IOException;
```

> 翻译过来意思就是： 通过该方法告诉客户端临时重定向到一个指定的URL，并且清空缓存区，之前还没有发送到客户端的数据．
并使用该方法设置的数据填充缓存区．
该方法设置http响应的状态码为302. 
如果重定向的地址为相对地址，该方法内部会将相对地址转为绝对地址．　
如果response已经committed，再次调用该方法会抛出`IllegalStateException`异常.
调用该方法后，　response对象的状态应该是`committed`，并且不应该再写入数据．

servlet-api已经详细说明了该方法的用法和需要注意的事项．但是不同的servlet容器在实现机制上可能不尽相同．
项目中发现的问题主要有两个原因：

1. 代码有bug．这个是主要原因．
2. servlet容器实现不同．

下面就分析下该方法在resin和tomcat中实现的细节：

### resin对302的处理

在resin中，`HttpServletResponse`接口的实现类是`HttpServletResponseImpl`．代码如下：

```java
abstract public class AbstractCauchoResponse implements CauchoResponse {

}

public interface CauchoResponse extends HttpServletResponse {
}

public final class HttpServletResponseImpl extends AbstractCauchoResponse
  implements CauchoResponse
{
    /**
   * Sends a redirect to the browser.  If the URL is relative, it gets
   * combined with the current url.
   *
   * @param url the possibly relative url to send to the browser
   */
  @Override
  public void sendRedirect(String url)
    throws IOException
  {
    if (url == null)
      throw new NullPointerException();

    if (isCommitted())
      throw new IllegalStateException(L.l("Can't sendRedirect() after data has committed to the client."));

    _responseStream.clearBuffer();

    // server/10c4
    // reset();
    resetBuffer();

    setStatus(SC_MOVED_TEMPORARILY);

    String encoding = getCharacterEncoding();
    boolean isLatin1 = "iso-8859-1".equals(encoding);
    
    String path = encodeAbsoluteRedirect(url);

    setHeader("Location", path);
    
    if (isLatin1)
      setHeader("Content-Type", "text/html; charset=iso-8859-1");
    else
      setHeader("Content-Type", "text/html; charset=utf-8");

    String msg = "The URL has moved <a href=\"" + path + "\">here</a>";

    // The data is required for some WAP devices that can't handle an
    // empty response.
    if (_writer != null) {
      _writer.println(msg);
    }
    else {
      ServletOutputStream out = getOutputStream();
      out.println(msg);
    }
    // closeConnection();

    _request.saveSession(); // #503
    // 非常重要，这个就是resion和tomcat的不同之处．
    // 已经关闭了，肯定不能再写入数据．
    close();
  }
   @Override
  public void close()
    throws IOException
  {
    // tck - jsp include
    AbstractHttpResponse response = _response;
    
    if (response != null) {
      response.close();
    }
  }

}

resin处理`302`方式其实非常简单，步骤如下：

1. 清空缓存区内容并进行重置
2. 设置302状态码
3. 设置`Location`　和　`Content-Type` 响应头
4. 写响应体数据
5. 保存session
6. 关闭连接

整个处理流程非常简单明了．

### tomcat对302的处理

在tomcat中，`HttpServletResponse`接口的实现类是`ResponseFacade`．该类指示一个Facade,
代码如下：

```java ResponseFacade
public class ResponseFacade implements HttpServletResponse {
    @Override
    public void sendRedirect(String location)
        throws IOException {

        if (isCommitted()) {
            throw new IllegalStateException
                (sm.getString("coyoteResponse.sendRedirect.ise"));
        }

        response.setAppCommitted(true);

        response.sendRedirect(location);

    }
}
```

真正的处理由`Response`来进行

```java
public class Response implements HttpServletResponse {
    /**
    * Send a temporary redirect to the specified redirect location URL.
    *
    * @param location Location URL to redirect to
    *
    * @exception IllegalStateException if this response has
    *  already been committed
    * @exception IOException if an input/output error occurs
    */
    @Override
    public void sendRedirect(String location) throws IOException {
        sendRedirect(location, SC_FOUND);
    }

    /**
     * Internal method that allows a redirect to be sent with a status other
     * than {@link HttpServletResponse#SC_FOUND} (302). No attempt is made to
     * validate the status code.
     *
     * @param location Location URL to redirect to
     * @param status HTTP status code that will be sent
     * @throws IOException an IO exception occurred
     */
    public void sendRedirect(String location, int status) throws IOException {
        if (isCommitted()) {
            throw new IllegalStateException(sm.getString("coyoteResponse.sendRedirect.ise"));
        }

        // Ignore any call from an included servlet
        if (included) {
            return;
        }

        // 清空缓存区内容并进行重置
        resetBuffer(true);

        // Generate a temporary redirect to the specified location
        try {
            String locationUri;
            // Relative redirects require HTTP/1.1
            if (getRequest().getCoyoteRequest().getSupportsRelativeRedirects() &&
                    getContext().getUseRelativeRedirects()) {
                locationUri = location;
            } else {
                locationUri = toAbsolute(location);
            }
            setStatus(status);
            setHeader("Location", locationUri);
            //　这里有个小魔法
            if (getContext().getSendRedirectBody()) {
                PrintWriter writer = getWriter();
                writer.print(sm.getString("coyoteResponse.sendRedirect.note",
                        Escape.htmlElementContent(locationUri)));
                flushBuffer();
            }
        } catch (IllegalArgumentException e) {
            log.warn(sm.getString("response.sendRedirectFail", location), e);
            setStatus(SC_NOT_FOUND);
        }

        // 设置缓存区的suspended标志位　
        // 从应用视图的角度看，该响应已经结束了． 但其实连接并没有关闭.
        setSuspended(true);
    }
}
```

tomcat处理`302`步骤如下：

1. 清空缓存区内容并进行重置
2. 设置302状态码
3. 设置`Location` 响应头
4. 写响应体数据
5. 保存session
6. 关闭连接

#### tomcat context配置

详细配置项参考:[https://tomcat.apache.org/tomcat-7.0-doc/config/context.html](https://tomcat.apache.org/tomcat-7.0-doc/config/context.html)

其中和302处理相关的一个配置项为:`sendRedirectBody`,文档解释如下：

```java
If true, redirect responses will include a short response body that includes details of the redirect as recommended by RFC 2616. This is disabled by default since including a response body may cause problems for some application component such as compression filters.
```

根据[RFC 2616](https://tools.ietf.org/html/rfc2616)规范，302跳转是可以带有响应体数据的(resin就按规范进行了实现).　tomcat默认处理是不带的，原因是可能与其它组件冲突，例如压缩组件.

如果将`sendRedirectBody`的值设为true,则tomcat在处理302时，在写完响应体数据后，会执行缓存区的刷新，客户端能收到对应的响应头数据，完成跳转，且不会应为后续继续写数据导致客户端不能正常跳转.

因为默认是false,导致302响应头数据没有及时发送给客户端，在`sendRedirect`后如果应用发生了异常，则已经设置了的302响应码会被500所替代，客户端不能正常跳转．

###　tomcat　`sendRedirect`后不能跳转的逻辑分析

tomcat　处理请求的流程一部分流程如下：

`StandardHostValve` -> `StandardContextValve` -> `StandardWrapperValve`

请求入口由`StandardWrapperValve`处理，　结束还是需要`StandardHostValve`来处理．

```java　StandardWrapperValve
@Override
public final void invoke(Request request, Response response)
    throws IOException, ServletException {

    // Allocate a servlet instance to process this request
    try {
        if (!unavailable) {
            servlet = wrapper.allocate();
        }
    } catch (UnavailableException e) {
       
    } catch (ServletException e) {
        
        exception(request, response, e);
    } catch (Throwable e) {
        exception(request, response, e);
        servlet = null;
    }

    //　当请求发送异常时, 已经设置的302状态码此时变为500
    private void exception(Request request, Response response,
                           Throwable exception) {
        request.setAttribute(RequestDispatcher.ERROR_EXCEPTION, exception);
        response.setStatus(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
        response.setError();
    }
}
```

StandardHostValve　处理逻辑

```java　StandardHostValve
@Override
public final void invoke(Request request, Response response)
    throws IOException, ServletException {

    try {
        // 省略代码
        try {
            if (!asyncAtStart || asyncDispatching) {
                context.getPipeline().getFirst().invoke(request, response);
            } else {
                // Make sure this request/response is here because an error
                // report is required.
                if (!response.isErrorReportRequired()) {
                    throw new IllegalStateException(sm.getString("standardHost.asyncStateError"));
                }
            }
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            container.getLogger().error("Exception Processing " + request.getRequestURI(), t);
            // If a new error occurred while trying to report a previous
            // error allow the original error to be reported.
            if (!response.isErrorReportRequired()) {
                request.setAttribute(RequestDispatcher.ERROR_EXCEPTION, t);
                throwable(request, response, t);
            }
        }
        // 在sendRedirect方法设置的suspended标志位此时又被置为false
        // 也就是说　response由可以使用了．这就是resin和tomcat设计实现的不同        
        // Now that the request/response pair is back under container
        // control lift the suspension so that the error handling can
        // complete and/or the container can flush any remaining data
        response.setSuspended(false);

        Throwable t = (Throwable) request.getAttribute(RequestDispatcher.ERROR_EXCEPTION);

        // Protect against NPEs if the context was destroyed during a
        // long running request.
        if (!context.getState().isAvailable()) {
            return;
        }

        // Look for (and render if found) an application level error page
        if (response.isErrorReportRequired()) {
            if (t != null) {
                throwable(request, response, t);
            } else {
                status(request, response);
            }
        }

        if (!request.isAsync() && !asyncAtStart) {
            context.fireRequestDestroyEvent(request.getRequest());
        }
    } finally {
        // Access a session (if present) to update last accessed time, based
        // on a strict interpretation of the specification
        if (ACCESS_SESSION) {
            request.getSession(false);
        }

        context.unbind(Globals.IS_SECURITY_ENABLED, MY_CLASSLOADER);
    }        
}
```

## 总结

1. 线上替换应用组件，尤其是底层的应用软件需要非常注意．不可全量替换．一定要逐步替换．虽然已经在测试环境，线上灰度替换了一台机器，
但是因为访问量小，导致问题发现的比较晚，直至大批量替换才发现问题．　还有就是：同一个规范，但不同的实现细节还是有差异．
2. 应用开发一定需要遵守API规范，否则会导致奇怪问题的发生．好多人总是说遇到的问题多，踩坑多，那是因为你从来不仔细阅读相关api规范文档或官方文档．不遵守规范导致的问题能叫坑吗？
3. 在使用任何一项技术时，优先查询官方文档．不要随手google或baidu．　实话说，网上的文章质量参差不齐，难免找到理解错误的文档．
4. 养成阅读源代码的习惯，好处不言而喻．如果３年前没有读过tomcat6的源代码，今天排查起来问题就非常困难了.