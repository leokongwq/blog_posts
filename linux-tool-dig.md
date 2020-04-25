---
layout: post
comments: true
title: linux-tool-dig
date: 2016-10-13 18:01:21
tags:
    - linux
categories:
    - linux
---
                                                
> dig是一个域名查询工具和nslookup类似。dig其实是一个缩写：即**Domain Information Groper**。一些专业的DNS管理员在追查DNS问题时，都乐于使用dig命令，是看中了dig设置灵活、输出清晰、功能强大的特点。

<!-- more -->

### 最简单的用法

```shell
dig 
输出如下：
; <<>> DiG 9.8.3-P1 <<>>
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16836
;; flags: qr rd ra; QUERY: 1, ANSWER: 13, AUTHORITY: 0, ADDITIONAL: 0
;; QUESTION SECTION:
;.				IN	NS
;; ANSWER SECTION:
.			370341	IN	NS	c.root-servers.net.
.			370341	IN	NS	d.root-servers.net.
.			370341	IN	NS	m.root-servers.net.
.			370341	IN	NS	j.root-servers.net.
.			370341	IN	NS	k.root-servers.net.
.			370341	IN	NS	g.root-servers.net.
.			370341	IN	NS	e.root-servers.net.
.			370341	IN	NS	a.root-servers.net.
.			370341	IN	NS	f.root-servers.net.
.			370341	IN	NS	i.root-servers.net.
.			370341	IN	NS	b.root-servers.net.
.			370341	IN	NS	l.root-servers.net.
.			370341	IN	NS	h.root-servers.net.
;; Query time: 771 msec
;; SERVER: 172.16.0.7#53(172.16.0.7)
;; WHEN: Fri Jun 24 09:58:17 2016
;; MSG SIZE  rcvd: 228
```

直接使用dig命令，不加任何参数和选项时，dig会向默认的本地DNS服务器查询“.”（根域）的NS记录。

### dig 查询根域名

```shell
$ dig .
输出如下：
; <<>> DiG 9.8.3-P1 <<>> .
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 10329
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 0
;; QUESTION SECTION:
;.				IN	A
;; AUTHORITY SECTION:
.			218	IN	SOA	a.root-servers.net. nstld.verisign-grs.com. 2016062301 1800 900 604800 86400
;; Query time: 43 msec
;; SERVER: 172.16.0.7#53(172.16.0.7)
;; WHEN: Fri Jun 24 10:00:15 2016
;; MSG SIZE  rcvd: 92
```

`.`表示根域，`dig .`就表示查询根域名的ip地址。    

### dig指定域名查询服务器

```shell
$ dig @8.8.8.8 www.baidu.com
输出如下：
; <<>> DiG 9.8.3-P1 <<>> @8.8.8.8 www.baidu.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 40812
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 0
;; QUESTION SECTION:
;www.baidu.com.			IN	A
;; ANSWER SECTION:
www.baidu.com.		179	IN	CNAME	www.a.shifen.com.
www.a.shifen.com.	236	IN	A	220.181.111.188
www.a.shifen.com.	236	IN	A	220.181.112.244
;; Query time: 59 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Fri Jun 24 10:09:38 2016
;; MSG SIZE  rcvd: 90
```

### dig命令的格式

dig命令的格式为：`dig @dnsserver domian_name query type`
如果你设置的`dnsserver`是一个域名，那么dig会首先通过默认的`local DNS`服务器去查询对应的IP地址，然后再以设置的`dnsserver`为`local DNS`服务器。如果你没有设置`@dnsserver`，那么dig就会依次使用`/etc/resolv.conf`里的地址作为`local DNS`服务器。
`query type`的`type`取值如下：

```shell
A 地址记录 
AAAA 地址记录 
AFSDB Andrew文件系统数据库服务器记录 
ATMA ATM地址记录 
CNAME 别名记录 
HINFO 硬件配置记录，包括CPU、操作系统信息 
ISDN 域名对应的ISDN号码 
MB 存放指定邮箱的服务器 
MG 邮件组记录 
MINFO 邮件组和邮箱的信息记录 
MR 改名的邮箱记录 
MX 邮件服务器记录 
NS 名字服务器记录 
PTR 反向记录 
RP 负责人记录 
RT 路由穿透记录 
SRV TCP服务器信息记录 
TXT 域名对应的文本信息 
X25 域名对应的X.25地址记录
```

### dig 常用的选项

`-c`:选项，可以设置协议类型（class），包括IN(默认)、CH和HS。
`-f`:选项，dig支持从一个文件里读取内容进行批量查询，这个非常体贴和方便。文件的内容要求一行为一个查询请求。来个实际例子吧：

```shell
$ cat querylist //文件内容，共有两个域名需要查询
www.baidu.com
www.sohu.com
$ dig -f querylist -c IN -t A//设置-f参数开始批量查询
     
; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.10.rc1.el6_3.2 <<>> www.baidu.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<> DiG 9.8.2rc1-RedHat-9.8.2-0.10.rc1.el6_3.2 <<>> www.sohu.com
;; Got answer:
;; ->>HEADER<
```

