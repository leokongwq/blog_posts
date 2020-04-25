---
layout: post
comments: true
title: nodeJs解决异步回调方案之module-async
date: 2016-10-17 10:28:22
tags:
- node
categories:
- web
---

### async简介
Aync是一个处理异步Javascript的功能强大工具模块。最初是为了Nodejs环境开发设计的，但是也可以直接在浏览器端使用。

### 安装

    bower install async
    component install caolan/async
    jam install async
    spm install async

Async 提供了接近70个函数，包括常用的功能性特性(map, reduce, filter, each…) 和一些常用的异步流程控制模式 (parallel, series, waterfall…)。所有的这些函数允许你遵循NodeJs惯例，提供一个回调函数作为异步函数的最后一个参数。该函数的第一个参数是Error类型的参数，并且只会被调用一次。

<!-- more -->

### 常见陷阱

#### 同步迭代函数

如果你在使用Async时得到一个像 `RangeError: Maximum call stack size exceeded`，或其他栈溢出错误, 你应该是在使用异步迭代器。我们知道在异步的情况下，一个函数是在javascript事件循环的同一个事件处理中调用它的回调函数，如果没有做任何I/O操作或使用定时器。在迭代中调用多个回调函数将会引起栈溢出错误。如果你遇到过这种错误, 只需要通过`async.setImmediate`将你的回调函数传入，以此在下一个事件循环的事件处理中开始一个新的调用占。

    async.eachSeries(hugeArray, function iteratee(item, callback) {
        if (inCache(item)) {
            callback(null, cache[item]); // if many items are cached, you'll overflow
        } else {
            doSomeIO(item, callback);
        }
    }, function done() {
        //...
    });

Just change it to:

    async.eachSeries(hugeArray, function iteratee(item, callback) {
        if (inCache(item)) {
            async.setImmediate(function () {
                callback(null, cache[item]);
            });
        } else {
            doSomeIO(item, callback);
            //...
        }
    });

#### 多次调用回调函数

确保在调用回调函数时及时添加return语句。否则会导致回调函数被多此调用，此时会导致意想不到的行为。

    async.waterfall([
        function (callback) {
            getSomething(options, function (err, result) {
                if (err) {
                    callback(new Error("failed getting something:" + err.message));
                    // we should return here
                }
                // since we did not return, this callback still will be called and
                // `processData` will be called twice
                callback(null, result);
            });
        },
        processData
    ], done)

### 下载

