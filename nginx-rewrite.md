---
layout: post
comments: true
title: nginx配置rewrite指令详解
date: 2016-11-23 22:25:11
tags:
- web
- nginx
categories:
- web
---

### 前言

> Nginx 的配置指令非常多, 其中`rewrite`指令的使用频率非常高. 以前只是对该指令有简单的了解,没有详细查看该指令详细的使用方法.在
双11的秒杀配置中使用了该配置指令来做URL的跳转,但是感觉配置的有问题,重复进行了跳转.借此机会翻看[Nginx](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#rewrite)官方文档学习该指令的用法.

<!-- more -->

### 语法

    Syntax:	rewrite regex replacement [flag];
    Default:	—
    Context:	server, location, if
    
如果请求的URI配置该指令指定的正则表达式`regex`, 则请求URI会被该指令指定 `replacement` 字符串进行格式转换. `rewrite`指令是按照它们出现的顺序执行的.通过该指令的`flag`选项,我们可以终止请求的处理过程.如果字符串`replacement`是以`http://`或`https://`开头, 则该请求的处理过程会停止,并告诉客户端进行重定向.

### 可选参数flag

- last 停止处理`ngx_http_rewrite_module`指令集, 并开始查找新的匹配重写后的location.

- break 停止处理`ngx_http_rewrite_module`指令集,继续执行下面的指令

- redirect 返回302临时跳转, 当`replacement`不是以“http://” 或 “https://”开头时使用该参数.

- permanent 返回301永久重定向.

完整的重定向URL是根据当前请求的`scheme` 和指令[server_name_in_redirect](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_name_in_redirect) 以及指令 [port_in_redirect directives](http://nginx.org/en/docs/http/ngx_http_core_module.html#port_in_redirect) 共同来计算的.

**例子:**

    server {
        ...
        rewrite ^(/download/.*)/media/(.*)\..*$ $1/mp3/$2.mp3 last;
        rewrite ^(/download/.*)/audio/(.*)\..*$ $1/mp3/$2.ra  last;
        return  403;
        ...
    }

如果这些指令实在`location`“/download/”中配置的, 则应该使用参数`last`而不是`break`, 否则会导致循环重定向, 10次以后, Nginx会返回500error.

    location /download/ {
        rewrite ^(/download/.*)/media/(.*)\..*$ $1/mp3/$2.mp3 break;
        rewrite ^(/download/.*)/audio/(.*)\..*$ $1/mp3/$2.ra  break;
        return  403;
    }

如果`replacement`字符串中指定了新请求URI的请求参数, 则原有的请求参数会被追加的新的参数后面.
如果这种行为是不希望发生的, 则只需要在新的URI后面添加一个`?`就可以了.例如:

    rewrite ^/users/(.*)$ /show?user=$1? last;
        
如果一个正则表达式包含“}”或“;”字符, 则整个表达式必须用`"`括起来. 原因很明显, "}" 和 ";" 都是Nginx配置文件中的特殊符号.
        
        
### 参考
        
[http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#rewrite](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#rewrite)        

   
