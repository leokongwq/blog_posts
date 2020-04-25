---
layout: post
comments: true
title: curl学习笔记
date: 2017-05-10 21:29:11
tags:
- curl
categories:
- web
---

### 背景

在日常的开发和问题处理中，经常会使用`curl`命令来测试http接口，今天偶尔了解到可以通过curl测试接口的性能耗时问题，特此完整的学习下`curl`命令的基本使用和高级用法。

### 基本使用

#### 命令格式

curl [options...] url

<!-- more -->

#### get请求

curl 命令不加任何命令选项，直接访问给定的URL：

    curl www.baidu.com 

#### post请求

发送post请求时需要使用`-X`选项

    curl -XPOST url

除了使用`POST`外，还可以使用http规范定义的其它请求谓词，如`PUT`,`DELETE`等

发送post请求时，通常需要指定请求体数据。可以使用`-d`或`--data`来指定发送的请求体。

    curl -XPOST -d "name=leo&age=12" url
    
如果需要对请求数据进行urlencode,可以使用下面的方式：
        
    curl -XPOST --data-urlencode "name=leo&age=12" url
    
此外发送post请求还可以有如下几种子选项：

--data-raw
--data-ascii
--data-binary

#### 文件上传

文件上传可以通过下面的格式来实现：

curl --form upload=@localfilename --form name=leo www.zhihu.com   

或

curl -F upload=@localfilename -F name=leo www.zhihu.com   

如果通过代理，上面的命令有可能会被代理拒绝，这时需要指定上传文件的MIME类型。

curl -x proxyIP:port -F "action=upload" -F "filename=@file.tar.gz;type=application/octet-stream" www.baidu.com
    
#### 显示请求的详细过程

如果你想了解http请求的发送和响应过程，你可以使用`-v`选项

curl -v www.baidu.com
    
#### 显示请求跟踪信息

curl --trace-ascii  a.out www.baidu.com

或

curl --trace a.out www.baidu.com

#### 只显示响应头信息        

curl -I www.baidu.com

或

curl --head www.baidu.com    

#### 指定Referer

curl --referer www.sina.com www.baidu.com

#### 指定请求头

curl -H "Content-Type:application/json" www.baidu.com

或

curl --header "Content-Type:application/json" www.baidu.com

#### 指定User-Agent

curl --user-agent "[User Agent]" www.baidu.com 

或

curl -A "[User Agent]" www.baidu.com

#### 发送Cookie

curl -b "name=xxx" www.baidu.com

或

curl --cookie "name=xxx" www.baidu.com

或

curl --cookie a.txt www.zhihu.com

#### 保存Cookie

curl -c a.txt www.zhihu.com

或

curl --cookie-jar a.txt www.zhihu.com

#### 重定向

curl -L www.zhihu.com

或

curl --location www.zhihu.com


#### 用户认证

curl -u leo:123 www.zhihu.com

或

curl --user leo:123 www.zhihu.com


### 高级用法


#### 分析接口耗时

`curl` 命令提供了 `-w` 参数，这个参数在 manpage 是这样解释的：

> -w, --write-out <format>
              Make curl display information on stdout after a completed transfer. The format is a string that may contain plain text mixed with any number of variables. The  format can  be  specified  as  a literal "string", or you can have curl read the format from a file with "@filename" and to tell curl to read the format from stdin you write "@-".The variables present in the output format will be substituted by the value or text that curl thinks fit, as described below. All variables are specified  as  %{variable_name} and to output a normal % you just write them as %%. You can output a newline by using \n, a carriage return with \r and a tab space with \t.
              
它能够按照指定的格式打印某些信息，里面可以使用某些特定的变量，而且支持 `\n`、`\t`和 `\r` 转义字符。提供的变量很多，比如 `status_code`、`local_port`、`size_download` 等等。我们只关注和请求时间有关的变量（以 time_ 开头的变量）。

先往文本文件 curl-format.txt 写入下面的内容：              

```shell
time_namelookup:  %{time_namelookup}\n
time_connect:  %{time_connect}\n
time_appconnect:  %{time_appconnect}\n
time_redirect:  %{time_redirect}\n
time_pretransfer:  %{time_pretransfer}\n
time_starttransfer:  %{time_starttransfer}\n
----------\n
time_total:  %{time_total}\n
```

我们先看看一个简单的请求，没有重定向，也没有 SSL 协议的时间：

```shell
curl -w "@curl-format.txt" -o /dev/null -s -L "http://cizixs.com"
time_namelookup:  0.012
time_connect:  0.227
time_appconnect:  0.000
time_redirect:  0.000
time_pretransfer:  0.227
time_starttransfer:  0.443
----------
time_total:  0.867
```

可以看到这次请求各个步骤的时间都打印出来了，每个数字的单位都是秒（seconds），这样可以分析哪一步比较耗时，
方便定位问题。这个命令各个参数的意义：

- w : 从文件中读取要打印信息的格式
- o : /dev/null：把响应的内容丢弃，因为我们这里并不关心它，只关心请求的耗时情况
- s : 静默模式，没有任何输出

从这个输出，我们可以算出各个步骤的时间：

- DNS 查询：12ms
- TCP 连接时间：pretransfter(227) - namelookup(12) = 215ms
- 服务器处理时间：starttransfter(443) - pretransfer(227) = 216ms
- 内容传输时间：total(867) - starttransfer(443) = 424ms

来个比较复杂的，访问某度首页，带有中间有重定向和 SSL 协议：

```
curl -w "@curl-format.txt" -o /dev/null -s -L "https://baidu.com"
time_namelookup:  0.012
time_connect:  0.018
time_appconnect:  0.328
time_redirect:  0.356
time_pretransfer:  0.018
time_starttransfer:  0.027
----------
time_total:  0.384
```         

### 一行命令搞定

```shell
curl -o /dev/null -s -w "time_namelookup:%{time_namelookup}\ntime_connect:%{time_connect}\ntime_appconnect:%{time_appconnect}\ntime_pretransfer:%{time_pretransfer}\ntime_starttransfer:%{time_starttransfer}\ntime_total:%{time_total}\n" "http://www.kklinux.com"
```





