---
layout: post
comments: true
title: linux中kill主进程和所有子进程
date: 2017-08-22 13:32:48
tags:
- linux
categories:
- linux
---

### 背景

在使用python多进程导数据时，发现有错误。此时需要kill所有的进程，一个个kill太浪费时间，不是程序员该有的解决问题的思路。肯定是获取所有的子进程`pid`并进行kill。这篇文章就是我想到的一个解决办法。

<!-- more -->

### 方法

查看主进程和所有子进程：

```shell
ps --ppid $1 

PID TTY          TIME CMD
23389 pts/1    00:01:55 python
23390 pts/1    00:01:53 python
23391 pts/1    00:01:52 python
23392 pts/1    00:01:53 python
23393 pts/1    00:01:52 python
23394 pts/1    00:01:52 python
23395 pts/1    00:01:52 python
23396 pts/1    00:01:52 python
23397 pts/1    00:01:53 python
23400 pts/1    00:01:52 python
23402 pts/1    00:01:53 python
23404 pts/1    00:01:54 python
23406 pts/1    00:01:53 python
23408 pts/1    00:01:53 python
23410 pts/1    00:01:53 python
23411 pts/1    00:01:53 python
```

根据上面的输出，很容易想到通过`awk`来提取所有子进程pid
              
```shell
ps --ppid $1 | awk '{print $1}'
PID
23389
23390
23391
23392
23393
23394
23395
23396
23397
23400
23402
23404
23406
23408
23410
23411
```

发现有一行数据是`PID` 这个肯定是需要剔除掉的，再结合`xargs`就可以完成整个操作。
              
```shell
#!/bin/sh  
# kill process and child process  
ps --ppid $1 | awk '{if($1~/[0-9]+/) print $1}'| xargs kill -9  
kill -9 $1  
echo 'over'  
```





