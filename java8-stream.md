---
layout: post
comments: true
title: java8-stream
date: 2016-12-25 14:58:52
tags:
- java8
categories:
- java
---

### stream 定义

> A sequence of elements supporting sequential and parallel aggregate operations.

上面的定义出自jdk8中Stream类的说明。意思是：Stream 是一个支持顺序和并行聚合操作的元素序列。

<!-- more -->
### 入门示例

```java
    int sum = widgets.stream().filter(w -> w.getColor() == RED).mapToInt(w -> w.getWeight()).sum();
    
```

**解释**：在上面的代码中，widgets的类型是`Collection<Widget>`, 通过`Collection`接口的`stream`方法创造了一个类型参数为`Widget`的`Stream`；通过`filter`方法生产一个只包含红颜色`Widget`的stream；然后通过`mapToInt`将`filter`方法生产的stream转为包含`int`元素的stream；最后进行求和运算。

### stream分类

除了针对引用类型的Stream外还有专门对应基本数据类型的Stream，例如：IntStream, LongStream。不论是何种类型的Stream都遵循Stream接口定义的规范。

### Stream详解

**Stream计算过程**

通常为了进行计算会将Stream组合成Stream管道，一个Stream管道是由一个输入源（数组，集合，生成器函数，I/O 通道等），零个或多个中间操作（如 filter， map 等操作）， 一个终结操作组成（该操作会会产生一个结果或副作用；例如Stream.count()或Stream.forEach(Consumer)）。

Stream都是惰性求值的，只有当最终调用了Stream中的终结方法才会正常执行从数据源获取值的操作和定义的中间操作。

**Stream 和 Collection 的区别**

Collections ：代表所有的Collection实现
Streams ：代表所有的Stream实现

Collections 和 Streams 表面上看起来有相似之处，但两者的目标不同。Collections 的焦点在于高效的管理和访问其中的元素。与Collection相比Stream并没有直接提供一些访问或修改其中元素的方法，它的关注点在于提供一些针对Stream数据输入源的聚合操作。

一个Stream管道可以被看做是针对它的数据源的查询操作，除非数据源是明确的设计用来针对并发操作的（例如：ConcurrentHashMap）。如果Stream的数据源在Stream的使用过程被外界所修改，则会导致不可预知的或错误的结果。也就是说Stream的数据源最好是不可变的。

Stream的大多数操作的参数都是用户自己可以定制的，就想上面例子提到的传入`mapToInt`方法的lambda表达式`w -> w.getWeight()`。为了保证Stream方法的正确行为，这些行为参数必须是无副作用的，大多数情况下是无状态的，也就是说不能修改Stream数据源中的数据。这种类型的参数总是函数式接口的实例，lambda表达式或方法引用。除非是另有说明，否则这些参数也必须是非null的。

### Stream 主要方法

#### empty

empty方法签名为：public static<T> Stream<T> empty()

empty方法放回一个空的顺序的Stream对象。

#### of(T t)

of方法的签名为：public static<T> Stream<T> of(T t)

of方法返回只包含一个元素顺序的Stream对象。

#### of(T... values)

方法签名： public static<T> Stream<T> of(T... values) 

该方法返回一个包含多个元素的顺序的Stream对象。

#### generate

generate方法的签名是：public static<T> Stream<T> generate(Supplier<T> s)

generate方法接受一个类型为`Supplier`的函数式接口并返回一个含有无穷个元素的无序的Stream，其中的元素是由参数 supplier生成的。

#### filter

filter方法的签名：Stream<T> filter(Predicate<? super T> predicate);

filter方法的作用是用来过滤Stream的元素的，参数是一个函数式接口`Predicate`。

例子：

```java Stream.filter
 final Collection<Integer> integers = Arrays.asList(3, 1, 4, 8, 5);
        Collection<Integer> result = integers.stream().filter(num -> num >= 5).collect(Collectors.toList());
System.out.println(result);
```

#### map

map方法的签名是：<R> Stream<R> map(Function<? super T, ? extends R> mapper);

map方法的作用是转换Stream中的元素类型， T 转为 R 并返回一个新的Stream对象。

