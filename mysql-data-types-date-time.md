---
layout: post
comments: true
title: mysql数据类型-日期&时间
date: 2017-08-17 20:20:08
tags:
- mysql
categories:
- mysql
---

### 前言

MySQL提供的日期和时间类型很多，如：DATE，TIME，DATETIME 和 TIMESTAMP。 每一种类型都有各自的特点和使用范围，而且随和MySQL版本的不同会有小的差异。这篇文章就就对MySQL的日期，时间类型进行总结。

<!-- more -->

### Year

`YEAR`类型是用一个字节来表示年的值的，可以用格式`YEAR` 或 `YEAR(4)`（4位显示宽度）来声明年类型。

> 注意
类型`YEAR(2)`已经过时了并且在MySQL 5.7.5版本中已经被移除了。将`YEAR(2)`类型的列转为`YEAR(4)`，可以参考:[YEAR(2) Limitations and Migrating to YEAR(4)](https://dev.mysql.com/doc/refman/5.7/en/migrating-to-year4.html)。

- YEAR 类型默认4位显示宽度
- YEAR(4) 和 YEAR(2) 拥有相同的取值范围
- YEAR(4) 显示年的格式是：YYYY（1901 到 2155 或 0000）
- YEAR(2) 只显示最后两位。70 (1970 或 2070) 或 69 (2069)

你可以用不同的格式来指定YEAR类型列的值：

- 使用范围为： 1901 到 2155 的四位数字
- 使用范围为： '1901' 到 '2155' 的四位字符
- 使用范围为1-99的两位数组。MySQL会将 `1 - 69` 转为 `2001 到 2069`，`70 - 99` 转为 `1970 到 1999` 
- 使用范围为1-99的两位数组。MySQL会将 `1 - 69` 转为 `2001 到 2069`，`70 - 99` 转为 `1970 到 1999`
- 对`YEAR(2)` 和 `YEAR(4)` 来说数字`0`的作用是不同的。针对`YEAR(2)` 显示的结果是`00`并且内部的值是`2000`；针对`YEAR(4)`，显示的结果是`0000` 内部的值是`0000`。 如果要指定`YEAR(4)`的零值，并且希望MySQL将其解释为`2000`，你需要将`'0'` 或 `'00'`指定为该列的值。
- 某些在年上下文中的函数返回值，例如`NOW()`。
- MySQL 将不合法的 `YEAR` 字段的值保存为`0000`。

### Date 

`DATE`类型用来保存只含有日期部分而不包含时间部分的值。MySQL 以格式`YYYY-MM-DD`来查询和展示`DATE`类型。
它支持的范围为：'1000-01-01' 到 '9999-12-31'。

### DATETIME

`DATETIME` 类型用来保存同时含有日期和时间部分的值。MySQL 以格式`YYYY-MM-DD HH:MM:SS`来查询和展示`DATETIME`类型。
它支持的范围为：`1000-01-01 00:00:00` 到 `9999-12-31 23:59:59`。

### TIMESTAMP

`TIMESTAMP` 类型同样用来保存同时含有日期和时间部分的值。`TIMESTAMP`保存的时间范围是UTC格式的，从`1970-01-01 00:00:01` 到 `2038-01-19 03:14:07`。

### DATETIME vs TIMESTAMP

`DATETIME`和`TIMESTAMP`类型中表示秒的部分都可以包含小数部分，该小数部分是用来表示毫秒的，最多有6个小数位。通常来说任何含有小数部分的值保存到`DATETIME`或`TIMESTAMP`类型的列中，小数部分都会被丢弃，除非在定义列类型时指定的时间格式为:`YYYY-MM-DD HH:MM:SS[.fraction]`。所以`DATETIME`值的范围可以是`1000-01-01 00:00:00.000000` 到 `9999-12-31 23:59:59.999999`，`TIMESTAMP`的范围可以是：`1970-01-01 00:00:01.000000` 到 `2038-01-19 03:14:07.999999`。小数部分总是应该和其它的时间部分用一个小数点进行区分。

从MySQL 5.6.4版本起，MySQL扩展了TIME，DATETIME，TIMESTAMP类型的秒部分，增加了对毫秒精度的支持，毫秒部分最多可以使用6位数字表示（0-6）。格式如：`type_name(fsp)` ，其中fsp的取值是：TIME，DATETIME，或TIMESTAMP。举个例子：

```sql
CREATE TABLE t1 (t TIME(3), dt DATETIME(6));
```

`fsp`默认值为0。如果指定`fsp`为0，相当于没有指定毫秒部分。与标准SQL默认值为6不同的原因是MySQL需要往前兼容。

如果指定的`fsp`位数小于保存的值，则多出的部分会进行四舍五入运行。例如：

```sql
mysql> CREATE TABLE fractest( c1 TIME(2), c2 DATETIME(2), c3 TIMESTAMP(2) );
Query OK, 0 rows affected (0.33 sec)

mysql> INSERT INTO fractest VALUES
     > ('17:51:04.777', '2014-09-08 17:51:04.777', '2014-09-08 17:51:04.777');
Query OK, 1 row affected (0.03 sec)

mysql> SELECT * FROM fractest;
+-------------+------------------------+------------------------+
| c1          | c2                     | c3                     |
+-------------+------------------------+------------------------+
| 17:51:04.78 | 2014-09-08 17:51:04.78 | 2014-09-08 17:51:04.78 |
+-------------+------------------------+------------------------+
1 row in set (0.00 sec)
```

这样的四舍五入运行没有任何警告和错误提示。该行为和SQL标准保持一致并且不受`sql_mode`设置的影响。

更多详情参考:[Fractional Seconds in Time Values](https://dev.mysql.com/doc/refman/5.7/en/fractional-seconds.html)

`DATETIME`和`TIMESTAMP`类型的列提供了自动初始和更新为当前时间的机制，具体参考： [Automatic Initialization and Updating for TIMESTAMP and DATETIME](https://dev.mysql.com/doc/refman/5.7/en/timestamp-initialization.html)。

在保存`TIMESTAMP`类型的值时，MySQL会将该值从当前时区转为`UTC`，在查询时又重新转为当前的时区。默认情况下，每个连接的时区都是取MySQL服务器端的时区。不过每个连接都可以设置自己的时区。只要时区设置保持一致，你就可以获取和你保存到数据时一样的值。如果你保存了一个`TIMESTAMP`类型的值并更改了MySQL服务端的时区信息，那么你查询出来的值就不是你当初保存进去的值。这种情况的出现是因为在时间转换是没有使用同样的时区信息。MySQL服务端的时区信息可以是系统环境变量`time_zone`的值。更多信息参考:[MySQL Server Time Zone Support](https://dev.mysql.com/doc/refman/5.7/en/time-zone-support.html)

不正确的`DATE`，`DATETIME` 或 `TIMESTAMP` 类型的值会被转为该类型对应的零值（'0000-00-00' or '0000-00-00 00:00:00'）。

**请注意MySQL中关于日期值某些属性的解释：**

- MySQL permits a “relaxed” format for values specified as strings, in which any punctuation character may be used as the delimiter between date parts or time parts. In some cases, this syntax can be deceiving. For example, a value such as '10:11:12' might look like a time value because of the :, but is interpreted as the year '2010-11-12' if used in a date context. The value '10:45:15' is converted to '0000-00-00' because '45' is not a valid month.
The only delimiter recognized between a date and time part and a fractional seconds part is the decimal point.
- The server requires that month and day values be valid, and not merely in the range 1 to 12 and 1 to 31, respectively. With strict mode disabled, invalid dates such as '2004-04-31' are converted to '0000-00-00' and a warning is generated. With strict mode enabled, invalid dates generate an error. To permit such dates, enable ALLOW_INVALID_DATES. See Section 5.1.8, “Server SQL Modes”, for more information.
- MySQL does not accept TIMESTAMP values that include a zero in the day or month column or values that are not a valid date. The sole exception to this rule is the special “zero” value '0000-00-00 00:00:00'.
- Dates containing two-digit year values are ambiguous because the century is unknown. MySQL interprets two-digit year values using these rules:
    - Year values in the range 00-69 are converted to 2000-2069.
    - Year values in the range 70-99 are converted to 1970-1999.

[Two-Digit Years in Dates](https://dev.mysql.com/doc/refman/5.7/en/two-digit-years.html)


### 存储

`TIME`，`DATETIME`和`TIMESTAMP`列类型的存储在MySQL版本5.6.4前和5.6.4后是不一样的。造成不同的原因是从5.6.4版本开始MySQL针对时间增加了时间小数部分的支持。这部分需要0到3个字节的长度。

| Data Type | Storage Required Before MySQL 5.6.4  | Storage Required as of MySQL 5.6.4 | |
| --- | --- | --- |
| YEAR | 1 byte | 1 byte |
| DATE | 3 byte | 3 byte |
| TIME | 3 byte | 3 bytes + fractional seconds storage  |
| DATETIME | 8 bytes  | 5 bytes + fractional seconds storage |
| TIMESTAMP | 4 bytes | 4 bytes + fractional seconds storage |

### 参考

[fractional-seconds](https://dev.mysql.com/doc/refman/5.6/en/fractional-seconds.html)

