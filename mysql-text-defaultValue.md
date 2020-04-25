---
layout: post
comments: true
title: mysql-text-defaultValue
date: 2017-01-09 16:34:10
tags:
categories:
---

### 前言

今天在修改MySQL表字段的类型为text类型时，修改语句设置了默认值，但是查看表结构信息时发现该字段还是没有默认值。Google得出以下结论。

<!-- more -->

### 结论

mysql text类型没有默认值，如果该字段没有值，则该字段是空，即`is null`

使用select语句时应注意：(test是表名，description是字段名，类型是text) 

    select * from test where description = null;   
    
等价为 

    select * from test where description = 'null'; 

即此时description 值是null才可以取出。 

如果description字段没有填入值，是系统设置的，则执行 

    select * from test where description is null;

即可。

当然了，MySQL也不是非得限制你不能为text类型字段设置默认值。可以通过修改my.cnf来改变该配置

    sql-mode="STRICT_TRANS_TABLES,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION"
    //改为
    sql_mode='MYSQL40'
    
    
### 参考资料

[http://stackoverflow.com/questions/3466872/why-cant-a-text-column-have-a-default-value-in-mysql](http://stackoverflow.com/questions/3466872/why-cant-a-text-column-have-a-default-value-in-mysql)

[http://love-love-l.blog.163.com/blog/static/2107830420106575332428/](http://love-love-l.blog.163.com/blog/static/2107830420106575332428/)

[http://dev.mysql.com/doc/refman/5.7/en/data-type-defaults.html](http://dev.mysql.com/doc/refman/5.7/en/data-type-defaults.html)
    

