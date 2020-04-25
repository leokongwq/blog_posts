---
layout: post
comments: true
title: golang打开系统浏览器
date: 2016-10-13 18:00:32
tags:
    - golang
categories:
    - golang
---

> 项目的日常接口测试中有个需求是: 输入需要测试买手id, 打开浏览器进行自动登录,于是就有了如下的代码

<!-- more -->

golang打开系统浏览器，代码如下：

{% codeblock lang:go %}    
    package desktop
    
    import (
        "fmt"
        "os/exec"
        "runtime"
    )
    
    var commands = map[string]string{
        "windows": "start",
        "darwin":  "open",
        "linux":   "xdg-open",
    }
    
    var Version = "0.1.0"
    
    // Open calls the OS default program for uri
    func Open(uri string) error {
        run, ok := commands[runtime.GOOS]
        if !ok {
            return fmt.Errorf("don't know how to open things on %s platform", runtime.GOOS)
        }
    
        cmd := exec.Command(run, uri)
        return cmd.Start()
    }
{% endcodeblock %}

从上面的代码可以看出：打开浏览器是通过调用各个平台打开文件的命令来实现。如果系统没有安装对应文件的打开工具，
`cmd.Start`应该会返回err。
osx苹果平台的`open`命令的参数如果是一个url，则会调用系统默认的浏览器打开指定的URL。

{% codeblock lang:go %}    
    open -h
    Usage: open [-e] [-t] [-f] [-W] [-R] [-n] [-g] [-h] [-b <bundle identifier>] [-a <application>] [filenames] [--args arguments]
    Help: Open opens files from a shell.
          By default, opens each file using the default application for that file.
          If the file is in the form of a URL, the file will be opened as a URL.
    Options:
          -a 通过指定的应用打开
          -b 通过指定标示符的应用打开
          -e 通过编辑器打开
          -t 通过默认的编辑器打开
          -f 从标准输入读取输入内容并用编辑器打开
          -F  --fresh       Launches the app fresh, that is, without restoring windows. Saved persistent state is lost, excluding Untitled documents.
          -R, --reveal      Selects in the Finder instead of opening.
          -W, --wait-apps   Blocks until the used applications are closed (even if they were already running).
              --args        All remaining arguments are passed in argv to the application's main() function instead of opened.
          -n, --new 新开一个进程打开指定的文件
          -j, --hide        Launches the app hidden.
          -g, --background  后台打开
          -h, --header      Searches header file locations for headers matching the given filenames, and opens them.                        
{% endcodeblock %}                    
                    
                    