---
layout: post
comments: true
title: mysql数据类型-枚举类型
date: 2017-08-15 10:41:16
tags:
- mysql
categories:
- mysql
---

### ENUM 类型

`ENUM`是一个字符串对象，它的取值范围是在创建表时指定的列表。它具有以下优点：

- 当一个列取值是有限范围时，可以压缩数据存储。输入的字符串会被自动编码为数字存储。具体参考：[Data Type Storage Requirements](https://dev.mysql.com/doc/refman/5.7/en/storage-requirements.html)
- 在查询和输出时可读性更好。在查询结果中，存储的数字会被还原为字符串。但是需要考虑一下潜在的问题：
    - 如果你将枚举类型字段的取值设置为数字时，很容易将字面量的值和MySQL内部存储的索引值混淆。详情参考:[枚举类型的限制](./mysql-data-types-enum.html#枚举类型的限制)
    - 枚举类型的列用在`order by`子句中需要特别注意，详情参考：[枚举列排序](./mysql-data-types-enum.html#枚举列排序).
    - [创建并使用枚举类型的列](./mysql-data-types-enum.html#创建并使用枚举类型的列)
    - [枚举元素的索引](./mysql-data-types-enum.html#枚举元素的索引)
    - [枚举字面量处理](./mysql-data-types-enum.html#枚举字面量处理)
    - [空值或NULL值](./mysql-data-types-enum.html#空值或NULL值)
    - [枚举列排序](./mysql-data-types-enum.html#枚举列排序)
    - [枚举类型的限制](./mysql-data-types-enum.html#枚举类型的限制)

<!-- more -->

### 创建并使用枚举类型的列

一个枚举值必须是一个引号括起来的字符串常量。例如你可以使用下面这种方式来创建一个带有枚举字段的表：

```sql
CREATE TABLE shirts (
    name VARCHAR(40),
    size ENUM('x-small', 'small', 'medium', 'large', 'x-large')
);
INSERT INTO shirts (name, size) VALUES ('dress shirt','large'), ('t-shirt','medium'),
  ('polo shirt','small');
SELECT name, size FROM shirts WHERE size = 'medium';
+---------+--------+
| name    | size   |
+---------+--------+
| t-shirt | medium |
+---------+--------+
UPDATE shirts SET size = 'small' WHERE size = 'large';
COMMIT;
```

向该表插入100万条记录，其中`size`的值为`medium`，所有记录size字段总共需要100万字节。如果存储对应的字符串值，则需要使用600万字节。

### 枚举元素的索引

每一个枚举值都有一个索引：

- 在创建表时什么的每一个枚举值都指定了一个从`1`开始的索引。
- 如果枚举字段的值为空字符串，或错误的值，该索引的值为0。这就以为这你可以使用下面这种查询方式找出取值不正常的记录：`mysql> SELECT * FROM tbl_name WHERE enum_col=0;`
- 字段值为NULL，对应的索引也为NULL
- 这里所所有的索引`index`仅仅指的是枚举字段取值列表的下标，和表的查询索引没有任何关系。

例如：一个枚举类型的字段`ENUM('Mercury', 'Venus', 'Earth')`可以拥有这里出现的任何值，对应的索引值如下所示：

| Value | index |
| --- | --- |
| NULL | NULL |
| '' | 0 |
| 'Mercury' | 1  |
| 'Venus' | 2 |
| 'Earth' | 3 |

一个枚举类型的列最多可以有65535个不同的值（最佳实践应该小于3000）。一个表中，所有set和enum类型的列不能超过超过255个。关于这些限制的详细信息请参考:[Limits Imposed by .frm File Structure](https://dev.mysql.com/doc/refman/5.7/en/limits-frm-file.html)

如果你在数字环境中获取枚举字段的值，则枚举元素的索引会被返回。例如：

```sql
mysql> SELECT enum_col+0 FROM tbl_name;
```

像`SUM()`和`AVG()`这类函数，它们需要的参数类型都是数值类型，如果需要它们会将参数转为数字类型。对枚举类型子弹来说使用的就是它的枚举值。

### 枚举字面量处理

在表定义时，枚举类型取值列表中的元素会字段被去掉控制。

在查询枚举字段时，展示的值是表定义是枚举元素的小写格式。需要注意的是枚举字段可以设置字符集和校验字符集。对于二进制或大小写敏感的校验集，

如果你将一个数字保存到枚举字段中，则该数字会被当做枚举值列表的索引（然而，该规则并不适用于`LOAD DATA xxxx 语句，因为它会将所有的输入当做字符串`）。需要注意的是这个数字不能超过枚举字段取值列表的长度，否则报`Data truncated for column 'xxx' at row 1`错误。如果该数字是被引号括起来的数字并且在取值列表中没有查找到该字符串，则该值任然会被当做索引。由于这些原因，建议在使用枚举类型的字段时不要讲枚举元素定义为数值类型，因为这样很容易使人感到困惑。例如，一个枚举类型的字段包含的元素是`0`,`1`,`2`，但是数值索引是1,2,3：

```sql
numbers ENUM('0','1','2')
```

如果将`2`保存到numbers字段，它会被当做索引`2`。当你在查询时，返回的字段值显示为`1`。如果你保存'2'到字段numbers，因为枚举值列表中有'2'这个元素，因此在查询时返回的就是'2'。

```sql
mysql> INSERT INTO t (numbers) VALUES(2),('2'),('3');
mysql> SELECT * FROM t;
+---------+
| numbers |
+---------+
| 1       |
| 2       |
| 2       |
+---------+
```

想要检测一个枚举字段所有可能的取值，你可以使用语句`SHOW COLUMNS FROM tbl_name LIKE 'enum_col'` and 

### 空值或NULL值

一个枚举字段的值在下面的情况下可以是一个空字符串''或NULL

- 如果将一个不存在于枚举值列表中的值保存到枚举字段时，一个空字符串作为一个特殊的错误值将被保存到该字段。该字符串可以和正常的空字符串区分开来，因为它的值是'0'。详情可以参考：[enum-indexes](https://dev.mysql.com/doc/refman/5.7/en/enum.html#enum-indexes)
- 如果一个枚举字段在创建表时声明为可以为NULL，则一个NULL值对该列来说就是一个合法的值，并且默认值就是NULL。如果枚举类型的列声明为不允许为NULL， 则它的默认值就是它的第一个元素。

### 枚举列排序

枚举类型的字段排序是基于它们的索引的，而每个元素的索引是基于建表时声明的元素顺序的。举个例子：对于字段`ENUM('b', 'a')` 'b' 排在 'a' 前面。空字符串排在飞空字符串前，`NULL`值排在所有枚举元素前面。

可以使用下面的办法来避免枚举字段在`order by`子句中的默认行为：

- 在指定枚举元素时按照字母序排序。
- 确保枚举字段排序时使用字典序而不是按照枚举元素的索引值，方法是：`ORDER BY CAST(col AS CHAR)` 或 `ORDER BY CONCAT(col)`。

### 枚举类型的限制

一个枚举值不能是一个表达式，即使表达式的计算结果是一个字符串。

举个例子，下面的建表语句不能正确执行，因为它在枚举列上使用了函数`concat`

```sql
CREATE TABLE sizes (
    size ENUM('small', CONCAT('med','ium'), 'large')
);
```

同样的，枚举列也能使用用户定义的变量：

```sql
SET @mysize = 'medium';

CREATE TABLE sizes (
    size ENUM('small', @mysize, 'large')
);
```

Duplicate values in the definition cause a warning, or an error if strict SQL mode is enabled.

强烈建议枚举列的的元素不要使用数字。如果在定义枚举列时有重复的元素出现，通常会触发一个警告，如果服务器在严格SQL模式下，则会触发一个错误。

### 参考

[https://dev.mysql.com/doc/refman/5.7/en/enum.html](https://dev.mysql.com/doc/refman/5.7/en/enum.html)







