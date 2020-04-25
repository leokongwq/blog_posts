---
layout: post
comments: true
title: JVM-codecache内存区域介绍
date: 2016-10-11 19:52:59
tags:
    - java
    - jvm
categories:
    - java

---

### JVM-codecache内存区域介绍
                        
> 大家都知道JVM在运行时会将频繁调用方法的字节码编译为本地机器码。这部分代码所占用的内存空间成为CodeCache区域。一般情况下我们是不会关心这部分区域的且大部分开发人员对这块区域也不熟悉。偶然的机会我们线上服务器Down了，在日志里面看到 `java.lang.OutOfMemoryError code cache`。通过查找资料来详细了解一下该快内存区域的使用。

<!-- more -->

** 所有测试数据都基于jdk8 **

### Codecache大小控制选项

| 选项        | 默认值          | 描述  |
| ------------- |:-------------:| ----- |
| InitialCodeCacheSize | 2555904 | 默认的CodeCache区域大小，单位为字节 |
| ReservedCodeCacheSize | 251658240 | CodeCache区域的最大值，单位为字节 |
| CodeCacheExpansionSize | 65536 | CodeCache每次扩展大小，单位为字节 |

### Codecache 刷新选项

| 选项        | 默认值          | 描述  |
| ------------- |:-------------:| ----- |
| ExitOnFullCodeCache | false | 当CodeCache区域满了的时候是否退出JVM |
| UseCodeCacheFlushing | false | 是否在关闭JIT编译前清除CodeCache |
| MinCodeCacheFlushingInterval | 30 | 刷新CodeCache的最小时间间隔 ，单位为秒 |
| CodeCacheMinimumFreeSpace | 512000 | 当CodeCache区域的剩余空间小于参数指定的值时停止JIT编译。剩余的空间不会再用来存放方法的本地代码, 可以存放本地方法适配器代码。 |

### 编译策略选项

| 选项        | 默认值          | 描述  |
| ------------- | :-------------: | ----- |
| CompileThreshold | 10000 | 指定方法在在被JIT编译前被调用的次数 |
|OnStackReplacePercentage | 140 | 该值为用于计算是否触发OSR（OnStackReplace）编译的阈值 |

### OSR编译的阈值计算

在client模式时，计算规则为`CompileThreshold * (OnStackReplacePercentage/100)`，在server模式时，计算规则为`(CompileThreshold * (OnStackReplacePercentage - InterpreterProfilePercentage))/100`。InterpreterProfilePercentage的默认值为33，当方法上的回边计数器到达这个值时，即触发后台的OSR编译，并将方法上累积的调用计数器设置为CompileThreshold的值，同时将回边计数器设置为CompileThreshold/2的值，一方面是为了避免OSR编译频繁触发；另一方面是以便当方法被再次调用时即触发正常的编译，当累积的回边计数器的值再次达到该值时，先检查OSR编译是否完成。如果OSR编译完成，则在执行循环体的代码时进入编译后的代码；如果OSR编译未完成，则继续把当前回边计数器的累积值再减掉一些，从这些描述可看出，默认情况下对于回边的情况，server模式下只要回边次数达到10 700次，就会触发OSR编译。

用以下一段示例代码来模拟编译的触发：

{% codeblock lang:java %}
    public class Foo{  
        public static void main(String[] args){  
        Foo foo=new Foo();  
            for(int i=0;i<10;i++){  
                foo.bar();  
            }  
        }  
        public void bar(){  
            // some bar code  
            for(int i=0;i<10700;i++){  
            bar2();  
            }  
        }  
        private void bar2(){  
            // bar2 method  
        }  
    }
{% endcodeblock %}

以上代码采用java -server方式执行，当main中第一次调用foo.bar时，bar方法上的调用计数器为1，回边计数器为0；当bar方法中的循环执行完毕时，bar方法的调用计数器仍然为1，回边计数器则为10 700，达到触发OSR编译的条件，于是触发OSR编译，并将bar方法的调用计数器设置为10 000，回边计数器设置为5 000。

当main中第二次调用foo.bar时，jdk发现bar方法的调用次数已超过compileThreshold，于是在后台执行JIT编译，并继续解释执行// some bar code，进入循环时，先检查OSR编译是否完成。如果完成，则执行编译后的代码，如果未编译完成，则继续解释执行。

当main中第三次调用foo.bar时，如果此时JIT编译已完成，则进入编译后的代码；如果编译未完成，则继续按照上面所说的方式执行。

由于Sun JDK的这个特性，在对Java代码进行性能测试时，要尤其注意是否事先做了足够次数的调用，以保证测试是公平的；对于高性能的程序而言，也应考虑在程序提供给用户访问前，自行进行一定的调用，以保证关键功能的性能。

**JIT编译限制选项**

| 选项        | 默认值          | 描述  |
| ------------- | :-------------: | ----- |
| MaxInlineLevel | 9 | 在进行方法内联前，方法的最多嵌套调用次数 |
| MaxInlineSize | 35 | 被内联方法的字节码最大值 |
| MinInliningThreshold | 9 | 方法被内联的最小调用次数 |
| InlineSynchronizedMethods | true | 是否对同步方法进行内联 |

### 诊断选项

| 选项        | 默认值          | 描述  |
| ------------- | :-------------: | ----- |
| PrintFlagsFinal | false | 是否打印所有的JVM参数 |
| PrintCodeCache | false | 是否在JVM退出前打印CodeCache的使用情况 |
| PrintCodeCacheOnCompilation | false | 是否在每个方法被JIT编译后打印CodeCache区域的使用情况 |
         
