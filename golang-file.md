---
layout: post
comments: true
title: golang文件操作
date: 2016-10-16 09:21:17
tags:
- go
categories:
- golang
---

> 文件操作是大多数编程语言必不可少的功能. 在GO中,它提供了文件操作所需的各种API, 包括文件的创建,删除,读写,权限设置等操作.下面就介绍下GO中常见的文件操作.

<!-- more -->

### 创建文件

创建文件是由`os.Create`函数实现的，

```golang
file, err := os.Create(fileName)
//创建指定名称的文件，如果创建失败，返回的err不为nil；成功则返回一个File指针，
// File represents an open file descriptor.
type File struct {
    *file
}
// file is the real representation of *File.
// The extra level of indirection ensures that no clients of os
// can overwrite this data, which could cause the finalizer
// to close the wrong file descriptor.
type file struct {
    fd      int
    name    string
    dirinfo *dirInfo // nil unless directory being read
    nepipe  int32    // number of consecutive EPIPE in Write
}
//如果指定的文件已经存在，则该文件的内容会被情况。新创建的文件是可读写的。
也可以通过os.OpenFile函数实现文件的创建：
func OpenFile(name string, flag int, perm FileMode) (*File, error)
事实上：os.Create(fileName)内部也是通过该函数实现的：
func Create(name string) (*File, error) {
    return OpenFile(name, O_RDWR|O_CREATE|O_TRUNC, 0666)
}
```

### 文件读写

文件读取是由`os`包实现的：

```golang
file, err := os.Open(fileName)
//如果打开失败，返回的err不为nil；成功则返回一个File指针，
通过file就可以读写文件了
//读
numRD, err := file.Read(make([]byte, 1024, 1024))
//从指定位置读取
numRD, err := file.ReadAt(make([]byte, 1024, 1024), 1024)
//读取目录文件下的所有文件， 返回一个FileInfo的切片，0或负数表示读取所有，
如果大于0，则只读取指定数据的文件
fileInfos, err := file.Readdir(0) 
//写
numWR, err := file.Write("hello go"([]byte))
numWR, err := file.WriteString("Hello World")
//写入指定位置
numWR, err := file.WriteAt("hello go"([]byte), 1024)
```

### 文件删除

```golang
//删除指定的文件或目录(目录为空), 出错返回*PathError
err := os.Remove(fileName) 
//删除指定目录和目录下的所有子目录；如果指定的目录不存在，返回nil
err := os.RemoveAll(fileName)    
```

### 文件是否存在

```golang
// 判断error表示的错误是因为指定的文件或目录已经存在
os.IsExist(error) 
// 判断error表示的错误是因为指定的文件或目录不存在
os.IsNotExist(error)
```

### 创建目录

```golang
//通过os包中Mkdir函数创建目录
//只能在已经存在的目录下创建子目录
func Mkdir(name string, perm FileMode) error
//可以递归的创建目录
func MkdirAll(path string, perm FileMode) error
```

**例子：**

```golang
crtUser, err := user.Current()
homeDir := crtUser.HomeDir
fmt.Println("home dir is " + homeDir)
err1 := os.Mkdir(homeDir + "/abc", os.ModePerm)
if err1 != nil {
    fmt.Println(err.Error())
}
os.MkdirAll(homeDir + "/lang/java",  os.ModePerm)
```


                        
                
                
                