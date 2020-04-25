---
layout: post
comments: true
title: mac-osx-kernel
date: 2016-10-20 19:30:49
tags:
- OS
---

> 有一天同事讨论苹果`OS X`系统是否属于UNIX系统, 于是有了这篇文章. 

### os x 内核架构图

{% asset_img Mac_OS_X_architecture.svg.png %}

<!-- more -->

### 结论

1. `os x`的内核称为xnu,是混合内核结构，由开源项目FreeBSD和Mach构建。
2. Darwin项目的负责人是老牌黑客[Jordan Hubbard](http://www.turbofuzz.com/jkh/)，他也是FreeBSD的创始人之一。
3. `os x`的操作系统核心称为darwin。
4. `os x`也是分层架构，由系统成和用户界面组成。
5. `ox x`的用户界面称为Aqua。