```java Stream.map
Stream.of("1", "2").map(str -> Integer.parseInt(str)).collect(Collectors.toList());
```

#### mapToInt

mapToInt的方法签名为： IntStream mapToInt(ToIntFunction<? super T> mapper);

mapToInt方法接受一个类型为`ToIntFunction`的函数接口为参数并返回一个`IntStream`。

类似的方法还有`mapToLong`,`mapToDouble`

#### flatMap

flatMap方法的签名为：<R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper);

flatMap方法本质和其它类型的map一样都是对Stream中的元素进行转换，但区别是flatMap将原Stream中的元素转为一个Stream对象。例如：

```java Stream.flatMap
Stream.of("123", "456").flatMap(str -> str.chars().boxed()).count();
//输出结果为：6
```

类似的还有flatMapToInt，flatMapToLong, flatMapToDouble

#### reduce 

reduce方法有三个，分别是：

**T reduce(T identity, BinaryOperator<T> accumulator);**

该方法接受一个初始值`identity`和一个相关的计算函数并返回reduced后的值。可以翻译为如下的代码

```java
T result = identity;
for (T element : this stream){
    result = accumulator.apply(result, element)
}
return result;
```
一个常见的使用该方法的例子就是累加：

```java
Stream.of(1, 2, 3).reduce(0, (res, num) -> res + num);
//或
Stream.of(1, 2, 3).reduce(0, Integer::sum);

//0就是初始值
```

这里需要注意的是参数`identity`对参数`accumulator`来说必须是一个恒等式。也就是说对于所有的`t`, `accumulator.apply(identity, t)` 和 `t` 是等价的。有点难以理解吧？举个例子就是：

0 + 1 = 1； 0就是identity。 因为0加上任何数都等于任何数。


**Optional<T> reduce(BinaryOperator<T> accumulator);**

该方法比上面的方法少了第一个参数，但是功能都是一样的，只是计算的逻辑稍有不同。翻译为普通的代码就是：

```java
boolean foundAny = false;
T result = null;
for (T element : this stream) {
    if (!foundAny) {
        foundAny = true;
        result = element;
    } else {
        result = accumulator.apply(result, element);
    }
}
return foundAny ? Optional.of(result) : Optional.empty();
```

还是累加的例子：

```java
Stream.of(1, 2, 3).reduce(Integer::sum);
```

**<U> U reduce(U identity,BiFunction<U, ? super T, U> accumulator,BinaryOperator<U> combiner);**

该方法和第一个方法的区别是多了第三个参数`combiner`并且参数`identity`对`combiner`来说必须满足恒等式。这个好第一个方法的规则是相同的。还有就是对所有的`u`和`t`来说下面的等式必须成立：

**combiner.apply(u, accumulator.apply(identity, t)) == accumulator.apply(u, t)**

例子：

```java
Stream.of(1, 2, 3, 4).parallel().reduce(0, Integer::sum, Integer::sum)
```
第三个参数`combiner`只有在并行的情况下才会起作用。

```java
Integer ageSum = persons.parallelStream().reduce(
        0,
        (sum, p) -> {
            System.out.format("accumulator: sum=%s; person=%s\n", sum, p);
            return sum += p.age;
        },
        (sum1, sum2) -> {
            System.out.format("combiner: sum1=%s; sum2=%s\n", sum1, sum2);
            return sum1 + sum2;
        });

// accumulator: sum=0; person=Pamela
// accumulator: sum=0; person=David
// accumulator: sum=0; person=Max
// accumulator: sum=0; person=Peter
// combiner: sum1=18; sum2=23
// combiner: sum1=23; sum2=12
// combiner: sum1=41; sum2=35
```

#### distinct

distinct方法是见名知意的， 直接上代码

```java
Stream.of(1, 2, 3, 2).distinct().count();
//输出 3
```

#### sorted

sorted 也是非常简单

```java
Stream.of(1,4,2,3).sorted().collect(Collectors.toList())

//输出 [1, 2, 3, 4]

Stream.of("aaa", "be").sorted((a, b) -> a.length() - b.length()).collect(Collectors.toList());

//输出 [aaa, be] 
```

#### peek

peek方法是非常有用的调试方法，对该方法的调用并不会影响我们正常的逻辑，在调试代码是可以在peek方法中下断点。

