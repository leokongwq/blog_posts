---
layout: post
comments: true
title: linux工具-strace
date: 2016-10-15 19:56:39
tags:
- go
categories:
- golang
---

> truss和strace都是用来跟踪一个进程的系统调用或信号产生的情况，而 ltrace用来 跟踪进程调用库函数的情况。

### strace用法

    usage: strace [-dDffhiqrtttTvVxx] [-a column] [-e expr] ... [-o file]
              [-p pid] ... [-s strsize] [-u username] [-E var=val] ...
              [command [arg ...]]
       or: strace -c [-D] [-e expr] ... [-O overhead] [-S sortby] [-E var=val] ...
              [command [arg ...]]                        

<!-- more -->

### strace 选项

- -c : 统计每个系统调用的时间，次数，错误并生成报告
- -f : 跟踪每个子进程
- -T : 输出每次系统调用的时间消耗
- -0 : 指定结果输出的文件路径
- -p : 指定需要跟踪的进程pid
- -u : 以指定的用户运行该命令
- -E : var = val , 设置命令执行环境中的键值对数据

### strace 实践用法

1.查询进程打开的配置文件

    strace -o php.trace php 2>&1
    grep php.ini php.trace
    //输出
    open("/usr/bin/php.ini", O_RDONLY)      = -1 ENOENT (No such file or directory)
    open("/etc/php.ini", O_RDONLY)          = 3

2.跟踪指定的系统调用
    
    strace -e open -o php.strace php 2>&1
    //文件内容(前3行)
    open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
    open("/lib64/libcrypt.so.1", O_RDONLY|O_CLOEXEC) = 3
    open("/lib64/libresolv.so.2", O_RDONLY|O_CLOEXEC) = 3

3.跟踪进程
strace不但能用在命令上，而且通过使用-p选项能用在运行的进程上

    sudo strace -f -p 21175
    //输出
    select(0, NULL, NULL, NULL, {0, 541763}) = 0 (Timeout)
    wait4(-1, 0x7ffffd8082cc, WNOHANG|WSTOPPED, NULL) = 0
    select(0, NULL, NULL, NULL, {1, 0})     = 0 (Timeout)
    wait4(-1, 0x7ffffd8082cc, WNOHANG|WSTOPPED, NULL) = 0
    select(0, NULL, NULL, NULL, {1, 0})     = 0 (Timeout)
    
4.strace的统计概要

    sudo strace -cf -p 21175
    //输出
    % time     seconds  usecs/call     calls    errors syscall
    ------ ----------- ----------- --------- --------- ----------------
     71.59    0.000126          16         8           select
     28.41    0.000050          50         1           sendmsg
      0.00    0.000000           0         1           close
      0.00    0.000000           0         1           socket
      0.00    0.000000           0         8           wait4
    ------ ----------- ----------- --------- --------- ----------------
    100.00    0.000176                    19           total

通过`-c`选项可以很容易的看出系统调用次数最多的系统调用和耗时。对排查线上问题很有帮助。

5.保存输出结果
通过使用-o选项可以把strace命令的输出结果保存到一个文件中

    sudo strace -f -o httpd.trace  -p 21175

6.显示时间戳

使用-t选项，可以在每行的输出之前添加时间戳。

    sudo strace -ft -o httpd.trace  -p 21175
    //输出
    21175 18:13:25 select(0, NULL, NULL, NULL, {0, 706785}) = 0 (Timeout)
    21175 18:13:26 wait4(-1, 0x7ffffd8082cc, WNOHANG|WSTOPPED, NULL) = 0
    21175 18:13:26 select(0, NULL, NULL, NULL, {1, 0}) = 0 (Timeout)

7.更精细的时间戳

-tt选项可以展示微秒级别的时间戳。
 
    21175 18:14:59.822442 select(0, NULL, NULL, NULL, {0, 422755}) = 0 (Timeout)
    21175 18:15:00.246102 wait4(-1, 0x7ffffd8082cc, WNOHANG|WSTOPPED, NULL) = 0
    21175 18:15:00.246209 select(0, NULL, NULL, NULL, {1, 0}) = 0 (Timeout)

8. 相对时间

-r选项展示系统调用之间的相对时间戳

    21175      0.000000 select(0, NULL, NULL, NULL, {0, 75340}) = 0 (Timeout)
    21175      0.075613 wait4(-1, 0x7ffffd8082cc, WNOHANG|WSTOPPED, NULL) = 0
    21175      0.000056 select(0, NULL, NULL, NULL, {1, 0}) = 0 (Timeout)

9. 耗时

`-T` 能输出每次系统调用的耗时

    21175 select(0, NULL, NULL, NULL, {0, 511764}) = 0 (Timeout) <0.512210>
    21175 wait4(-1, 0x7ffffd8082cc, WNOHANG|WSTOPPED, NULL) = 0 <0.000017>
    21175 select(0, NULL, NULL, NULL, {1, 0}) = 0 (Timeout) <1.001134>
    
                    