---
layout: post
title: mysql load data infile 简介
date: 2015-06-11
categories:
- database
tags:
- mysql
---

### mysql load data infile 简介

### load data 语法介绍

<!-- more -->

```sql
LOAD DATA [LOW_PRIORITY | CONCURRENT] [LOCAL] INFILE 'file_name.txt'

   [REPLACE | IGNORE]

  INTO TABLE tbl_name

   [FIELDS

    [TERMINATED BY 'string']

   [[OPTIONALLY] ENCLOSED BY 'char']

   [ESCAPED BY 'char' ]

 ]

   [LINES

   [STARTING BY 'string']

  [TERMINATED BY 'string']

  ]

   [IGNORE number LINES]

  [(col_name_or_user_var,...)]

   [SET col_name = expr,...]]
```

#### load data的作用

在日常的开发中我们都是通过JDBC连接mysql进行数据的写入操作，但是如果需要把从其他系统，数据库等获得的**大量**数据导入到mysql的需求时，如果再利用JDBC的话，速度很难满足需求，而且需要我们写代码解析文件，工作量比较大。此次利用**mysql load data infile** 功能的话就非常方便，且速度较快（官方的说法是20倍的差距）。

load data infile语句从一个文本文件中以很高的速度读入一个表中。使用这个命令之前，mysqld进程（服务）必须已经在运行。为了安全原因，当读取位于服务器上的文本文件时，文件必须处于数据库目录或可被所有人读取。另外，为了对服务器上文件使用load data infile，在服务器主机上你必须有file的权限。

#### load data 使用事项

1. 如果你指定关键词**low_priority**，那么MySQL将会等到没有其他人读这个表的时候，才把插入数据。可以使用如下的命令：
 
	load data low_priority infile "/home/mark/data sql" into table Orders;

2. 如果指定local关键词，则表明从客户主机读文件。如果local没指定，文件必须位于服务器(mysqld)上。

3. replace和ignore关键词控制对现有的唯一键记录的重复的处理。如果你指定replace，新行将代替有相同的唯一键值的现有行。如果你指定ignore，跳过有唯一键的现有行的重复行的输入。如果不指定二者中的任一个，则操作行为将依赖是否指定了LOCAL 关键字。没有指定LOCAL，则如果发现有重复的键值，将产生一个错误，并忽略文本文件的其余部分。如果指定了LOCAL，则缺省的操作行为将与指定了IGNORE 的相同；这是因为，在操作过程中，服务器没有办法终止文件的传送。例如：
	load data low_priority infile "/home/mark/data sql" replace into table Orders;

4. 分隔符
	1. fields关键字指定了文件记段的分割格式，如果用到这个关键字，MySQL剖析器希望看到至少有下面的一个选项： terminated by分隔符：意思是以什么字符作为分隔符;enclosed by字段括起字符;escaped by转义字符,默认的是反斜杠（backslash：\ ),例如：
		load data infile "/home/mark/Orders txt" replace into table Orders fields terminated by',' enclosed by '"'; 
	2. lines 关键字指定了每条记录的分隔符,默认为'\n';即为换行符如果两个字段都指定了那fields必须在lines之前。如果不指定fields关键字缺省值与如果你这样写的相同： 
		fields terminated by'\t' enclosed by ’ '' ‘ escaped by'\\' 
5. load data infile 可以按指定的列把文件导入到数据库中。 当我们要把数据的一部分内容导入的时候，，需要加入一些栏目（列/字段/field）到MySQL数据库中，以适应一些额外的需要。比方说，我们要从Access数据库升级到MySQL数据库的时候
下面的例子显示了如何向指定的栏目(field)中导入数据： 
	load data infile "/home/Order txt" into table Orders(Order_Number, Order_Date, Customer_ID); 
6. 当在服务器主机上寻找文件时，服务器使用下列规则： 
	1. 如果给出一个绝对路径名，服务器使用该路径名。 
	2. 如果给出一个有一个或多个前置部件的相对路径名，服务器相对服务器的数据目录搜索文件。
	3. 如果给出一个没有前置部件的一个文件名，服务器在当前数据库的数据库目录寻找文件。 
	例如： /myfile txt”给出的文件是从服务器的数据目录读取，而作为“myfile txt”给出的一个文件是从当前数据库的数据库目录下读取。
	注意：字段中的空值用\N表示

#### load data infile 安全问题

LOAD DATA 默认读的是服务器上的文件，但是加上LOCAL参数后，就可以将本地具有访问权限的文件加载到数据库中。这在带来方便的同时。也带来了以下安全问题：
1. 可以任意加载本地文件到数据库。
2. 在WEB环境中，客户从WEB服务器连接，用户可以使用LOAD DATA LOCAL语句来读取WEB服务器进程有读访问权限的任何文件(假定用户可以运行SQL服务器的任何命令)。在这种环境中，MySQL服务器的客户实际上是WEB服务器，而不是连接WEB服务器的用户运行的程序。

**解决方法**

1. 可以用--local-infile=0选项启动mysqld从服务器端禁用所有LOAD DATA LOCAL命令。
即是在/etc/my.cnf的[mysqld]下面添加local-infile=0选项。

2. 对于mysql命令行的客户端，可以通指定--local-infile[=1]选项启用LOAD DATA LOCAL命令，或通过--local-infile=0选项禁用。
类似地，对于mysqlimport,--local or -L选项启用本地数据库文件装载。在任何情况下，成功进行本地装载需要服务器启有相关选项。
