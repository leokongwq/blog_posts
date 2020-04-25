---
layout: post
title: 诡异的异常IllegalMonitorStateException
comments: true
date: 2015-06-12 15:31:33
categories:
- java
---

### 诡异的异常IllegalMonitorStateException

### 诡异的异常IllegalMonitorStateException

今天的一段代码抛出了java.lang.IllegalMonitorStateException，代码如下：
<!-- more -->

```java
private boolean wait = false;
    
public boolean pleaseWait() {  
	synchronized (this.wait) {  
    	if (this.wait == true) {  
    	    return false;  
    	}  
  
    	this.wait =true;  
      
    	try {  
    	    this.wait.wait();  
    	} catch (InterruptedException e) {  

    	}  
      	return true;  
	}
}  
```

JavaDoc上关于IllegalMonitorStateException的解释是：

	Thrown to indicate that a thread has attempted to wait on an object's monitor or to notify other threads waiting on an object's monitor without owning the specified monitor.

看起来有些晦涩难懂，比如object's monitor。其实，在Object.notify()这个函数的JavaDoc中有相关的解释：

A thread becomes the owner of the object's monitor in one of three ways:

1. By executing a synchronized instance method of that object.
2. By executing the body of a synchronized statement that synchronizes on the object.
3. For objects of type Class, by executing a synchronized static method of that class. 

说白了，就是需要在调用wait()或者notify()之前，必须使用synchronized语义绑定住被wait/notify的对象。

可问题是，在上面的代码中，已经对this.wait这个变量使用了synchronzied，然后才调用的this.wait.wait()。按理不应该抛出这个异常。

**上网查了很久，终于找到了答案：**

真正的问题在于this.wait这个变量是一个Boolean，并且，在调用this.wait.wait()之前，this.wait执行了一次赋值操作：

	this.wait = true; 

Boolean型变量在执行赋值语句的时候，其实是创建了一个新的对象。简单的说，在赋值语句的之前和之后，this.wait并不是同一个对象。
synchronzied(this.wait)绑定的是旧的Boolean对象，而this.wait.wait()使用的是新的Boolean对象。由于新的Boolean对象并没有使用synchronzied进行同步，所以系统抛出了IllegalMonitorStateException异常。

相同的悲剧还有可能出现在this.wait是Integer或者String类型的时候。

一个解决方案是采用java.util.concurrent.atomic中对应的类型，比如这里就应该是AtomicBoolean。采用AtomicBoolean类型，可以保证对它的修改不会产生新的对象。

**正确的代码：**

```java
private AtomicBoolean wait = new AtomicBoolean(false);  

public boolean pleaseWait() {  
    synchronized (this.wait) {  
        if (this.wait.get() == true) {  
            return false;  
        }  
  
        this.wait.set(true);  
  
        try {  
            this.wait.wait();  
        } catch (InterruptedException e) {  
  
        }  
  
        return true;  
    }  
}  
```

### 备注

该blog内容来自[ifeve.com](http://ifeve.com)