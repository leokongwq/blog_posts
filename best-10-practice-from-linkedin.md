---
layout: post
comments: true
title: LinkedIn关于提高Node.JS服务的10个建议
date: 2016-11-02 17:45:30
tags:
categories:
- nodejs
---

> LinkedIn 最近从 Rails转移到 Node.js 获得了巨大的成功，它砍掉了之前90%的服务器，并使性能提升了20倍。这个消息令很多人把 Node.js 看成了葵花宝典一样的神功，可是练习神功也不是一朝一夕的事，光练招式没有内功也是不成的，更何况还得…那啥…总之不容易啊！那么除了Node.js，LinkedIn 的性能提升还有什么秘密？LinkedIn 的软件工程师 Shravya Garlapati 写的这篇文章可以给我们一些启示。

<!-- more -->

### 1. 避免同步的代码

Node.js 从一开始的设计就是单线程的。要让一个线程处理很多并发请求，你就永远不要让这个线程在阻塞、同步或运行时间很长的操作中等待。Node.js的一个超凡脱俗的特性就是它从上到下都是设计和实现为异步模式。这使它对事件驱动的应用是绝配。

不幸的是，在Node.js里进行同步/阻塞式的调用还是有可能的。例如，很多文件系统操作都提供了异步和同步版本，例如writeFile和writeFileSync。即便你在自己的代码里避免了同步方法，却有可能漫不经心地引入了一个外部库，而在该库中使用了阻塞式调用。当你这么做了之后，它对性能的影响是巨大的。

我们最初的日志实现就偶然性地引入了一个写入磁盘的同步调用。在我们进行性能测试之前，没人注意到它的存在。当我们在一台开发机上对比单个Node.js服务器性能的时候，这一个同步调用导致性能从每秒处理1000个请求下降到几十个！

```javascript
// Good: write files asynchronously
fs.writeFile('message.txt', 'Hello Node', function (err) {
  console.log("It's saved and the server remains responsive!");
});
 
// BAD: write files synchronously
fs.writeFileSync('message.txt', 'Hello Node');
console.log("It's saved, but you just blocked ALL requests!");
```

### 2.关闭socket池

Node.js的http模块会自动使用socket池，缺省是每台主机限制为5个socket。虽然这种socket复用方式可能对控制资源增长的速度有利，但如果你需要处理很多同时需要从同一台主机获取数据的并发请求，这个socket池会成为严重的性能瓶颈。在这种情况下，好的办法是增大maxSockets参数或者干脆把socket池disable掉。

```javascript
// Disable socket pooling
 
var http = require('http');
var options = {.....};
options.agent = false;
var req = http.request(options)
```

### 3. 不要用Node.js处理静态资源
   
对于静态资源，例如CSS和图像，使用标准的web服务器而不是Node.js来处理。例如，LInkedIn mobile使用nginx。我们还利用内容分发网络（CDN），它能把静态资源复制到分布在全世界的很多服务器上。这样做有两大益处：①减少了Node.js服务器的负载，②CDN能让静态内容从离用户比较近的服务器上传输过去，这样减少了延迟。


### 4.在客户端渲染页面

让我们快速比较一下服务器端渲染和客户端渲染页面。如果我们让Node.js在服务器端渲染，我们会对所有请求发回一个类似于本文的HTML页面。

```html 
<!DOCTYPE html>
<html>
  <head>
    <title>LinkedIn Mobile</title>
  </head>
  <body>
    <div class="header">
      <img src="http://mobile-cdn.linkedin.com/images/linkedin.png" alt="LinkedIn"/>
    </div>
    <div class="body">
      Hello John!
    </div>
  </body>
</html>
```

请注意，在这个页面上，除了用户名之外，所有内容都是静态的：也就是说，对于每个用户和每次刷新都是完全相同的。所以更有效率的方法是让Node.js只以JSON格式返回页面所需的动态数据。

    {"name": "John"}
    
