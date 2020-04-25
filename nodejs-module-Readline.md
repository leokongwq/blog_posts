---
layout: post
comments: true
title: nodejs模块之readline
date: 2016-10-17 10:58:54
tags:
- node
categories:
- nodejs
---

> Readline 模块提供了从输入流中一行一行的读取数据的能力。需要注意的是：一旦你调用该模块的读数据方法，你的Node程序将一直运行，直到关闭该数据输入接口，下面的例子展示了如何优雅的推出程序：

<!-- more -->

```javascript
    const readline = require('readline');
    const rl = readline.createInterface({
      input: process.stdin,
      output: process.stdout
    });
    rl.question('What do you think of Node.js? ', (answer) => {
    // TODO: Log the answer in a database
    console.log('Thank you for your valuable feedback:', answer);
    rl.close();
    });
```

### Class: Interface

该类表示一个拥有输入流和输出流的`readline`接口

### rl.close()

关闭接口实例, 并释放和该实例相关的输入输出流。同时会触发`close` 事件。

### rl.pause()

暂停`readline`的输入流，如果需要的话，稍后可以恢复。

需要注意的值，该方法不会立即停止输入流。可能在几个事件处理后才能停止。这是和Node的工作机制决定的。

### rl.prompt([preserveCursor])

Readies readline for input from the user, putting the current setPrompt options on a new line, giving the user a new spot to write. Set preserveCursor to true to prevent the cursor placement being reset to 0.This will also resume the input stream used with createInterface if it has been paused.If output is set to null or undefined when calling createInterface, the prompt is not written.

### rl.resume()

恢复暂停的`readline`输入流

                        
                    
                    
                    