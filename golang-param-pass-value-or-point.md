---
layout: post
comments: true
title: 值传递, 指针传递 这是一个问题
date: 2017-01-22 12:03:58
tags:
- GO
categories:
- golang
---

本文转自：http://colobu.com/2017/01/05/-T-or-T-it-s-a-question/

这是我看过的关于值传递和指针传递(或引用传递)理解的比较透彻且有自己思考，观点的好文。

在编程语言深入讨论中，经常被大家提起也是争论最多的讨论之一就是按值(by value)还是按引用传递(by reference， by pointer)，你可以在C/C++或者Java的社区经常看到这样的讨论，也会看到很多这样的面试题。

<!-- more -->

对于Go语言，严格意义上来讲，只有一种传递，也就是按值传递(by value)。当一个变量当作参数传递的时候，会创建一个变量的副本，然后传递给函数或者方法，你可以看到这个副本的地址和变量的地址是不一样的。

当变量当做指针被传递的时候，一个新的指针被创建，它指向变量指向的同样的内存地址，所以你可以将这个指针看成原始变量指针的副本。当这样理解的时候，我们就可以理解成Go总是创建一个副本按值转递，只不过这个副本有时候是变量的副本，有时候是变量指针的副本。

这是Go语言中你理解后续问题的基础。

但是Go语言的情况比较复杂，我们什么时候选择 `T` 作为参数类型，什么时候选择 `*T`作为参数类型？ `[]T`是传递的指针还是值？选择`[]T`还是`[]*T`? 哪些类型复制和传递的时候会创建副本？什么情况下会发生副本创建？

本文将详细介绍Go语言的变量的副本创建还是变量指针的副本创建的case以及各种类型在这些case的情况。


### 副本的创建

前面已经讲到，`T`类型的变量和`*T`类型的变量在当做函数或者方法的参数时会传递它的副本。我们先看看例子。

#### T的副本创建

首先看一下 参数类型为`T`的函数调用的情况：

```golang
package main
import "fmt"
type Bird struct {
	Age  int
	Name string
}
func passV(b Bird) {
	b.Age++
	b.Name = "Great" + b.Name
	fmt.Printf("传入修改后的Bird:\t %+v, \t内存地址：%p\n", b, &b)
}
func main() {
	parrot := Bird{Age: 1, Name: "Blue"}
	fmt.Printf("原始的Bird:\t\t %+v, \t\t内存地址：%p\n", parrot, &parrot)
	passV(parrot)
	fmt.Printf("调用后原始的Bird:\t %+v, \t\t内存地址：%p\n", parrot, &parrot)
}
```

运行后输入结果(每次运行指针的值可能不同):

```golang
原始的Bird:      {Age:1 Name:Blue},      内存地址:0xc420012260
传入修改后的Bird: {Age:2 Name:GreatBlue}, 内存地址：0xc4200122c0
调用后原始的Bird: {Age:1 Name:Blue},      内存地址：0xc420012260
```

可以看到，在`T`类型作为参数的时候，传递的参数parrot会将它的副本(内存地址0xc4200122c0)传递给函数passV,在这个函数内对参数的改变不会影响原始的对象。

#### *T的副本创建

修改上面的例子，将函数的参数类型由`T`改为`*T`:

```golang
package main
import "fmt"
type Bird struct {
	Age  int
	Name string
}
func passP(b *Bird) {
	b.Age++
	b.Name = "Great" + b.Name
	fmt.Printf("传入修改后的Bird:\t %+v, \t内存地址：%p, 指针的内存地址: %p\n", *b, b, &b)
}
func main() {
	parrot := &Bird{Age: 1, Name: "Blue"}
	fmt.Printf("原始的Bird:\t\t %+v, \t\t内存地址：%p, 指针的内存地址: %p\n", *parrot, parrot, &parrot)
	passP(parrot)
	fmt.Printf("调用后原始的Bird:\t %+v, \t内存地址：%p, 指针的内存地址: %p\n", *parrot, parrot, &parrot)
}
```

