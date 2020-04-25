---
layout: post
comments: true
title: 常见DNS记录解释
date: 2018-05-24 14:18:58
tags:
categories:
- web
---

### A记录

A记录，又称IP指向，用户可以设置域名到目标主机IP地址的映射，从而实现通过域名找到服务器。

#### 负载均衡

可以配置多条A记录，通过域名解析来实现`轮询`负载均衡。

#### 泛域名解析

泛域名解析指的是将一个域名所有未指定的子域名统一解析到一个地址。

在`主机名`中填入`*`，类型为`A`，`IP地址/主机名`中填入web服务器的IP地址，点击“新增”按钮即可。

<!-- more -->

### NS 记录

域名服务器 (NS) 记录用于确定由哪些服务器来解析域名。

例如用户希望由12.34.56.78这台服务器解析news.mydomain.com，则需要设置`news.mydomain.com`的NS记录。

说明：`优先级`中的数字越小表示级别越高；`IP地址/主机名`中既可以填写IP地址，也可以填写像`ns.mydomain.com`这样的主机地址，但必须保证该主机地址有效。

如将`news.mydomain.com`的NS记录指向到`ns.mydomain.com`，在设置NS记录的同时还需要设置`ns.mydomain.com`的指向（因为`ns.mydomain.com`也是一个域名，需要解析）。

否则NS记录将无法正常解析；NS记录优先于A记录。即，如果一个主机地址同时存在NS记录和A记录，则A记录不生效。这里的NS记录只对子域名生效。 

### MX记录 

MX记录: 邮件交换记录。用于将以该域名为结尾的电子邮件指向对应的邮件服务器以进行处理。如：用户所用的邮件是以域名`mydomain.com`为结尾的，则需要在管理界面中添加该域名的MX记录来处理所有以`@mydomain.com`结尾的邮件。 

说明：MX记录可以使用主机名或IP地址；MX记录可以通过设置优先级实现主辅服务器设置，“优先级”中的数字越小表示级别越高。也可以使用相同优先级达到负载均衡的目的；如果在`主机名`中填入子域名则此MX记录只对该子域名生效。

### CNAME 记录

`CNAME`(Canonical Name)通常称别名指向。你可以为一个主机设置别名。比如设置test.mydomain.com，用来指向一个主机www.rddns.com那么以后就可以用test.mydomain.com来代替访问www.rddns.com了。

说明：CNAME的目标主机地址只能使用主机名，不能使用IP地址；主机名前不能有任何其他前缀，如：`http://`等是不被允许的；A记录优先于CNAME记录。即如果一个主机地址同时存在A记录和CNAME记录，则CNAME记录不生效。  

```
nslookup  www.baidu.com
Server:		10.1.30.51
Address:	10.1.30.51#53

Non-authoritative answer:
www.baidu.com	canonical name = www.a.shifen.com.
Name:	www.a.shifen.com
Address: 61.135.169.121
Name:	www.a.shifen.com
Address: 61.135.169.125
```

从上面的命令输出可以知道：`www.baidu.com`的别名是`www.a.shifen.com.`

使用 CNAME 的好处就是解耦了域名和 IP 的直接联系, 这样假如服务器 IP 发生变更, 只需要改变CNAME记录中别名的A记录中的IP。

### TXT记录

TXT记录一般是为某条记录设置说明，比如你新建了一条`a.ezloo.com`的TXT记录，TXT记录内容"this is a test TXT record."，然后你用 `nslookup -qt txt a.ezloo.com` ，你就能看到"this is a test TXT record"的字样。

除外，TXT还可以用来验证域名的所有，比如你的域名使用了Google的某项服务，Google会要求你建一个TXT记录，然后Google验证你对此域名是否具备管理权限。

在命令行下可以使用`nslookup -qt=txt a.ezloo.com`来查看TXT记录。

### AAAA记录

AAAA记录是一个指向IPv6地址的记录。

可以使用nslookup -qt=aaaa a.ezloo.com来查看AAAA记录。

### 参考资料

[DNS 基础知识](https://support.google.com/a/answer/48090?hl=zh-Hans)

[DNS articles](https://support.dnsimple.com/categories/dns/)


