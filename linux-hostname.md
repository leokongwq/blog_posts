---
layout: post
comments: true
title: linux命令hostname详解
date: 2016-10-17 10:28:45
tags:
- linux
- OS
categories:
- linux
---

查看Linux主机名对使用过Linux系统的开发人员来说是`so easy`的，设置主机名也很方便,通过下面的命令：

    hostname 主机名

就可以设置主机名了。但是我们可以很容易的想到，通常这种方式设置的结果就是主机重启以后就恢复原来的值；
那我们马上就可以想到，将主机名写入系统配置文件。那这个文件在哪里呢？

<!-- more -->

    /etc/sysconfig/network

这个就是配置文件，但是我打开该文件查看里面的内容发现：

    NETWORKING=yes
    HOSTNAME=localhost.localdomain

但是我通过命令`hostname`查看到的主机名是`walle128131`。这是为什么呢？上面的文件不是主机名的配置文件？还是配置文件就是这个，只是后期通过命令修改了主机名，但是配置文件没有更新？ 带着这些问题，Google之。结论如下：

1. hostname 的值是Linux的一个内核参数，它保存在`/proc/sys/kernel/hostname`下。
 
2. 该值是在系统启动时从`rc.sysinit`读取的。

3. 而`/etc/rc.d/rc.sysinit`中hostname的值是根据文件`/etc/sysconfig/network`中的配置值而来的。`/etc/rc.d/rc.sysinit`中的部分代码如下：

        HOSTNAME=$(/bin/hostname)

        set -m

        if [ -f /etc/sysconfig/network ]; then
            . /etc/sysconfig/network
        fi
        if [ -z "$HOSTNAME" -o "$HOSTNAME" = "(none)" ]; then
            HOSTNAME=localhost
        fi

### 结论

文件`/etc/sysconfig/network`确实是hostname的值的配置文件。但是hostname的值在系统启动后可以通过`hostname`命令修改。我们平时通过`hostname`命令查看到的主机名值是文件`/proc/sys/kernel/hostname` 中的值。

                        
                    
                    
                    
                    
                    