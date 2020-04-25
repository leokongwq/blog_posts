---
layout: post
comments: true
title: golang中import语句详解
date: 2016-10-16 09:21:55
tags:
- go
categories:
- golang
---

### go语言import语句的各种用法

### 1.导入系统包或第三方的软件包

    import "fmt"
    import "github.com/astaxie/beego"    

### 2.使用import语句调用导入包的init函数。

这个第一次在beego框架中见到。

    _ "github.com/go-sql-driver/mysql"

<!-- more -->

### import 的路径问题

    import "./controller" // 相对路径 不推荐
    import "models/User" // 相对gopath src目录下 推荐

### 点操作

    import( . “fmt” ) 

这个点操作的含义就是这个包导入之后在你调用这个包的函数时，你可以省略前缀的包名，也就是前面你调用的fmt.Println(“hello world”)  可以省略的写成Println(“hello world”)

### 别名操作

    import( f “fmt” ) 

别名操作顾名思义可以把包命名成另一个用起来容易记忆的名字

### 包的导入过程说明

程序的初始化和执行都起始于main包。如果main包还导入了其它的包，那么就会在编译时将它们依次导入。有时一个包会被多个包同时导入，那么它只会被导入一次（例如很多包可能都会用到fmt包，但它只会被导入一次，因为没有必要导入多次）。当一个包被导入时，如果该包还导入了其它的包，那么会先将其它包导入进来，然后再对这些包中的包级常量和变量进行初始化，接着执行init函数（如果有的话），依次类推。等所有被导入的包都加载完毕了，就会开始对main包中的包级常量和变量进行初始化，然后执行main包中的init函数（如果存在的话），最后执行main函数。下图详细地解释了整个执行过程：

![go-pkg-import](http://www.jiexiu.com/images/posts/go-import.jpeg)







                    
                
                
                