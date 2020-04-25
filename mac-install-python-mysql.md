---
layout: post
comments: true
title: mac上安装Python-Mysql
date: 2016-12-28 09:35:30
tags:
- python
categories:
- python
---

### 背景

因为项目中需要修复一部分数据，不想写Java代码并配置任务来修复（主要是代码只使用一次），所以就考虑使用python来修了。我本机再带的Python是2.7.10版本，要操作MySQL当然需要python-mysql包了。当时当我使用pip命令是，系统提示我没有改命令。原来是mac针对该Python版本没有安装pip命令。

<!-- more -->

### 安装pip

    curl https://bootstrap.pypa.io/get-pip.py | sudo python
    
需要注意的是如果不使用sudo的话会报没有权限的错误。    

    OSError: [Errno 13] Permission denied: '/Library/Python/2.7/site-packages/pip'
    
### Upgrading pip

    pip install -U pip  
    
### 安装MySQL-python    

    pip install MySQL-python    
    
该命令还是提示没有权限：
    
    IOError: [Errno 13] Permission denied: '/Library/Python/2.7/site-packages/_mysql.so'
    
使用sudo执行解决问题

    sudo  pip install MySQL-python
    
    //输出结果
    Collecting MySQL-python
    Installing collected packages: MySQL-python
    Successfully installed MySQL-python-1.2.5
    
### 总结

看似简单的问题如果不亲自动手试试就不要说很简单。还有就是注意看日历，如果我仔细查看了每一个日志就不会再Google上浪费几分钟时间了。    
        


