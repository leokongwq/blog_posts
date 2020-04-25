---
layout: post
comments: true
title: Python常用的代码
date: 2017-07-22 08:43:08
tags:
categories:
- python
---

### 前言

对python不熟。但工作中偶尔需要除了一些和数据（mysql,mongodb等），文件等相关的问题。此时第一个想到的就是python（不喜欢shell的语法）。遇到需要使用的功能每次都要查API，这个就比较翻了。这篇文章就是讲自己常用的功能代码整理起来，方便以后使用。

<!-- more -->

### dict操作

#### dict遍历

dict的遍历有三种方式，分别如下所示：

```python
person = {
    "name":"lily",
    "age":12
}
#方式一
for entry in person:
    print "%s's is %s" % (entry, person[entry])
	
## 方式二	， 返回dict的一个(key, value)对list的拷贝		
for (k, v) in person.items():
    print "%s's is %s" % (k, v)
	
## 方式三， 返回一个(key, value)的迭代器
for (k, v) in person.iteritems():
    print "%s's is %s" % (k, v)	
```

#### 判断key是否存在

```python
person = {
    "name":"lily",
    "age":12
}
## 方法一
print person.has_key("school")
## 方法二
if ('school' in person):
    print True
else:
    print False    
```

#### 获取key对应的value

直接访问一个dict中的key对应的值是有风险的，有可能key是不存在。可以通过下面的方式安全的访问：

方式零：

先通过上面的方法判断，再访问。

方式一 ： 如果不存在，则返回默认的值

```python 
person = {
    "name":"lily",
    "age":12
}
print person.get('school', 'MIT')
```

方式二 ： 设置key的默认值

```python
person = {
    "name":"lily",
    "age":12
}
print person.setdefault('school', 'UCLA')
print person['school']
```

方式三：向类dict增加__missing__()方法，当key不存在时，会转向__missing__()方法处理，而不触发KeyError,如：

```python
person = {
    "name":"lily",
    "age":12
}
class Counter(dict):
    def __missing__(self, key):
        return None
c = Counter(t)
print(c['d'])
```

方式四：利用collections.defaultdict([default_factory[,...]])对象，实际上这个是继承自dict，而且实际也是用到的__missing__()方法，其default_factory参数就是向__missing__()方法传递的，不过使用起来更加顺手：如果default_factory为None，则与dict无区别，会触发KeyError错误，如：

```python
import collections
t = {
    'a': '1',
    'b': '2',
    'c': '3',
}
t = collections.defaultdict(None, t)
print(t['d'])
```
会出现：

```python
    KeyError: 'd'
```

但如果真的想返回None也不是没有办法：

```python
import collections
t = {
    'a': '1',
    'b': '2',
    'c': '3',
}

def handle():
    return None
    
t = collections.defaultdict(handle, t)
print(t['d'])
```
会出现：

```python
None
```

如果default_factory参数是某种数据类型，则会返回其默认值，如：

```python
import collections
t = {
    'a': '1',
    'b': '2',
    'c': '3',
}
t = collections.defaultdict(int, t)
print(t['d'])
```

会出现：

```python
0
```

又如：

```python
import collections
t = {
    'a': '1',
    'b': '2',
    'c': '3',
}
t = collections.defaultdict(list, t)
print(t['d'])
```

会出现：

```python
[]
```
**注意：**
如果dict内又含有dict，key嵌套获取value时，如果中间某个key不存在，则上述方法均失效，一定会触发KeyError

```python
import collections
t = {
    'a': '1',
    'b': '2',
    'c': '3',
}
t = collections.defaultdict(dict, t)
print(t['d']['y'])
```

会出现

```python
KeyError: 'y'
```

### 文件操作

#### 文件或文件夹是否存在

```python
import os
## 判断文件夹或文件是否存
print os.path.exists('/data/logs/')
print os.path.exists('/data/logs/a.txt')
```

#### 文件类型判断

```python
import os

print os.path.isfile('/data/logs/a.txt')
print os.path.isdir('/data/logs')
```

#### 文件创建

python中创建文件非常简单，如下所示：

```python
f = open('/home/tom/data/a.txt', 'w')
f.writelines('hello world')
f.close()
```

