---
layout: post
comments: true
title: golang 用户信息获取
date: 2016-10-16 09:12:01
tags:
- go
categories:
- golang
---

### golang 用户信息获取

### 获取当前用户

```golang
    import "os/user"
    
    func main(){
        crtUser, err := user.Current()
    }
    crtUser 的类型是：*User, 具体如下：
    type User struct {
    	Uid      string // user id
    	Gid      string // primary group id
    	Username string
    	Name     string
    	HomeDir  string
    }
```
<!-- more -->

### 查询用户

```golang
   import "os/user"
    
    func main(){
        usr1, err1 := user.Lookup(userName) //通过用户名查询
        usr2, err2 := user.LookupId(123) //通过用户id查询
    }
```    
                            
                    
                    
                    