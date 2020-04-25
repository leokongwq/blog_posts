---
layout: post
comments: true
title: linux中最大文件描述符数
date: 2016-11-09 18:49:47
tags:
- OS
categories:
- linux
---

### 前言

> 关于Linux下系统,进程能最大能打开的文件描述符数看过好多文章,但大都没有完整,详细说明每个值表示什么意思,在实践中该怎么设置.今天刚好有时间就通过Google来整理了如下的内容.如有错误请指出,谢谢.

### 系统级别
Linux系统级别限制所有用户进程能打开的文件描述符总数可以通过如下的命令查看

    $ cat /proc/sys/fs/file-max 
    2259544

<!-- more -->

有2中方法修改系统级别的限制：
1. 通过命令动态修改(重启后失效)

    sysctl -w fs.file-max=102400
    
2.通过配置文件修改
    
    vi /etc/sysctl.conf
    在文件末尾添加
    fs.file-max=102400
    保存退出后使用sysctl -p 命令使其生效

和`fs.file-max`有关的一个参数是`file-nr`, 该参数是只读的

    $ cat /proc/sys/fs/file-nr 
    3296    0       2259544    

`file-nr`的值由3部分组成：1，已经分配的文件描述符数；2，已经分配但未使用的文件描述符数；
3，内核最大能分配的文件描述符数

**注意：** 只要你的内存足够大，file-max的值可以非常大。

### 用户级别

用户级别的限制是通过可以通过命令`ulimit`命令和文件`/etc/security/limits.conf`

    $ ulimit  -n
    655350
    //查看硬件资源限制
    $ ulimit  -Hn
    655350
    //软件资源限制
    $ ulimit  -Sn
    655350
    //设置软/硬件资源限制
    ulimit  -Sn 655350 或 ulimit  -Hn 655350
    
查看limits.conf文件

    cat /etc/security/limits.conf
    //输出
    * hard nofile 655350
    * soft nofile 655350    

limits.conf 文件的格式是：

    <domain> <type> <item> <value>

每个域的取值可以[参考](https://linux.die.net/man/5/limits.conf)

如果domain的值是一个用户名，则可以限制该用户下的所有进程能打开的文件描述符总数，如果是`*`则表示针对每个用户都起作用

**注意** 针对同一个item取值， soft的值不能大于hard

### nr_open

> This denotes the maximum number of file-handles a process can 
allocate. Default value is 1024*1024 (1048576) which should be 
enough for most machines. Actual limit depends on RLIMIT_NOFILE 
resource limit.

就是说nr_open表示一个进程做多能分配的文件句柄数，默认值是1048576。针对大多数的情况该值是足够的。

### NR_FILE

> NR_FILE is the limit on total number of files in the system at any given point in time

NR_FILE 是系统在某一给定时刻，限制的文件总数

> While initializing the kernel we setup the vfs cache with start_kernel
vfs_caches_init(num_physpages);
files_init(mempages);
fs/file_table.c says
/* One file with associated inode and dcache is very roughly 1K.
* Per default don't use more than 10% of our memory for files.
n = (mempages * (PAGE_SIZE / 1024)) / 10;
this n can never be greater than NR_FILE   

### ulimit 命令

1.只对当前tty（终端有效），若要每次都生效的话，可以把ulimit参数放到对应用户的.bash_profile里面；
2.ulimit命令本身就有分软硬设置，加-H就是硬，加-S就是软；
3.默认显示的是软限制，如果运行ulimit命令修改的时候没有加上的话，就是两个参数一起改变.生效；

命令参数
-H 设置硬件资源限制.
-S 设置软件资源限制.
-a 显示当前所有的资源限制.
-c size:设置core文件的最大值.单位:blocks
-d size:设置数据段的最大值.单位:kbytes
-f size:设置创建文件的最大值.单位:blocks
-l size:设置在内存中锁定进程的最大值.单位:kbytes
-m size:设置可以使用的常驻内存的最大值.单位:kbytes
-n size:设置内核可以同时打开的文件描述符的最大值.单位:n
-p size:设置管道缓冲区的最大值.单位:kbytes
-s size:设置堆栈的最大值.单位:kbytes
-t size:设置CPU使用时间的最大上限.单位:seconds
-v size:设置虚拟内存的最大值.单位:kbytes
unlimited 是一个特殊值，用于表示不限制


### 总结 file-max, nr_open, nofile之间的关系

1. 针对用户打开最大文件数的限制，可以通过修改文件`limits.conf`来实现
2. nofile中soft的值小于hard, 最大值由nr_open来决定
3. file-max表示内核针对整个系统，限制能所有进程能打开的文件描述符数
4. nofile < nr_open < file-max

### 参考
[https://linux.die.net/man/5/limits.conf](https://linux.die.net/man/5/limits.conf)
[https://www.kernel.org/doc/Documentation/sysctl/fs.txt](https://www.kernel.org/doc/Documentation/sysctl/fs.txt)
[http://blog.chinaunix.net/uid-24807808-id-3077199.html](http://blog.chinaunix.net/uid-24807808-id-3077199.html)
[http://blog.csdn.net/gatieme/article/details/51058797](http://blog.csdn.net/gatieme/article/details/51058797)
[http://www.cyberciti.biz/faq/linux-increase-the-maximum-number-of-open-files/](http://www.cyberciti.biz/faq/linux-increase-the-maximum-number-of-open-files/)


    


    
    
        
    
  