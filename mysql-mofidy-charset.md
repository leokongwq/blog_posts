---
layout: post
comments: true
title: mysql修改表-字段-库的字符集
date: 2016-12-27 16:13:38
tags:
- mysql
categories:
- database
---

### 修改数据库字符集

    ALTER DATABASE db_name DEFAULT CHARACTER SET character_name [COLLATE ...];
    
### 修改表默认字符集

    ALTER TABLE tbl_name DEFAULT CHARSET character_name [COLLATE...];
    
### 修改表和字段默认的字符集    

    ALTER TABLE tbl_name CONVERT TO CHARACTER SET character_name [COLLATE ...]

DBA建议修改表的字符集如下操作：

1. 先修改表的默认字符集
2. 再进行字符集转换

例如：

        ALTER TABLE test DEFAULT CHARSET utf8;
        ALTER TABLE test CONVERT TO CHARACTER SET utf8; 
        
### 修改字段的字符集：               

    ALTER TABLE tbl_name CHANGE c_name c_name CHARACTER SET character_name [COLLATE ...];
    如：ALTER TABLE logtest CHANGE title title VARCHAR(100) CHARACTER SET utf8 COLLATE utf8_general_ci;
    
### 查看数据库编码：

    SHOW CREATE DATABASE db_name;

### 查看表编码：

    SHOW CREATE TABLE tbl_name;

### 查看字段编码：

    SHOW FULL COLUMNS FROM tbl_name;    
        


