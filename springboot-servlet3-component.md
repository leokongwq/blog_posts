---
layout: post
comments: true
title: springboot 之 Servlet3 web组件
date: 2018-11-17 11:32:28
tags:
- springboot
- servlet3
categories:
- springboot
---

### 前言

[Servlet3.0](https://jcp.org/en/jsr/detail?id=315) 规范新增了很多注解，例如：`@WebFilter`, `@WebServlet`, `@WebListener`, 可以帮助我们简化Web应用的开发，不在使用繁琐的xml配置。

但是在SpringBoot环境和支持Servlet3.0规范的容器下使用有些许的区别。

<!-- more -->

### springboot环境下使用Servlet3.0注解


#### Servlet

```java
/**
 * @author jiexiu
 */
@WebServlet(name = "authServlet", urlPatterns = {"/abc"}, loadOnStartup=1)
public class AuthServlet extends HttpServlet {

    public AuthServlet() {
        super();
    }

    @Override
    public void init() throws ServletException {
        System.out.println("AuthServlet init");
        super.init();
    }

    @Override
    public void doGet(HttpServletRequest req, HttpServletResponse resp) {
        try {
            resp.getOutputStream().write("hello".getBytes());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

#### Filter

```java
/**
 * @author : jiexiu
 * DateTime: 2018/3/7 下午10:32
 * Mail:leokongwq@gmail.com   
 * Description: desc
 */
@WebFilter("/*")
public class AuthFilter implements Filter {

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("AuthFilter init");
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println(request.getLocalName());
        chain.doFilter(request, response);
    }

    @Override
    public void destroy() {

    }
}
```

#### Application 启动类

```java
/**
 * @author jiexiu
 */
@EnableDiscoveryClient
@SpringBootApplication
@ServletComponentScan
public class BookServiceApplication {

	public static void main(String[] args) {
		SpringApplication.run(BookServiceApplication.class, args);
	}
}
```

### @ServletComponentScan

`@ServletComponentScan`这个注解很重要，它告诉SpringBoot从哪里加载Servlet组件。

如果不加该注解，则在SpringBoot内置的Servlet容器中不能正常加载注解指定的组件。

原因参见：[https://github.com/spring-projects/spring-boot/issues/2290](https://github.com/spring-projects/spring-boot/issues/2290)

### 原理解析

`@ServletComponentScan`注解是被 `ServletComponentRegisteringPostProcessor`进行处理的。 代码如下：

```java
class ServletComponentRegisteringPostProcessor
  implements BeanFactoryPostProcessor, ApplicationContextAware {
   
    private static final List<ServletComponentHandler> HANDLERS;
 
    static {
        List<ServletComponentHandler> handlers = new ArrayList<>();
        handlers.add(new WebServletHandler());
        handlers.add(new WebFilterHandler());
        handlers.add(new WebListenerHandler());
        HANDLERS = Collections.unmodifiableList(handlers);
    }
     
    //...
     
    private void scanPackage(
      ClassPathScanningCandidateComponentProvider componentProvider, 
      String packageToScan){
        //...
        for (ServletComponentHandler handler : HANDLERS) {
            handler.handle(((ScannedGenericBeanDefinition) candidate),
              (BeanDefinitionRegistry) this.applicationContext);
        }
    }
}
```


