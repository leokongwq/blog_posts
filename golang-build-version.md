---
layout: post
comments: true
title: golang自动生成版本信息
date: 2016-10-15 19:22:19
tags:
- go
categories:
---

> 如果需要知道使用的软件的版本信息,我们使用`cmd --version`或者`cmd -v`就可以查看当前的版本信息了.我们用GO开发一个工具,如果要提供工具的版本信息,最先能想到的方法就是读取命令行参数.那有没有更好的办法的呢?答案肯定是了.下面介绍的方式是以前没有想到的.

<!-- more -->

### 需求

golang程序在build时自动生成版本信息，使用 ./helloworld –version可以查看版本和build时间

### 实现原理

使用链接选项-X设置一个二进制文件中可以访问的变量                    

例子1：

```golang
    package main
    import "fmt"
     
    var Version = "No Version Provided"
     
    func main() {
        fmt.Println("HelloWorld Version is:", Version)
    }
```

**测试输出1:**

```
    go run -ldflags "-X main.Version 1.5" helloworld.go
     
    HelloWorld Version is: 1.5
```
    
例子2：

```golang
    package main
    import \"fmt\"
     
    var buildstamp = \"no timestamp set\"
    var githash = \"no githash set\"
     
    func main() {
        fmt.Println(\"HelloWorld buildstamp is:\", buildstamp)
        fmt.Println(\"HelloWorld buildgithash is:\", githash)
    }
```

**测试输出2:**

```
    go build -ldflags \"-X main.buildstamp `date \'+%Y-%m-%d_%I:%M:%S\'` -X main.githash `git rev-parse HEAD`\" helloworld.go
     
    ./helloworld
     
    HelloWorld buildstamp is: 2015-09-08_05:58:49
    HelloWorld buildgithash is: 1adb00d88d832687eb4148a3871829fb73021c29
```

原文： [Golang自动生成版本信息](http://www.trueeyu.com/2015/09/18/golang-version/)
    


    
                    