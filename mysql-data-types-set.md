---
layout: post
comments: true
title: mysql数据类型-集合
date: 2017-08-15 20:53:40
tags:
- mysql
categories:
- mysql
---

`SET`是一个字符串对象，可以包含0个或多个值，每一个值都必须是在建表是指定的取值列表中出现的。`SET`类型的列的值可以是建表是取值列表中的一个或多个值，每个值用','分割。由此可以得出一个结论，集合中的元素不能包含','。

举个例子，一个列被声明为`SET('one', 'two') NOT NULL`，该列所有可能的取值如下所示：

```sql
''
'one'
'two'
'one,two'
```

<!-- more -->

一个`SET`类型的列最多能拥有64个不同的元素。一个表中，所有的枚举类型和集合类型的列加起来不能超过255。详细信息可以参考:[Limits Imposed by .frm File Structure](https://dev.mysql.com/doc/refman/5.7/en/limits-frm-file.html)

如果在定义集合类型的列时，如果有重复的元素出现，通常会触发一个警告，如果服务器在严格SQL模式下，则会触发一个错误。

在创建表时，结合中元素的空白字符会被自动剔除。

查询时，存储在SET类型列中的值将使用列定义中元素的字符串格式显示。 **注意:**`SET`类型的列可以分配一个字符集和排序规则。 对于二进制或区分大小写的排序规则，在将值保存到该列时，会考虑到字母的大小写格式。

MySQL以数字方式存储`SET`值，字节序的低位（1）对应于第一个成员，以此类推。 如果在数字上下文中查询SET类型列的值，则查询到的列值为所有bit位组成的数字。 例如，你可以以下面所示的方式查询SET类型的列：

```sql
mysql> SELECT set_col+0 FROM tbl_name;
```

如果直接将一个数字保存的SET类型的列中，首先会将计算该数字的二进制表示，然后查找所有的元素。例如一个SET类型的列`SET('a','b','c','d')`，集合中的每一个元素都有下面所示的10进行和二进制表示形式。


| SET Member | Decimal Value | Binary Value |
| --- | --- | --- |
| 'a' | 1 | 0001 |
| 'b' | 2 | 0010 |
| 'c' | 4 | 0100 |
| 'd' | 8 | 1000 |

如果你将数字9保存到该列上，对应的二进制格式为`1001`，根据上表的规则我们可以知道该列的值就是'a,b'。

如果将多于一个元素的值保存到SET类型的列上，不论插入值元素的顺序如何，同样，不论同一个元素在一个值中出现多少次。在查询时重复的元素会被删除，而且元素的显示顺序就是按建表时声明的顺序那样。看下面的例子：

```sql
mysql> CREATE TABLE myset (col SET('a', 'b', 'c', 'd'));
```

假设你插入的值为： 'a,d', 'd,a', 'a,d,d', 'a,d,a', and 'd,a,d':

```sql
mysql> INSERT INTO myset (col) VALUES 
-> ('a,d'), ('d,a'), ('a,d,a'), ('a,d,d'), ('d,a,d');
Query OK, 5 rows affected (0.01 sec)
Records: 5  Duplicates: 0  Warnings: 0
```

在查询时，所有的值都显示为：'a,d'

```sql
mysql> SELECT col FROM myset;
+------+
| col  |
+------+
| a,d  |
| a,d  |
| a,d  |
| a,d  |
| a,d  |
+------+
5 rows in set (0.04 sec)
```

如果一个插入的值包含一个SET类型的列声明时所不存在的元素，通常情况下会触发一个警告：

```sql
mysql> INSERT INTO myset (col) VALUES ('a,d,d,s');
Query OK, 1 row affected, 1 warning (0.03 sec)

mysql> SHOW WARNINGS;
+---------+------+------------------------------------------+
| Level   | Code | Message                                  |
+---------+------+------------------------------------------+
| Warning | 1265 | Data truncated for column 'col' at row 1 |
+---------+------+------------------------------------------+
1 row in set (0.04 sec)

mysql> SELECT col FROM myset;
+------+
| col  |
+------+
| a,d  |
| a,d  |
| a,d  |
| a,d  |
| a,d  |
| a,d  |
+------+
6 rows in set (0.01 sec)
```

如果设置了严格的SQL 模式， 则上面的SQL会触发一个错误。

SET values are sorted numerically. NULL values sort before non-NULL SET values.

集合类型的值是以数字保存的，NULL值总是排在非NULL值前面。

Functions such as SUM() or AVG() that expect a numeric argument cast the argument to a number if necessary. For SET values, the cast operation causes the numeric value to be used.

Normally, you search for SET values using the FIND_IN_SET() function or the LIKE operator:

```sql
mysql> SELECT * FROM tbl_name WHERE FIND_IN_SET('value',set_col)>0;
mysql> SELECT * FROM tbl_name WHERE set_col LIKE '%value%';
```

The first statement finds rows where set_col contains the value set member. The second is similar, but not the same: It finds rows where set_col contains value anywhere, even as a substring of another set member.

The following statements also are permitted:

```sql
mysql> SELECT * FROM tbl_name WHERE set_col & 1;
mysql> SELECT * FROM tbl_name WHERE set_col = 'val1,val2';
```

The first of these statements looks for values containing the first set member. The second looks for an exact match. Be careful with comparisons of the second type. Comparing set values to 'val1,val2' returns different results than comparing values to 'val2,val1'. You should specify the values in the same order they are listed in the column definition.

To determine all possible values for a SET column, use SHOW COLUMNS FROM tbl_name LIKE set_col and parse the SET definition in the Type column of the output.

In the C API, SET values are returned as strings. For information about using result set metadata to distinguish them from other strings, see Section 27.8.5, “C API Data Structures”.