`-4和-6`:两个选项，用于设置仅适用哪一种作为查询包传输协议，分别对应着`IPv4`和`IPv6`  
-`t`:选项，用来设置查询类型，默认情况下是`A`，也可以设置`MX`等类型，来一个例子：

```shell
$ dig www.baidu.com -t MX
; <<>> DiG 9.8.3-P1 <<>> www.baidu.com -t MX
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 56599
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 0
;; QUESTION SECTION:
;www.baidu.com.			IN	MX
;; ANSWER SECTION:
www.baidu.com.		433	IN	CNAME	www.a.shifen.com.
;; AUTHORITY SECTION:
a.shifen.com.		355	IN	SOA	ns1.a.shifen.com. baidu_dns_master.baidu.com. 1606230006 5 5 86400 3600
;; Query time: 50 msec
;; SERVER: 172.16.0.7#53(172.16.0.7)
;; WHEN: Fri Jun 24 10:36:06 2016
;; MSG SIZE  rcvd: 115
```

`-q`:选项，其实它本身是一个多余的选项，但是它在复杂的`dig`命令中又是那么的有用。`-q`选项可以显式设置你要查询的域名，这样可以避免和其他众多的参数、选项相混淆，提高了命令的可读性，来个例子：    

```shell
dig -q www.baidu.com
; <<>> DiG 9.8.3-P1 <<>> -q www.baidu.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 64015
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 0
;; QUESTION SECTION:
;www.baidu.com.			IN	A
;; ANSWER SECTION:
www.baidu.com.		332	IN	CNAME	www.a.shifen.com.
www.a.shifen.com.	156	IN	A	115.239.210.27
www.a.shifen.com.	156	IN	A	115.239.211.112
;; Query time: 28 msec
;; SERVER: 172.16.0.7#53(172.16.0.7)
;; WHEN: Fri Jun 24 10:37:48 2016
;; MSG SIZE  rcvd: 90
```

`-x`:选项，是逆向查询选项。可以查询IP地址到域名的映射关系。举一个例子：   

```shell
$ dig -x 193.0.14.129
; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.10.rc1.el6_3.2 <<>> -x 193.0.14.129
;; global options: +cmd
;; Got answer:
;; ->>HEADER<
```

### dig特有的查询选项（query option）

和刚才的选项不同，dig还有一批所谓的“查询选项”，这批选项的使用与否，会影响到dig的查询方式或输出的结果信息，因此对于这批选项，dig要求显式的在其前面统一的加上一个“+”（加号），这样dig识别起来会更方便，同时命令的可读性也会更强。
dig总共有42个查询选项，涉及到DNS信息的方方面面，如此多的查询选项，本文不会一一赘述，只会挑出最最常用的几个重点讲解。

#### TCP代替UDP

众所周知，DNS查询过程中的交互是采用UDP的。如果你希望采用TCP方式，需要这样：

    $ dig +tcp www.baidu.com

### 默认追加域

大家直接看例子，应该就能理解“默认域”的概念了，也就能理解+domain=somedomain的作用了：

```shell
dig +domain=baidu.com image
; <<>> DiG 9.8.3-P1 <<>> +domain=baidu.com image
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 8216
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 0
;; QUESTION SECTION:
;image.baidu.com.		IN	A
;; ANSWER SECTION:
image.baidu.com.	847	IN	CNAME	image.n.shifen.com.
image.n.shifen.com.	237	IN	A	115.239.210.36
;; Query time: 116 msec
;; SERVER: 172.16.0.7#53(172.16.0.7)
;; WHEN: Fri Jun 24 10:45:52 2016
;; MSG SIZE  rcvd: 78
```

#### 跟踪dig全过程