运行后输出结果：

```golang
原始的Bird:	 {Age:1 Name:Blue},      内存地址：0xc420076000, 指针的内存地址: 0xc420074000
传入修改后的Bird:	 {Age:2 Name:GreatBlue}, 内存地址：0xc420076000, 指针的内存地址: 0xc420074010
调用后原始的Bird:	 {Age:2 Name:GreatBlue}, 内存地址：0xc420076000, 指针的内存地址: 0xc420074000
```
可以看到在函数`passP`中，参数`p`是一个指向Bird的指针，传递参数给它的时候会创建指针的副本(`0xc420074010`)，只不过指针`0xc420074000`和`0xc420074010`都指向内存地址0xc420076000。 函数内对`*T`的改变显然会影响原始的对象，因为它是对同一个对象的操作。

当然，一位对Go有深入了解的读者都已经对这个知识有所了解，也明白了`T`和`*T`作为参数的时候副本创建的不同。

### 如何选择 `T` 和 `*T`

在定义函数和方法的时候，作为一位资深的Go开发人员，一定会对函数的参数和返回值定义成T和`*T`深思熟虑，有些情况下可能还会有些苦恼。
那么什么时候才应该把参数定义成类型T,什么情况下定义成类型`*T`呢。

一般的判断标准是看副本创建的成本和需求。

1. 不想变量被修改。 如果你不想变量被函数和方法所修改，那么选择类型T。相反，如果想修改原始的变量，则选择`*T`
2. 如果变量是一个大的struct或者数组，则副本的创建相对会影响性能，这个时候考虑使用`*T`，只创建新的指针，这个区别是巨大的
3. (不针对函数参数，只针对本地变量／本地变量)对于函数作用域内的参数，如果定义成T,Go编译器尽量将对象分配到栈上，而*T很可能会分配到对象上，这对垃圾回收会有影响

### 什么时候发生副本创建

上面举的例子都是作为函数参数时发生的副本的创建，还有很多情况下会发生副本的创建，甚至有些“隐蔽”的情况。
编程的时候如何小心这些情况呢，一条原则就是：

> A go assignment is a copy of the value itself 

赋值的时候就会创建对象副本

Assignment的语法表达式如下

> Assignment = ExpressionList assign_op ExpressionList .
assign_op = [ add_op | mul_op ] "=" .

> Each left-hand side operand must be addressable, a map index expression, or (for = assignments only) the blank identifier. Operands may be parenthesized.

#### 最常见的case

最常见的赋值的例子是对变量的赋值，包括函数内和函数外:

```golang
package main
import "fmt"
type Bird struct {
	Age  int
	Name string
}
type Parrot struct {
	Age  int
	Name string
}
var parrot1 = Bird{Age: 1, Name: "Blue"}
var parrot2 = parrot1
func main() {
	fmt.Printf("parrot1:\t\t %+v, \t\t内存地址：%p\n", parrot1, &parrot1)
	fmt.Printf("parrot2:\t\t %+v, \t\t内存地址：%p\n", parrot2, &parrot2)
	parrot3 := parrot1
	fmt.Printf("parrot2:\t\t %+v, \t\t内存地址：%p\n", parrot3, &parrot3)
	parrot4 := Parrot(parrot1)
	fmt.Printf("parrot4:\t\t %+v, \t\t内存地址：%p\n", parrot4, &parrot4)
}
```

输出结果:

```golang
parrot1:		 {Age:1 Name:Blue}, 		内存地址：0xfa0a0
parrot2:		 {Age:1 Name:Blue}, 		内存地址：0xfa0c0
parrot2:		 {Age:1 Name:Blue}, 		内存地址：0xc42007e0c0
parrot4:		 {Age:1 Name:Blue}, 		内存地址：0xc42007e100
```

可以看到这几个变量的内存地址都不相同，说明发生了赋值。

