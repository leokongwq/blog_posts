---
layout: post
comments: true
title: golang代码测试
date: 2016-10-13 18:01:31
tags:
    - golang
categories:
    - golang
---

> 开发程序其中很重要的一点是测试，我们如何保证代码的质量，如何保证每个函数是可运行，运行结果是正确的，又如何保证写出来的代码性能是好的，我们知道单元测试的重点在于发现程序设计或实现的逻辑错误，使问题及早暴露，便于问题的定位解决，而性能测试的重点在于发现程序设计上的一些问题，让线上的程序能够在高并发的情况下还能保持稳定。本小节将带着这一连串的问题来讲解Go语言中如何来实现单元测试和性能测试。Go语言中自带有一个轻量级的测试框架testing和自带的go test命令来实现单元测试和性能测试，testing框架和其他语言中的测试框架类似，你可以基于这个框架写针对相应函数的测试用例，也可以基于该框架写相应的压力测试用例，那么接下来让我们一一来看一下怎么写。

<!-- more -->

### 如何编写测试用例

由于go test命令只能在一个相应的目录下执行所有文件，所以我们接下来新建一个项目目录gotest,这样我们所有的代码和测试代码都在这个目录下。

接下来我们在该目录下面创建两个文件：gotest.go和gotest_test.go

gotest.go:这个文件里面我们是创建了一个包，里面有一个函数实现了除法运算:

{% codeblock lang:go %}
    package gotest
    import (
        "errors"
    )
    func Division(a, b float64) (float64, error) {
        if b == 0 {
            return 0, errors.New("除数不能为0")
        }
        return a / b, nil
    }
{% endcodeblock %}

`gotest_test.go` : 这是我们的单元测试文件，但是记住下面的这些原则：

 - 文件名必须是`_test.go`结尾的，这样在执行go test的时候才会执行到相应的代码
 - 你必须import testing这个包
 - 所有的测试用例函数必须是Test开头
 - 测试用例会按照源代码中写的顺序依次执行
 - 测试函数TestXxx()的参数是testing.T，我们可以使用该类型来记录错误或者是测试状态
 - 测试格式：func TestXxx (t*testing.T),Xxx部分可以为任意的字母数字的组合，
   但是首字母不能是小写字母[a-z]，例如Testintdiv是错误的函数名。
 - 函数中通过调用testing.T的Error, Errorf, FailNow, Fatal, FatalIf方法，说明测试不通过，
   调用Log方法用来记录测试的信息。 

下面是我们的测试用例的代码：

{% codeblock lang:go %}
    package gotest
    import (
        "testing"
    )
    func Test_Division_1(t *testing.T) {
        if i, e := Division(6, 2); i != 3 || e != nil { //try a unit test on function
            t.Error("除法函数测试没通过") // 如果不是如预期的那么就报错
        } else {
            t.Log("第一个测试通过了") //记录一些你期望记录的信息
        }
    }
    func Test_Division_2(t *testing.T) {
        t.Error("就是不通过")
    }
{% endcodeblock %}
    
我们在项目目录下面执行go test,就会显示如下信息：

{% codeblock lang:go %}
    FAIL: Test_Division_2 (0.00 seconds)
    gotest_test.go:16: 就是不通过
    FAIL
    exit status 1
    FAIL    gotest  0.013s
{% endcodeblock %}

从这个结果显示测试没有通过，因为在第二个测试函数中我们写死了测试不通过的代码t.Error，那么我们的第一个函数执行的情况怎么样呢？默认情况下执行go test是不会显示测试通过的信息的，我们需要
带上参数`go test -v`，这样就会显示如下信息：

{% codeblock lang:go %}
    === RUN Test_Division_1
    --- PASS: Test_Division_1 (0.00 seconds)
        gotest_test.go:11: 第一个测试通过了
    === RUN Test_Division_2
    --- FAIL: Test_Division_2 (0.00 seconds)
        gotest_test.go:16: 就是不通过
    FAIL
    exit status 1
    FAIL    gotest  0.012s
{% endcodeblock %}
    
上面的输出详细的展示了这个测试的过程，我们看到测试函数1Test_Division_1测试通过，而测试函数2Test_Division_2测试失败了，最后得出结论测试不通过。接下来我们把测试函数2修改成如下代码：

{% codeblock lang:go %}
    func Test_Division_2(t *testing.T) {
        if _, e := Division(6, 0); e == nil { //try a unit test on function
            t.Error("Division did not work as expected.") // 如果不是如预期的那么就报错
        } else {
            t.Log("one test passed.", e) //记录一些你期望记录的信息
        }
    } 
{% endcodeblock %}

然后我们执行go test -v，就显示如下信息，测试通过了：

{% codeblock lang:go %}
=== RUN Test_Division_1
--- PASS: Test_Division_1 (0.00 seconds)
    gotest_test.go:11: 第一个测试通过了
=== RUN Test_Division_2
--- PASS: Test_Division_2 (0.00 seconds)
    gotest_test.go:20: one test passed. 除数不能为0
PASS
ok      gotest  0.013s
{% endcodeblock %}

### 如何编写压力测试

压力测试用来检测函数(方法）的性能，和编写单元功能测试的方法类似,此处不再赘述，但需要注意以下几点：
压力测试用例必须遵循如下格式，其中XXX可以是任意字母数字的组合，但是首字母不能是小写字母

    func BenchmarkXXX(b *testing.B) { ... }

`go test`默认不会执行压力测试的函数，如果要执行压力测试需要带上参数`-test.bench`，语法:-test.bench="test_name_regex",例如go test -test.bench=".*"表示测试全部的压力测试函数

在压力测试用例中,请记得在循环体内使用testing.B.N,以使测试可以正常的运行
文件名也必须以_test.go结尾
下面我们新建一个压力测试文件webbench_test.go，代码如下所示：

{% codeblock lang:go %}
    package gotest
    import (
        "testing"
    )
    func Benchmark_Division(b *testing.B) {
        for i := 0; i < b.N; i++ { //use b.N for looping 
            Division(4, 5)
        }
    }
    func Benchmark_TimeConsumingFunction(b *testing.B) {
        b.StopTimer() //调用该函数停止压力测试的时间计数
        //做一些初始化的工作,例如读取文件数据,数据库连接之类的,
        //这样这些时间不影响我们测试函数本身的性能
        b.StartTimer() //重新开始时间
        for i := 0; i < b.N; i++ {
            Division(4, 5)
        }
    }
{% endcodeblock %}
    
我们执行命令`go test -file webbench_test.go -test.bench=".*"`，可以看到如下结果：

{% codeblock lang:go %}
    PASS
    Benchmark_Division  500000000            7.76 ns/op
    Benchmark_TimeConsumingFunction 500000000            7.80 ns/op
    ok      gotest  9.364s  
{% endcodeblock %}

上面的结果显示我们没有执行任何TestXXX的单元测试函数，显示的结果只执行了压力测试函数，
第一条显示了Benchmark_Division执行了500000000次，每次的执行平均时间是7.76纳秒，
第二条显示了Benchmark_TimeConsumingFunction执行了500000000，每次的平均执行时间是7.80纳秒。
最后一条显示总共的执行时间。

### 小结

通过上面对单元测试和压力测试的学习，我们可以看到testing包很轻量，编写单元测试和压力测试用例非常简单，
配合内置的go test命令就可以非常方便的进行测试，这样在我们每次修改完代码,执行一下`go test`就可以简单的完成回归测试了。

               
                    
                    