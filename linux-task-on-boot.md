---
layout: post
comments: true
title: linux开机启动任务
date: 2017-06-09 16:10:13
tags:
- linux
categories:
- linux
---

### 背景

有各种各样的需要导致需要在开机执行一些任务，例如：当虚拟机重启时启动应用或某些服务。
此时就需要用来Linux的启动任务机制了。

<!-- more -->

### Linux运行级别

Linux总共有7中运行级别[0 - 6],每种级别的功能说明如下：

- 0 halt (Do NOT set initdefault to this)
- 1 Single user mode
- 2 Multi user, without NFS(The same as 3,if you do not have net working)
- 3 Full multi user mode
- 4 unused
- 5 X11
- 6 reboot(Do NOT set initdefault to this)

我们最常设置的级别是3或者5，服务器都是3。可以通过文件`/etc/inittab`来查看或修改运行级别。

如果你的系统使用的是`systemd`，则可以通过命令:`systemctl get-default`来获取当前的运行级别。

### 启动脚本

每一种启动级别都有对应的脚本文件，系统的启动的过程中会执行这些脚本。这些文件位于目录`/etc/rc.d`下：

```
drwxr-xr-x. 2 root root 102 Jun  1 10:54 init.d
drwxr-xr-x. 2 root root  58 Apr 25 16:41 rc0.d
drwxr-xr-x. 2 root root  58 Apr 25 16:41 rc1.d
drwxr-xr-x. 2 root root  85 Apr 25 16:41 rc2.d
drwxr-xr-x. 2 root root  85 Apr 25 16:41 rc3.d
drwxr-xr-x. 2 root root  85 Apr 25 16:41 rc4.d
drwxr-xr-x. 2 root root  85 Apr 25 16:41 rc5.d
drwxr-xr-x. 2 root root  58 Apr 25 16:41 rc6.d
-rw-r--r--. 1 root root 637 Apr 19 09:44 rc.local
```

`rc[0-6].d`文件夹下。

**注意**：`/etc/rc[0~6].d`其实是`/etc/rc.d/rc[0~6].d`的软连接，主要是为了保持和Unix的兼容性才做此策。

这7个目录中，每个目录分别存放着对应运行级别加载时需要关闭或启动的服务

由详细信息可以知道，其实每个脚本文件都对应着`/etc/init.d/`目录下具体的服务

`K`开头的脚本文件代表运行级别加载时需要关闭的，`S`开头的代表需要执行

因此，当我们需要开机执行自己的脚本时，只需要将可执行脚本丢在`/etc/init.d`目录下，然后在`/etc/rc.d/rc*.d`中建立软链接即可。

```shell
[root@localhost ~]# ln -s /etc/init.d/sshd /etc/rc.d/rc3.d/S100ssh
```

此处sshd是具体服务的脚本文件，S100ssh是其软链接，S开头代表加载时自启动

如果需要在多个运行级别下设置自启动，则需建立多个软链接

这种方式比较繁琐，适用于自定义的服务脚本

如果系统中已经存在某些服务（比如安装apache时就会有httpd服务项），可以使用下面的两种方式

### chkconfig

如果需要自启动某些服务，只需使用`chkconfig 服务名 on`即可，若想关闭，将`on`改为`off`

在默认情况下，`chkconfig`会自启动2345这四个级别，如果想自定义可以加上`--level`选项

Tips：`--list` 选项可查看指定服务的启动状态，chkconfig不带任何选项则查看所有服务状态

跟多信息参考:[http://sdgdfgdsfgfg.iteye.com/blog/1159906](http://sdgdfgdsfgfg.iteye.com/blog/1159906)

### 参考

[http://www.cnblogs.com/nerxious/archive/2013/01/18/2866548.html](http://www.cnblogs.com/nerxious/archive/2013/01/18/2866548.html)

