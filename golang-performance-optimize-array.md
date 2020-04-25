---
layout: post
comments: true
title: Go性能优化技巧-array
date: 2016-10-15 19:55:45
tags:
- go
categories:
- golang
---

### 前言  

> 对于一些初学者，自知道 Go 里面的 array 以 pass-by-value 方式传递后，就莫名地引起“恐慌”。外加诸多文章未作说明，就建议用 slice 代替 array，企图避免数据拷贝，提升性能。实际上，此做法有待商榷。某些时候怕会适得其反，倒造成不必要的性能损失。

<!-- more -->

### 结论：

对于一些短小的对象，复制成本远小于在堆上分配和回收操作。大的对象还是通过指针来传递吧。但是这个大小究竟是多大呢？恐怕需要反复测试，而且不同的运行状态下也不一定相同。很难抉择吧？

原文地址：[http://www.jianshu.com/p/f5aada70e9a6](http://www.jianshu.com/p/f5aada70e9a6)

                            
                    
                    
                    