Async的源代码可以在[github](https://github.com/caolan/async)下载。你也可以通过`npm`安装： 

    npm install --save async

同样的方式也适用于：Bower:

    bower install async

安装完成后可以通过正常的方式引用Async:

    var async = require("async");

或引用Async里面的单个方法：

    var waterfall = require("async/waterfall");
    var map = require("async/map");

### 在浏览器使用

Async 可以在任何的ES5环境中使用(IE9 或更高的版本)。

用法：

    <script type="text/javascript" src="async.js"></script>
    <script type="text/javascript">
    
        async.map(data, asyncProcess, function(err, results){
            alert(results);
        });
    
    </script>

### 作为ES模块使用

Async提供了一个名为`async-es`的npm包，可以在ES5环境中使用：

```
npm i -S async-es
import waterfall from 'async-es/waterfall';
import async from 'async-es';
```

### 文档

一些格式如下的Async函数同样是可用的：

 - <name>Series - the same as <name> but runs only a single async operation at a time
 - <name>Limit - the same as <name> but runs a maximum of limit async operations at a time
 
### 集合
 - each, eachSeries, eachLimit
 - forEachOf, forEachOfSeries, forEachOfLimit
 - map, mapSeries, mapLimit
 - filter, filterSeries, filterLimit
 - reject, rejectSeries, rejectLimit
 - reduce, reduceRight
 - detect, detectSeries, detectLimit
 - sortBy
 - some, someLimit, someSeries
 - every, everyLimit, everySeries
 - concat, concatSeries

### 流程控制

 - series
 - parallel, parallelLimit
 - whilst, doWhilst
 - until, doUntil
 - during, doDuring
 - forever
 - waterfall
 - compose
 - seq
 - applyEach, applyEachSeries
 - queue, priorityQueue
 - cargo
 - auto
 - autoInject
 - retry
 - retryable
 - iterator
 - times, timesSeries, timesLimit
 - race

### 工具

 - apply
 - nextTick
 - memoize
 - unmemoize
 - ensureAsync
 - constant
 - asyncify
 - wrapSync
 - log
 - dir
 - noConflict
 - timeout
 - reflect
 - reflectAll

### 集合类方法
集合类方法可以迭代数组，对象，映射，集合和任何实现了ES2015迭代协议的对象。 

### each(coll, iteratee, [callback])

该方法针对集合中的每一个元素并行调用函数`iteratee`。函数`iteratee`接受2个参数，第一个是集合中的元素，第二个参数是一个回调函数，当函数`iteratee`执行完毕后调用该参数指定的函数。如果函数`iteratee`调用它的回调函数时，传了一个error对象，则主回调函数会被立刻调用，参数就是该error对象。

**注意**, 由于每个`iteratee`函数是并行调用的，所以该函数不会保证每个`iteratee`函数的执行顺序 
例子：

    //假设 openFiles是一个包含文件名称的数组并且 saveFile 是一个函数
    
    async.each(openFiles, saveFile, function(err){
        // if any of the saves produced an error, err would equal that error
    });
    // assuming openFiles is an array of file names
    
    async.each(openFiles, function(file, callback) {
    
      // Perform operation on file here.
      console.log('Processing file ' + file);
    
      if( file.length > 32 ) {
        console.log('This file name is too long');
        callback('File name too long');
      } else {
        // Do work to process file here
        console.log('File processed');
        callback();
      }
    }, function(err){
        // if any of the file processing produced an error, err would equal that error
        if( err ) {
          // One of the iterations produced an error.
          // All processing will now stop.
          console.log('A file failed to process');
        } else {
          console.log('All files have been processed successfully');
        }
    });


相关的函数：

 - eachSeries(coll, iteratee, [callback])
 - eachLimit(coll, limit, iteratee, [callback])
 
### forEachOf(coll, iteratee, [callback])

该函数和`each`函数类似, 区别是该函数会将元素的索引传递给迭代函数的第二个参数。

例子：

    var obj = {dev: "/dev.json", test: "/test.json", prod: "/prod.json"};
    var configs = {};
    async.forEachOf(obj, function (value, key, callback) {
        fs.readFile(__dirname + value, "utf8", function (err, data) {
            if (err) return callback(err);
            try {
                configs[key] = JSON.parse(data);
            } catch (e) {
                return callback(e);
            }
            callback();
        });
    }, function (err) {
        if (err) console.error(err.message);
        // configs is now a map of JSON data
        doSomethingWith(configs);
    })

相关的函数：

 - forEachOfSeries(coll, iteratee, [callback])
 - forEachOfLimit(coll, limit, iteratee, [callback])

### map(coll, iteratee, [callback])

该函数会生成一个新的集合，集合中的元素是通过迭代函数`iteratee`针对参数`coll`指定的集合中的元素生成。迭代函数的每个回调函数都接受2个参数，第一个参数是error对象，第二个是经过`iteratee`进行转换的元素。 

**注意**： 由于每个迭代函数是并行执行的，因此并不保证每个迭代函数会按顺序完成。然而结果数组中元素的顺序和在参数集合中出现的顺序是一致的。

例子：

    async.map(['file1','file2','file3'], fs.stat, function(err, results){
        // results is now an array of stats for each file
    });

相关的函数：

 - mapSeries(coll, iteratee, [callback])
 - mapLimit(coll, limit, iteratee, [callback])

### filter(coll, iteratee, [callback])

**别名**: select

该函数针对参数集合中的每个元素并行调用`iteratee`函数，`iteratee`函数返回true或false, 返回true,则参数指定元素会被添加的结果数组中。结果数组中的元素顺序和参数`coll`中出现的顺序一致。

例子：

```javascript
    async.filter(['file1','file2','file3'], function(filePath, callback) {
      fs.access(filePath, function(err) {
        callback(null, !err)
      });
    }, function(err, results){
        // results now equals an array of the existing files
    });
```
相关的函数：

 - filterSeries(coll, iteratee, [callback])
 - filterLimit(coll, limit, iteratee, [callback])

### reject(coll, iteratee, [callback])

和函数`filter`的功能正好相反。

相关的函数：

 - rejectSeries(coll, iteratee, [callback])
 - rejectLimit(coll, limit, iteratee, [callback])

### reduce(coll, memo, iteratee, [callback])

**别名**: inject, foldl

该函数将参数`coll`指定集合转为单个值。`memo`是每个reduce操作的初始状态。该函数只能串行执行。

For performance reasons, it may make sense to split a call to this function into a parallel map, and then use the normal Array.prototype.reduce on the results. This function is for situations where each step in the reduction needs to be async; if you can get the data before reducing it, then it's probably a good idea to do so.

例子：

```javascript
    async.reduce([1,2,3], 0, function(memo, item, callback){
        // pointless async:
        process.nextTick(function(){
            callback(null, memo + item)
        });
    }, function(err, result){
        // result is now equal to the last value of memo, which is 6
    });
```

### every(coll, iteratee, [callback])

别名: all

如果集合中的每个元素都通过异步的测试，则该函数返回true。如果任意一个迭代函数返回false,则主回调函数会立刻被调用。

例子：

```javascript
    async.every(['file1','file2','file3'], function(filePath, callback) {
      fs.access(filePath, function(err) {
        callback(null, !err)
      });
    }, function(err, result){
        // if result is true then every file exists
    });
```

相关的函数：

 - everySeries(coll, iteratee, callback)
 - everyLimit(coll, limit, iteratee, callback)

### concat(coll, iteratee, [callback])

该函数针对集合中的每个元素并行的调用函数`iteratee`,并把所有的结果进行连接。callback接受2个参数，第一个是error；第二个是一个结果数组。

例子：

    async.concat(['dir1','dir2','dir3'], fs.readdir, function(err, files){
        // files is now a list of filenames that exist in the 3 directories
    });

相关的函数：

 - concatSeries(coll, iteratee, [callback])

### 流程控制

### series(tasks, [callback])

该函数串行执行参数`tasks`指定的任务数组。

参数说明：

 - tasks ： 一个包含需要执行函数的数组, 每个函数接受一个回调函数`callback(err, result)`,每个函数必须在完成后调用该回调函数，给回调函数传一个error对象（error对象可以为null）和一个可选的返回值。 
 - callback(err, results) ： 一个可选的回调函数，在所有任务执行完成后只调用一次，可以在该回调函数中获取每个任务的返回结果值组成的数组或对象。

例子：

```javascript
    async.series([
        function(callback){
            // do some stuff ...
            callback(null, 'one');
        },
        function(callback){
            // do some more stuff ...
            callback(null, 'two');
        }
    ],
    // optional callback
    function(err, results){
        // results is now equal to ['one', 'two']
    });
    // an example using an object instead of an array
    async.series({
        one: function(callback){
            setTimeout(function(){
                callback(null, 1);
            }, 200);
        },
        two: function(callback){
            setTimeout(function(){
                callback(null, 2);
            }, 100);
        }
    },
    function(err, results) {
        // results is now equal to: {one: 1, two: 2}
    });    
```
 
### parallel(tasks, [callback])

并行执行参数`tasks`指定的函数数组，每个函数不用等待前一个函数是否执行完毕。如果任何一个任务函数返回一个error对象，则主回调函数会立刻调用。当所有的任务函数都执行完毕后，结果数组会被传给主回调函数。

例子：

```javascript
    async.parallel([
        function(callback){
            setTimeout(function(){
                callback(null, 'one');
            }, 200);
        },
        function(callback){
            setTimeout(function(){
                callback(null, 'two');
            }, 100);
        }
    ],
    // optional callback
    function(err, results){
        // the results array will equal ['one','two'] even though
        // the second function had a shorter timeout.
    });
    // an example using an object instead of an array
    async.parallel({
        one: function(callback){
            setTimeout(function(){
                callback(null, 1);
            }, 200);
        },
        two: function(callback){
            setTimeout(function(){
                callback(null, 2);
            }, 100);
        }
    },
    function(err, results) {
        // results is now equals to: {one: 1, two: 2}
    });
```
    
### whilst(test, fn, callback)

重复调用参数`fn`指定的函数，直到参数`test`指定的函数返回true。结束后或出错后调用函数`callback`。

例子：

```javascript
    var count = 0;
    async.whilst(
        function () { return count < 5; },
        function (callback) {
            count++;
            setTimeout(function () {
                callback(null, count);
            }, 1000);
        },
        function (err, n) {
            // 5 seconds have passed, n = 5
        }
    );
```

### doWhilst(fn, test, callback)

该函数是`whilst`的后检查版本，参数`test` 和 `fn` 进行了位置交换。

### until(test, fn, callback)

重复调用参数`fn`指定的函数，直到参数`test`指定的函数返回true。结束后或出错后调用函数`callback`。


doUntil(fn, test, callback)

参照：doWhilst

                    