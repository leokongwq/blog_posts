---
layout: post
comments: true
title: 微服务之网关
date: 2017-12-06 14:40:03
tags:
- microservices
categories:
---

本文翻译自：[http://microservices.io/patterns/apigateway.html](http://microservices.io/patterns/apigateway.html)


## API Gateway / Backend for Front-End

### 背景

想象一下，你使用`微服务架构`构建了一个在线商城并且你在实现一个商品详情页功能。你需要开发不同版本的详情页服务接口：

- 利用HTML5和JavaScript开发的针对PC浏览器和手机浏览器。HTML页面是由后端的web服务生成的。
- 原生 Android 和 iPhone 客户端 - 这些客户端通过REST API和服务端进行通信。

此外，在线商城的商品详情页服务必须通过 REST API 暴露给第三方应用程序。

一个商品详情页页面需要展示的商品信息是非常多的。例如：[Amazon.com](http://www.amazon.com) 上的一本书的商品详情页：[POJOs in Action](https://www.amazon.com/POJOs-Action-Developing-Applications-Lightweight/dp/1932394583)需要展示的信息如下：

<!-- more -->

- 书的基本信息，如 标题，作者，价格等
- 你的书籍购买记录
- 库存
- 购买选项
- 购买本书后经常购买的其它书籍
- 购买此书的顾客购买的其他物品
- 顾客评论
- 卖家等级
- 等等

由于在线商城是基于`微服务架构`设计并开发的，一个商品详情页需要的书籍是通过多个微服务来获取的。例如：

- 商品信息服务 - 商品的基本信息，如 标题，作者
- 价格服务 - 商品价格
- 订单服务 - 商品的购买记录
- 库存服务 - 商品库存
- 评论服务 - 顾客评论

因此，显示产品详细信息的代码需要从所有这些服务中获取信息。

### 问题

基于微服务的应用程序客户端应用如何访问独立的服务呢？

### 限制条件

- 微服务提供的API粒度往往不同于客户需要的。 微服务通常提供细粒度的API，这意味着客户需要与多个服务进行交互。 例如，如上所述，需要产品细节的客户端需要从众多服务中获取数据。
- 不同的客户需要不同的数据 例如，商品详细信息页面桌面的桌面浏览器版本通常比移动版本更精细。
- 不同类型的客户端的网络性能是不同的。 例如，移动网络通常比非移动网络慢得多，并且具有高得多的延迟。 当然，任何广域网都比局域网慢得多。 这意味着本地移动客户端使用的网络与服务器端Web应用程序使用的LAN具有非常不同的性能特征。 服务器端Web应用程序可以向后端服务发出多个请求，而不会影响用户的体验，因为移动客户端只能使用少数几个。
- 微服务实例的数量及其位置（主机+端口）动态变化
- 随着时间的推移，服务的分区变化应该隐藏起来
- 不同的服务可能使用不同的协议，可能有些协议对web来说并不友好。

### 解决办法

实现一个API网关，它是所有客户端的单一入口点。 API网关以两种方式之一处理请求。 一些请求只是代理/路由到适当的服务。 通过合并多个服务的结果来处理请求。

{% asset_img apigateway.jpg %}

API网关可以为每个客户端提供一个不同的API，而不是提供一种万能API风格的API。 例如，Netflix API网关运行特定于客户端的适配器代码，为每个客户端提供最适合其需求的API。

API网关还可以实现安全性，例如， 验证客户端是否有权执行请求。

### 变体: Backend for front-end

这种模式的一个变种是：`专门针对前端请求的服务后端`。 它为每种客户端定义一个单独的API网关。

{% asset_img bffe.png %}

在这个例子中，有三种客户端：Web应用程序，移动应用程序和外部第三方应用程序。 有三个不同的API网关。 每个人都为其客户提供一个API。

### API网关例子

- [Netflix API gateway](http://techblog.netflix.com/2013/01/optimizing-netflix-api.html)
- A simple [Java/Spring API gateway](https://github.com/cer/event-sourcing-examples/tree/master/java-spring/api-gateway-service) from the [Money Transfer example application](https://github.com/cer/event-sourcing-examples).

### 结论

#### 使用一个API网关有如下的好处：

- 将客户端和构成应用的微服务进行隔离
- 将客户端和微服务的发现和定位进行隔离
- 针对不同的客户端提供不同的API
- 减少请求次数。 例如，API网关可以使客户端只通过一次请求就可以获取多个服务的数据。更少的请求次数同时意味着更低的延迟和更好的用户体验。API网关对于移动应用程序至关重要
- 通过将客户端调用多个微服务的逻辑移动到API网关来简化客户端逻辑
- 从“标准的”公共网络友好API协议转换为内部使用的任何协议

#### API网关模式的一些缺点：

- 增加复杂性 - API网关是另一个必须开发，部署和管理的应用
- 由于通过API网关的额外网络跳跃而增加了响应时间 - 但是，对于大多数应用来说，额外往返的成本是微不足道的。

### 问题：

如何实现API网关？ 如果需要按比例缩放以处理高负载，则最好采用`事件驱动/响应式`方法。 在JVM上，基于NIO的库如Netty，Spring Reactor是非常有用的。 NodeJS是另一种选择。

### 相关模式

- [微服务架构模式](http://microservices.io/patterns/microservices.html) 催生了API网关模式
- API网关必须使用 [客户端发现模式](http://microservices.io/patterns/client-side-discovery.html) 或 [服务端发现模式](http://microservices.io/patterns/server-side-discovery.html) 来将请求路由到可用的服务实例。
- API网关可以对用户进行身份验证，并将包含用户信息的访问令牌传递给后端的服务
- API网关将使用[断路器](http://microservices.io/patterns/reliability/circuit-breaker.html)来调用服务
- API网关通常实现[API组合模式](http://microservices.io/patterns/data/api-composition.html)