#### map、slice和数组

slice，map和数组在初始化和按索引设置的时候也会创建副本:

```golang
package main
import "fmt"
type Bird struct {
	Age  int
	Name string
}
var parrot1 = Bird{Age: 1, Name: "Blue"}
func main() {
	fmt.Printf("parrot1:\t\t %+v, \t\t内存地址：%p\n", parrot1, &parrot1)
//slice
	s := []Bird{parrot1}
	s = append(s, parrot1)
	parrot1.Age = 3
	fmt.Printf("parrot2:\t\t %+v, \t\t内存地址：%p\n", s[0], &(s[0]))
	fmt.Printf("parrot3:\t\t %+v, \t\t内存地址：%p\n", s[1], &(s[1]))
	parrot1.Age = 1
	//map
	m := make(map[int]Bird)
	m[0] = parrot1
	parrot1.Age = 4
	fmt.Printf("parrot4:\t\t %+v\n", m[0])
	parrot1.Age = 5
	parrot5 := m[0]
	fmt.Printf("parrot5:\t\t %+v, \t\t内存地址：%p\n", parrot5, &parrot5)
	parrot1.Age = 1
	//array
	a := [2]Bird{parrot1}
	parrot1.Age = 6
	fmt.Printf("parrot6:\t\t %+v, \t\t内存地址：%p\n", a[0], &a[0])
	parrot1.Age = 1
	a[1] = parrot1
	parrot1.Age = 7
	fmt.Printf("parrot7:\t\t %+v, \t\t内存地址：%p\n", a[1], &a[1])
}
```

输出结果

```golang
parrot1:		 {Age:1 Name:Blue}, 		内存地址：0xfa0a0
parrot2:		 {Age:1 Name:Blue}, 		内存地址：0xc4200160f0
parrot3:		 {Age:1 Name:Blue}, 		内存地址：0xc420016108
parrot4:		 {Age:1 Name:Blue}
parrot5:		 {Age:1 Name:Blue}, 		内存地址：0xc420012320
parrot6:		 {Age:1 Name:Blue}, 		内存地址：0xc420016120
parrot7:		 {Age:1 Name:Blue}, 		内存地址：0xc420016138
```

可以看到 slice/map/数组 的元素全是原始变量的副本， *副本*。

#### for－range循环

for-range循环也是将元素的副本赋值给循环变量，所以变量得到的是集合元素的副本。

```golang
package main
import "fmt"
type Bird struct {
	Age  int
	Name string
}
var parrot1 = Bird{Age: 1, Name: "Blue"}

func main() {
	fmt.Printf("parrot1:\t\t %+v, \t\t内存地址：%p\n", parrot1, &parrot1)
	//slice
	s := []Bird{parrot1, parrot1, parrot1}
	s[0].Age = 1
	s[1].Age = 2
	s[2].Age = 3
	parrot1.Age = 4
	for i, p := range s {
		fmt.Printf("parrot%d:\t\t %+v, \t\t内存地址：%p\n", (i + 2), p, &p)
	}
	parrot1.Age = 1
	//map
	m := make(map[int]Bird)
	parrot1.Age = 1
	m[0] = parrot1
	parrot1.Age = 2
	m[1] = parrot1
	parrot1.Age = 3
	m[2] = parrot1
	parrot1.Age = 4
	for k, v := range m {
		fmt.Printf("parrot%d:\t\t %+v, \t\t内存地址：%p\n", (k + 2), v, &v)
	}
	parrot1.Age = 4
	//array
	a := [...]Bird{parrot1, parrot1, parrot1}
	a[0].Age = 1
	a[1].Age = 2
	a[2].Age = 3
	parrot1.Age = 4
	for i, p := range a {
		fmt.Printf("parrot%d:\t\t %+v, \t\t内存地址：%p\n", (i + 2), p, &p)
	}
}
```

输出结果