页面的其他部分，也就是所有的静态HTML标记，可以放到一个JavaScript模板里（例如一个underscore.js模板）。

```html
<!DOCTYPE html>
<html>
  <head>
    <title>LinkedIn Mobile</title>
  </head>
  <body>
    <div class="header">
      <img src="http://mobile-cdn.linkedin.com/images/linkedin.png" alt="LinkedIn"/>
    </div>
    <div class="body">
      Hello <%= name %>!
    </div>
  </body>
</html>
```

性能的受益是这么来的：根据秘籍3，静态JavaScript模板可以从你的web服务器（例如nginx）或者CDN获取。而且，JavaScript模板可以缓存在客户端或保存在本地存储，这样初始加载页面之后，唯一需要传给客户端的就是动态JSON数据，这是最高效率的做法。该方法极大地减少了Node.js服务器上占用的CPU、 IO和负载。

### 5. 使用gzip压缩

大部分服务器和客户端都支持gzip压缩请求和响应的数据。一定要在对客户端发送响应和对远程服务器发送请求时利用它。

{% asset_img 63918611gw1ep36j8g1shj20ez0bggmd.jpg %}

### 6. 并行

尽量让你所有的阻塞式操作（比如，对远程服务的请求，数据库调用，和文件系统访问）并发进行。这样会把总延迟减少为最慢的哪个阻塞式操作的时间，而不是每个操作顺序进行的延迟总和。为了保持回调函数和错误处理整洁清楚，我们使用了Step来进行流程控制。

{% asset_img 63918611gw1ep36j8zt3zj20ei090my1.jpg %}

### 7. 避免使用session

LinkedIn mobile使用Express框架来管理请求/响应循环。

缺省情况下，session数据是保存在内存里的，这样会给服务器增加相当大的开销，特别是在用户数增长的情况下。你可以改用外部session存储，例如MongoDB或Redis，可是这样每个请求又增加了获取session数据的远程调用的开销。在可能的情况下，最好的办法是根本就不在服务器端保存状态信息。通过去掉Express里类似于 “app.use(express.session()); ” 的配置，实现无session后，你会看到更好的性能。（【译者注】原文缺了具体配置，这条是译者自己琢磨着加的）

    app.use(express.session({ secret: "keyboard cat" }));
    
### 8. 使用已编译的模块

尽可能使用已编译的模块而不是JavaScript模块。例如，当我们把SHA模块从一个用JavaScript写的版本转到一个包含在Node.js里的已编译版本，我们看到了一个巨大的性能飞跃
    
```javascript
// Use built in or binary modules
var crypto = require('crypto');
var hash = crypto.createHmac("sha1",key).update(signatureBase).digest("base64");
```

### 9. 使用标准V8 JavaScript而不是客户端的库

大部分JavaScript库是做出来用在web浏览器上的。而在浏览器上JavaScript环境真是千差万别：例如，某个浏览器可能支持类似于forEach, map/reduce这样的功能，而其他浏览器却不支持。结果，客户端的库通常包含了很多低效率的代码来克服浏览器带来的差别。另一方面，在Node.js里，你能明确知道有哪些JavaScript函数，因为支撑Node.js的V8 JavaScript引擎是按第5版 ECMA-262 标准描述的ECMAScript标准实现的。通过直接使用标准的V8函数而不是客户端库里的东西，你可以看到相当大的性能改进。

### 10. 保持你的代码小规模、轻量级

在设备慢、延迟高的移动客户端环境下工作，你会学着保持代码的小规模和轻量级。把这个思路也带到你服务器端的代码中去。经常重新思考你的决定，问自己一些问题，例如：“我们真的需要这个模块吗？”，“我们为啥要用这个框架？与其产生的开销相比它是否值得用？”，“我们能不能用更简单的方式来做这件事？”。更小，更轻量的代码总是会更有效率，也会更快。

本文转自:[http://blog.jobbole.com/40135/](http://blog.jobbole.com/40135/)

    