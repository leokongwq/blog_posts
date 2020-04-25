---
layout: post
comments: true
title: HTTP301和302的区别
date: 2016-11-09 15:14:17
tags:
categories:
- web
---

### 官方解释

#### 301 Moved Permanently

The requested resource has been assigned a new permanent URI and any future references to this resource SHOULD use one of the returned URIs. Clients with link editing capabilities ought to automatically re-link references to the Request-URI to one or more of the new references returned by the server, where possible. This response is cacheable unless indicated otherwise.

The new permanent URI SHOULD be given by the Location field in the response. Unless the request method was HEAD, the entity of the response SHOULD contain a short hypertext note with a hyperlink to the new URI(s).

If the 301 status code is received in response to a request other than GET or HEAD, the user agent MUST NOT automatically redirect the request unless it can be confirmed by the user, since this might change the conditions under which the request was issued.

      Note: When automatically redirecting a POST request after
      receiving a 301 status code, some existing HTTP/1.0 user agents
      will erroneously change it into a GET request.
      
<!-- more -->
      
#### 302 Found

The requested resource resides temporarily under a different URI. Since the redirection might be altered on occasion, the client SHOULD continue to use the Request-URI for future requests. This response is only cacheable if indicated by a Cache-Control or Expires header field.

The temporary URI SHOULD be given by the Location field in the response. Unless the request method was HEAD, the entity of the response SHOULD contain a short hypertext note with a hyperlink to the new URI(s).

If the 302 status code is received in response to a request other than GET or HEAD, the user agent MUST NOT automatically redirect the request unless it can be confirmed by the user, since this might change the conditions under which the request was issued.

      Note: RFC 1945 and RFC 2068 specify that the client is not allowed
      to change the method on the redirected request.  However, most
      existing user agent implementations treat 302 as if it were a 303
      response, performing a GET on the Location field-value regardless
      of the original request method. The status codes 303 and 307 have
      been added for servers that wish to make unambiguously clear which
      kind of reaction is expected of the client.      
### 日常理解
在日常的开发中我们可以通过这种方式实现请求的跳转，服务端（response.redirect），浏览器端（js控制）, 我们也知道`301`表示永久跳转，`302`表示临时跳转；更深入的区别也没有细究过，觉得只有实现了跳转功能就好（主动学习能力不足，没有一探究竟）。
      
### 区别和联系

#### 对用户
对我们普通用户来说，我们只是看到了浏览器中地址栏的改变，我们的请求被转到了新的地址， 一般我们也不用且不会关心它们之间的区别，只要能看到我想要的东西。

#### 对搜索引擎

当网页A用301重定向转到网页B时，搜索引擎可以肯定网页A永久的改变位置，或者说实际上不存在了，搜索引擎就会把网页B当作唯一有效目标；这样搜索引擎在抓取新内容的同时也将旧的网址替换为重定向之后的网址，并且会把旧页面的PR等信息转移到新页面。

302转向可能会有URL规范化及网址劫持的问题。可能被搜索引擎判为可疑转向，甚至认为是作弊。

网址规范化：[http://www.seozac.com/seo/url-canonicalization/](http://www.seozac.com/seo/url-canonicalization/)

网址劫持:302重定向和网址劫持（URL hijacking）有什么关系呢？这要从搜索引擎如何处理302转向说起。从定义来说，从网址A做一个302重定向到网址B时，主机服务器的隐含意思是网址A随时有可能改主意，重新显示本身的内容或转向其他的地方。大部分的搜索引擎在大部分情况下，当收到302重定向时，一般只要去抓取目标网址就可以了，也就是说网址B。
实际上如果搜索引擎在遇到302转向时，百分之百的都抓取目标网址B的话，就不用担心网址URL劫持了。问题就在于，有的时候搜索引擎，尤其是Google，并不能总是抓取目标网址。为什么呢？比如说，有的时候A网址很短，但是它做了一个302重定向到B网址，而B网址是一个很长的乱七八糟的URL网址，甚至还有可能包含一些问号之类的参数。很自然的，A网址更加用户友好，而B网址既难看，又不用户友好。这时Google很有可能会仍然显示网址A。
由于搜索引擎排名算法只是程序而不是人，在遇到302重定向的时候，并不能像人一样的去准确判定哪一个网址更适当，这就造成了网址URL劫持的可能性。也就是说，一个不道德的人在他自己的网址A做一个302重定向到你的网址B，出于某种原因， Google搜索结果所显示的仍然是网址A，但是所用的网页内容却是你的网址B上的内容，这种情况就叫做网址URL劫持。你辛辛苦苦所写的内容就这样被别人偷走了。

      