```golang
parrot1:		 {Age:1 Name:Blue}, 		内存地址：0xfb0a0
parrot2:		 {Age:1 Name:Blue}, 		内存地址：0xc4200122a0
parrot3:		 {Age:2 Name:Blue}, 		内存地址：0xc4200122a0
parrot4:		 {Age:3 Name:Blue}, 		内存地址：0xc4200122a0
parrot2:		 {Age:1 Name:Blue}, 		内存地址：0xc420012320
parrot3:		 {Age:2 Name:Blue}, 		内存地址：0xc420012320
parrot4:		 {Age:3 Name:Blue}, 		内存地址：0xc420012320
parrot2:		 {Age:1 Name:Blue}, 		内存地址：0xc4200123a0
parrot3:		 {Age:2 Name:Blue}, 		内存地址：0xc4200123a0
parrot4:		 {Age:3 Name:Blue}, 		内存地址：0xc4200123a0
```

*注意循环变量是重用的，所以你看到它们的地址是相同的。*

#### channel

往channel中send对象的时候也会创建对象的副本:

```golang
package main
import "fmt"
type Bird struct {
	Age  int
	Name string
}
var parrot1 = Bird{Age: 1, Name: "Blue"}

func main() {
	ch := make(chan Bird, 3)
	fmt.Printf("parrot1:\t\t %+v, \t\t内存地址：%p\n", parrot1, &parrot1)
	ch <- parrot1
	parrot1.Age = 2
	ch <- parrot1
	parrot1.Age = 3
	ch <- parrot1
	parrot1.Age = 4
	p := <-ch
	fmt.Printf("parrot%d:\t\t %+v, \t\t内存地址：%p\n", 2, p, &p)
	p = <-ch
	fmt.Printf("parrot%d:\t\t %+v, \t\t内存地址：%p\n", 3, p, &p)
	p = <-ch
	fmt.Printf("parrot%d:\t\t %+v, \t\t内存地址：%p\n", 4, p, &p)
}
```

输出结果：

```golang
parrot1:		 {Age:1 Name:Blue}, 		内存地址：0xfa0a0
parrot2:		 {Age:1 Name:Blue}, 		内存地址：0xc4200122a0
parrot3:		 {Age:2 Name:Blue}, 		内存地址：0xc4200122a0
parrot4:		 {Age:3 Name:Blue}, 		内存地址：0xc4200122a0
```

注意：因为变量`p`是重复使用的，所有地址都是相同的。

#### 函数参数和返回值

将变量作为参数传递给函数和方法会发生副本的创建。
对于返回值，将返回值赋值给其它变量或者传递给其它的函数和方法，就会创建副本。

#### Method Receiver

因为方法(method)最终会产生一个receiver作为第一个参数的函数(参看规范)，所以就比较好理解method receiver的副本创建的规则了。
当receiver为`T`类型时，会发生创建副本，调用副本上的方法。
当receiver为`*T`类型时,只是会创建对象的指针，不创建对象的副本，方法内对receiver的改动会影响原始值。

### 不同类型的副本创建

#### bool,数值和指针

bool和数值类型一般不必考虑指针类型，原因在于这些对象很小，创建副本的开销可以忽略。只有你在想修改同一个变量的值的时候才考虑它们的指针。

指针类型就不用多说了，和数值类型类似。

#### 数组

数组是值类型，赋值的时候会发生原始数组的复制，所以对于大的数组的参数传递和赋值，一定要慎重。

```golang
package main
import "fmt"
func main() {
	a1 := [3]int{1, 2, 3}
	fmt.Printf("a1:\t\t %+v, \t\t内存地址：%p\n", a1, &a1)
	a2 := a1
	a1[0] = 4
	a1[1] = 5
	a1[2] = 6
	fmt.Printf("a2:\t\t %+v, \t\t内存地址：%p\n", a2, &a2)
}
```

输出

