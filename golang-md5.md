---
layout: post
comments: true
title: golang计算字符串MD5
date: 2016-10-13 18:00:52
tags:
    - golang
categories:
    - golang

---

#### golang计算字符串MD5

```golang
    package main
    
    import (
        "crypto/md5"
        "fmt"
    )
    
    func main() {
        data := []byte("hello")
        fmt.Printf("%x", md5.Sum(data))
    }                        
```
                    
                    