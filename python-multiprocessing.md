---
layout: post
comments: true
title: python多进程简介
date: 2017-08-11 22:21:07
tags:
- python
categories:
- python
---

### 前言

由于CPython中`GIL(Global Interpreter Lock)`存在，导致Python多线程的效率针对CPU密集型的应用不友好。具体原因可以参考优秀博文:[Python的GIL是什么鬼，多线程性能究竟如何](http://cenalulu.github.io/python/gil-in-python/)。解决这个问题的办法很多，换一个Python运行环境或者换一种语言。哈哈..... 当然了你也可以使用`multiprocessing`

<!-- more -->


### multiprocessing 简介

> multiprocessing is a package that supports spawning processes using an API similar to the threading module. The multiprocessing package offers both local and remote concurrency, effectively side-stepping the Global Interpreter Lock by using subprocesses instead of threads. Due to this, the multiprocessing module allows the programmer to fully leverage multiple processors on a given machine. It runs on both Unix and Windows.

意思就是该模块可以用来生成多个进程，以此来避免`GIL`的限制，从而允许程序员更好的利用CPU资源。该模块同时支持Windows和Linux。

The multiprocessing module also introduces APIs which do not have analogs in the threading module. A prime example of this is the Pool object which offers a convenient means of parallelizing the execution of a function across multiple input values, distributing the input data across processes (data parallelism). The following example demonstrates the common practice of defining such functions in a module so that child processes can successfully import that module. This basic example of data parallelism using Pool,

```python
from multiprocessing import Pool
import time

def f(x):
    time.sleep(20)
    return x*x

if __name__ == '__main__':
    p = Pool(5)
    print(p.map(f, [1, 2, 3]))
```

will print to standard output

```
[1, 4, 9]
```

`multiprocessing`模块提供了进程池功能。上面代码的功能是：

1. 创建了含有5个进程的进程池对象
2. 将list=[1,2,3]作为每个进程计算任务的输入参数
3. 将每个子进程的计算结果汇总起来并返回。

为了更好的演示代码的功能，我添加了`time.sleep(20)`语句。执行上面的代码我们可以看到系统会存在多个进程。如下图：

{% asset_img multi-processor.png %}

### Process类简介

在`multiprocessing`包中，创建一个子进程其实就是创建一个`Process`对象，并调用创建对象的`start()`方法。
`Process`提供了和`threading.Thread`一致的API。下面就是一个多进程的小例子：

```python
from multiprocessing import Process

def f(name):
    print 'hello', name

if __name__ == '__main__':
    p = Process(target=f, args=('bob',))
    p.start()
    p.join()
```

上面代码的作用如下：

1. 创建一个`Process`对象
2. 调用Process对象的`start()`方法
3. 在主进程中等待子进程结束

下面的代码演示了如何获取主子进程的ID：

```python
from multiprocessing import Process
import os

def info(title):
    print title
    print 'module name:', __name__
    if hasattr(os, 'getppid'):  # only available on Unix
        print 'parent process:', os.getppid()
    print 'process id:', os.getpid()

def f(name):
    info('function f')
    print 'hello', name

if __name__ == '__main__':
    info('main line')
    p = Process(target=f, args=('bob',))
    p.start()
    p.join()
```

### 进程间交换数据

`multiprocessing` 支持两种进程间通信的通道：

#### Queues

类`Queue`类似`Queue.Queue`的一个克隆。下面的例子演示了主子进程间如何通过`Queue`进行通信：

```python
from multiprocessing import Process, Queue

def f(q):
    q.put([42, None, 'hello'])

if __name__ == '__main__':
    q = Queue()
    p = Process(target=f, args=(q,))
    p.start()
    print q.get()    # prints "[42, None, 'hello']"
    p.join()
```

Queue 是线程和进程间安全的

#### Pipes

函数`Pipe()`返回一对通过默认全双工的`pipe`进行通信的连接对象。

```python
from multiprocessing import Process, Pipe

def f(conn):
    conn.send([42, None, 'hello'])
    conn.close()

if __name__ == '__main__':
    parent_conn, child_conn = Pipe()
    p = Process(target=f, args=(child_conn,))
    p.start()
    print parent_conn.recv()   # prints "[42, None, 'hello']"
    p.join()
```

`Pipe()`返回的两个连接对象代表了`pipe`的两端。每一个连接对象都有一个`send()`和`recv()`方法。需要注意的是：`pipe`中的数据有可能变的不可用，这是可能是因为两个连接对象同时在通道的**一端**进行读取或写入操作。当然了同时在通道的两端进行操作是没有任何风险的。

### 进程间同步

`multiprocessing`包含了与`threading`中所有同步元语等价的实现。一个例子就是：可以通过一个🔐确保在同一时刻只能有一个进程打印数据到标准输出。

```python
from multiprocessing import Process, Lock

def f(l, i):
    l.acquire()
    print 'hello world', i
    l.release()

if __name__ == '__main__':
    lock = Lock()

    for num in range(10):
        Process(target=f, args=(lock, num)).start()
```

如果没有同步，所有进程的输出将会是乱序的。

### 进程间共享状态

就像上面提到的，在并发编程中最好尽可能的避免状态共享。这条规则同样适用于多进程。然而，如果你必须在多个进程间共享数据，`multiprocessing`也提供了下面的几种方式来实现。

#### Shared memory

共享数据可以使用`Value`或`Array`保存在共享内存映射中。例如下面代码所示：

```python
from multiprocessing import Process, Value, Array

def f(n, a):
    n.value = 3.1415927
    for i in range(len(a)):
        a[i] = -a[i]

if __name__ == '__main__':
    num = Value('d', 0.0)
    arr = Array('i', range(10))

    p = Process(target=f, args=(num, arr))
    p.start()
    p.join()

    print num.value
    print arr[:]
```

输出如下：

```
3.1415927
[0, -1, -2, -3, -4, -5, -6, -7, -8, -9]
```

上面代码中，创建变量num和变量arr使用的参数`d`,`i`都是类型代码。在数组中使用时，`d`表示双精度的浮点数，`i`表示有符号的整数类型。这些变量在使用时都是线程安全的。

如果需要更加灵活的使用共享内存, 可以使用`multiprocessing.sharedctypes`模块，该模块支持在共享内存中创建任意的对象。

#### Server process

一个`Manager()`函数返回的manager对象可以控制一个拥有python对象的服务器进程，并允许其它的进程使用代理来修改这写Python对象。

`Manager()`函数返回的对象支持的类型有: list, dict, Namespace, Lock, RLock, Semaphore, BoundedSemaphore, Condition, Event, Queue, Value and Array。举个例子：

```python
from multiprocessing import Process, Manager

def f(d, l):
    d[1] = '1'
    d['2'] = 2
    d[0.25] = None
    l.reverse()

if __name__ == '__main__':
    manager = Manager()

    d = manager.dict()
    l = manager.list(range(10))

    p = Process(target=f, args=(d, l))
    p.start()
    p.join()

    print d
    print l
```

输出如下：

```
{0.25: None, 1: '1', '2': 2}
[9, 8, 7, 6, 5, 4, 3, 2, 1, 0]
```

服务器进程管理器比使用共享内存对象更灵活，因为它们可以用于支持任意对象类型。 此外，单个管理器可以通过网络在不同计算机上的进程共享。 但是，它们比使用共享内存要慢。

### Using a pool of workers

类`Pool`表示一个工作进程池。 它具有几种不同方法将任务分配到工作进程上。

For example:

```python
from multiprocessing import Pool, TimeoutError
import time
import os

def f(x):
    return x*x

if __name__ == '__main__':
    pool = Pool(processes=4)              # start 4 worker processes

    # print "[0, 1, 4,..., 81]"
    print pool.map(f, range(10))

    # print same numbers in arbitrary order
    for i in pool.imap_unordered(f, range(10)):
        print i

    # evaluate "f(20)" asynchronously
    res = pool.apply_async(f, (20,))      # runs in *only* one process
    print res.get(timeout=1)              # prints "400"

    # evaluate "os.getpid()" asynchronously
    res = pool.apply_async(os.getpid, ()) # runs in *only* one process
    print res.get(timeout=1)              # prints the PID of that process

    # launching multiple evaluations asynchronously *may* use more processes
    multiple_results = [pool.apply_async(os.getpid, ()) for i in range(4)]
    print [res.get(timeout=1) for res in multiple_results]

    # make a single worker sleep for 10 secs
    res = pool.apply_async(time.sleep, (10,))
    try:
        print res.get(timeout=1)
    except TimeoutError:
        print "We lacked patience and got a multiprocessing.TimeoutError"
```

**请注意**，`Pool`类对象的方法只能由创建它的进程使用。

> 此包中的功能要求__main__模块可由子级导入。 这在编程指南中有所涉及，但值得一提的是这里。 这意味着一些示例，例如Pool示例将无法在交互式解释器中使用。 例如：
```
>>> from multiprocessing import Pool
>>> p = Pool(5)
>>> def f(x):
...     return x*x
...
>>> p.map(f, [1,2,3])
Process PoolWorker-1:
Process PoolWorker-2:
Process PoolWorker-3:
Traceback (most recent call last):
AttributeError: 'module' object has no attribute 'f'
AttributeError: 'module' object has no attribute 'f'
AttributeError: 'module' object has no attribute 'f'
```
(如果你尝试这样做，实际上它会半随机的输出三个完整的堆栈信息， 然后你可能不得不停止主进程。)
    
### 参考

[https://docs.python.org/2/library/multiprocessing.html](https://docs.python.org/2/library/multiprocessing.html)



