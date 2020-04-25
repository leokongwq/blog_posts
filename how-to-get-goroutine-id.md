---
layout: post
comments: true
title: 如何获取goroutine的id
date: 2016-10-15 19:22:09
tags:
- go
categories:
- golang
---

>  使用Java的时候很容易得到线程的名字，比如"Thread.currentThread().getName"，这样就可以进行一些监控操作或者设置线程相关的一些数据。当转向Golang开发的时候，却发现Go语言并没有提供获取当前goroutine id的操作。这是Golang的开发者故意为之，避免开发者滥用goroutine id实现goroutine local storage (类似java的"thread-local" storage)， 因为goroutine local storage很难进行垃圾回收。因此尽管以前暴露出了相应的方法，现在已经把它隐藏了。当然Go的这种隐藏的做法还是有争议的，有点因噎废食。在debug log的时候goroutine id是很好的一个监控信息。本文介绍了两种获取goroutine id的方法。

<!-- more -->

### 1、C代码扩展
先前的一种方式是使用C语言暴露出获取goroutine方法,如

```c
    #include "runtime.h"
    int64 ·Id(void) {
    	return g->goid;
    }
    package id
    // Id returns the id of the current goroutine.
    // If you call this function you will go straight to hell.
    func Id() int64
```
    
但是Go 1.4已经不支持在包中包含C代码 (go1.4#gocmd)。

滴滴打车的huandu贡献了一个开源的项目goroutine,可以在新的Go版本下获取goroutine的id (至少go 1.5.1, go 1.6下可用):
首先获取这个项目的最新版本

go get -u github.com/huandu/goroutine

然后你就可以在代码中调用相应的方法了：

```golang
    // Get id of current goroutine.
    var id int64 = goroutine.GoroutineId()
    println(id)
```
    
实际使用中 goroutine.GoroutineId()并不能正确的获得当前的goroutine id, 似乎有些问题

2、第二种方法
还有一种花招，可以用纯Go语言获取当前的goroutine id。代码如下所示：

```golang
    package main
    import (
    	"fmt"
    	"runtime"
    	"strconv"
    	"strings"
    	"sync"
    )
    
    func GoID() int {
    	var buf [64]byte
    	n := runtime.Stack(buf[:], false)
    	idField := strings.Fields(strings.TrimPrefix(string(buf[:n]), "goroutine "))[0]
    	id, err := strconv.Atoi(idField)
    	if err != nil {
    		panic(fmt.Sprintf("cannot get goroutine id: %v", err))
    	}
    	return id
    }
    func main() {
    	fmt.Println("main", GoID())
    	var wg sync.WaitGroup
    	for i := 0; i < 10; i++ {
    		i := i
    		wg.Add(1)
    		go func() {
    			defer wg.Done()
    			fmt.Println(i, GoID())
    		}()
    	}
    	wg.Wait()
    }
```
    
go run main.go输出：

    main 1
    9 14
    0 5
    1 6
    2 7
    5 10
    6 11
    3 8
    7 12
    4 9
    8 13
    
它利用runtime.Stack的堆栈信息。runtime.Stack(buf []byte, all bool) int会将当前的堆栈信息写入到一个slice中，堆栈的第一行为goroutine #### […,其中####就是当前的gororutine id,通过这个花招就实现GoID方法了。

但是需要注意的是，获取堆栈信息会影响性能，所以建议你在debug的时候才用它。                        
                    

转自：[http://colobu.com/2016/04/01/how-to-get-goroutine-id/](http://colobu.com/2016/04/01/how-to-get-goroutine-id/)                    
                    