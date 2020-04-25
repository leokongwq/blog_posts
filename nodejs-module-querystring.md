---
layout: post
comments: true
title: nodejs模块之querystring
date: 2016-10-17 11:25:55
tags:
- node
categories:
- nodejs
---

> 该模块提供了一些工具方法用来处理查询字符串，具体的方法如下所示：

### querystring.escape

解码查询字符串

### querystring.parse(str[, sep][, eq][, options])

将一个查询字符串转为一个JS对象。可以覆盖默认的分隔符`&`和`=`。

<!-- more -->

参数`Options`是一个JS对象，可以包含key：

 1. maxKeys (默认值是1000)，该值用来限制处理键值对的最大个数。如果`maxKeys`设为0，则没有限制。
 2. decodeURIComponent(默认值是:querystring.unescape)， 该key对应的值用来解码非utf-8编码的字符串。

```javascript
querystring.parse('foo=bar&baz=qux&baz=quux&corge')
// returns
{
    foo: 'bar',
    baz: ['qux', 'quux'],
    corge: ''
}
// Suppose gbkDecodeURIComponent function already exists,
// it can decode `gbk` encoding string
querystring.parse('w=%D6%D0%CE%C4&foo=bar', null, null, {
    decodeURIComponent: gbkDecodeURIComponent
})
//returns
{
    w: '中文',
    foo: 'bar'
}
```    

### querystring.stringify(obj[, sep][, eq][, options])

将一个对象转化为查询字符串。可以覆盖默认的键值分隔符('&')和等号('=')。

参数`Options`是一个JS对象，该对象可以包含一个名为`encodeURIComponent`(默认值是：querystring.escape),该key对应的值用来将非utf-8编码的字符串进行编码。

```javascript
querystring.stringify({ foo: 'bar', baz: ['qux', 'quux'], corge: '' })
// returns
'foo=bar&baz=qux&baz=quux&corge='

querystring.stringify({foo: 'bar', baz: 'qux'}, ';', ':')
// returns
'foo:bar;baz:qux'

// Suppose gbkEncodeURIComponent function already exists,
// it can encode string with `gbk` encoding
querystring.stringify({ w: '中文', foo: 'bar' }, null, null,
  { encodeURIComponent: gbkEncodeURIComponent })
// returns
'w=%D6%D0%CE%C4&foo=bar'
```
                        
### querystring.unescape

解码查询字符串                    
                    
                    
                    
                    
                    
                    
                    
                   