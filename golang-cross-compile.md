---
layout: post
comments: true
title: golang交叉编译
date: 2016-10-15 19:56:02
tags:
- go
categories:
- golang
---

### 前言

> 我的开发环境是Mac，已经安装了go1.5; 测试和生产环境都是CentOS; 则需要在Mac下编译可以在CentOS上运行的Binary文件；下面是处理办法：

<!-- more -->

**1. 先进入Go的源代码目录中**

如果Go是源码编译安装的，则该目录可能是：/usr/local/go/src;
如果Go是通过Homebrew安装的，则该目录是：/usr/local/Cellar/go/1.5/libexec/src;

    cd /usr/local/go/src

**2. 配置好编译环境**
    
    源码安装go开发环境后，只有对应平台的编译工具链，这些工具在：
    
    cd /usr/local/go/pkg/tool && ls -l
    
    drwxr-xr-x  19 root  wheel  646  5 27 13:21 darwin_amd64 (我本机)
    drwxr-xr-x  19 root  wheel  646  5 27 13:21 linux_amd64 (生成的对应平台交叉编译环境)

    #如果配置Linux平台下编译环境请执行：
    $ CGO_ENABLED=0 GOOS=linux GOARCH=amd64 ./make.bash; 

    #如果配置Windows平台下编译环境请执行：
    $ CGO_ENABLED=0 GOOS=windows GOARCH=amd64 ./make.bash;

    如果是32位，则把amd64改为386即可

**3. 回到项目所在目录，开始编译**

    #linux
    $ GOROOT_BOOTSTRAP=/usr/local/go1.4 CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build 您的项目名称;

    #windows
    $ GOROOT_BOOTSTRAP=/usr/local/go1.4 CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build 您的项目名称;


**4.总结**

 - `CGO_ENABLED=0` 禁用CGO，交叉编译不支持CGO
 
 - go1.5版本的编译器是用go编写的。如果要生产其它平台的编译工具链，需要<=go1.4的go开发环境（我又装了go1.4环境）

 - GOOS 指定了目标平台的操作系统类型

 - GOARCH 指定了目标平台的架构

                    
                    