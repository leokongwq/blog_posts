---
layout: post
comments: true
title: golang的unsafe包简介
date: 2016-10-13 22:00:31
tags:
categories:
    - golang
---

                        
> 包简介：`unsafe`包包含一些可以安全的完成golang中类型转换工作的操作。导入`unsafe`包的包可能是不可移植的并且是违背了Go 1兼容性规则的。

<!-- more -->

### Pointer
// Pointer 代表了一个指向任意类型的指针。针对Pointer有如下四个特殊的操作，这些操作只针对`Pointer`类型有效，对其它类型是不可用的

 1. 任何类型的指针都可以转为`Pointer`类型
 2. `Pointer`可以转为任何类型的指针
 3. 一个`uintptr`可以转为`Pointer`
 4. 一个`Pointer`可以转为一个`uintptr`
 
`Pointer`允许程序跳过类型系统，并且可以读写任意的内存。因此使用`Pointer`需要非常小心，仔细。 

下面介绍的`Pointer`使用方式是推荐的，如果不通过下面的范式使用`Pointer`则会导致代码行为错误或将来的某个时候不可以用。即使是下面介绍的是否方式也需要谨慎使用，尽量避免使用。

执行"go vet"可以获取不被下面这些范式肯定的`Pointer`使用方式。但是`go vet`的静默输出并不保证代码是否可用。

 1. **转换 *T1 为 *T2**

Provided that T2 is no larger than T1 and that the two share an equivalent memory layout,   this conversion allows reinterpreting data of one type as data of another type. An          example is the implementation of math.Float64bits:

{% codeblock lang:golang %}
    func Float64bits(f float64) uint64 {
        return *(*uint64)(unsafe.Pointer(&f))
    }
{% endcodeblock %}

 2.**将一个指针转为`uintptr`(but not back to Pointer).**

将一个`Pointer`转为`uintptr`会缩小该Pointer值的内存地址。通常情况下如此使用时为了打印该值。 

将一个`uintptr`转回为Pointer通常是不可以的。

一个`uintptr`是一个整数而不是引用。将一个`Pointer`转为一个`uintptr`会创建一个没有指针语意的integer值。及时该`uintptr`的值是个对象的地址，对垃圾回收器来说，当Pointer指向的对象删除或，垃圾回收器也不会更新uintptr的值。 nor will that uintptr keep the object from being reclaimed.

其余的范式列举了the only valid conversions from uintptr to Pointer.

 
3 . **通过算术将一个`Pointer`转为`uintptr`并转回来**
如果p指向一个分配了内存地址的对象, 可以通过将p转为uintptr然后在加上一个偏移量offset,并可以转回Pointer

{% codeblock lang:golang %}
    p = unsafe.Pointer(uintptr(p) + offset)
{% endcodeblock %}

该模式最长见得使用场景是：方法结构体的字段或数组中的元素

{% codeblock lang:golang %}
    // equivalent to f := unsafe.Pointer(&s.f)
    f := unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Offsetof(s.f))
    // equivalent to e := unsafe.Pointer(&x[i])
    e := unsafe.Pointer(uintptr(unsafe.Pointer(&x[0])) + i*unsafe.Sizeof(x[0]))
{% endcodeblock %}

通过这种方式给指针添加或减少偏移量都是可行的，但是结果必须还是指向原始分配的对象。和 **C** 语言不同， 指针的值添加了偏移量以后不能超过原始对象分配空间的地址范围。

{% codeblock lang:golang %}
    // INVALID: end points outside allocated space.
    var s thing
    end = unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Sizeof(s))
    // INVALID: end points outside allocated space.
    b := make([]byte, n)
    end = unsafe.Pointer(uintptr(unsafe.Pointer(&b[0])) + uintptr(n))
{% endcodeblock %}

**需要注意的是**：所有的转换必须在一个表达式中。

{% codeblock lang:golang %}
    // INVALID: uintptr cannot be stored in variable
    // before conversion back to Pointer.
    u := uintptr(p)
    p = unsafe.Pointer(u + offset)   
{% endcodeblock %}

4 . **当执行`syscall.Syscall`时将一个`Pointer`转为`uintptr`**

`syscall`包中的`Syscall`函数会直接将它们的`uintptr`参数传递给操作系统, 根据系统调用的具体信息会将传入的参数重新解释为指针。这就是为啥系统调用的实现会显示的将确定的参数从`1uintptr`转为pointer。

如果一个`pointer`参数必须被转为`uintptr`作为系统调用函数的参数, 则转换必须在系统调用表达式中：

{% codeblock lang:golang %}
    syscall.Syscall(SYS_READ, uintptr(fd), uintptr(unsafe.Pointer(p)), uintptr(n))
{% endcodeblock %}
    
编译器会将函数调用参数列表中的`Pointer`转为`uintptr`。

对编译器来说如何识别该模式呢？答案是：转换表达式必须出现在参数列表中：

{% codeblock lang:golang %}
    // INVALID: uintptr cannot be stored in variable
    // before implicit conversion back to Pointer during system call.
    u := uintptr(unsafe.Pointer(p))
    syscall.Syscall(SYS_READ, uintptr(fd), u, uintptr(n))
{% endcodeblock %}

5 . **将`reflect.Value.Pointer` 或 `reflect.Value.UnsafeAddr` 的结果从`uintptr`转为`Pointer`**

`reflect`包中`Value`类型的方法中名称为`Pointer` 和 `UnsafeAddr`的方法的返回值类型是`uintptr` 而不是`unsafe.Pointer`,目的是为了使调用者可以将结果转为任意类型而不用dai'aoarbitrary type导入`unsafe`包。然而，这意味着调用结果必须马上再调用完成后转为`Pointer`,并且是在同一个表达式中完成；如下：

{% codeblock lang:golang %}
    p := (*int)(unsafe.Pointer(reflect.ValueOf(new(int)).Pointer()))
{% endcodeblock %}

如上面的例子, 在转换前保存调用的结果不合法的是不和法的：

{% codeblock lang:golang %}
    // INVALID: uintptr cannot be stored in variable
    // before conversion back to Pointer.
    u := reflect.ValueOf(new(int)).Pointer()
    p := (*int)(unsafe.Pointer(u))    
{% endcodeblock %}

6 . **将`reflect.SliceHeader` 或 `reflect.StringHeader` 的数据字段转为`Pointer`或从`Pointer`转为`uintptr`**

As in the previous case, the reflect data structures SliceHeader and StringHeader declare the field Data as a 
uintptr to keep callers from changing the result to an arbitrary type without first importing "unsafe". However, 
this means that SliceHeader and StringHeader are only valid when interpreting the content of an actual slice or string value.

{% codeblock lang:golang %}
    var s string
    hdr := (*reflect.StringHeader)(unsafe.Pointer(&s)) // case 1
    hdr.Data = uintptr(unsafe.Pointer(p))              // case 6 (this case)
    hdr.Len = uintptr(n)
{% endcodeblock %}

In this usage hdr.Data is really an alternate way to refer to the underlying pointer in the slice header,
 not a uintptr variable itself.In general, reflect.SliceHeader and reflect.StringHeader should be used only 
 as *reflect.SliceHeader and *reflect.StringHeader pointing at actual slices or strings, never as plain structs. 
A program should not declare or allocate variables of these struct types.

{% codeblock lang:golang %}
    // INVALID: a directly-declared header will not hold Data as a reference.
    var hdr reflect.StringHeader
    hdr.Data = uintptr(unsafe.Pointer(p))
    hdr.Len = uintptr(n)
    s := *(*string)(unsafe.Pointer(&hdr)) // p possibly already lost
{% endcodeblock %}

                    