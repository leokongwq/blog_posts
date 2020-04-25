---
layout: post
comments: true
title: Go性能优化技巧-string
date: 2016-10-15 19:55:52
tags:
categories:
---

 ### 概述   

> 字符串（string）作为一种不可变类型，在与字节数组（slice, [ ]byte）转换时需付出 “沉重” 代价，根本原因是对底层字节数组的复制。这种代价会在以万为单位的高并发压力下迅速放大，所以对它的优化常变成 “必须” 行为。

首先，须了解 string 和 [ ]byte 数据结构，并确认默认方式的复制行为。 

<!-- more -->

```golang
    pacakge main 
    
    import (
        "fmt"
    )
    
    func main() {
        s := "Hello World"
        b := []byte(s)
        fmt.Println(s, b)
    }
```

编译并调试：
    
    go build -gcflags "-N -l" -o test main.go
    sudo gdb test
    l main.main
    b 11
    r
    info locals
    ptype s //输出
    type = struct string {
        uint8 *str;
        int len;
    }
    ptype b //输出
    type = struct []uint8 {
        uint8 *array;
        int len
        int cap
    }
    
    x/2xg &s //输出
    
    x/3xg &b //输出
    
    
从 GDB 输出结果可看出，转换后 [ ]byte 底层数组与原 string 内部指针并不相同，以此可确定数据被复制。那么，如不修改数据，仅转换类型，是否可避开复制，从而提升性能？

从 ptype 输出的结构来看，string 可看做 [2]uintptr，而 [ ]byte 则是 [3]uintptr，这便于我们编写代码，无需额外定义结构类型。如此，str2bytes 只需构建 [3]uintptr{ptr, len, len}，而 bytes2str 更简单，直接转换指针类型，忽略掉 cap 即可。

优化代码：

```golang
    func StrToBytes(s string) []byte {
	    x := (*[2]uintptr)(unsafe.Pointer(&s))
	    h := [3]uintptr{x[0], x[1], x[1]}
	    return *(*[]byte)(unsafe.Pointer(&h))
    }

    func BytesToStr(b []byte) string {
	    return *(*string)(unsafe.Pointer(&b))
    }
```

用 unsafe 完成指针类型转换，所以得自行为底层数组生命周期做出保证。好在这两个函数都很简单，编译器会完成内联处理，并未发生逃逸行为。

    -> % go build -gcflags "-m" -o test main.go
    # command-line-arguments
    ./main.go:14: can inline StrToBytes
    ./main.go:20: can inline BytesToStr
    ./main.go:11: s escapes to heap
    ./main.go:11: b escapes to heap
    ./main.go:10: ([]byte)(s) escapes to heap
    ./main.go:11: main ... argument does not escape
    ./main.go:14: StrToBytes s does not escape
    ./main.go:15: StrToBytes &s does not escape
    ./main.go:17: StrToBytes &h does not escape
    ./main.go:20: leaking param: b to result ~r1 level=0
    ./main.go:21: BytesToStr &b does not escape

性能测试：

```golang
    package main

    import (
    	"testing"
    	"strings"
    )
    
    var s = strings.Repeat("s", 1024)
    
    func test()  {
    	b := []byte(s)
    	_ = string(b)
    }
    
    func test2()  {
    	b := StrToBytes(s)
    	_ = BytesToStr(b)
    }
    
    func BenchmarkTest(b *testing.B)  {
    	for i := 0; i < b.N; i++ {
    		test()
    	}
    }
    
    func BenchmarkTestBlock(b *testing.B)  {
    	for i := 0; i < b.N; i++ {
    		test2()
    	}
    }
```
   
性能测试结果：
    
    go test -v -bench . -benchmem
    testing: warning: no tests to run
    PASS
    BenchmarkTest-4     	 2000000	       606 ns/op	    2048 B/op	       2 allocs/op
    BenchmarkTestBlock-4	300000000	         5.55 ns/op	       0 B/op	       0 allocs/op

观察上面的测试数据，性能提升明显，最关键的是 zero-garbage。    
    
原文地址：[http://www.jianshu.com/p/0b8387057050](http://www.jianshu.com/p/0b8387057050)                        
                    
                    
                    
                    