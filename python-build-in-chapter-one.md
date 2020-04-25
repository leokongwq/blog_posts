---
layout: post
comments: true
title: python内置函数-第一篇
date: 2017-08-22 21:32:37
tags:
- python
categories:
- python
---

### zip

#### zip 函数简介

Python内置`zip`函数的签名如下：

```python
future_builtins.zip(*iterables)
```

和`itertools.izip()`的功能类似。 从函数签名可以知道该函数的参数接收一个或多个可以迭代的对象。

我们通过一个例子来看下`zip`函数的作用是啥:

<!-- more -->

```python
#!/usr/bin/python

print zip([1,2,3], [4,5,6])
# 输出： [(1, 4), (2, 5), (3, 6)]
print zip([1,2,3], [4,5])
# 输出： [(1, 4), (2, 5)]
print zip((1,2,3), (4,5,6))
# 输出： [(1, 4), (2, 5), (3, 6)]
print zip([1,2], [4,5,6], [7])
# 输出：[(1, 4, 7)]
print zip([1,2], [4,5,6], [])
print zip({'a':1}, {'b':2})
# 输出：[('a', 'b')]
print zip({'a':1})
# 输出：[('a',)]
# 输出：[]
print zip()
# 输出：[]
print zip({'a':1}, [1,2])
# 输出：[('a', 1)]
```

从上面的代码和输出结果我们可以得出如下的结论：

1. `zip` 函数的参数可以是零个或多个可迭代的参数（eg. tuple, list, map）。
2. 当参数是tuple或list类型时，它将每个参数同一个索引位置的元素组成一个tuple，放入结果list的对应位置
3. 如果参数为空，或参数中有一个为空的参数，则返回一个空list
4. 如果参数类型是dict类型，则它将每个dict的key按顺序组成tuple，放入结果list的对应位置

#### 使用zip构造字典

```python
keys = ['spam','eggs','toast']
vals = [1,3,5]
dict_1 = {}
for (k,v) in zip(keys,vals):
       dict_1[k] = v
print(dict_1)
```

运行结果为：`{'toast': 5, 'eggs': 3, 'spam': 1}`

在python2.2和后续版本中，可以完全跳过for循环，直接把zip过的健/值列表传给内置的dict构造函数:

```python
dict_2 = dict(zip(keys,vals)
```

#### 并行遍历

```python
l1 = [1,2,3]
l2 = [4,5,6]
for (x,y) in zip(l1,l2):
    print(x,'+',y,'=',x + y)
```

### map

map函数的签名是：

```python
map(function, iterable, …)
```
官方文档说明：
> Apply function to every item of iterable and return a list of the results. If additional iterable arguments are passed, function must take that many arguments and is applied to the items from all iterables in parallel. If one iterable is shorter than another it is assumed to be extended with None items. If function is None, the identity function is assumed; if there are multiple arguments, map() returns a list consisting of tuples containing the corresponding items from all iterables (a kind of transpose operation). The iterable arguments may be a sequence or any iterable object; the result is always a list.

我们分解开来学习该函数的用法：

> Apply function to every item of iterable and return a list of the results.

该函数的作用是将参数`function`应用的参数`iterable`指定序列的每一个上，并将结果作为一个list返回。

```python
result = map(lambda x: x * x, [1, 2])
print result
输出：[1, 4]
```

> If additional iterable arguments are passed, function must take that many arguments and is applied to the items from all iterables in parallel.

如果传递了额外的可迭代参数，则该函数必须接收多个来自每个可迭代参数中的值进行并行处理。举个例子：

```python
result = map(lambda x, y: x + y, [1, 2], [3, 4])
print result
输出：[4, 6]
```

> If one iterable is shorter than another it is assumed to be extended with None items. 

如果某个可迭代对象的长度小于其它的对象，则会使用None进行元素补齐。

```python
result = map(lambda x, y: str(x) + str(y), [1, 2], [3])
print result
输出：['13', '2None']
```

> If function is None, the identity function is assumed;
 
如果参数`function`是`None`，自动假定一个`identity`函数。

```python
print map(None, [1, 3])
输出：[1, 3]
```