```golang
a1:		 [1 2 3], 		内存地址：0xc420012260
a2:		 [1 2 3], 		内存地址：0xc4200122c0
```

对于`[...]T`和`[...]*T`的区别，我想你也应该清楚了，`[...]*T`创建的副本的元素时元数组元素指针的副本。

#### map、slice 和 channel

网上一般说， 这三种类型都是指向指针类型，指向一个底层的数据结构。
因此呢，在定义类型的时候就不必定义成`*T`了。

当然你可以这么认为，不过我认为这是不准确的，比如slice,其实你可以看成是`SliceHeader`对象，只不过它的数据`Data`是一个指针，所以它的副本的创建对性能的影响可以忽略。

#### 字符串

string类型类似slice,它等价`StringHeader`。所以很多情况下会用`unsafe.Pointer`与`[]byte`类型进行更有效的转换，因为直接进行类型转换`string([]byte)`会发生数据的复制。

字符串比较特殊，它的值不能修改，任何想对字符串的值做修改都会生成新的字符串。

大部分情况下你不需要定义成`*string`。唯一的例外你需要 `nil`值的时候。我们知道，类型string的空值/缺省值为"",但是如果你需要`nil`，你就必须定义`*string`。举个例子，在对象序列化的时候""和`nil`表示的意义是不一样的，""表示字段存在，只不过字符串是空值，而`nil`表示字段不存在。

#### 函数

函数也是一个指针类型，对函数对象的赋值只是又创建了一个对次函数对象的指针。

```golang
package main
import "fmt"
func main() {
	f1 := func(i int) {}
	fmt.Printf("f1:\t\t %+v, \t\t内存地址：%p\n", f1, &f1)
	f2 := f1
	fmt.Printf("f2:\t\t %+v, \t\t内存地址：%p\n", f2, &f2)
}
```

输出结果:

```golang
f1:		 0x2200, 		内存地址：0xc420028020
f2:		 0x2200, 		内存地址：0xc420028030
```

### 参考文档

1. [https://www.reddit.com/r/golang/comments/5lheyg/returning_t_vs_t/?](https://www.reddit.com/r/golang/comments/5lheyg/returning_t_vs_t/?)
2. [https://github.com/google/go-github/issues/180](https://github.com/google/go-github/issues/180)
3. [http://openmymind.net/Things-I-Wish-Someone-Had-Told-Me-About-Go/](http://openmymind.net/Things-I-Wish-Someone-Had-Told-Me-About-Go/)
4. [http://goinbigdata.com/golang-pass-by-pointer-vs-pass-by-value/](http://goinbigdata.com/golang-pass-by-pointer-vs-pass-by-value/)
5. [https://groups.google.com/forum/#!topic/golang-nuts/__BPVgK8LN0](https://groups.google.com/forum/#!topic/golang-nuts/__BPVgK8LN0)
6. [https://golang.org/ref/spec](https://golang.org/ref/spec)
7. [https://golang.org/doc/faq](https://golang.org/doc/faq)
8. [https://golang.org/doc/effective_go.html](https://golang.org/doc/effective_go.html)
9. [https://nathanleclaire.com/blog/2014/08/09/dont-get-bitten-by-pointer-vs-non-pointer-method-receivers-in-golang/](https://nathanleclaire.com/blog/2014/08/09/dont-get-bitten-by-pointer-vs-non-pointer-method-receivers-in-golang/)
10. [https://dhdersch.github.io/golang/2016/01/23/golang-when-to-use-string-pointers.html](https://dhdersch.github.io/golang/2016/01/23/golang-when-to-use-string-pointers.html)
11. [https://dave.cheney.net/2016/03/19/should-methods-be-declared-on-t-or-t](https://dave.cheney.net/2016/03/19/should-methods-be-declared-on-t-or-t)
12. [http://colobu.com/2016/10/28/When-are-function-parameters-passed-by-value/](http://colobu.com/2016/10/28/When-are-function-parameters-passed-by-value/)

