---
layout: post
comments: true
title: nodejs模块path简介
date: 2016-10-17 11:26:47
tags:
categories:
---

> 该模块提供了一些用来处理和转换文件路径的工具方法

### path.basename(p[, ext])

返回路径最后一部分，和Unix命令`basename`很相似。

<!-- more -->

例如：

    path.basename('/foo/bar/baz/asdf/quux.html')
    // returns
    'quux.html'
    path.basename('/foo/bar/baz/asdf/quux.html', '.html')
    // returns
    'quux'

### path.delimiter

`path.delimiter` 获取系统相关的路径分隔符
如：

    console.log(process.env.PATH)
    // '/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin'

    process.env.PATH.split(path.delimiter)
    // returns
    ['/usr/bin', '/bin', '/usr/sbin', '/sbin', '/usr/local/bin']

### path.dirname(p)

返回参数指定的路径的文件夹名称，如：

    path.dirname('/foo/bar/baz/asdf/quux')
    // returns
    '/foo/bar/baz/asdf'

### path.extname(p)

返回路径的扩展名称，从最后一个`.`到路径字符串结尾之间的部分。如果路径的最后一部分不包含`.`或者第一个字符是`.`，则返回一个空字符串。
如：

    path.extname('index.html')
    // returns
    '.html'

    path.extname('index.coffee.md')
    // returns
    '.md'

    path.extname('index.')
    // returns
    '.'

    path.extname('index')
    // returns
    ''

    path.extname('.index')
    // returns
    ''

### path.format(pathObject)

根据参数指定的路径对象，返回一个格式化字符串；相对的是`path.parse`

### path.isAbsolute(path)

检查参数指定的路径是否一个绝对路径，

Posix 环境：

    path.isAbsolute('/foo/bar') // true
    path.isAbsolute('/baz/..')  // true
    path.isAbsolute('qux/')     // false
    path.isAbsolute('.')        // false

Windows 环境：

    path.isAbsolute('//server')  // true
    path.isAbsolute('C:/foo/..') // true
    path.isAbsolute('bar\\baz')  // false
    path.isAbsolute('.')         // false

### path.join([path1][, path2][, ...])

将参数指定的字符串组成一个格式化路径。
如：

    path.join('/foo', 'bar', 'baz/asdf', 'quux', '..')
    // returns
    '/foo/bar/baz/asdf'

    path.join('foo', {}, 'bar')
    // throws exception
    TypeError: Arguments to path.join must be strings

如果参数中有长度为0的字符串，则会忽略该参数；如果结果字符串长度为0，则返回`.`表示当前路径。

### path.normalize(p)

转化参数路径为正常格式的路径, 该方法会处理参数中出现的`..`和`.`字符。

    path.normalize('/foo/bar//baz/asdf/quux/..')
    // returns
    '/foo/bar/baz/asdf'

如果参数字串长度为0，则返回`.`表示当前路径。

### path.parse(pathString)

根据参数路径字符串返回一个表示该路径的对象，如：

    path.parse('/home/user/dir/file.txt')
    // returns
    {
        root : "/",
        dir : "/home/user/dir",
        base : "file.txt",
        ext : ".txt",
        name : "file"
    }

### path.relative(from, to)

解析参数`form`到参数`to`间的相对路径，例如，我们有2个绝对路径，想在想获取从from到to的相对路径，就可以使用该方法。

如：

    path.relative('C:\\orandea\\test\\aaa', 'C:\\orandea\\impl\\bbb')
    // returns
    '..\\..\\impl\\bbb'

    path.relative('/data/orandea/test/aaa', '/data/orandea/impl/bbb')
    // returns
    '../../impl/bbb'


**注意**: 如果参数中有空字符串，则使用当前路径来替代；如果2个路径一样的话，则返回一个空字符串。

### path.resolve([from ...], to)

将参数`to`指定的路径解析为一个绝对路径。如下：

    path.resolve('/foo/bar', './baz')
    // returns
    '/foo/bar/baz'

    path.resolve('/foo/bar', '/tmp/file/')
    // returns
    '/tmp/file'

    path.resolve('wwwroot', 'static_files/png/', '../gif/image.gif')
    // if currently in /home/myself/node, it returns
    '/home/myself/node/wwwroot/static_files/gif/image.gif'

**注意**: 如果参数中有空字符串，则使用当前路径来替代。

### path.sep

平台相关的文件路径分隔符： '\\' or '/'.

例子：

    'foo/bar/baz'.split(path.sep)
    // returns
    ['foo', 'bar', 'baz']
    An example on Windows:

    'foo\\bar\\baz'.split(path.sep)
    // returns
    ['foo', 'bar', 'baz']

                        
                    
                    
                    