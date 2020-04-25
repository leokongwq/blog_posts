---
layout: post
comments: true
title: Nodejs全局对象
date: 2016-10-16 09:21:33
tags:
    - node
categories:
    - nodejs
---

> 所谓全局对象，也就在所有模块都可以使用的对象，不用通过require导入的。

### __dirname

该变量在所有的模块都可以使用，但是在不同的模块取值不同，在及定义文件模块中，返回当前文件模块所在的目录路径。

### __filename

方法当前文件的绝对文件路径

<!-- more -->

### console

该对象用来将消息输出到标准输入或标准错误。

### exports

A reference to the module.exports that is shorter to type. See [module system documentation](https://nodejs.org/api/modules.html) for details on when to use exports and when to use module.exports.

exports isn't actually a global but rather local to each module.

See the [module system documentation](https://nodejs.org/api/modules.html) for more information.


### global

 - Object 全局命名空间对象
在浏览器端，最顶级的作用域是全局作用域。这就意味着在浏览器端如果你在global中定义一个变量，则该变量是全局变量。在Node.js中情况就不一样了。顶级作用域不是全局作用域； 在一个模块中定义一个变量，该变量的作用域在该模块中。

### module

 - Object

当前模块的一个引用。通常情况下 `module.exports` 用来定义一个模块导入的内容，可以通过 require()来访问。

### process

 - Object

表示当前进程的对象

### require()

 - Function

用来加载模块的函数。

### require.cache

 - Object

已经加载的模块会缓存在该对象中。可以通过删除该对象的一个键值对，使下一次`require`时重新加载该模块。需要注意的是：该操作并不会作用于核心模块。

### setImmediate(callback[, arg][, ...])

[setImmediate](https://nodejs.org/api/timers.html#timers_setimmediate_callback_arg) is described in the [timers](https://nodejs.org/api/timers.html) section.

### setInterval(callback, delay[, arg][, ...])#

[setInterval](https://nodejs.org/api/timers.html#timers_setinterval_callback_delay_arg) is described in the [timers](https://nodejs.org/api/timers.html) section.

### setTimeout(callback, delay[, arg][, ...])#

[setTimeout](https://nodejs.org/api/timers.html#timers_settimeout_callback_delay_arg) is described in the [timers](https://nodejs.org/api/timers.html) section.

### clearImmediate(immediateObject)

[clearImmediate](https://nodejs.org/api/timers.html#timers_clearimmediate_immediateobject) is described in the [timers](https://nodejs.org/api/timers.html)  section.

### clearInterval(intervalObject)

[clearInterval](https://nodejs.org/api/timers.html#timers_clearinterval_intervalobject) is described in the [timers](https://nodejs.org/api/timers.html)  section.

### clearTimeout(timeoutObject)

[clearTimeout](https://nodejs.org/api/timers.html#timers_cleartimeout_timeoutobject) is described in the [timers](https://nodejs.org/api/timers.html)  section.

                        
                    
                    
                    