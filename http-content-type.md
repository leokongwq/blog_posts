---
layout: post
comments: true
title: http协议之content-type
date: 2017-11-22 16:50:30
tags:
- http
categories:
- web
---

### 背景

最近在学习vert.x，准备使用vert.x重构之前的网关应用。在开发的过程中发现post请求，没有添加请求头`Content-Type`时，后端的tomcat应用不能获取参数。查了一些文档后对这个问题有了结论：

<!-- more -->

servlet 3.1 规范 - When Parameters Are Available

```javaThe following are the conditions that must be met before post form data will be populated to the parameter set:1. The request is an HTTP or HTTPS request.2. The HTTP method is POST.3. The content type is application/x-www-form-urlencoded.4. The servlet has made an initial call of any of the getParameter family of methods on the request object.If the conditions are not met and the post form data is not included in the parameter set, the post data must still be available to the servlet via the request object’s input stream. If the conditions are met, post form data will no longer be available for reading directly from the request object’s input stream.
```

规范里已经明确的声明当请求满足: 

1. 协议是 http/https
2. 请求方法是 POST
3. 请求头 `Content-Type` 的值是 `application/x-www-form-urlencoded`
4. 调用过`getParameter`方法；则数据会被当做请求的`paramaters`，而不能再通过`request` 的 `inputstream` 直接读取。

所以不论是tomcat还是其它的servlet容器都遵循这个方式。以前查看tomcat的源代码时也了解到第一次调用`getParameter`方法才会导致tomcat解析参数，并不是提前解析的。

### http协议之Content-Type

#### 作用
`Content-Type`请求头使用来指定资源的[媒体类型](https://developer.mozilla.org/en-US/docs/Glossary/MIME_type)的。

在http响应中，响应头：`Content-Type` 用来告诉客户端，服务端返回的真实内容类型是什么。浏览器或客户端会根据这个值来做一些操作。

在http请求中（如：POST，PUT）该值用来告诉服务端，客户端发送的内容类型是什么。

#### 语法

> Content-Type: text/html; charset=utf-8
Content-Type: multipart/form-data; boundary=something

#### 指令

- media-type：  资源或数据的[MIME type](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types)
- charset： 标准的字符编码值
- boundary：对于multi-part实体来说，boundary指令是必需的，它由1到70个字符组成，这些字符通过电子邮件网关已经非常健壮，而不是以空格结束。 它被用来封装消息的多个部分的边界。

### 结论

1. 要让servlet容器自动解析post请求参数，请求头`Content-Type` 的值必须是 `application/x-www-form-urlencoded`
2. get请求是通过对查询字符串解析获取的，属于`请求行`的内容。请求头`Content-Type`通常是不需要的（应该没有请求体，当然了通过get请求你也可以带请求体， curl 命令查询 ES）。
3. 如果需要我们自己解析请求体的内容，例如：post的请求体是XML或json格式的数据，此时我们应该将`Content-Type`的值设为对应的内容类型，`application/json`或`application/xml`，并通过`getInputStream`方法获取输入流，读取流进行处理。

### 参考

servlet3.1规范

[https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Type](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Type)
[https://zhuanlan.zhihu.com/p/22536382](https://zhuanlan.zhihu.com/p/22536382)
[四种常见的 POST 提交数据方式](https://imququ.com/post/four-ways-to-post-data-in-http.html)

