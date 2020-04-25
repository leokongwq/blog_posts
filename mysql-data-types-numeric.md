---
layout: post
comments: true
title: mysql数据类型-数字类型
date: 2017-08-16 20:50:14
tags:
- mysql
categories:
- mysql
---

### 前言

MySQL支持的数字类型很多，具体有如下类型：

整形：INTEGER, INT, SMALLINT, TINYINT, MEDIUMINT, BIGINT
浮点型：FLOAT, DOUBLE
定点型：DECIMAL, NUMERIC
位类型：BIT

<!-- more -->

### 整数类型

MySQL支持SQL标准的整数类型：INTEGER (或 INT) 和 SMALLINT。作为对SQL标准的扩展，MySQL同时支持的整数类型有 TINYINT，MEDIUMINT和BIGINT。下面的表格汇总MySQL支持的所有整数类型和每个类型需要的存储空间大小以及取值范围。

#### 整数类型范围和存储

| Type | Storage | Minimum Value | Maximum Value |
| --- | --- | --- | --- |
|  | (Bytes) | (Signed/Unsigned) | (Signed/Unsigned) |
| TINYINT | 1 | -128 | 127 |
|  |  | 0 | 255 |
| SMALLINT | 2 | -32768 | 32767  |
|  |  | 0 | 65535 |
| MEDIUMINT | 3 | -8388608 | 8388607 |
|  |  | 0 | 16777215 |
| INT | 4 | -2147483648 | 2147483647 |
|  |  | 0 | 4294967295 |
| BIGINT | 8 | -9223372036854775808 | 9223372036854775807  |
|  |  | 0 | 18446744073709551615 |

#### 整数使用

有时候我们能看到表字段类型声明时使用这样的方式：`int(11)`，括号里面的`11`是什么意思呢?

MySQL的官方文档是这样解释的(TINYINT[(M)] [UNSIGNED] [ZEROFILL])：

> M indicates the maximum display width for integer types. The maximum display width is 255. Display width is unrelated to the range of values a type can contain, as described in Section 11.2, “Numeric Types”. For floating-point and fixed-point types, M is the total number of digits that can be stored.

意思就是`M`是用来指示列的最大显示宽度的。最大的显示宽度是255。而且这个值和该字段能保存的值范围没有任何关系。针对浮点类型或定点类型字段，`M`表示该字段能保存的所有数字字符数。

举个例子：一个int类型声明为：

```sql
age int(3) unsigned default 0
```

这样的类型声明并不影响`age`字段能保存多余3位的数字。此时还看不出来`M`的作用，如果字段声明的格式如下：

```sql
age int(3) unsigned zerofill default 0
```

我们查询该列：

```sql
mysql> select age from emp where id > 1;
+------+
| age  |
+------+
|  018 |
|  122 |
+------+
2 rows in set (0.00 sec)
```

从上面的查询结果可以知道，如果该字段的字符数小于字段什么的显示宽度`M`并且在创建表时指定了`zerofill`属性，则在保存时，如果值的长度小于宽度`M`则会在左边补齐0。

#### zerofill的作用

1. 插入数据时，当该字段的值的长度小于定义的长度时，会在该值的前面补上相应的0
2. `zerofill`默认为`int(10)`
3. 当使用`zerofill`时，默认会自动加`unsigned`（无符号）属性，使用unsigned属性后，数值范围是原值的2倍，例如，有符号为-128~+127，无符号为0~256

### 浮点类型

`FLOAT`，`DOUBLE`类型用来表示一个数据的近似值。MySQL使用4字节保存单精度的值，8字节保存双精度的值。

FLOAT类型用于表示近似数值数据类型。SQL标准允许在关键字FLOAT后面的括号内选择用位指定精度(但不能为指数范围)。MySQL还支持可选的只用于确定存储大小的精度规定。0到23的精度对应FLOAT列的4字节单精度。24到53的精度对应DOUBLE列的8字节双精度。

MySQL允许使用非标准语法：`FLOAT(M,D)`或`REAL(M,D)`或`DOUBLE PRECISION(M,D)`。这里，`(M,D)`表示该值一共显示M位整数，其中D位位于小数点后面。例如，定义为`FLOAT(7,4)`的一个列可以显示为-999.9999。MySQL保存值时进行四舍五入，因此如果在`FLOAT(7,4)`列内插入999.00009，近似结果是999.0001。

MySQL将`DOUBLE`视为`DOUBLE PRECISION(非标准扩展)`的同义词。MySQL还将REAL视为DOUBLE PRECISION(非标准扩展)的同义词，除非SQL服务器模式包括REAL_AS_FLOAT选项。

