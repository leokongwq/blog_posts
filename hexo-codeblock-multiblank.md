---
layout: post
comments: true
title: hexo代码块前后空白行问题
date: 2016-10-14 15:55:57
tags:
    - hexo
    - web
categories:
    web
---

### 使用hexo3.2.2时解决代码块空行的问题

> 最近使用hexo3.2.2版本搭建blog时, 发现只要的代码块中有空行,则这些空行机会平均的添加好代码块的前后部分网上搜索的答案,官方github给出的答案[795](https://github.com/hexojs/hexo/pull/795)并没有解决我的问题.但是给我了启发,最终经过调试将答案写到本blog中

<!-- more -->

### 解决办法

- 1.找到hexo-util/lib/highlight.js文件
- 2.修改highlight.js代码35~38行为

```js
numbers += '<span class="line">' + (firstLine + i) + '</span>\n';
content += '<span class="line';
content += (mark.indexOf(firstLine + i) !== -1) ? ' marked' : '';
content += '">' + line + '</span>\n';
```

网上大部分答案说是可以通过如下的代码解决:

```js
// 修改文件 lib/util/highlight.js
numbers += '<div class="line">' + (i + firstLine) + '</div>';
content += '<div class="line">' + item + '</div>';
改为
numbers += '<span class="line">' + (i + firstLine) + '</span>\n';
content += '<span class="line">' + item + '</span>\n';
```

但是这种办法对3.2.2版本是不起作用的.

### 总结

使用开源软件就需要有修改它bug的耐心.同样的bug可能在一个版本中修复了,但在新的版本中有可能又出现
且修复的办法不尽相同.
    
