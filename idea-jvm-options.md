---
layout: post
comments: true
title: mac下idea虚拟机参数设置
date: 2017-05-16 16:24:04
tags:
- devtools
categories:
---

### 背景

idea目前应该算是Java开发首选的IDE了，功能非常强大。在日常开发中通常需要打开多个项目进行开发联调，此时idea默认的虚拟机设置就不够好的，需要你手动调整它的JVM参数。

<!-- more -->

### idea jvm参数设置

mac下idea的安装目录通常位于：`/Applications/<Product><version>.app/Contents`。它的jvm参数配置文件是`bin/idea.vmoptions`。修改该文件就可以调整idea的启动jvm参数了。下面是我的配置信息：

```shell
-ea
-server
-Xms1g
-Xmx1g
-XX:ReservedCodeCacheSize=240m
-XX:+UseConcMarkSweepGC
-XX:SoftRefLRUPolicyMSPerMB=50
-Dsun.io.useCanonCaches=false
-Djava.net.preferIPv4Stack=true
-XX:+HeapDumpOnOutOfMemoryError
-XX:-OmitStackTraceInFastThrow
-XX:MaxJavaStackTraceDepth=-1
```

**注意**： 这种修改方式可能破坏idea的签名信息，不推荐使用。

还有一种办法是推荐使用的：将上述文件`copy`一份到目录`~/Library/Preferences/<Product><version>/`, 然后修改该配置文件即可。 

### 参考信息

[https://intellij-support.jetbrains.com/hc/en-us/articles/206544869-Configuring-JVM-options-and-platform-properties](https://intellij-support.jetbrains.com/hc/en-us/articles/206544869-Configuring-JVM-options-and-platform-properties)

[https://intellij-support.jetbrains.com/hc/en-us/articles/206544519](https://intellij-support.jetbrains.com/hc/en-us/articles/206544519)

[http://stackoverflow.com/questions/13578062/how-to-increase-ide-memory-limit-in-intellij-idea-on-mac/13581526#13581526](http://stackoverflow.com/questions/13578062/how-to-increase-ide-memory-limit-in-intellij-idea-on-mac/13581526#13581526)