因为单精度的值是一个近似值，如果尝试将它们用在精确比较中就可能产生问题。并且这还和具体的平台和实现有关。更多信息可以参考：[Problems with Floating-Point Values](https://dev.mysql.com/doc/refman/5.5/en/problems-with-float.html)

为了保证最大可能的可移植性，需要使用近似数值数据值存储的代码应使用FLOAT或DOUBLE PRECISION，不规定精度或位数。

DECIMAL和NUMERIC类型在MySQL中视为相同的类型。它们用于保存必须为确切精度的值，例如货币数据。当声明该类型的列时，可以(并且通常要)指定精度和标度；例如：

    salary DECIMAL(5,2)

在该例子中，5是精度，2是标度。精度表示保存值的主要位数，标度表示小数点后面可以保存的位数。

double 和 float 的区别是double精度高，有效数字16位（float精度7位）。但double消耗内存是float的两倍，占8字节，double的运算速度比float慢得多。

```sql
msyql> create table tc_float(fid int primary key auto_increment,f_float float, f_float10 float(10), f_float25 float(25), f_float7_3 float(7,3), f_float9_2 float(9,2), f_float30_3 float(30,3), f_decimal9_2 decimal(9,2));
mysql> insert into tc_float(f_float,f_float10,f_float25) values(123456,123456,123456);
mysql> insert into tc_float(f_float,f_float10,f_float25) values(1234567.89,12345.67,1234567.89);
mysql> select * from tc_float;
+-----+----------+-----------+------------+------------+------------+-------------+--------------+
| fid | f_float  | f_float10 | f_float25  | f_float7_3 | f_float9_2 | f_float30_3 | f_decimal9_2 |
+-----+----------+-----------+------------+------------+------------+-------------+--------------+
|   1 |   123456 |    123456 |     123456 | NULL       | NULL       | NULL        | NULL         |
|   2 |  1234570 |   12345.7 | 1234567.89 | NULL       | NULL       | NULL        | NULL         |
+-----+----------+-----------+------------+------------+------------+-------------+--------------+
```

可以看到float与float(10)是没区别的，float默认能精确到6位有效数字

```sql
mysql> insert into tc_float(f_float9_2,f_decimal9_2) values(123456.78,123456.78);
mysql> insert into tc_float(f_float9_2,f_decimal9_2) values(1234567.1,1234567.125);
Query OK, 1 row affected, 1 warning (0.00 sec)
mysql> show warnings;
+-------+------+---------------------------------------------------+
| Level | Code | Message                                           |
+-------+------+---------------------------------------------------+
| Note  | 1265 | Data truncated for column 'f_decimal9_2' at row 1 |
+-------+------+---------------------------------------------------+
1 row in set (0.00 sec)
mysql> select * from tc_float;
+-----+----------+-----------+------------+------------+------------+-------------+--------------+
| fid | f_float  | f_float10 | f_float25  | f_float7_3 | f_float9_2 | f_float30_3 | f_decimal9_2 |
+-----+----------+-----------+------------+------------+------------+-------------+--------------+
|   6 | NULL     | NULL      | NULL       | NULL       |  123456.78 | NULL        |    123456.78 |
|   9 | NULL     | NULL      | NULL       | NULL       | 1234567.12 | NULL        |   1234567.13 |
+-----+----------+-----------+------------+------------+------------+-------------+--------------+
mysql> insert into tc_float(f_float7_3) values(12345.1);
ERROR 1264 (22003): Out of range value for column 'f_float7_3' at row 1
```

- float(9,2)与decimal(9,2)是很像的，并没有前面提到24位一下6位有效数字的限制
- 他们俩之间的差别就在精度上，f_float9_2本应该是 1234567.10，结果小数点变成 .12 。f_decimal9_2因为标度为2，所以 .125 四舍五入成 .13
- 将 12345.1 插入f_float7_3列，因为转成标度3时 12345.100，整个位数大于7，所以 out of range 了

### 定点类型

The DECIMAL and NUMERIC types store exact numeric data values. These types are used when it is important to preserve exact precision, for example with monetary data. In MySQL, NUMERIC is implemented as DECIMAL, so the following remarks about DECIMAL apply equally to NUMERIC.

MySQL stores DECIMAL values in binary format. See Section 12.18, “Precision Math”.

In a DECIMAL column declaration, the precision and scale can be (and usually is) specified; for example:

```sql
salary DECIMAL(5,2)
```

In this example, 5 is the precision and 2 is the scale. The precision represents the number of significant digits that are stored for values, and the scale represents the number of digits that can be stored following the decimal point.

Standard SQL requires that DECIMAL(5,2) be able to store any value with five digits and two decimals, so values that can be stored in the salary column range from -999.99 to 999.99.

In standard SQL, the syntax DECIMAL(M) is equivalent to DECIMAL(M,0). Similarly, the syntax DECIMAL is equivalent to DECIMAL(M,0), where the implementation is permitted to decide the value of M. MySQL supports both of these variant forms of DECIMAL syntax. The default value of M is 10.

If the scale is 0, DECIMAL values contain no decimal point or fractional part.

The maximum number of digits for DECIMAL is 65, but the actual range for a given DECIMAL column can be constrained by the precision or scale for a given column. When such a column is assigned a value with more digits following the decimal point than are permitted by the specified scale, the value is converted to that scale. (The precise behavior is operating system-specific, but generally the effect is truncation to the permissible number of digits.)

### BIT

The BIT data type is used to store bit values. A type of BIT(M) enables storage of M-bit values. M can range from 1 to 64.

To specify bit values, b'value' notation can be used. value is a binary value written using zeros and ones. For example, b'111' and b'10000000' represent 7 and 128, respectively. See Section 9.1.5, “Bit-Value Literals”.

If you assign a value to a BIT(M) column that is less than M bits long, the value is padded on the left with zeros. For example, assigning a value of b'101' to a BIT(6) column is, in effect, the same as assigning b'000101'.

NDB Cluster.  The maximum combined size of all BIT columns used in a given NDB table must not exceed 4096 bits.


### 参考资料

[numeric-type-overview](https://dev.mysql.com/doc/refman/5.7/en/numeric-type-overview.html)
[integer-types](https://dev.mysql.com/doc/refman/5.7/en/integer-types.html)

[MySQL 数据类型（float）的注意事项](http://www.cnblogs.com/zhoujinyi/archive/2013/04/26/3043160.html)


