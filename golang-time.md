---
layout: post
comments: true
title: golang时间操作
date: 2016-10-16 09:09:19
tags:
- go
categories:
- golang
---

> 对时间操作的操作是日常编码中长的操作, 一个好的日期时间操作库能提供编码的效率并减少bug的产生; golang有自己的时间操作库, 在它的`time`包,下面就介绍少在GO中如何操作时间.

<!-- more -->

#### 包

go中时间的操作是在`time`包中

#### 数据结构

```golang
type Time struct {
    // sec gives the number of seconds elapsed since
    // January 1, year 1 00:00:00 UTC.
    sec int64
    // nsec specifies a non-negative nanosecond
    // offset within the second named by Seconds.
    // It must be in the range [0, 999999999].
    nsec int32
    // loc specifies the Location that should be used to
    // determine the minute, hour, month, day, and year
    // that correspond to this Time.
    // Only the zero Time has a nil Location.
    // In that case it is interpreted to mean UTC.
    loc *Location
}

// A Month specifies a month of the year (January = 1, ...).
type Month int

// A Duration represents the elapsed time between two instants
// as an int64 nanosecond count.  The representation limits the
// largest representable duration to approximately 290 years.
type Duration int64
```

#### 相关函数

**time.Now()**

time.Now返回当前时间，返回类型是Time, 该返回值是操作时间的关键。

```golang    
now := time.Now()
fmt.Println(now.Year())
//Month 函数返回的类型是:Month(其实是int) //type Month int
// String returns the English name of the day ("Sunday", "Monday", ...).
func (d Weekday) String() string { return days[d] }
fmt.Println(now.Month()) //Month类型实现了String方法，在fmt输出时会转为数字对应的string类型的表示形式
//	
month := now.Month()
fmt.Println(int(month)) //强制转为int,则会输出数字月份
fmt.Println(now.Day())
fmt.Println(now.Hour())
fmt.Println(now.Minute())
fmt.Println(now.Second())
fmt.Println(now.Unix())
fmt.Println(now.UnixNano())
```

**时间计算**	

计算时间差：

```golang
startTime := time.Now()
endTime := time.Now()
duration := endTime.Sub(startTime) 
duration.Seconds()
duration.Nanoseconds()
fmt.Println(duration.String())
```

//Sub的返回值类型是Duration（type Duration int64）， 表示2个时间的纳秒级别的差值

**sleep**

```golang
time.Sleep(time.Second * 3)
//将当前的goroutine 暂定指定的时间，单位是纳秒
```
### 时间格式化

```golang
now := time.Now()
timeFmt := "2006-01-02 15:04:05" //这个时间很特殊，
fmt.Println(now.Format(time.RFC822))
fmt.Println(now.Format(timeFmt))
```

### 时间解析

```golang
t := time.Parse(timeFmt, "2015-04-12 14:22:12")
t2 = time.Unix(1461758975, 0)
```

其它的方法可以参考文档！
          
                
                
                
