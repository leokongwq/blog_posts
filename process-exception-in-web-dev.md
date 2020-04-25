---
layout: post
comments: true
title: web开发异常统一处理
date: 2016-10-12 16:30:09
tags:
    - java
    - web
categories:
    - java
    - web
    
---

### web开发异常统一处理
                        
> web开发中异常一定要进行处理。通常根据请求的类型不同进行不同的处理。如果是`http+json`接口请求的异常，需要根据错误构造对应的错误响应。如果是500异常，则需要跳转到指定的错误页面。其它的异常类型需要根据不同的HTTP状态码进行处理。目前项目中主要是使用spring-MVC进行开发web开发。通常的处理方式有一下几种。

<!-- more -->

### web.xml
    
    这种错误处理方式是最常见的，比较通用的处理方式。通过在`web.xml`文件中配置如下内容：

{% codeblock lang:xml %}    
    <error-page>
        <error-code>404</error-code>
        <location>/404</location>
    </error-page>
    <error-page>
        <error-code>500</error-code>
        <location>/500</location>
    </error-page>
    <!-- 未捕获的错误，同样可指定其它异常类，或自定义异常类 -->
    <error-page>
        <exception-type>java.lang.Exception</exception-type>
        <location>/uncaughtException</location>
    </error-page>
{% endcodeblock %}
    
`web.xml`这种错误处理方式只能显示一些静态的页面，但通常我们可能需要在错误页面输出一些更有用的数据，例如电商网站可能会有商品推荐等。其实这种方式可以和spring进行集成。

{% codeblock lang:xml %}
    <!-- 错误路径和错误页面，注意指定viewResolver -->
    <mvc:view-controller path="/404" view-name="404"/>
    <mvc:view-controller path="/500" view-name="500"/>
    <mvc:view-controller path="/uncaughtException" view-name="uncaughtException"/>
{% endcodeblock %}

使用这种方式，当我们访问`/404 /500 /uncaughtException` 这些路径时，我们配置的视图解析器会渲染`view-name`指定的视图文件并进行输出。


### Spring全局异常，代码入侵方式

**异常抛出**

{% codeblock lang:java %}
    @Controller
    public class MainController {
        @ResponseBody
        @RequestMapping("/")
        public String main(){
            throw new NullPointerException("NullPointerException Test!");
        }
    }
{% endcodeblock %}
    
**异常捕获**

{% codeblock lang:java %}
    //注意使用注解@ControllerAdvice作用域是全局Controller范围
    //可应用到所有@RequestMapping类或方法上的@ExceptionHandler、@InitBinder、@ModelAttribute，在这里是@ExceptionHandler
    @ControllerAdvice
    public class AControllerAdvice {
        @ExceptionHandler(NullPointerException.class)
        @ResponseStatus(HttpStatus.BAD_REQUEST)
        @ResponseBody
        public String handleIOException(NullPointerException ex) {
            return ClassUtils.getShortName(ex.getClass()) + ex.getMessage();
        }
    }
{% endcodeblock %}
    
### Sping全局异常，自定义异常类和异常解析    

**自定义异常类：**     

{% codeblock lang:java %}
    public class SimbaException extends RuntimeException {
    
        public SimbaException(){
            super();
        }
    
        public SimbaException(String msg, Throwable cause){
            super(msg, cause);
            //Do something...
        }
    }
{% endcodeblock %}

**抛出异常**    

{% codeblock lang:java %}
    @ResponseBody
    @RequestMapping("/ce")
    public String ce(CustomException e){
        throw new CustomException("msg",e);
    }
{% endcodeblock %}

**实现异常捕获接口HandlerExceptionResolver**    

{% codeblock lang:java %}
    public class CustomHandlerExceptionResolver implements HandlerExceptionResolver{
    
        @Override
        public ModelAndView resolveException(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, Exception e) {
            Map<String, Object> model = new HashMap<String, Object>();
            model.put("e", e);
            //这里可根据不同异常引起类做不同处理方式，本例做不同返回页面。
            String viewName = ClassUtils.getShortName(e.getClass());
            return new ModelAndView(viewName, model);
        }
    }
{% endcodeblock %}

**配置异常处理器**   

    <bean class="com.meiliinc.mls.web.handler.SimbaExceptionResolver"/>                        
                    
                    