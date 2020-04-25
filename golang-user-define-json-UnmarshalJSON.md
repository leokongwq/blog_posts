---
layout: post
comments: true
title: golang进行json数据的解析
date: 2016-10-12 18:00:34
tags:
    - golang
    - json
categories:
    - golang
    
---

> json作为非常流行的数据交换格式,每个语言都有自己的解析工具, golang作为互联网时代的C语言当然也有自己的工具包,本篇文章就介绍下如何在golang中解析json数据,以及如何将结构体序列化为json字符串.

<!-- more -->

### golang进行json数据的解析 

{% codeblock lang:go%}
    package main
    import (
        "encoding/json"
        "fmt"
        "log"
    )
    type IntBool bool
    func (p *IntBool) UnmarshalJSON(b []byte) error {
        str := string(b)
        switch str {
        case "1":
            *p = true
        case "0":
            *p = false
        default:
            return fmt.Errorf("unexpected bool: %s", str)
        }
        return nil
    }
    func main() {
        var account struct {
            Gender IntBool `json:"gender"`
        }
        err := json.Unmarshal([]byte(`{"gender":1}`), &account)
        if err != nil {
            log.Fatal(err)
        }
        log.Println(account.Gender == true)
    }                        
{% endcodeblock %}

具体原理需要参考包:encoding/json 中关于Unmarshaler接口的描述信息
转自:[http://www.golangtc.com/t/5791e275b09ecc76e300000b](http://www.golangtc.com/t/5791e275b09ecc76e300000b)                    
                    
                    