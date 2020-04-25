---
layout: post
comments: true
title: nodejs模块-os模块
date: 2016-10-17 11:26:55
tags:
- node
categories:
- nodejs
---

### os.EOL 

`os.EOL` 用来获取和具体操作系统相关的行结束符

### os.arch()

`os.arch()` 获取当前系统的CPU架构，可能的返回值是：`x64`, `arm` 和 `ia32`。

<!-- more -->

### os.cpus()

放回一个包含当前系统CPU信息的数组，如：

    [
    {
        "model": "Intel(R) Core(TM) i5-5257U CPU @ 2.70GHz", 
        "speed": 2700, 
        "times": {
            "user": 176720760, 
            "nice": 0, 
            "sys": 82254200, 
            "idle": 535195920, 
            "irq": 0
        }
    }, 
    {
        "model": "Intel(R) Core(TM) i5-5257U CPU @ 2.70GHz", 
        "speed": 2700, 
        "times": {
            "user": 76765780, 
            "nice": 0, 
            "sys": 30186250, 
            "idle": 687215360, 
            "irq": 0
        }
    }, 
    {
        "model": "Intel(R) Core(TM) i5-5257U CPU @ 2.70GHz", 
        "speed": 2700, 
        "times": {
            "user": 176613050, 
            "nice": 0, 
            "sys": 70448080, 
            "idle": 547106310, 
            "irq": 0
        }
    }, 
    {
        "model": "Intel(R) Core(TM) i5-5257U CPU @ 2.70GHz", 
        "speed": 2700, 
        "times": {
            "user": 79569910, 
            "nice": 0, 
            "sys": 30051190, 
            "idle": 684546240, 
            "irq": 0
        }
    }
]


### os.endianness()

判断当前系统是大端(`BE`)还是小端(`LE`)

#### os.freemem()

返回当前系统的空闲内存，单位是字节

### os.homedir()

返回当前用户的home目录

### os.hostname()

返回主机名

### os.loadavg()

返回当前系统的平均负载，

### os.networkInterfaces()

返回当前主机的网卡信息，如下：

    {
    "lo0": [
        {
            "address": "::1", 
            "netmask": "ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff", 
            "family": "IPv6", 
            "mac": "00:00:00:00:00:00", 
            "scopeid": 0, 
            "internal": true
        }, 
        {
            "address": "127.0.0.1", 
            "netmask": "255.0.0.0", 
            "family": "IPv4", 
            "mac": "00:00:00:00:00:00", 
            "internal": true
        }, 
        {
            "address": "fe80::1", 
            "netmask": "ffff:ffff:ffff:ffff::", 
            "family": "IPv6", 
            "mac": "00:00:00:00:00:00", 
            "scopeid": 1, 
            "internal": true
        }
    ], 
    "en0": [
        {
            "address": "fe80::aebc:32ff:fe8a:183b", 
            "netmask": "ffff:ffff:ffff:ffff::", 
            "family": "IPv6", 
            "mac": "ac:bc:32:8a:18:3b", 
            "scopeid": 4, 
            "internal": false
        }, 
        {
            "address": "192.168.0.104", 
            "netmask": "255.255.255.0", 
            "family": "IPv4", 
            "mac": "ac:bc:32:8a:18:3b", 
            "internal": false
        }
    ], 
    "awdl0": [
        {
            "address": "fe80::545c:5eff:fe6e:4847", 
            "netmask": "ffff:ffff:ffff:ffff::", 
            "family": "IPv6", 
            "mac": "56:5c:5e:6e:48:47", 
            "scopeid": 8, 
            "internal": false
        }
    ]
  

### os.platform()

放回当前系统的平台，可能的返回值：`darwin`, `freebsd`, `linux`, `sunos` 或 `win32`。

### os.release()

返回当前系统的版本信息

### os.tmpdir()

返回当前系统的临时目录

### os.totalmem()

返回当前系统总的内存大小

### os.type()

返回当前系统的类型，如：`Linux`,`Darwin`,`Windows_NT`

### os.uptime()

返回当前系统的运行时间，单位是秒

                        
                    
                    
                    
                    
                    
                    