```java
Stream.of(1, 2, 3).peek(num -> System.out.println(num)).count();    
```

#### skip 和 limit

让代码来解释吧：

```java
Arrays.stream(arr).sorted().skip(10).limit(5);
```

#### forEach

forEach方法的签名：void forEach(Consumer<? super T> action);

该方法针对Stream中的每一个元素执行参数`action`指定的动作。需要指出的是针对并行的Stream来说该操作的行为是不确定的，该操作也不保证按照元素在Stream中出现的顺序来处理否则会影响并行处理的性能。
对于该Stream中的任何元素来说，该动作都应该在任何时候任意的线程中执行，如果在多个执行线程中需要方法共享的状态数据则需要手动添加必须的同步代码。

```java
IntStream.range(1, 10).forEach(num -> {
    System.out.print(Thread.currentThread().getName() + "-->");
    System.out.println(num);
});
//输出
main-->1
main-->2
main-->3
main-->4
main-->5
main-->6
main-->7
main-->8
main-->9
//并行
IntStream.range(1, 10).parallel().forEach(num -> {
    System.out.print(Thread.currentThread().getName() + "-->");
    System.out.println(num);
});
//输出
ForkJoinPool.commonPool-worker-1-->8
ForkJoinPool.commonPool-worker-1-->9
ForkJoinPool.commonPool-worker-2-->2
ForkJoinPool.commonPool-worker-3-->3
ForkJoinPool.commonPool-worker-2-->1
ForkJoinPool.commonPool-worker-1-->7
ForkJoinPool.commonPool-worker-2-->5
ForkJoinPool.commonPool-worker-3-->4
main-->6
```    

#### forEachOrdered

forEachOrdered方法的签名：void forEachOrdered(Consumer<? super T> action);

从名字就能看出来，该方法是按元素在Stream中出现的顺序来处理的，一次处理一个。

#### toArray

toArray方法签名：Object[] toArray();

该方法返回一个包含该Stream中所有元素的数组。

#### toArray(IntFunction<A[]> generator);

方法签名：<A> A[] toArray(IntFunction<A[]> generator);

该方法与前面的一个方法功能是相同的，都是返回一个元素数组，但不同的是前一个方法返回的是一个Object[]数组，后一个方法需要使用者提供一个生产结果数组的方法。

    Stream.of(1, 2, 3).toArray();
    Stream.of(1, 2, 3).toArray(Integer[]::new);

#### count

方法签名：long count();

放回该Stream中的元素个数。

### max和min

二者的方法签名如下：

Optional<T> min(Comparator<? super T> comparator);
Optional<T> max(Comparator<? super T> comparator);

这2个方法分别返回该Stream中的最小元素和最大元素；如何使用就不用多说了吧。需要注意的是如果Stream是空的则需要判断返回值是否有值。

#### anyMatch

方法签名：boolean anyMatch(Predicate<? super T> predicate);

如果该Stream有一个元素满足参数指定的条件则返回true，如果Stream为空或者都不满足则返回false。

#### allMatch

方法签名：boolean allMatch(Predicate<? super T> predicate);

如果该Stream中的所有元素都满足指定的条件则返回true，如果Stream为空或有一个不满足就返回false。

#### noneMatch

方法签名：boolean noneMatch(Predicate<? super T> predicate);

如果该Stream中的所有元素**都不**满足指定的条件则返回true，如果Stream为空也返回true。

#### findFirst

方法签名：Optional<T> findFirst();

返回该Stream中的第一个元素，如果该Stream是无序的则可能返回其中的任意一个元素。如果为空则返回一个空的Optional

#### findAny

方法签名：Optional<T> findAny();

返回一个表示Stream中任意一个元素的Optional对象。如果该Stream为空则返回一个空的Optional对象。

#### Collector

顾名思义，Collector就是收集器的意思。用来收集流式计算的最终结果。例如：

```java
    List<String> collect = list.stream().map(Person::getName).collect(Collectors.toList());
```

以上代码最后的Collectors.toList()，就是Java8原生的收集器，用于把流结果放到一个List中。

原生的收集器还有很多，大多定义在Collectors下的静态方法。

