---
layout: post
comments: true
title: 压测工具siege简介
date: 2016-11-07 23:01:42
tags:
categories:
- web
---

### 简介

> `siege`是一个http负载测试和基准测试工具。它旨在让web开发人员了解自己代码在压力测试中的执行性能。`siege`支持基本身份验证,coockie,HTTP、HTTPS和FTP协议。用户也可以通过配置来模拟访问服务器的并发用户数.

<!-- more -->

### 参数

```shell
-> % siege -h
SIEGE 3.1.3
用法: siege [options]
       siege [options] URL
       siege -g URL
选项:
  -V, --version           打印siege版本信息
  -h, --help                打印帮助信息
  -C, --config             打印当前siege的配置信息,siege的配置信息一般在mac:/usr/local/Cellar/siege/3.1.3/etc, linux:$HOME/.siegerc
  -v, --verbose          打印详细信息
  -q, --quiet              静默执行,不会输出压测过程的详细信息
  -g, --get                 输出HTTP的头信息,该选项可以用来调试
  -c, --concurrent=NUM    并发用户数, 默认为10
  -i, --internet           模拟互联网用户访问情景,随机的选取urls文件中的URL进行访问
  -b, --benchmark      请求之间不延迟
  -t, --time=NUMm    执行压测的时间, eg: 10S 表示10秒, 10M 表示10分钟, 10H 表示10小时(一般不会用到)
  -r, --reps=NUM        重复执行的次数; 不可以和-t参数同时使用
  -f, --file=FILE           指定需要压测的URL列表文件
  -R, --rc=FILE             指定`siegerc`文件,不使用默认的配置文件
  -l, --log[=FILE]          指定日志文件, 如果文件不存在, 则默认的日志文件是: PREFIX/var/siege.log
  -m, --mark="text"         MARK, mark the log file with a string.
  -d, --delay=NUM           指定每个请求间的延迟时间,范围在 001 到 -d 指定的值
  -H, --header="text"      添加指定的请求头信息
  -A, --user-agent="text"   使用指定的UA信息
  -T, --content-type="text" 设置请求头的Content-Type字段值
```

### 用法示例

    siege -c 300 -t 5S -r 2 -f -b -q www.baidu.com

### 输出信息
    
    Transactions:  		          94 hits //总共请求次数 420
    Availability:  		      100.00 % // 成功请求百分百
    Elapsed time:  		        4.07 secs //总耗时
    Data transferred:      	        0.31 MB //总传输数据量
    Response time: 		        0.02 secs  // 响应时间
    Transaction rate:      	       23.10 trans/sec // 每秒处理请求数
    Throughput:    		        0.08 MB/sec // 吞吐量
    Concurrency:   		        0.51 // 并发数
    Successful transactions:          94 //成功处理次数
    Failed transactions:   	           0 //请求失败数
    Longest transaction:   	        0.11 // 请求最长耗时
    Shortest transaction:  	        0.01 //请求最短耗时
    
    
    

        
        
    
    


