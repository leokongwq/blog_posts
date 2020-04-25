---
layout: post
comments: true
title: python包机制简介
date: 2017-08-07 18:08:54
tags:
categories:
- python
---

### 前言

每种编程语言都有自己的代码组织方式，Python也不例外。如果只是写一点简单的脚本，一般也不用不上Python的包和模块机制，当你积累的工具代码或工程代码越来越多的时候，为了更好的代码复用和结构组织就需要它的包和模块机制。

### 包

Python程序由包（package）、模块（module）和函数组成。包是由一系列模块组成的集合。模块是处理某一类问题的函数和类的集合。

<!-- more -->

Python提供了许多有用的工具包，如字符串处理、图形用户接口、Web应用、图形图像处理等。这些自带的工具包和模块安装在Python的安装目录下的Lib子目录中。
**注意：**
包必须至少含有一个`__int__.py`文件按，该文件的内容可以为空。`__int__.py`用于标识当前文件夹是一个包。

### 模块

Python的程序是由一个个模块组成的。

#### 模块的创建

每一个`.py`文件都是一个模块。一个模块将强相关的代码：类，函数，常量等组织在一起。创建一个`person.py`文件就是创建了名为`person`的模块。

```python person.py
#!/usr/bin/python
# -*- coding: UTF-8 -*-

def sayHi(msg):
    print "Hello %s" % msg

class Person:

    def sing(self, music):
        print "I'm singing %s" % music
```

```python callPerson.py
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import person

person.sayHi("Python")

p = person.Person()

p.sing("我的祖国")
```

**注意：**

`person.py`和`callPerson.py`必须放在同一个目录下或放在`sys.path`所列出的目录下，否则，`Python解释器`找不到自定义的模块。

当Python导入一个模块时会按照如下的顺序来查找模块：
1. 首先查找当前路径
2. 查找`lib`目录(`/usr/lib64/python2.7`)、`site-packages`目录（`/usr/lib64/python2.7/site-packages`）
3. 查找环境变量`PATHONPATH`设置的目录。
4. 如果通过以上的路径模块还没有找到，则可以通过`sys.path`语句的输出的值作为模块的查找路径。

**注意：** 我们可以通过操作`sys.path`的值控制查找路径，但一般情况是不需要的。

#### 模块的使用

当模块创建好了之后就可以使用了。使用模块时需要通过`import`关键字来导入需要使用的模块。例如：

```python
import datetime
```

导入模块后就可以通过模块名来使用模块里面的函数和类。例如：

```python
import datetime

print datetime.datetime.now()
```

如果想直接使用模块中的函数或类，可以通过如下的方式来实现：

```python
from datetime import datetime

print datetime.now()
```

**注意：** 通过这中方式可能导致代码的可读性变差。

如果需要导入该模块下所有的类和函数，可以通过`from module_name import *`这样格式实现。

Python中的`import`语句比Java的`import`语句更加灵活。Python的`import`语句可以置于程序中任意的位置，甚至可以放在条件语句中。

#### 模块的属性

模块有一些内置属性，用于完成特定的任务，如`__name__`、`__doc__`。每个模块都有一个名称。
例如：
`__name__`:用于判断当前模块是否是程序的入口，如果当前程序正在被使用，`__name__`的值为“__main__”。

### 内置模块

Python提供了一个内联模块bulidin，该模块中定义了一些软件开发总经常用到的模块，利用这些函数可以实现数据类型的转换、数据的计算、序列的处理等功能。

#### apply()

`apply()`可以实现调用可变参数列表的函数，把函数的参数存放在一个元组或序列中。
apply()的声明如下：

```python
apply(func[,args[,kwargs]])
```
**说明:**
- 参数func是自定义的函数名称；
- 参数args是一个包含自定义函数参数的列表或元组。args如果省略，则表示被执行的函数没有任何参数。
- 参数kwargs是一个字典。字典中的key值为函数的参数名称，value值为实际参数的值。

**例子:**

```python
def sum(x = 1,y = 2):
    return x + y

print apply(sum,(1,3))
```

**注意：** : apply()的元组参数是有序的。

#### fiter()

`filter()`可以对某个序列做过滤处理，对自定义函数的参数返回的结果是否为“真”来过滤，并一次性返回处理结果。
`filter()`的声明如下所示：

```python
filter(func or None, sequence) -> list, tuple, or string
```

**说明**

- 参数func是自定义的过滤函数，在函数`func(item)`中定义过滤的规则。如果func为None，则过滤项item都为真，返回所有的序列元素。
- 参数sequence为待处理的序列。

filter()的返回值是由func()的返回值组成的序列，返回的类型与参数sequence的类型相同。例如，参数sequence为元组，则返回的类型也是元组。
例子：从给定的列表中过滤出大于0的数字。

```python
def func(x):
    if x > 0:
        return x
print filter(func, range(-9,10))
```

#### reduce()

对序列中元素的连续操作可以通过循环来处理（例如对某个序列中的元素累加操作）。Python提供的`reduce()`也可以实现连续处理的功能。
`reduce()`的声明如下所示：

```python
reduce(func, sequence[,initial])->value
```
说明：
- 参数func是自定义的函数，在函数func()中实现对参数sequence的连续操作。
- 参数sequence为待处理的序列。
- 参数initial可以省略，如果initial不为空，则initial的值将首先传入func()进行计算。如果sequence为空，则对initial的值进行处理。
reduce()的返回值是func()计算后的结果。

例子：实现对一个列表中的数字进行累加的操作。

```python

def sum(x, y):
    return x + y

print reduce(sum,range(0,10))#0+1+2+3+4+5+6+7+8+9
print reduce(sum,range(0,10),10)#10+0+1+2+3+4+5+6+7+8+9
print reduce(sum,range(0,0),10)#range(0,0)返回空列表，所以返回结果就是10
```

#### map()

map()可以对多个序列的每个元素都执行相同的操作，并组成列表返回。
map()的声明如下所示：

```python
map(func,sequence[,sequence,...])->list
```

- 参数func是自定义的函数，实现对序列每个元素的操作。
- 参数sequence为待处理的序列，参数sequence的个数可以是多个。

map()的返回值是对序列元素处理后的列表。

例子：实现列表中数字的幂运算。

```python
def power(x):return x**x
print map(power,range(1,5))
def power2(x,y):return x**y
print map(power2,range(1,5),range(5,1,-1))
```

**注意：**
如果map()中提供了多个序列，则每个序列中的元素一一对应进行计算。如果每个序列的长度不相同，则短的序列后补充None，再进行计算。
常用的内联函数如下表所示：

{% asset_img 20141008225103462.jpg %}

### 自定义包

包就是一个至少包含`__int__.py`文件的文件夹。Python包和Java包的作用是相同的，都是为了实现程序的重用。把实现一个常用功能的代码组合到一个包中，调用包提供的服务从而实现重用。
例如，定义一个包parent。在parent包中创建两个子包pack和pack2.pack包中定义一个模块myModule，pack2包中定义一个模块myModule2.最后在包parent中定义一个模块main，调用子包pack和pack2.如下图所示：
包与模块的树形关系图

{% asset_img 20141008225117549.jpg %}

包pack的__int__.py程序如下所示：

```python
#!/usr/bin/env python
# -*- coding=utf-8 -*-
if __name__ == '__main__':
    print '作为主程序运行'
else:
    print 'pack初始化'
```

这段代码初始化pack包，这里直接输出一段字符串。当pack包将其他模块调用时，将输出“pack包初始化”。
包pack的myModule模块如下所示：





