---
layout: post
title: 有用的JVM工具简介
comments: true
date: 2015-06-14
categories:
- java
tags:
- jdk
---

### 有用的JVM工具简介

通常我们在安装完JDK后只使用bin目录下的的java,javac命令，其实jdk还给我们提供了很多非常有用的工具，下面我们学习一些监控和故障处理的工具。

<!-- more -->

<table>
<tr>                                                                                                                         
	<td>名称</td>
	<td>作用</td>
</tr>
<tr>
	<td>jps</td>
	<td>JVM process status tool，显示指定系统内所有的 HotSpot 虚拟机进程</td>
</tr>
<tr>
	<td>jstat</td>
	<td>JVM statistics monitoring tool，用于收集 HotSpot 虚拟机各方面的运行数据</td>
</tr>
<tr>
	<td>jinfo</td>
	<td>显示虚拟机配置信息</td>
</tr>
<tr>
	<td>jmap</td>
	<td>生产虚拟机的内存快照dump文件</td>
</tr>
<tr>
	<td>jhat</td>
	<td>分析堆dump文件</td>
</tr>
<tr>
	<td>jstack</td>
	<td>显示虚拟机的线程快照</td>
</tr>
</table>

### jps 虚拟机进程状况工具

**jps 的命令格式：**

jps [options] [hostid]

jps 有如下主要的选项：

-q	只输出 LVMID，省略主类的名称

-m	输出虚拟机启动时候传递给 main 方法的参数

-l	输出类的全名

-v	输出虚拟机进程启动时 JVM 参数

示例：
<pre>
>jps -l
25330 sun.tools.jps.Jps
25296

>jps -lv
25356 sun.tools.jps.Jps -Dapplication.home=/Library/Java/JavaVirtualMachines/jdk1.7.0_71.jdk/Contents/Home -Xms8m
25296  -Dosgi.requiredJavaVersion=1.6 -XstartOnFirstThread -Dorg.Eclipse.swt.internal.carbon.smallFonts -XX:MaxPermSize=256m -Xms40m -Xmx512m -Xdock:icon=../Resources/Eclipse.icns -XstartOnFirstThread -Dorg.eclipse.swt.internal.carbon.smallFonts
</pre>

jps 可以查看通过 rmi 协议查询开启了 rmi 服务的原创虚拟机进程状态，hostid 是 rmi 注册表中注册的主机。


### jstat 虚拟机统计信息监视工具

jstat 可以显示本地或者远程虚拟机进程中的类装载、内存、垃圾收集、 JIT 编译等运行数据。

**jstat 的命令格式：**

jstat [option vmid [interval] [count]]

示例：
<pre>
>jstat -gcutil 25296 1000 5
  S0     S1     E      O      P     YGC     YGCT    FGC    FGCT     GCT
  0.00  99.54  90.43  93.70  95.23     55    1.156     5    1.990    3.146
  0.00  99.54  90.43  93.70  95.23     55    1.156     5    1.990    3.146
  0.00  99.54  90.43  93.70  95.23     55    1.156     5    1.990    3.146
  0.00  99.54  90.43  93.70  95.23     55    1.156     5    1.990    3.146
  0.00  99.54  90.43  93.70  95.23     55    1.156     5    1.990    3.146
</pre>

查询 25296 进程的虚拟机状况，并且每隔 1000 毫秒一次，显示 5 次。

看下主要选项的含义：

选项	
-class	监视类装载、卸载数量、总看见以及类装载消耗的时间
-gc	监视 java 堆状况，包括 eden 区、两个 survivor 区、年老代、永久代等的容量、已用空间、gc 时间合计等
-gccapacity	内容与 -gc 基本相同，输出主要关注 java 堆各个区使用到的最大、最小空间
-gcutil	内容与 -gc 基本相同，关注已使用区域占总空间的百分比
-gccause	内容与 -gcutil 一样，并且多输出导致上一次 gc 产生的原因
-gcnew	监视新生代状况
-gcnewcapacity	与 -gcnew 相同，主要关注使用到的最大、最小空间
-compiler	输出 JIT 编译器编译过的方法、耗时等信息
下面解读下 -gcutil 所产生的内容：