> if there are multiple arguments, map() returns a list consisting of tuples containing the corresponding items from all iterables (a kind of transpose operation). The iterable arguments may be a sequence or any iterable object; 

如果参数`function`是`None`，并且有多个`iterables`参数，此时`map`将返回一个元组类型的列表。其中每个元组中的元素都是取自每个`iterable`参数相同位置的值（该操作类似矩阵转换）。

```python
print map(None, [1, 3], [2, 4])
输出：[(1, 2), (3, 4)]
# 类似矩阵转换
1, 3    ==> 1, 2
2, 4        3, 4

```

> the result is always a list.

map函数的返回值总是一个list。

### reduce 

reduce函数的签名如下：

```python
reduce(function, iterable[, initializer])
```

reduce函数的官方说明：

> Apply function of two arguments cumulatively to the items of iterable, from left to right, so as to reduce the iterable to a single value. For example, `reduce(lambda x, y: x+y, [1, 2, 3, 4, 5])` calculates `((((1+2)+3)+4)+5)`. The left argument, x, is the accumulated value and the right argument, y, is the update value from the iterable. If the optional initializer is present, it is placed before the items of the iterable in the calculation, and serves as a default when the iterable is empty. If initializer is not given and iterable contains only one item, the first item is returned. Roughly equivalent to:

```python
def reduce(function, iterable, initializer=None):
    it = iter(iterable)
    if initializer is None:
        try:
            initializer = next(it)
        except StopIteration:
            raise TypeError('reduce() of empty sequence with no initial value')
    accum_value = initializer
    for x in it:
        accum_value = function(accum_value, x)
    return accum_value
```

reduce 函数的第一个参数`function`是一个接收两个参数并返回一个计算的函数，该函数对参数`iterable`中相邻的每两个元素进行连续的计算，最终将生产一个值作为reduce函数的返回值。

### filter 

filter函数的签名如下：

```python
filter(function, iterable)
```

官方说明：

> Construct a list from those elements of iterable for which function returns true. iterable may be either a sequence, a container which supports iteration, or an iterator. If iterable is a string or a tuple, the result also has that type; otherwise it is always a list. If function is None, the identity function is assumed, that is, all elements of iterable that are false are removed.
Note that filter(function, iterable) is equivalent to [item for item in iterable if function(item)] if function is not None and [item for item in iterable if item] if function is None.
See itertools.ifilter() and itertools.ifilterfalse() for iterator versions of this function, including a variation that filters for elements where the function returns false.

filte函数的作用很简单，针对参数`iterable`中的每个元素应用函数`function`，并将返回true的元素放入结果list。

### max & min

max 函数签名：

```python
max(iterable[, key])
max(arg1, arg2, *args[, key])
```

> Return the largest item in an iterable or the largest of two or more arguments.
If one positional argument is provided, iterable must be a non-empty iterable (such as a non-empty string, tuple or list). The largest item in the iterable is returned. If two or more positional arguments are provided, the largest of the positional arguments is returned.
The optional key argument specifies a one-argument ordering function like that used for list.sort(). The key argument, if supplied, must be in keyword form (for example, max(a,b,c,key=func)).


min 函数签名：

```python
min(iterable[, key])
min(arg1, arg2, *args[, key])
```

> Return the smallest item in an iterable or the smallest of two or more arguments.
If one positional argument is provided, iterable must be a non-empty iterable (such as a non-empty string, tuple or list). The smallest item in the iterable is returned. If two or more positional arguments are provided, the smallest of the positional arguments is returned.
The optional key argument specifies a one-argument ordering function like that used for list.sort(). The key argument, if supplied, must be in keyword form (for example, min(a,b,c,key=func)).

函数max是用来计算一个序列中元素的最大值，或计算两个或多个参数中的最大值的。其中可选参数`key`是接收一个参数的排序函数，如果要指定参数`key`，则必须使用格式`max(iterable, key=func)`。举个例子：

```python
print max([3,2,1])
print max(4, 8)
输出：
3
8
print max(range(10), key = lambda x: x > 5)
输出：6
```

从上面的代码可以推断，如果参数`key`不为空，那么最大值就一该返回的返回值为准。

min函数的工作原理和max相同，就不在赘述。













