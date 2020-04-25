---
layout: post
comments: true
title: mockito学习总结
date: 2018-12-06 14:33:01
tags:
- mockito
categories:
- 测试
---

### 前言

在开发工作中，单元测试时必不可少的一环。而在编写单元测试代码中，你肯定会遇到依赖的服务，接口和一些运行环境中的对象无法构造的情况。此时你就需要一个功能强大的mock框架，让它来帮你完成这些功能，而[Mockito](https://site.mockito.org/)就是Java开发环境中一个功能强大的Mock框架。类似的还有powermock, easymock等。

本文是对以往工作中使用到Mockito的一些功能做一次总结，方便以后翻看，并帮助需要使用Mockito的人。

<!-- more -->

### 安装

Mockito不需要安装，根据不同的项目管理工具（maven, gradle）引入Mockito的依赖即可。

```maven
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-all</artifactId>
    <version>1.10.19</version>
    <scope>test</scope>
</dependency>
```

### 创建Mock对象

```java
HttpServletRequest  mockedRequest = Mockito.mock(HttpServletRequest.class);
```

### mock对象方法

#### mock方法方法返回值

```java
Mockito.when(mockedRequest.getHeader("X-Forwarded-For")).thenReturn("127.0.0.1");

List mockedList = mock(List.class);
when(mockedList.get(anyInt())).thenReturn(1, 2, 3);
```

#### mock方法异常

```java
List mockedList = mock(List.class);
when(mockedList.get(0)).thenReturn();
when(mockedList.get(1)).thenThrow(new IndexOutOfBoundsException());

//输出:0
 System.out.println(mockedList.get(0));

 //抛异常
 System.out.println(mockedList.get(1));

 //输出null, 这是因为get(999)没有被mock
 System.out.println(mockedList.get(999));
 
 
doThrow(new RuntimeException()).when(mockedList).clear();

//following throws RuntimeException:
mockedList.clear();
```

### mock对象行为测试。

Mockito 会追踪 Mock 对象的所用方法调用和调用方法时所传递的参数，可以通过 `verify()`来测试mock对象行为。例如方法是否被调用，方法被调用的次数等。

```java
List mockedList = mock(List.class);
mockedList.add("one");
mockedList.add("two");
mockedList.add("three times");
mockedList.add("three times");
mockedList.add("three times");

when(mockedList.size()).thenReturn(5);

Assert.assertEquals(mockedList.size(), 5);

//add("one") 执行被调用一次，否则会抛异常
verify(mockedList, atLeastOnce()).add("one");
//add("two") 被调用恰好一次，否则会抛异常
verify(mockedList, times(1)).add("two");
//add("three times") 被调用恰好三次，否则会抛异常
verify(mockedList, times(3)).add("three times");
//isEmpty 从来没有被调用过
verify(mockedList, never()).isEmpty();
```

### 方法调用顺序测试

```java
List<String> firstMock = mock(List.class);
List<String> secondMock = mock(List.class);

firstMock.add("was called first");
firstMock.add("was called first");
secondMock.add("was called second");
secondMock.add("was called third");

/* 如果mock方法的调用顺序和InOrder中verify的顺序不同，那么测试将执行失败。 */

InOrder inOrder = inOrder(firstMock, secondMock);
//验证firstMock是否调用了2次add("was called first")，如果把2修改为3则会抛异常
inOrder.verify(firstMock, times(2)).add("was called first");
//验证 secondMock是否分别调用add("was called second")，add("was called third")一次
inOrder.verify(secondMock).add("was called second");
inOrder.verify(secondMock).add("was called third");

// 因为在secondMock.add("was called third")之后已经没有多余的方法调用了。
inOrder.verifyNoMoreInteractions();// 表示此方法调用后再没有多余的交互
```

### 参数匹配

Mockito使用`equals()`方法验证参数值。 当需要更加灵活的验证方式时，可以使用参数匹配器：

```java
//使用Mockito内置的 anyInt() 参数匹配器
 when(mockedList.get(anyInt())).thenReturn("element");

 //使用自定义的参数匹配器
 when(mockedList.contains(argThat(isValid()))).thenReturn("element");

 //输出：element
 System.out.println(mockedList.get(999));

 //you can also verify using an argument matcher
 verify(mockedList).get(anyInt());

 //argument matchers can also be written as Java 8 Lambdas
 verify(mockedList).add(argThat(someString -> someString.length() > 5));
```

### 自定义Answer接口

```java
 List<String> mock = mock(List.class);
when(mock.get(4)).thenAnswer(new Answer() {
    public String answer(InvocationOnMock invocation) throws Throwable {
        Object[] args = invocation.getArguments();
        Integer num = (Integer) args[0];
        if (num > 3) {
            return "yes";
        }
        throw new RuntimeException();
    }
});
System.out.println(mock.get(4));
```

### mock `void` 方法

doReturn()|doThrow()| doAnswer()|doNothing()|doCallRealMethod()

#### doThrow 

当需要mock一个返回值是void的方法，调用该方法返回异常的情况，使用doThrow

```java
doThrow(new RuntimeException()).when(mockedList).clear();

//following throws RuntimeException:
mockedList.clear();
```

#### doReturn

有些特殊情况下，不能直接使用`when(...).thenReturn`，此时只能使用`doReturn`了。

##### When spying real objects and calling real methods on a spy brings side effects

```java
List list = new LinkedList();
List spy = spy(list);

//Impossible: real method is called so spy.get(0) throws IndexOutOfBoundsException (the list is yet empty)
when(spy.get(0)).thenReturn("foo");

//You have to use doReturn() for stubbing:
doReturn("foo").when(spy).get(0);
```

##### Overriding a previous exception-stubbing:

```java
when(mock.foo()).thenThrow(new RuntimeException());

//Impossible: the exception-stubbed foo() method is called so RuntimeException is thrown.
when(mock.foo()).thenReturn("bar");

//You have to use doReturn() for stubbing:
doReturn("bar").when(mock).foo();
```

##### doAnswer

Use doAnswer() when you want to stub a void method with generic Answer.
Stubbing voids requires different approach from when(Object) because the compiler does not like void methods inside brackets...

Example:

```java
doAnswer(new Answer() {
      public Object answer(InvocationOnMock invocation) {
          Object[] args = invocation.getArguments();
          Mock mock = invocation.getMock();
          return null;
      }})
  .when(mock).someMethod();
```

##### doNothing

Use doNothing() for setting void methods to do nothing. Beware that void methods on mocks do nothing by default! However, there are rare situations when doNothing() comes handy:

- Stubbing consecutive calls on a void method:

```java
doNothing().
doThrow(new RuntimeException())
.when(mock).someVoidMethod();

//does nothing the first time:
mock.someVoidMethod();

//throws RuntimeException the next time:
mock.someVoidMethod();
```

- When you spy real objects and you want the void method to do nothing:

```java
List list = new LinkedList();
List spy = spy(list);

//let's make clear() do nothing
doNothing().when(spy).clear();

spy.add("one");

//clear() does nothing, so the list still contains "one"
spy.clear();
```

##### doCallRealMethod

当需要调用真是对象的方法时，使用`doCallRealMethod`

```java
@Test  
public void callRealMethodTest() {  
    Jerry jerry = mock(Jerry.class);  
  
    doCallRealMethod().when(jerry).goHome();  
    doCallRealMethod().when(jerry).doSomeThingB();  
  
    jerry.goHome();  
  
    verify(jerry).doSomeThingA();  
    verify(jerry).doSomeThingB();  
}  
  
class Jerry {  
    public void goHome() {  
        doSomeThingA();  
        doSomeThingB();  
    }  
  
    // real invoke it.  
    public void doSomeThingB() {  
        System.out.println("good day");  
  
    }  
  
    // auto mock method by mockito  
    public void doSomeThingA() {  
        System.out.println("you should not see this message.");  
  
    }  
} 
```

通过代码可以看出Jerry是一个mock对象， goHome()和doSomeThingB()是使用了实际调用技术，而doSomeThingA()被mockito执行了默认的answer行为（这里是个void方法，so，什么也不干）。

### @Mock 简化mock对象的创建

Mockito提供了注解`@Mock`来简化Mock对象的创建：

```java
public class ArticleManagerTest {

    @Mock private ArticleCalculator calculator;
    @Mock private ArticleDatabase database;
    @Mock private UserProvider userProvider;
    
    private ArticleManager manager;
    
    @BeforeClass
    public void setup() {    
        // 需要在单元测试方法前执行
        MockitoAnnotations.initMocks(testClass);
    }
}
```


### spy真实对象

可以创建真实对象的spy对象。当代用spy对象的方法时，真实对象的方法会被调用（如果被调用的方法没有被mock的话）。

spy对象可以被用在部分mock场景下。

```java
List list = new LinkedList();
List spy = spy(list);

//可以mock部分方法调用
when(spy.size()).thenReturn(100);

//using the spy calls *real* methods
spy.add("one");
spy.add("two");

//输出：one
System.out.println(spy.get(0));

//size() 返回100
System.out.println(spy.size());

//optionally, you can verify
verify(spy).add("one");
verify(spy).add("two");
```

### 参考

https://heipark.iteye.com/blog/1496603
http://static.javadoc.io/org.mockito/mockito-core/2.23.4/org/mockito/Mockito.html

