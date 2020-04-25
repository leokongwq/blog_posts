---
layout: post
comments: true
title: nodejs中exports和module.expots异同
date: 2016-10-17 11:27:02
tags:
- node
categories:
- nodejs
---

> 为了更好的理解 exports 和 module.exports 的关系，我们先来补点 js 基础。示例：

<!-- more -->

app.js

    var a = {name: 'nswbmw 1'};
    var b = a;

    console.log(a);
    console.log(b);

    b.name = 'nswbmw 2';
    console.log(a);
    console.log(b);

    var b = {name: 'nswbmw 3'};
    console.log(a);
    console.log(b);


运行 app.js 结果为：

    D:\>node app
    { name: 'nswbmw 1' }
    { name: 'nswbmw 1' }
    { name: 'nswbmw 2' }
    { name: 'nswbmw 2' }
    { name: 'nswbmw 2' }
    { name: 'nswbmw 3' }



解释一下：a 是一个对象，b 是对 a 的引用，即 a 和 b 指向同一个对象，即 a 和 b 指向同一块内存地址，所以前两个输出一样。当对 b 作修改时，即 a 和 b 指向同一块内存地址的内容发生了改变，所以 a 也会体现出来，所以第三四个输出一样。当对 b 完全覆盖时，b 就指向了一块新的内存地址（并没有对原先的内存块作修改），a 还是指向原来的内存块，即 a 和 b 不再指向同一块内存，也就是说此时 a 和 b 已毫无关系，所以最后两个输出不一样。

明白了上述例子后，我们进入正题。
我们只需知道三点即可知道 exports 和 module.exports 的区别了：

`exports` 是指向的 module.exports 的引用
`module.exports` 初始值为一个空对象 {}，所以 exports 初始值也是 {}
require() 返回的是 module.exports 而不是 exports
所以：

我们通过

     var name = 'nswbmw';
     exports.name = name;
     exports.sayName = function() {
         console.log(name);
     }


给 exports 赋值其实是给 module.exports 这个空对象添加了两个属性而已，上面的代码相当于：

    var name = 'nswbmw';
    module.exports.name = name;
    module.exports.sayName = function() {
        console.log(name);
    }

  
我们通常这样使用 exports 和 module.exports

一个简单的例子，计算圆的面积：

使用 exports

app.js

    var circle = require('./circle');
    console.log(circle.area(4));
    circle.js

    exports.area = function(r) {
        return r * r * Math.PI;
    }

使用 module.exports

app.js

      var area = require('./area');
      console.log(area(4));
      area.js

      module.exports = function(r) {
          return r * r * Math.PI;
      }

上面两个例子输出是一样的。你也许会问，为什么不这样写呢？

app.js

      var area = require('./area');
          console.log(area(4));
      area.js

      exports = function(r) {
        return r * r * Math.PI;
      }

运行上面的例子会报错。这是因为，前面的例子中通过给 exports 添加属性，只是对 exports 指向的内存做了修改，而

     exports = function(r) {
        return r * r * Math.PI;
     }

其实是对 exports 进行了覆盖，也就是说 exports 指向了一块新的内存（内容为一个计算圆面积的函数），也就是说 exports 和 module.exports 不再指向同一块内存，也就是说此时 exports 和 module.exports 毫无联系，也就是说 module.exports 指向的那块内存并没有做任何改变，仍然为一个空对象 {} ，也就是说 area.js 导出了一个空对象，所以我们在 app.js 中调用 area(4) 会报 TypeError: object is not a function 的错误。

所以，一句话做个总结：当我们想让模块导出的是一个对象时， exports 和 module.exports 均可使用（但 exports 也不能重新覆盖为一个新的对象），而当我们想导出非对象接口时，就必须也只能覆盖 module.exports 。

我们经常看到这样的用写法：

    exports = module.exports = somethings

上面的代码等价于

     module.exports = somethings
     exports = module.exports

原因也很简单， module.exports = somethings 是对 module.exports 进行了覆盖，此时 module.exports 和 exports 的关系断裂，module.exports 指向了新的内存块，而 exports 还是指向原来的内存块，为了让 module.exports 和 exports 还是指向同一块内存或者说指向同一个 “对象”，所以我们就 exports = module.exports 。

[转自](https://cnodejs.org/topic/5231a630101e574521e45ef8)

                    
                    
                    
                    
                    
                    
                    
                    
                    
                    
                    