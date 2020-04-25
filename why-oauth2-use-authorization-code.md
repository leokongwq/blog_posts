---
layout: post
comments: true
title: 为什么在oauth2认证中需要使用授权码
date: 2017-02-28 19:03:38
tags:
- oauth2
categories:
- web
---

### 前言

曾经被问到过为什么Oauth2的认证中需要一个授权码，当时的分析觉得是为了安全性。当时开发时也没有阅读过Oauth2的规范。没有仔细分析过原因，只是知道Oauth2分了好几种认证方式。今天就看看究竟是什么原因。


### Oauth2认证分类和流程参考资料

[https://blog.yorkxin.org/2013/09/30/oauth2-4-1-auth-code-grant-flow](https://blog.yorkxin.org/2013/09/30/oauth2-4-1-auth-code-grant-flow)
[https://tools.ietf.org/html/draft-ietf-oauth-v2-22#section-1.3.1](https://tools.ietf.org/html/draft-ietf-oauth-v2-22#section-1.3.1)
[理解OAuth 2.0](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)
[https://oauth.net/2/](https://oauth.net/2/)
[http://stackoverflow.com/questions/8666316/facebook-oauth-2-0-code-and-token](http://stackoverflow.com/questions/8666316/facebook-oauth-2-0-code-and-token)

### 结论

1. 从以上的参考资料可以了解到，为什么使用授权码，这个问题本身就是不成立的。因为简化版这种授权类型就不用授权码。这中类型的授权是针对JS客户端的（没有后端服务器）。
2. 基于授权码这种认证方式存在的原因是：安全性。

> The authorization code provides a few important security benefits such as the ability to authenticate the client, and the transmission of the access token directly to the client without passing it through the resource owner's user-agent, potentially exposing it to others, including the resource owner.





