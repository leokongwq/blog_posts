---
layout: post
comments: true
title: mysql常用工具SQL
date: 2017-11-10 09:57:06
tags:
categories:
- mysql
---

### 背景

在工作中经常会遇到一些非常有用的SQL，记录下来以备不时之需。


### 查看数据库大小

```sql
select concat(round(sum(data_length) /1024/1024/1024, 2), 'GB') as '数据大小', concat(round(sum(index_length) /1024/1024/1024, 2), 'GB') as '索引大小' 
from information_schema.tables 
where table_schema = 'test';
```

### 查看数据库每个表的大小

```sql
select table_name, concat(round(data_length /1024/1024/1024, 2), 'GB') as 'table_size', concat(round(index_length /1024/1024/1024, 2), 'GB') as 'index_size' 
from information_schema.tables 
where table_schema = 'test'
order by table_size desc;
```