这种方式的前提是文件所在的目录必须存在，如果目前不存在，则会抛出如下的异常：

```python
IOError: [Errno 2] No such file or directory: '/data/a/a.txt'
```

可以通过下面的方式来实现安全的文件创建：

```python
import os
filePath = '/data/logs/a.txt'
if (not os.path.exists(os.path.dirname(filePath))):
    os.mkdir(os.path.dirname(filePath))
f = open(filePath, 'w')
f.writelines('hello world')
f.close()
```

#### 文件夹创建

```python
if (not os.path.exists(filePath)):
    os.mkdir(filePath)              
```

#### 文件夹遍历

方法一：使用`os.listdir`递归的处理文件夹

```python
def walkDir(dirPath):
    if(not os.path.exists(dirPath)):
        return
    for f in  os.listdir(dirPath):
        if os.path.isdir(dirPath + os.sep + f):
            printTree(dirPath + os.sep + f)
        else:
            print dirPath + os.sep + f
```

方法二：使用`os.walk`

`os.walk()` 原型为：

```python
os.walk(top, topdown=True, onerror=None, followlinks=False)
```

我们一般只使用第一个参数。（topdown指明遍历的顺序）
该方法对于每个目录返回一个三元组，(dirpath, dirnames, filenames)：
- 第一个是根路径
- 第二个是路径下面的目录
- 第三个是路径下面的非目录（对于windows来说也就是文件）。

例子：

```python
import os
from os.path import join, getsize
for root, dirs, files in os.walk(rootDir):
    print root, "consumes",
    print sum(getsize(join(root, name)) for name in files),
    print "bytes in", len(files), "non-directory files"
    if 'CVS' in dirs:
    	dirs.remove('CVS')  # don't visit CVS directories
```

#### 逐行读取文件

方法一：

```python
f = open("foo.txt")             # 返回一个文件对象  
line = f.readline()             # 调用文件的 readline()方法  
while line:  
    print line,                 # 后面跟 ',' 将忽略换行符  
    # print(line, end = '')　　　# 在 Python 3中使用  
    line = f.readline()  

f.close()  
```

方法二：

```python
for line in open("foo.txt"):  
    print line,
```

方法三：

```python
f = open("foo.txt", "r")
cnt = 0
for line in f.readlines():
    cnt = cnt + 1		

f.close()   
```

方法一和方法二速度太慢了。推荐使用方法三。

### 操作MySQL

#### 安装必要软件

CentOS 安装easy_install的方法

```shell
wget -q http://peak.telecommunity.com/dist/ez_setup.py
python ez_setup.py
```

CentOS安装python包管理安装工具pip的方法如下：

```shell
wget --no-check-certificate https://github.com/pypa/pip/archive/1.5.5.tar.gz
```

**注意**：wget获取https的时候要加上：--no-check-certificate

```shell
tar zvxf 1.5.5.tar.gz    #解压文件
cd pip-1.5.5/
python setup.py install
```

centos 安装MySQL-python

```shell
yum -y install mysql-dev
wget http://downloads.sourceforge.net/project/mysql-python/mysql-python-test/1.2.4b4/MySQL-python-1.2.4b4.tar.gz?r=http%3A%2F%2Fsourceforge.net%2Fprojects%2Fmysql-python%2F&ts=1364895531&use_mirror=nchc
 
tar zxvf  MySQL-python-1.2.4b4.tar.gz
cd MySQL-python-1.2.4b4
python setup.py build
python setup.py install
```

**如果　报`EnvironmentError: mysql_config not found` python 版本是2.7.3 **

此时执行 find / -name mysql_config 在/usr/bin/下发现了这个文件
然后修改MySQL-python-1.2.3目录下的site.cfg文件
去掉mysql_config=XXX这行的注释，并改成mysql_config=/usr/bin/mysql_config（以mysql_config文件所在机器上的目录为准）

#### centos6环境安装

```shell
yum -y install mysql-devel python-devel 
curl https://bootstrap.pypa.io/get-pip.py > get-pip.py   && python get-pip.py
pip install MySQL-python
```

#### 连接MySQL

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import MySQLdb

