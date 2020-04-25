---
layout: post
comments: true
title: Go性能优化技巧-map
date: 2016-10-15 19:55:36
tags:
- go
categories:
- golang
---

### 前言  

> golang内置 map类型是必须的。首先，该类型使用频率很高；其次，可借助 runtime 实现深层次优化（比如说字符串转换，以及 GC 扫描等）。可尽管如此，也不意味着万事大吉，依旧有很多需特别注意的地方。

<!-- more -->

### 1.预设容量

> 这个比较容易理解，map是基于hash算法实现的，通过计算key的hash值来分布和查找对象，如果key的hash值相同的话，一般会通过拉链法解决冲突（Java）。如果容量太小，冲突就比较严重。数据查询速度难免降低；如果需要提供数据查询速度，需要以空间换时间，加大容量。如果初始容量太小，而你需要存入大量的数据，一定就会发生数据复制和rehash（很有可能发生多次，go map 的负载因子是：6.5 // Maximum average load of a bucket that triggers growth.loadFactor =6.5）。所以预估容量就比较重要了。既能减少空间浪费，同时能避免运行时多次内存复制和rehash。

**性能测试**

```golang
    package main
    
    import (
    	"testing"
    )
    
    func test(m map[int]int){
    	for i :=0; i < 10000; i++ {
    		m[i] = i;
    	}
    }
    
    func BenchmarkMap(b *testing.B)  {
    	for i := 0; i < b.N; i++ {
    		b.StopTimer()
    		m := make(map[int]int)
    		b.StartTimer()
    		test(m)
    	}
    }
    
    func BenchmarkMapCap(b *testing.B)  {
    	for i := 0; i < b.N; i++ {
    		b.StopTimer()
    		m := make(map[int]int, 10000)
    		b.StartTimer()
    		test(m)
    	}
    }
    
    BenchmarkMap-4   	    1000	   1356876 ns/op	  686640 B/op	     626 allocs/op
    BenchmarkMapCap-4	    2000	    677001 ns/op	   20664 B/op	     134 allocs/op
```

### 2.直接存储

> 对于小对象，直接将数据交由 map 保存，远比用指针高效。这不但减少了堆内存分配，关键还在于垃圾回收器不会扫描非指针类型 key/value 对象。

**代码：**

```golang
    package main
    
    import (
    	"runtime"
    	"time"
    )
    
    const capacity int   = 50000
    
    var d interface{}
    
    func value() map[int]int {
    	m := make(map[int]int, capacity)
    	for i := 0; i < capacity; i++ {
    		m[i] = i
    	}
    	return m
    }
    
    func pointer() map[int]*int  {
    	m := make(map[int]*int, capacity)
    	for i := 0; i < capacity; i++ {
    		v := i
    		m[i] = &v
    	}
    	return m
    }
    
    func main()  {
    	//d = value()
    	d = pointer()
    
    	for i := 0; i < 20; i++  {
    		runtime.GC()
    	}
    	time.Sleep(time.Second)
    }
```

运行结果：

    // pointer 
    -> % GODEBUG="gctrace=1" ./test
    gc 1 @0.008s 17%: 0.006+0+0+0+2.5 ms clock, 0.019+0+0+0/0/0+7.7 ms cpu, 2->2->2 MB, 2 MB goal, 4 P (forced)
    gc 2 @0.011s 34%: 0.007+0+0+0+3.1 ms clock, 0.028+0+0+0/0/0+12 ms cpu, 2->2->2 MB, 2 MB goal, 4 P (forced)
    gc 3 @0.014s 40%: 0.003+0+0+0+2.6 ms clock, 0.011+0+0+0/0/0+7.8 ms cpu, 2->2->2 MB, 2 MB goal, 4 P (forced)
    gc 4 @0.017s 48%: 0.001+0+0+0+2.7 ms clock, 0.007+0+0+0/0/0+10 ms cpu, 2->2->2 MB, 2 MB goal, 4 P (forced)
    gc 5 @0.020s 54%: 0.001+0+0+0+2.4 ms clock, 0.005+0+0+0/0/0+9.8 ms cpu, 2->2->2 MB, 2 MB goal, 4 P (forced)

    // value
    gc 1 @0.007s 3%: 0.009+0+0+0+0.32 ms clock, 0.027+0+0+0/0/0+0.97 ms cpu, 1->1->1 MB, 1 MB goal, 4 P (forced)
    gc 2 @0.008s 5%: 0.001+0+0+0+0.24 ms clock, 0.007+0+0+0/0/0+0.98 ms cpu, 1->1->1 MB, 1 MB goal, 4 P (forced)
    gc 3 @0.008s 7%: 0.002+0+0+0+0.14 ms clock, 0.008+0+0+0/0/0+0.59 ms cpu, 1->1->1 MB, 1 MB goal, 4 P (forced)
    gc 4 @0.008s 8%: 0.001+0+0+0+0.15 ms clock, 0.006+0+0+0/0/0+0.63 ms cpu, 1->1->1 MB, 1 MB goal, 4 P (forced)
    gc 5 @0.008s 10%: 0.001+0+0+0+0.15 ms clock, 0.005+0+0+0/0/0+0.63 ms cpu, 1->1->1 MB, 1 MB goal, 4 P (forced)

从两次输出里 GC 所占时间百分比，就可看出 “巨大” 差异。
 
### 3.空间收缩

很遗憾，map 不会收缩 “不再使用” 的空间。就算把所有键值删除，它依然保留内存空间以待后用。
如果一个非常大的map里面的元素很少的话，可以考虑新建一个map将老的map元素手动复制到新的map中。

原文参考：[http://www.jianshu.com/p/c34e3a787de4](http://www.jianshu.com/p/c34e3a787de4)

                        
                    
                    