通过收集器的组合，能产生很复杂的收集效果。


#### collect-1

方法签名：  <R> R collect(Supplier<R> supplier,
                  BiConsumer<R, ? super T> accumulator,
                  BiConsumer<R, R> combiner);

先看代码：

```java
Stream.of(1, 2, 3).collect((Supplier<ArrayList<Integer>>)ArrayList::new, (left, integer) -> left.add(integer), (left, right) -> left.addAll(right))
```

**supplier**: 提供处理后结果的容器； 在上面的代码中容器是一个List

**accumulator**: 进行累计计算的函数，该函数接受2个参数，第一个参数是结果容器，第二个参数是Stream中的元素

**combiner**: 从名字也可以看出来是对多个结果进行组合的函数，接受的2个参数都是结果容器类型，在上面的代码中2个参数都是ArrayList.

该方法是比较复杂的，需要理解其中的工作原理。


#### collect-2

方法签名：<R, A> R collect(Collector<? super T, A, R> collector);

该方法使用的频率比较的高，见下面的代码：

```java
Stream.of("1", "2").map(str -> Integer.parseInt(str)).collect(Collectors.toList());
```

如上所示，我们经常使用的是Collectors工具类提供的收集器生成方法来生成参数收集器，一般也不需要我们自己开发。但是总有需要自己定制的收集器，一般也不难实现。

#### 自定义收集器

自定义收集器不算难，实现接口`Collector`就可以了。

    public interface Collector<T, A, R> 
    
Collector接口有三个泛型参数：

1. T 定义流中的数据类型

2. A 表示定义一个初始容器的类型

3. R 表示最终转换的类型

Collector接口主要定义了如下几个方法：

- supplier：这个方法主要是生成一个初始容器，用于存放转换的数据。它返回一个Supplier<A>类型，用Lambda表示为() -> A
- accumulator: 这个方法是将初始容器与Stream中的每个元素进行计算，它返回一个BiConsumer<A, T>类型，用Lambda表示为(A, T) -> void
- combiner: 这个方法用于在并发Stream中，将多个容器组合成一个容器，它返回一个BinaryOperator<A>类型，用Lambda表示为(A, A) -> A
- finisher：这个方法用于将初始容器转换成最终的值，它返回一个Function<A, R>类型，用Lambda表示为A -> R
- characteristics: 这个方法返回该Collector具有的哪些特征，返回的是一个Set, 分别是CONCURRENT(并发), UNORDERED(未排序)，IDENTITY_FINISH(finisher方法直接返回初始容器)等特征的组合。

在Collectors中有一个joining方法，它是将Stream中的字符序列类型的元素串接在一起，下面是一个例子：

```java
// 输出 abcd
System.out.println(Stream.of("a", "b", "c", "d").collect(Collectors.joining()));
```

下面通过自己实现joining方法来理解如何实现Collector接口。

定义自己实现的JoinCollector类    

```java
public class JoinCollector implements Collector<CharSequence, StringBuilder, String> {
    @Override
    public Supplier<StringBuilder> supplier() {
        return StringBuilder::new;
    }

    @Override
    public BiConsumer<StringBuilder, CharSequence> accumulator() {
        return StringBuilder::append;
    }

    @Override
    public BinaryOperator<StringBuilder> combiner() {
        return StringBuilder::append;
    }

    @Override
    public Function<StringBuilder, String> finisher() {
        return StringBuilder::toString;
    }

    @Override
    public Set<Characteristics> characteristics() {
        return Collections.emptySet();
    }
}
```

其中，supplier方法返回一个StringBuilder对象，用于表示初始容器。
accumulator方法返回的是StringBuilder的append方法的方法引用， 用于将Stream中的元素加入到初始容器StringBuilder中。
combiner方法用于将多个StringBuilder容器通过append方法组合成一个StringBuilder对象。
finisher方法是将StringBuilder对象通过toString方法最终转换成String对象。
使用自定义的Collector的例子如下

```java
// 输出 abcd
System.out.println(Stream.of("a", "b", "c", "d").collect(new JoinCollector()));
```

得到的结果与Collectors.joining方法是一样的

### 总结

未完待续。