# 打开数据库连接
db = MySQLdb.connect("localhost","testuser","test123","TESTDB" )

# 使用cursor()方法获取操作游标 
cursor = db.cursor()

# 使用execute方法执行SQL语句
cursor.execute("SELECT VERSION()")

# 使用 fetchone() 方法获取一条数据库。
data = cursor.fetchone()

print "Database version : %s " % data

# 关闭数据库连接
db.close()
```

#### 创建表

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import MySQLdb

# 打开数据库连接
db = MySQLdb.connect("localhost","testuser","test123","TESTDB" )

# 如果数据表已经存在使用 execute() 方法删除表。
cursor.execute("DROP TABLE IF EXISTS EMPLOYEE")

# 创建数据表SQL语句
sql = """CREATE TABLE EMPLOYEE (
         FIRST_NAME  CHAR(20) NOT NULL,
         LAST_NAME  CHAR(20),
         AGE INT,  
         SEX CHAR(1),
         INCOME FLOAT )"""

cursor.execute(sql)

# 关闭数据库连接
db.close()

```

#### 数据库插入操作

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import MySQLdb

# 打开数据库连接
db = MySQLdb.connect("localhost","testuser","test123","TESTDB" )

# 使用cursor()方法获取操作游标 
cursor = db.cursor()

# SQL 插入语句
sql = """INSERT INTO EMPLOYEE(FIRST_NAME,
         LAST_NAME, AGE, SEX, INCOME)
         VALUES ('Mac', 'Mohan', 20, 'M', 2000)"""
try:
   # 执行sql语句
   cursor.execute(sql)
   # 提交到数据库执行
   db.commit()
except:
   # Rollback in case there is any error
   db.rollback()
finally:
    # 关闭数据库连接
    db.close()
```

以上例子也可以写成如下形式：

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import MySQLdb

# 打开数据库连接
db = MySQLdb.connect("localhost","testuser","test123","TESTDB" )

# 使用cursor()方法获取操作游标 
cursor = db.cursor()

# SQL 插入语句
sql = "INSERT INTO EMPLOYEE(FIRST_NAME, \
       LAST_NAME, AGE, SEX, INCOME) \
       VALUES ('%s', '%s', '%d', '%c', '%d' )" % \
       ('Mac', 'Mohan', 20, 'M', 2000)
try:
   # 执行sql语句
   cursor.execute(sql)
   # 提交到数据库执行
   db.commit()
except:
   # 发生错误时回滚
   db.rollback()
finally:
    # 关闭数据库连接
    db.close()
```

#### 数据库查询操作

Python查询Mysql使用 `fetchone()` 方法获取单条数据, 使用`fetchall()` 方法获取多条数据。

- fetchone(): 该方法获取下一个查询结果集。结果集是一个对象
- fetchall():接收全部的返回结果行.
- rowcount: 这是一个只读属性，并返回执行execute()方法后影响的行数。

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import MySQLdb

# 打开数据库连接
db = MySQLdb.connect("localhost","testuser","test123","TESTDB" )

# 使用cursor()方法获取操作游标 
cursor = db.cursor()

# SQL 查询语句
sql = "SELECT * FROM EMPLOYEE \
       WHERE INCOME > '%d'" % (1000)
try:
   # 执行SQL语句
   cursor.execute(sql)
   # 获取所有记录列表
   results = cursor.fetchall()
   for row in results:
      fname = row[0]
      lname = row[1]
      age = row[2]
      sex = row[3]
      income = row[4]
      # 打印结果
      print "fname=%s,lname=%s,age=%d,sex=%s,income=%d" % \
             (fname, lname, age, sex, income )
except:
   print "Error: unable to fecth data"
finally:
    # 关闭数据库连接
    db.close()
```

#### 数据库更新操作

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import MySQLdb

# 打开数据库连接
db = MySQLdb.connect("localhost","testuser","test123","TESTDB" )

# 使用cursor()方法获取操作游标 
cursor = db.cursor()

# SQL 更新语句
sql = "UPDATE EMPLOYEE SET AGE = AGE + 1 WHERE SEX = '%c'" % ('M')
try:
   # 执行SQL语句
   cursor.execute(sql)
   # 提交到数据库执行
   db.commit()
except:
   # 发生错误时回滚
   db.rollback()
finally:
    # 关闭数据库连接
    db.close()
```