```shell
$ dig +trace www.baidu.com
; <<>> DiG 9.8.3-P1 <<>> +trace www.baidu.com
;; global options: +cmd
.			456164	IN	NS	f.root-servers.net.
.			456164	IN	NS	h.root-servers.net.
.			456164	IN	NS	c.root-servers.net.
.			456164	IN	NS	k.root-servers.net.
.			456164	IN	NS	i.root-servers.net.
.			456164	IN	NS	d.root-servers.net.
.			456164	IN	NS	a.root-servers.net.
.			456164	IN	NS	j.root-servers.net.
.			456164	IN	NS	b.root-servers.net.
.			456164	IN	NS	e.root-servers.net.
.			456164	IN	NS	m.root-servers.net.
.			456164	IN	NS	l.root-servers.net.
.			456164	IN	NS	g.root-servers.net.
;; Received 496 bytes from 172.16.0.7#53(172.16.0.7) in 34 ms
com.			172800	IN	NS	h.gtld-servers.net.
com.			172800	IN	NS	l.gtld-servers.net.
com.			172800	IN	NS	i.gtld-servers.net.
com.			172800	IN	NS	c.gtld-servers.net.
com.			172800	IN	NS	m.gtld-servers.net.
com.			172800	IN	NS	f.gtld-servers.net.
com.			172800	IN	NS	b.gtld-servers.net.
com.			172800	IN	NS	e.gtld-servers.net.
com.			172800	IN	NS	g.gtld-servers.net.
com.			172800	IN	NS	j.gtld-servers.net.
com.			172800	IN	NS	k.gtld-servers.net.
com.			172800	IN	NS	d.gtld-servers.net.
com.			172800	IN	NS	a.gtld-servers.net.
;; Received 491 bytes from 192.228.79.201#53(192.228.79.201) in 177 ms
baidu.com.		172800	IN	NS	dns.baidu.com.
baidu.com.		172800	IN	NS	ns2.baidu.com.
baidu.com.		172800	IN	NS	ns3.baidu.com.
baidu.com.		172800	IN	NS	ns4.baidu.com.
baidu.com.		172800	IN	NS	ns7.baidu.com.
;; Received 201 bytes from 192.54.112.30#53(192.54.112.30) in 188 ms
www.baidu.com.		1200	IN	CNAME	www.a.shifen.com.
a.shifen.com.		1200	IN	NS	ns4.a.shifen.com.
a.shifen.com.		1200	IN	NS	ns2.a.shifen.com.
a.shifen.com.		1200	IN	NS	ns3.a.shifen.com.
a.shifen.com.		1200	IN	NS	ns1.a.shifen.com.
a.shifen.com.		1200	IN	NS	ns5.a.shifen.com.
;; Received 228 bytes from 119.75.219.82#53(119.75.219.82) in 3 ms
```

#### NS 记录的查询

`dig`命令可以单独查看每一级域名的NS记录。

```shell
dig ns baidu.com
```

#### 精简dig输出

使用+nocmd的话，可以节省输出dig版本信息

```shell
dig +nocmd www.baidu.com
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 8425
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 0
;; QUESTION SECTION:
;www.baidu.com.			IN	A
;; ANSWER SECTION:
www.baidu.com.		496	IN	CNAME	www.a.shifen.com.
www.a.shifen.com.	188	IN	A	115.239.210.27
www.a.shifen.com.	188	IN	A	115.239.211.112
;; Query time: 31 msec
;; SERVER: 172.16.0.7#53(172.16.0.7)
;; WHEN: Fri Jun 24 10:47:36 2016
;; MSG SIZE  rcvd: 90
```

使用`+short`的话，仅会输出最精简的CNAME信息和A记录，其他都不会输出。就像这样：

```shell
$ dig +short  www.baidu.com
www.a.shifen.com.
115.239.211.112
115.239.210.27
```
    
使用`+nocomment`的话，可以节省输出dig的详情注释信息

```shell
dig +nocomment www.baidu.com
; <<>> DiG 9.8.3-P1 <<>> +nocomment www.baidu.com
;; global options: +cmd
;www.baidu.com.			IN	A
www.baidu.com.		329	IN	CNAME	www.a.shifen.com.
www.a.shifen.com.	21	IN	A	115.239.211.112
www.a.shifen.com.	21	IN	A	115.239.210.27
;; Query time: 4 msec
;; SERVER: 172.16.0.7#53(172.16.0.7)
;; WHEN: Fri Jun 24 10:50:24 2016
;; MSG SIZE  rcvd: 90
```

使用`+nostat`的话，最后的统计信息也不会输出。当`+nocmd`、`+nocomment`和`+nostat`都是用上，是这样:

```shell
dig +nocmd +nocomment +nostat www.baidu.com
;www.baidu.com.			IN	A
www.baidu.com.		880	IN	CNAME	www.a.shifen.com.
www.a.shifen.com.	212	IN	A	115.239.210.27
www.a.shifen.com.	212	IN	A	115.239.211.112
```

### DNS的记录类型

域名与IP之间的对应关系，称为"记录"（record）。根据使用场景，"记录"可以分成不同的类型（type），前面已经看到了有`A`记录和`NS`记录。

常见的DNS记录类型如下。

1. A：地址记录（Address），返回域名指向的IP地址。
2. NS：域名服务器记录（Name Server），返回保存下一级域名信息的服务器地址。该记录只能设置为域名，不能设置为IP地址。
3. MX：邮件记录（Mail eXchange），返回接收电子邮件的服务器地址。
4. CNAME：规范名称记录（Canonical Name），返回另一个域名，即当前查询的域名是另一个域名的跳转，详见下文。
5. PTR：逆向查询记录（Pointer Record），只用于从IP地址查询域名，详见下文。

一般来说，为了服务的安全可靠，至少应该有两条NS记录，而A记录和MX记录也可以有多条，这样就提供了服务的冗余性，防止出现单点失败。

`CNAME`记录主要用于域名的内部跳转，为服务器配置提供灵活性，用户感知不到。举例来说，`facebook.github.io`这个域名就是一个`CNAME`记录。

### 参考

[http://www.ruanyifeng.com/blog/2016/06/dns.html](http://www.ruanyifeng.com/blog/2016/06/dns.html)



            
                                     
                    
                    
                    

