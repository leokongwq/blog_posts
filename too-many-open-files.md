---
layout: post
comments: true
title: too-many-open-files
date: 2016-10-15 19:56:48
tags:
    - linux
    - java
categories:
    - java
---

### 故障总结：

> 不要被经验主义羁绊；生产环境的进程管理工具Supervisor等的原理和限制需要理解清楚。

昨天，项目的 ElasticSearch 服务挂了，我说的挂可不是进程没了，因为有 Supervisor 保护，而是服务不可用了。以前曾经出现过一次因为 ES_HEAP_SIZE 设置不当导致的服务不可用故障，于是我惯性的判断应该还是 ES_HEAP_SIZE 的问题，不过登录服务器后发现日志里显示大量的「Too many open files」错误信息。那么 ElasticSearch 设置的最大文件数到底是多少呢？可以通过 proc 确认：

    shell> cat /proc/<PID>/limits

结果是「4096」，我们还可以进一步看看 ElasticSearch 打开的都是什么东西：

    shell> ls /proc/<PID>/fd

<!-- more -->

问题看上去非常简单，只要加大相应的配置项应该就可以了。此配置在 ElasticSearch 里叫做 MAX_OPEN_FILES，可惜配置后发现无效。按我的经验，通常此类问题多半是由于操作系统限制所致，可是检查结果一切正常：

    shell> cat /etc/security/limits.conf

    * soft nofile 65535
    * hard nofile 65535

问题进入了死胡同，于是我开始尝试找一些奇技淫巧看看能不能先尽快缓解一下，我搜索到 @-神仙- 的一篇文章：动态修改运行中进程的 rlimit，里面介绍了如何动态修改阈值的方法，虽然我测试时都显示成功了，可惜 ElasticSearch 还是不能正常工作：

    shell> echo -n 'Max open files=65535:65535' > /proc/<PID>/limits

此外，我还检查了系统内核参数 fs.file-nr 及 fs.file-max，总之一切和文件有关的参数都查了，甚至在启动脚本里硬编码「ulimit -n 65535」，但一切努力都显得毫无意义。正当山穷水尽疑无路的时候，同事 @轩脉刃 一语道破玄机：关闭 Supervisor 的进程管理机制，改用手动方式启动 ElasticSearch 进程试试看。结果一切恢复正常。为什么会这样呢？因为使用 Supervisor 的进程管理机制，它会作为父进程 FORK 出子进程，也就是 ElasticSearch 进程，鉴于父子关系，子进程允许打开的最大文件数不能超过父进程的阈值限制，但是 Supervisor 中 minfds 指令缺省设置的允许打开的最大文件数过小，进而导致 ElasticSearch 进程出现故障。此故障原因本来非常简单，但我却陷入了经验主义的固定思维，值得反思。

转载自：[火丁](http://huoding.com/2015/08/02/460)
                    
                    
                    