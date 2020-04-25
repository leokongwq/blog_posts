---
layout: post
title: JVM常用参数和调优
date: 2015-04-16
comments: true
categories:
- JVM
tags:
- java
- jvm
---

### JVM常用参数和调优

#### 前言

下面所有内容都是针对Sun JDK1.6 JVM	
	
####	jvm常用参数 

* -Xms4096m：设置JVM堆初始大小为4096M

* -Xmx4096m：设置JVM堆最大为4096M

* -Xmn2g（-xx:NewSize=2g）：设置年轻代大小为2G。

* -Xss128k：设置每个线程的堆栈大小。JDK5.0以后每个线程堆栈大小为1M，以前每个线程堆栈大小为 256K。根据应用的线程所需内存大小进行调整。在相同物理内存下，减小这个值能生成更多的线程。

* -xx:PermSize=128m 设置持久代的初始大小

* -XX:MaxPermSize=128m 设置持久代的最大值

* -XX:NewRatio=4:设置年轻代（包括Eden和两个Survivor区）与年老代的比值（除去持久代）。设置为4，	则年轻代与年老代所占比值为1：4，年轻代占整个堆栈的1/5

* -XX:SurvivorRatio=4：设置年轻代中Eden区与Survivor区的大小比值。设置为4，则两个Survivor区	与一个Eden区的比值为2:4，一个Survivor区占整个年轻代的1/6

* -XX:MaxTenuringThreshold=0：设置垃圾最大年龄。如果设置为0的话，则年轻代对象不 经过Survivor	区，直接进入年老代。对于年老代比较多的应用，可以提高效率。如果将此值设置为一个较大值，则年轻代	对象会在Survivor区进行 多次复制，这样可以增加对象再年轻代的存活时间，增加在年轻代即被回收的概论。


#### jvm性能调优

###### JVM性能调优的重点

* 内存分配
		
	为了满足性能指标，合理分配应用占用的堆内存空间和堆内各个区块的比例
		
* 垃圾收集
		
	根据应用的特点选择最合适的满足需求的垃圾收集器
	
###### JVM性能调优步骤

1. 添加JVM参数，打印垃圾收集日志并分析	
	
	-Xloggc:filename
	
	-XX:+PrintGCDetails
	
	-XX:+PrintGCTimeStamps
	
2. 根据分析结果调整堆空间和堆内个部分的比例

3. 根据垃圾收集日志挑选符合自己应用的收集器

4. 根据垃圾收集日志继续调整堆空间和垃圾收集器参数

5. 根据垃圾收集日志重复3，4步骤

#### jvm参数详解

[jvm参数](http://www.oracle.com/technetwork/java/javase/tech/vmoptions-jsp-140102.html) 	

***未完待续....***

		
	
