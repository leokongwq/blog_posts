---
layout: post
comments: true
title: mysql数据类型-字符串类型
date: 2017-08-12 17:44:56
tags:
- mysql
categories:
- mysql
---

### 字符串类型

字符串类型指CHAR、VARCHAR、BINARY、VARBINARY、BLOB、TEXT、ENUM和SET。该节描述了这些类型如何工作以及如何在查询中使用这些类型。

<!-- more -->

| 类型 | 大小(字符) | 用途 |
| --- | --- | --- |
| char | 0-255字符 | 定长字符串 |
| varchar | 0-65535字符 | 变成字符串 |
| TINYBLOB | 0-255字符 | 不超过 255 个字符的二进制字符串 |
| BLOB | 0-65535字符 | 二进制形式的长文本数据 |
| MEDIUMBLOB | 0-16777215字符 | 二进制形式的中等长度文本数据 |
| LONGBLOB | 0-4294967295字符 | 二进制形式的极大文本数据 |
| TINYTEXT | 0-255字符 | 短文本字符串 |
| TEXT | 0-65535字符 | 长文本数据 |
| MEDIUMTEXT | 0-16777215字符 | 中等长度文本数据 |
| LONGTEXT | 0-4294967295字符 | 极大文本数据 |


### char和varchar

- char类型是定长的，如果字段值长度小于声明的长度则右边补齐空格，查询的时候会去掉空格。
- varchar类型是变长的，如果字段长度小于声明的长度则不会**补齐空格**，**存储和查询的时候也会保留末尾的空格**。
- char和varchar存储的时候，最前面会有1个(char)，或1-2个(varchar)字节来存储字符串的长度。
- 在非严格SQL模式下，给一个char或varchar字段赋值时，如果char和varchar字段的长度超过了声明的长度，则字段值会被截断，此时会有警告信息，如果使用严格SQL Mode 则会触发一个错误。具体参考[Server SQL Modes](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html)
- char类型字段在发生截断的时候，如果截断的是空格，则什么提示都没有。如果是varchar类型的字段发生空格截断，则不论SQL MODE都会触发警告信息。

下面的表格用来展示char和varchar的在存储不同字符串时结果的不同：

