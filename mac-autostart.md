---
layout: post
comments: true
title: mac上禁止jenkins自启动
date: 2018-01-10 15:43:30
tags:
- osx
categories:
---

### 背景

本机安装了Jenkins后，每次开机都会自启动。浪费资源不说，主要是占用了`8080`这个人见人爱的端口。 通过下面的办法可以禁止它的自启动。

### 解决办法

```shell
launchctl unload -w /Library/LaunchDaemons/org.jenkins-ci.plist
```

通过这个例子指定，OSX上启动的的脚本都是放在目录`/Library/LaunchDaemons`下。该目录下的每个文件都是有特殊格式的。

如果想要制作自启动脚本，可以参考这个目录下的脚本。

