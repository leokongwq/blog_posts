---
layout: post
comments: true
title: nodejs调试工具之heapdump简介
date: 2016-11-08 16:53:47
tags:
categories:
- nodejs
---

### heapdump工具介绍

> `heapdump` 是一个非常有用的node内存调试工具. 它能够在运行时将V8的堆dump到文件中. 这样我们就可以通过chrome的带的的开发者工具
查看当时的内存占用情况,帮助我们分析一些内存问题.

<!-- more -->

### 安装

`heapdump`的安装时非常简单的,和安装普通的npm包是一样的

    npm install heapdump --save
    
### 使用
    
一旦我们安装了`heapdump`, 我们就可以在代码中使用`heapdump`了.
    
    var heapdump = require('heapdump');

### 生产堆快照
    
有两种方法来主动生成堆快照:

1. 主动调用方法 `writeSnapshot([filename], [callback])`

2. 通过给进程发送信号 `kill -USR2 pid`

默认情况下node进程是可以接收并处理`USR2`信号的, 如果想要禁止,可以通过添加启动参数来实现:
    
    env NODE_HEAPDUMP_OPTIONS=nosignal node app.js

### 原理

在以前的实现中,在生成块照时, `heapdump` 会fork一个新的node进程异步将老的内存快照信息写入文件. 通过这种方式生成的快照信息不能再chrome的开发者工具中进行比较, 并且和node.js v0.12 版本完全不兼容. 如果你确实想通过这种方式生成快照信息,可以通过添加参数来实现:

    env NODE_HEAPDUMP_OPTIONS=fork node script.js
    
### 分析快照信息
    
1. 打开chrome控制台

{% asset_img chrome-devtool-profile.png %}
    
2. 加载快照文件

点击`load`按钮, 选择文件就可以加载快照文件了.

{% asset_img chrome-nodejs-heapdump.png %}

### 配套工具 memwatch-next

线上运行的程序我们不会时不时的主动生产堆快照信息的, 如果能在发生内存泄露的时候生成快照信息,这样就最好了. 幸运的是有这种运行时内存
监控包:`memwatch-next`

### 安装 memwatch

    npm install memwatch-next --save
    
### 使用 memwatch

#### 内存泄露监听

    var memwatch = require('memwatch-next');
    memwatch.on('leak', function(info) { 
        // 发生内存泄露时的回调函数
        console.error('Memory leak detected: ', info);
    });

### 内存使用监控

监控内存使用情况最好的方法是在V8的GC后统计内存使用信息, memwatch就是这样做的.当V8执行垃圾回收的时候, memwatch会触发一个事件, 我们可以通过监听该事件来收集内存使用情况.

    memwatch.on('stats', function(stats) { ... });
    
内存状态数据格式如下:
    
    {
      "num_full_gc": 17,
      "num_inc_gc": 8,
      "heap_compactions": 8,
      "estimated_base": 2592568,
      "current_base": 2592568,
      "min": 2499912,
      "max": 2592568,
      "usage_trend": 0
    }

`estimated_base` 和 `usage_trend` 是随事件变动的. 如果`usage_trend`的值一直是正数,则表明V8的堆大小是一直增长的, 也就是说程序可能有内存泄露.

V8 有自己执行 GC 的策略, 如当负载比较高的时, V8可能延迟进行GC, `memwatch` 提供了一个方法`gc()`可以让我们主动触发GC动作(V8执行full gc 并进行内存整理).

### Heap Diffing

针对内存泄露 `memwatch`提供了方法`HeapDiff`, 可以让我们对2个heap快照进行比较,找出差异.

    // Take first snapshot 
    var hd = new memwatch.HeapDiff();
     
    // do some things ... 
     
    // Take the second snapshot and compute the diff 
    var diff = hd.end();

diff 的内容格式是:

    {
      "before": { "nodes": 11625, "size_bytes": 1869904, "size": "1.78 mb" },
      "after":  { "nodes": 21435, "size_bytes": 2119136, "size": "2.02 mb" },
      "change": { "size_bytes": 249232, "size": "243.39 kb", "freed_nodes": 197,
        "allocated_nodes": 10007,
        "details": [
          { "what": "String",
            "size_bytes": -2120,  "size": "-2.07 kb",  "+": 3,    "-": 62
          },
          { "what": "Array",
            "size_bytes": 66687,  "size": "65.13 kb",  "+": 4,    "-": 78
          },
          { "what": "LeakingClass",
            "size_bytes": 239952, "size": "234.33 kb", "+": 9998, "-": 0
          }
        ]
      }
    }

用法: 我们可以在memwatch的`stats`回调事件中调用HeapDiff, 这样就能比较
    
### 配合使用

在每一次发现内存泄漏的时候，我们都将此时的内存堆栈快照写入磁盘中

```javascript
memwatch.on('leak', function(info) {
 console.error(info);
 var file = '/tmp/myapp-' + process.pid + '-' + Date.now() + '.heapsnapshot';
 heapdump.writeSnapshot(file, function(err){
   if (err) console.error(err);
   else console.error('Wrote snapshot: ' + file);
  });
}); 
```




    




    