| Value | char(40） | Storage Required | varchar(4) | Storage Required |
| --- | --- | --- | --- | --- |
| '' | ‘    ' | 4 bytes | '' | 1 byte |
| ‘ab' | ‘ab  ' | 4 bytes | 'ab' | 3 bytes |
| ‘abcd' | ‘abcd' | 4 bytes | ‘abcd' | 5 bytes |
| ‘abcdefg' | ‘abcd' | 4 bytes | ‘abcd' | 5 bytes |

表格最后一行中的内容表示在不使用严格模式的SQL MODE时需要的空间，否则在保存数据时会报错。

InnoDB将字节长度可能大于或等于768字节的固定长度字段编码为可变长字段存储。举个例子:`char(255)`，该列在编码为`utf8mb4`时，实际的字节长度可能超过768。

如果一个给定的值插入到`char(4)`和`varchar(4)`类型的列中，在随后的查询结果中这两个列不总是相同的，这是因为在查询中`char`类型类的末尾空格会被去掉。看下面的列子：

```sql
mysql> CREATE TABLE vc (v VARCHAR(4), c CHAR(4));
Query OK, 0 rows affected (0.01 sec)

mysql> INSERT INTO vc VALUES ('ab  ', 'ab  ');
Query OK, 1 row affected (0.00 sec)

mysql> SELECT CONCAT('(', v, ')'), CONCAT('(', c, ')') FROM vc;
+---------------------+---------------------+
| CONCAT('(', v, ')') | CONCAT('(', c, ')') |
+---------------------+---------------------+
| (ab  )              | (ab)                |
+---------------------+---------------------+
1 row in set (0.06 sec)
```

存储在`char`和`varchar`类型列中的值的排序和比较是根据建表是设置的字符集校验规则进行的。

所有MySQL的校验规则都是`PAD SPACE`类型的，这也就意味着所有的`char`,`varcha`,`text`类型的字段比较都是忽略末尾的空格的。这个上下文中所说的比较是不包含`LIKE`这种模式匹配操作符的。详情请看下面的例子：

```sql
mysql> CREATE TABLE names (myname CHAR(10));
Query OK, 0 rows affected (0.03 sec)

mysql> INSERT INTO names VALUES ('Monty');
Query OK, 1 row affected (0.00 sec)

mysql> SELECT myname = 'Monty', myname = 'Monty  ' FROM names;
+------------------+--------------------+
| myname = 'Monty' | myname = 'Monty  ' |
+------------------+--------------------+
|                1 |                  1 |
+------------------+--------------------+
1 row in set (0.00 sec)

mysql> SELECT myname LIKE 'Monty', myname LIKE 'Monty  ' FROM names;
+---------------------+-----------------------+
| myname LIKE 'Monty' | myname LIKE 'Monty  ' |
+---------------------+-----------------------+
|                   1 |                     0 |
+---------------------+-----------------------+
1 row in set (0.00 sec)
```

这个规则对每个版本的MySQL都是适用的，并且不受服务的SQL模式的影响。

### BLOB 和 TEXT 

一个`BLOB`表示一个二进制大对象，可以用来保存可变的数据。总共有四种类型的`BLOB`，分别是`TINYBLOB`,`BLOB`,`MEDIUMBLOB`,`LONGBLOB`。它们之间的区别仅在于能保存的最大数据长度不同。四个`TEXT`类型分别是`TINYTEXT`,`TEXT`,`MEDIUMTEXT`,`LONGTEXT`。这四个文本类型和四个BLOB相对应，拥有相同的最大长度限制和存储需要。详情可以参考:[Data Type Storage Requirements](https://dev.mysql.com/doc/refman/5.7/en/storage-requirements.html)

`BLOB`的值被当做二进制字符串。它们拥有二进制字符集和校验规则，比较和排序都是基于列中保存字节的数字值来进行。`TEXT`被当做非二进制字符串。它们拥有和二进制不同的字符集，比较和排序也是基于列字符集的校验规则。

如果在没有启用严格SQL模式下，给一个`BLOG`或`TEXT`列进行赋值，并超过了列声明的最大长度，则列的值会被截断来满足列的最大值限制，并且会有警告提示。针对非空白字符截断的情况，你可以通过设置SQL模式来出发一个错误（而不是一个警告）并且可以阻止这样值得插入。具体参考:[Server SQL Modes](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html)

不论哪种类型的SQL模式，截断将要插入到`TEXT`类型字段值的超长空白字符总是会发出告警信息。

对 `TEXT` 和 `BLOB` 类型的列来说， 它们在插入时没有补齐空格操作，在查询时也没有空格被去掉。

如果一个`TEXT`类型的列被索引了，索引记录会被进行右补齐操作。这意味着：如果索引需要唯一的值，则在插入只有空白字符长度不同的值时会触发一个`duplicate-key`错误。举个例子：如果一个表包含'a'，并想要插入一条'a '的记录，则会触发重复键错误。但这个规则不使用于`BLOB`类型的列。

在大多数请情况下你都可以将一个`BLOB`类型的列当做`VARBINARY`类型的列来使用。类似的，你也可以将`TEXT`类型的列当做`VARCHAR`来使用。 `BLOB` 和 `TEXT` 与 `VARBINARY` 和 `VARCHAR` 有一下几点不同：

- 在`BLOB` 和 `TEXT` 类型的列上建索引，你必须指定索引的前缀长度。而对`CHAR` 和 `VARCHAR`来说，前缀长度是可选的。具体参考:[Column Indexes](https://dev.mysql.com/doc/refman/5.7/en/column-indexes.html)
- `BLOB` 和 `TEXT` 类型的列不能有默认值。

如果你使用的是一个拥有`BINARY`属性的`TEXT`类型的字段，则会给该列分配二进制（_bin）的校验字符集。

`LONG`和`LONG VARCHAR`映射到`MEDIUMTEXT`数据类型。 这是兼容性功能。

MySQL `Connector/ODBC` 将 `BLOB` 类型的值作为`LONGVARBINARY`，将`TEXT`作为`LONGVARCHAR`。

###  BINARY 和 VARBINARY

BINARY和VARBINARY类型与CHAR和VARCHAR类似，但它们包含二进制字符串而不是非二进制字符串。 也就是说，它们包含字节字符串而不是字符串。 这意味着它们具有二进制字符集和排序规则，并且比较和排序基于值中的字节的数值。

对于BINARY和VARBINARY，允许的最大长度与CHAR和VARCHAR相同，但BINARY和VARBINARY的**长度以字节为单位**，而不是字符长度。

BINARY和VARBINARY数据类型与CHAR BINARY和VARCHAR BINARY数据类型不同。 对于后一种类型，BINARY属性不会导致列被视为二进制字符串列，它只会导致该列使用二进制（_bin）字符集的排序规则，而列本身包含非二进制字符串而不是二进制字节字符串。 例如，`CHAR(5)BINARY`被视为`CHAR(5)CHARACTER SET latin1 COLLATE latin1_bin`，假设默认字符集为`latin1`。 这与`BINARY(5)`不同，`BINARY(5)`存储具有二进制字符集和排序规则的5字节二进制字符串。 有关二进制字符串与非二进制字符串的二进制排序规则之间的差异的信息可以参考[https://dev.mysql.com/doc/refman/5.7/en/charset-binary-collations.html](https://dev.mysql.com/doc/refman/5.7/en/charset-binary-collations.html)

### 参考

[https://dev.mysql.com/doc/refman/5.7/en/string-types.html](https://dev.mysql.com/doc/refman/5.7/en/string-types.html)