#### 删除操作

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import MySQLdb

# 打开数据库连接
db = MySQLdb.connect("localhost","testuser","test123","TESTDB" )

# 使用cursor()方法获取操作游标 
cursor = db.cursor()

# SQL 删除语句
sql = "DELETE FROM EMPLOYEE WHERE AGE > '%d'" % (20)
try:
   # 执行SQL语句
   cursor.execute(sql)
   # 提交修改
   db.commit()
except:
   # 发生错误时回滚
   db.rollback()
finally:
    # 关闭连接
    db.close()
```

#### 执行事务

Python DB API 2.0 的事务提供了两个方法 commit 或 rollback。

```python
# SQL删除记录语句
sql = "DELETE FROM EMPLOYEE WHERE AGE > '%d'" % (20)
try:
   # 执行SQL语句
   cursor.execute(sql)
   # 向数据库提交
   db.commit()
except:
   # 发生错误时回滚
   db.rollback()
```   

对于支持事务的数据库， 在Python数据库编程中，**当游标建立之时，就自动开始了一个隐形的数据库事务**。
commit()方法游标的所有更新操作，rollback（）方法回滚当前游标的所有操作。每一个方法都开始了一个新的事务。

#### Cursor

签名我们是有的`Cursor`类型是`MySQLdb.cursors.Cursor`, 这种类型的`Cursor`在查询时将返回的结果集保存在客户端，并以`tuple`的形式来保存行。例如：

```python
(
    (1L, u'sky', 27L, u'123', u'123'), 
    (2L, u'tom', 18L, None, None)
)
```
可以看到每一个行都用一个`tuple`来表示，所有的行在一个总的`tuple`中。

#### DictCursor

如果我们不喜欢用`tuple`格式来作为返回行的表示格式，我们可以选择使用`MySQLdb.cursors.DictCursor`这种类型的`Cursor`。它返回的结果格式如下：

```python
(
    {'email': u'123', 'age': 27L, 'address': u'123', 'id': 1L, 'name': u'sky'}, 
    {'email': None, 'age': 18L, 'address': None, 'id': 2L, 'name': u'tom'}
)
```

可以看到每一行都是一个`Dict`，这样我们就可以通过字段名来访问每一行的指定字段。

#### SSCursor 和 SSDictCursor

这两个`Cursor`的类型和`Cursor`与`DictCursor`是一一对应的，所不同的是它将查询结果是保存在服务器端的。


#### Cursor 查询参数

Cursor的`execute`方法签名如下：

```python
execute(self, query, args=None)
````
query: string, 需要在服务器上执行的SQL
args: 针对该SQL，可选的序列或映射

**注意：** 如果查询参数类型是序列，则必须使用 `%s`作为查询参数占位符。如果查询参数类型是映射，则需要使用格式为：`%(key)s`来作为占位符。如果执行的是更新语句，则返回一个长整形作为影响的行数。

一个例子：

```python
// 序列
cursor.execute("select * from emp where id = %s", [1])
// 映射
cursor.execute("select * from emp where id = %(id)s", {"id":1})
```
**注意：** 使用查询参数有限制：
1.表名不能作为参数传入
2.不支持list等复杂类型 (TypeError: not all arguments converted during string formatting)


### mongodb操作

#### 驱动安装

```shell
$ python -m pip install pymongo
or 
$ python -m easy_install pymongo
```

#### 示例

```python
from pymongo import MongoClient

# mongodb配置
client = MongoClient('连接串')
# 选择db
db = client['users']
# 认证
db.authenticate('用户名', '密码', source='认证数据库')
# 集合对象
user = db['user']
# 查询
userObj = user.find_one({"id":1})

```

### 参考资料

[python逐行读取文件内容的三种方法](http://www.jb51.net/article/45956.htm)
[python操作mysql数据库](https://www.runoob.com/python/python-mysql.html)
[pymongo install](https://pypi.python.org/pypi/pymongo)
[pymongo doc](http://api.mongodb.com/python/current/)