S0、S1 分别代表了 Survivor0 和 Survivor1，E 代表 Eden 区，O 代表老年区， P 代表永久代。YGC 代表 Young GC 的次数，YGCT 代表时间，后面一样解释。

### jinfo 查看 java 配置信息工具

这个命令比较简单，直接看自带的描述：
<pre>
Usage:
  jinfo [option] <pid>
    (to connect to running process)
  jinfo [option] <executable <core>
    (to connect to a core file)
  jinfo [option] [server_id@]<remote server IP or hostname>
    (to connect to remote debug server)
where <option> is one of:
  -flag <name>		 to print the value of the named VM flag
  -flag [+|-]<name>	to enable or disable the named VM flag
  -flag <name>=<value> to set the named VM flag to the given value
  -flags			   to print VM flags
  -sysprops			to print Java system properties
  <no option>		  to print both of the above
  -h | -help		   to print this help message
</pre>

### jmap 生产jvm堆dump

jmap 除了可以生成 dump 文件外，还可以查询 finalize 执行队列，java堆和永久代的详细信息，如空间使用率和当前用的是哪种收集器等。

具体的 jamp 如何操作部多介绍，看下面提供的说明，比较简单：
<pre>
Usage:
  jmap [option] <pid>
    (to connect to running process)
  jmap [option] <executable <core>
    (to connect to a core file)
  jmap [option] [server_id@]<remote server IP or hostname>
    (to connect to remote debug server)
where <option> is one of:
  <none>			   to print same info as Solaris pmap
  -heap				to print java heap summary
  -histo[:live]		to print histogram of java object heap; if the "live"
             suboption is specified, only count live objects
  -permstat			to print permanent generation statistics
  -finalizerinfo	   to print information on objects awaiting finalization
  -dump:<dump-options> to dump java heap in hprof binary format
             dump-options:
               live		 dump only live objects; if not specified,
                    all objects in the heap are dumped.
               format=b	 binary format
               file=<file>  dump heap to <file>
             Example: jmap -dump:live,format=b,file=heap.bin <pid>
  -F				   force. Use with -dump:<dump-options> <pid> or -histo
             to force a heap dump or histogram when <pid> does not
             respond. The "live" suboption is not supported
             in this mode.
  -h | -help		   to print this help message
  -J<flag>			 to pass <flag> directly to the runtime system
</pre>

### jhat 虚拟机堆快照分析工具

我们可以使用 jhat 来分析 jmap 生成的 dump 文件

<pre>
>jhat tmp.dump
Reading from tmp.dump...
Dump file created Sat May 09 17:10:52 CST 2015
Snapshot read, resolving...
Resolving 0 objects...
WARNING:  hprof file does not include java.lang.Class!
WARNING:  hprof file does not include java.lang.String!
WARNING:  hprof file does not include java.lang.ClassLoader!
Chasing references, expect 0 dots
Eliminating duplicate references
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
</pre>

默认会开 7000 端口进行 web 访问。一般不使用这个命令来分析，会使用专业的工具来分析 dump 文件，如 eclipse memory analyzer 等。

### jstack 生成jvm线程信息快照

使用方式如下：
<pre>
Usage:
  jstack [-l] <pid>
    (to connect to running process)
  jstack -F [-m] [-l] <pid>
    (to connect to a hung process)
  jstack [-m] [-l] <executable> <core>
    (to connect to a core file)
  jstack [-m] [-l] [server_id@]<remote server IP or hostname>
    (to connect to a remote debug server)
Options:
  -F  to force a thread dump. Use when jstack <pid> does not respond (process is hung)
  -m  to print both java and native frames (mixed mode)
  -l  long listing. Prints additional information about locks
  -h or -help to print this help message
</pre>

jstack 也可以帮助我们分析线程死锁