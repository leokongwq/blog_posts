---
layout: post
comments: true
title: JWT简介
date: 2018-05-22 12:52:27
tags:
categories:
- web
---

### What is JSON Web Token?

JSON Web Token（JWT）是一个开放式标准（[RFC 7519](https://tools.ietf.org/html/rfc7519)），它定义了一种紧凑且自包含的方式，用于在各方之间以JSON对象安全的传输信息。 这些信息可以通过数字签名进行验证和信任。 JWT可以使用一个秘钥（HMAC签名算法）或使用RSA的公钥/私钥对对JWT进行签名。

虽然JWT可以加密以提供各方之间数据传递的保密性，但我们将重点关注已签名的令牌。 签名的令牌可以验证其中包含的信息的完整性，而加密令牌隐藏来自其他方的信息。 当令牌使用公钥/私钥对进行签名时，签名还证明只有持有私钥的方是签名方。

下面深入了解下JWT中的概念

- 紧凑: 因为JWT的大小比较小，因为它可以通过URL, POST参数 或者HTTP请求头来进行传递。从另一方面来说说，因为它小，所以传递速度也比较快（占用带宽小）。
- 自包含: JWT的负载包含了该用户所需的所有信息，从而避免了对DB的多次查询。

<!-- more -->

### When should you use JSON Web Tokens?

以下是JSON Web Tokens有用的一些场景：

- 认证: 这是使用JWT最常见的情况。 一旦用户登录，每个后续请求都将包含JWT，允许用户访问该令牌允许的路由，服务和资源。 单点登录（SSO）是当今广泛使用JWT的一项功能，因为它的开销很小，并且能够轻松地跨不同域使用.
- 信息交换: JWT也是一个在各方之间安全传输信息的好方法。 因为JWT可以签名 - 例如使用公钥/私钥对，所以可以确定发件人是他们自称的人。 此外，由于使用请求头和有效载荷来参加签名的计算，因此你还可以验证内容是否未被篡改。

### What is the JSON Web Token structure?

在JWT的紧凑格式中, JWT由三部分构成，每个部分以`.`号进行分割。如下所示：

- Header
- Payload
- Signature

因此一个典型的JWT可能看起来是下面的样子：

```
xxxxx.yyyyy.zzzzz
```

### Header

header部分通常包含两部分：令牌的类型，和其使用的哈希算法，例如：HMAC，SHA256 或 RSA.

例如：

```javascript
{
  "alg": "HS256",
  "typ": "JWT"
}
```

然后，这个JSON被Base64Url编码，形成JWT的第一部分。

### Payload

令牌的第二部分是包含声明的有效负载。 声明是关于实体（通常是用户）和其它元数据的声明。 有三种类型的claim：注册的claim，公开的claim和私有的claim。

- 已登记的claims：这些是一组预先定义的claims，这些claims不是强制性的，但建议提供一套有用的，可互操作的claims。 其中一些是：iss（发行者），exp（到期时间），sub（主题），aud（受众）等。

> Notice that the claim names are only three characters long as JWT is meant to be compact.

- 公共的claims: 这些可以由使用JWT的人员随意定义。 但为避免冲突，应在[IANA JSON Web令牌注册表](https://www.iana.org/assignments/jwt/jwt.xhtml)中定义它们，或将其定义为包含防冲突命名空间的URI。
- 私有的claims: 这些是为了同意使用它们并且既没有登记也没有公开声明的各方之间共享信息而创建的定制声明。

一个有效的负载可以是:

```
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

然后将有效载荷Base64Url进行编码以形成JSON Web令牌的第二部分。

> 请注意，对于已签名的令牌，此信息尽管受到篡改保护，但任何人都可以阅读。 除非加密，否则不要将关键信息放在JWT的payload或header中。

#### 签名

要创建签名部分，你必须拥用已经编码的header，编码的有效载荷，秘钥，header中指定的算法并签名。

例如，如果你想使用HMAC SHA256算法，签名将按照以下方式创建：

```
HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)
```

该签名用于验证消息在一路上没有改变，并且在使用私钥签名的令牌的情况下，它还可以验证JWT的发件人是谁说的。


#### 组合起来

输出是三个由`.`号分隔的Base64-URL字符串，可以在HTML和HTTP环境中轻松传递，而与基于XML的标准（如SAML）相比，它更加紧凑。

以下显示了一个JWT，它具有签名通过编码的header和有效负载，并且使用秘钥进行签名。

{% asset_img encoded-jwt3.png %}

如果你想要使用JWT并将这些概念付诸实践，则可以使用[jwt.io调试器](https://link.jianshu.com/?t=https://jwt.io/#debugger-io)来解码，验证和生成JWT。


### JWT 如何工作

在认证场景中，相较于传统的模式(在服务端生成会话并返回一个cookie)，当用户使用他们的凭据成功登录以后，一个JSON Web Token将会被返回，该令牌必须被保存在本地(典型的场景是保存在本地存储中，不过cookie也常常用来保存这一类信息)。

无论何时，当用户想要访问一个受保护的资源，他必须将JWT发送到服务端，典型的发送方式是通过 Authorization 请求头字段，并指定 Bearer 模式。请求头的内容看起来会像下面这样：

```
Authorization: Bearer <token>
```

这是一个无状态的认证机制，用户信息永远也不会保存在服务器的内存中。服务器受保护的路由会检查通过Authorization头传递的令牌是否是一个正确的令牌，如果检查通过，用户将被允许访问受保护的资源。由于JWT是自包含的，所有必要的信息都包含在令牌中，进而减少了查询数据库所需要的时间。

正因为如此，JWT允许你的服务完全依赖于无状态的数据接口。它不关心你的APIs寄宿在哪个域名之下，因此跨域访问(CORS)将不会成为一个问题(使用cookie就不行)

下面的图表展示了整个处理流程

{% asset_img jwt-diagram.png %}

### 安全总结

1. 预防XSS可以通过cookie存储JWT， http-only, secure
2. 预防CSRF可以通过给请求添加CSRF token， 该token可以放在WebStorage中。如此，攻击者网站不能获取该token。

当然了，如果攻击者第一步通过XSS获取了CSRF token，第二步通过CSRF攻击，则漏洞还是存在的。单此种情况发生的概率能小一点。只能通过缩短token的时间

### 参考

[hello-jwt](https://mozillazg.com/2015/06/hello-jwt.html)
[where-to-store-your-jwts-cookies-vs-html5-web-storage](https://stormpath.com/blog/where-to-store-your-jwts-cookies-vs-html5-web-storage/)
[jwt-the-right-way](https://stormpath.com/blog/jwt-the-right-way/)
[ten-things-you-should-know-about-tokens-and-cookies](https://auth0.com/blog/2014/01/27/ten-things-you-should-know-about-tokens-and-cookies/)
[where-to-store-jwt-in-browser-how-to-protect-against-csrf](http://stackoverflow.com/questions/27067251/where-to-store-jwt-in-browser-how-to-protect-against-csrf)
[别再使用JWT](http://hippoom.github.io/blogs/stoping-using-jwt-for-sessions.html)
[基于JWT的Token认证机制及安全问题](https://bbs.huaweicloud.com/blogs/06607ea7b53211e7b8317ca23e93a891)
[My Experience with JSON Web Tokens](https://x-team.com/blog/my-experience-with-json-web-tokens/)


