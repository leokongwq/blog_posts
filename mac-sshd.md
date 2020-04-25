---
layout: post
title: mac-sshd
date: 2015-05-14
comments: true
categories:
- linux
- mac
tags:
- mac
- ssh
- linux
---

### mac启用sshd服务

本启动苹果的sshd服务，通过crt链接，但发现竟然报错
报错内容：

	joedeMacBook-Pro:etc joe$ /usr/sbin/sshd
	Could not load host key: /etc/ssh_host_rsa_key
	Could not load host key: /etc/ssh_host_dsa_key
	
发现没有这两个文件

    /etc/ssh_host_rsa_key
    /etc/ssh_host_dsa_key

解决方法：

	sudo ssh-keygen -t rsa -f /etc/ssh_host_rsa_key
	sudo ssh-keygen -t dsa -f /etc/ssh_host_dsa_key
	
	