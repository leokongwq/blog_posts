---
layout: post
comments: true
title: python文件读写的几种方式
date: 2017-08-05 16:52:46
tags:
categories:
- python
---

### 前言

Python读取文件的方式有多种，但每种方式都有各自的不同的特点，在使用的时候需要注意这细节的不同。

### 文件读取

#### read([size])

read是最常用的文件读取方式。API的说明如下：

> Read at most size uncompressed bytes, returned as a string. If the size argument is negative or omitted, read until EOF is reached.

<!-- more -->

意思就是：如果参数`size`的值是负数或者没有指定，则该方法读取整个文件直到遇到`EOF`。如果指定了`size`参数，则至多读取参数`size`指定数量的未压缩的字节数，返回一个字符串。

```python
## 文件a.txt中的数据为：1428411693,201707041515072520023
f = open('/data/vodOrders.txt')
print f.read(10）
f.close()
//输出：1428411693
## 如果 参数是：11 
//输出：1428411693，
```

**注意**：由于该方法有可能读取整个文件，所以在处理大文件时需要注意，有可能撑满整个内存。

#### readline([size])

API说明如下：

> Return the next line from the file, as a string, retaining newline. A non-negative size argument limits the maximum number of bytes to return (an incomplete line may be returned then). Return an empty string at EOF.

意思是：该方法将文件中的下一行作为字符串进行返回，然后定位到新的一行。如果指定一个非负数的参数，则可以限制读取的最大字节数（此时可能返回不完整的行）。如果读取到文件末尾则会返回一个空字符串。

```python
## 文件a.txt中的数据为：
//1428411693,201707041515072520023
//1428411694,201707041515072520024

f = open('/data/vodOrders.txt')
print f.readline(10）
f.close()
//输出：1428411693
## 如果 参数是：32
//输出：1428411694,201707041515072520024
## 如果不指定参数
//输出：1428411694,201707041515072520024
```

`readline`方法通常用法如下所示：

```python
f = open('/data/vodOrders.txt')

while True:
	line = f.readline()
	if len(line) == 0:
		break
	print line
	
f.close()	
```

#### readlines([size])

`readlines`在处理文件时是非常有用的方法。API文档说明如下：

> Return a list of lines read. The optional size argument, if given, is an approximate bound on the total number of bytes in the lines returned.

意思是：该方法读取整个文件中行并返回一个列表。参数`size`是可选的。如果指定了参数`size`，则会限制最大读取的字节数，不过需要注意的是该值只能限制一个大概的范围。

```python
f = open('/data/vodOrders.txt')

for line in f.readlines(1):
	print line
	
f.close()	
```

**注意**：由于该方法有可能读取整个文件，所以在处理大文件时需要注意，有可能撑满整个内存。

### 写文件

#### write(data)

将字符串数据写入文件。需要注意的是由于缓存的存在，记得调用`close`来关闭文件将缓存中的数据写入磁盘。

#### writelines(sequence_of_strings)

将字符串序列写入文件。需要注意的是该方法并不会主动添加换行符。字符串序列可以是任何可以迭代处理的对象。该方法于针对每个字符串调用`write`方法是等效的。


