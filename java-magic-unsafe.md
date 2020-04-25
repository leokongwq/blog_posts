---
layout: post
comments: true
title: java魔法之unsafe
date: 2016-12-31 18:40:51
tags:
categories:
- java
---

本文翻译自[http://mishadoff.com/blog/java-magic-part-4-sun-dot-misc-dot-unsafe/](http://mishadoff.com/blog/java-magic-part-4-sun-dot-misc-dot-unsafe/)

### 前言

Java是一个安全的编程语言，它能最大程度的防止程序员犯一些低级的错误（大部分是和内存管理有关的）。但凡是不是绝对的，使用Unsafe程序员就可以操作内存，因此可能带来一个安全隐患。

这篇文章是就快速学习下`sun.misc.Unsafe`的公共API和一些有趣的使用例子。

<!-- more -->

### Unsafe 实例化

在使用Unsafe之前我们需要先实例化它。但我们不能通过像`Unsafe unsafe = new Unsafe()`这种简单的方式来实现Unsafe的实例化，这是由于Unsafe的构造方法是私有的。Unsafe有一个静态的getUnsafe()方法，但是如果天真的以为调用该方法就可以的话，那你将遇到一个`SecurityException`异常，这是由于该方法只能在被信任的代码中调用。

```java
public static Unsafe getUnsafe() {
    Class cc = sun.reflect.Reflection.getCallerClass(2);
    if (cc.getClassLoader() != null)
        throw new SecurityException("Unsafe");
    return theUnsafe;
}
```

那Java是如何判断我们的代码是否是受信的呢？它就是通过判断加载我们代码的类加载器是否是根类加载器。 

我们可是通过这种方法将我们自己的代码变为受信的，使用jvm参数`bootclasspath`。如下所示：

    java -Xbootclasspath:/usr/jdk1.7.0/jre/lib/rt.jar:. com.mishadoff.magic.UnsafeClient
    
**但这种方式太难了**
    
Unsafe类内部有一个名为`theUnsafe`的私有实例变量，我们可以通过反射来获取该实例变量。

```java
Field f = Unsafe.class.getDeclaredField("theUnsafe");
f.setAccessible(true);
Unsafe unsafe = (Unsafe) f.get(null);
```

**注意**: 忽略你的IDE提示. 例如, eclipse可能会报这样的错误"Access restriction..." 单如果你运行你的代码，会发现一切正常。如果还是还是提示错误，你可以通过如下的方式关闭该错误提示：

    Preferences -> Java -> Compiler -> Errors/Warnings ->
    Deprecated and restricted API -> Forbidden reference -> Warning           

### Unsafe API

类 [sun.misc.Unsafe](http://www.docjar.com/docs/api/sun/misc/Unsafe.html) 由150个方法组成。事实上这些方法只有几组是非常重要的用来操作不同的对象。下面我们就来看下这些方法中的一部分。

1. **Info** 仅仅是返回一个低级别的内存相关的信息
    * addressSize
    * pageSize

1. **Objects**. 提供操作对象和对象字段的方法

    * allocateInstance
    * objectFieldOffset

1. **Classes**. 提供针对类和类的静态字段操作的方法

    * staticFieldOffset
    * defineClass
    * defineAnonymousClass
    * ensureClassInitialized

1. **Arrays**. 数组操作

    * arrayBaseOffset
    * arrayIndexScale

1. Synchronization. 低级别的同步原语

    * monitorEnter
    * tryMonitorEnter
    * monitorExit
    * compareAndSwapInt
    * putOrderedInt

1. Memory. 直接访问内存的方法

    * allocateMemory
    * copyMemory
    * freeMemory
    * getAddress
    * getInt
    * putInt

### 有趣的使用case

#### 跳过构造初始化
    
allocateInstance方法可能是有用的,当你需要在构造函数中跳过对象初始化阶段或绕过安全检查又或者你想要实例化哪些没有提供公共构造函数的类时就可以使用该方法。考虑下面的类:

```java
class A {
    private long a; // not initialized value

    public A() {
        this.a = 1; // initialization
    }

    public long a() { return this.a; }
}
```

通过构造函数，反射，Unsafe分别来实例化该类结果是不同的：

```java
A o1 = new A(); // constructor
o1.a(); // prints 1

A o2 = A.class.newInstance(); // reflection
o2.a(); // prints 1

A o3 = (A) unsafe.allocateInstance(A.class); // unsafe
o3.a(); // prints 0
```

思考一下这些确保对[Singletons](http://en.wikipedia.org/wiki/Singleton_pattern)模式的影响。

#### 内存泄露

对C程序员来说这中情况是很常见的。

思考一下一些简单的类是如何坚持访问规则的：

```java
class Guard {
    private int ACCESS_ALLOWED = 1;

    public boolean giveAccess() {
        return 42 == ACCESS_ALLOWED;
    }
}
```

客户端代码是非常安全的,调用giveAccess()检查访问规则。不幸的是对所有的客户端代码,它总是返回false。只有特权用户在某种程度上可以改变ACCESS_ALLOWED常量并且获得访问权限。

事实上,这不是真的。这是证明它的代码:

```java
Guard guard = new Guard();
guard.giveAccess();   // false, no access

// bypass
Unsafe unsafe = getUnsafe();
Field f = guard.getClass().getDeclaredField("ACCESS_ALLOWED");
unsafe.putInt(guard, unsafe.objectFieldOffset(f), 42); // memory corruption

guard.giveAccess(); // true, access granted
```

现在所有的客户端都没有访问限制了。

事实上同样的功能也可以通过反射来实现。但有趣的是, 通过上面的方式我们修改任何对象，即使我们没有持有对象的引用。

举个例子, 在内存中有另外的一个Guard对象，并且地址紧挨着当前对象的地址，我们就可以通过下面的代码来修改该对象的`ACCESS_ALLOWED`字段的值。

    unsafe.putInt(guard, 16 + unsafe.objectFieldOffset(f), 42); // memory corruption
    
**注意**，我们没有使用任何指向该对象的引用，16是Guard对象在32位架构上的大小。我们也可以通过`sizeOf`方法来计算Guard对象的大小。

#### sizeOf

使用`objectFieldOffset`方法我们可以实现C风格的sizeof方法。下面的方法实现返回对象的表面上的大小

```java
public static long sizeOf(Object o) {
    Unsafe u = getUnsafe();
    HashSet<Field> fields = new HashSet<Field>();
    Class c = o.getClass();
    while (c != Object.class) {
        for (Field f : c.getDeclaredFields()) {
            if ((f.getModifiers() & Modifier.STATIC) == 0) {
                fields.add(f);
            }
        }
        c = c.getSuperclass();
    }

    // get offset
    long maxSize = 0;
    for (Field f : fields) {
        long offset = u.objectFieldOffset(f);
        if (offset > maxSize) {
            maxSize = offset;
        }
    }

    return ((maxSize/8) + 1) * 8;   // padding
}
```

算法逻辑如下：收集所有包括父类在内的非静态字段，获得每个字段的偏移量，发现最大并添加填充。也许,我错过了一些东西，但是概念是明确的。

更简单的sizeof方法实现逻辑是：我们只读取该对象对应的class对象中关于大小的字段值。在`JVM 1.7 32 位`版本上该表示大小的字段偏移量是12。

```java
public static long sizeOf(Object object){
    return getUnsafe().getAddress(
        normalize(getUnsafe().getInt(object, 4L)) + 12L);
}
```

`normalize`是一个将有符号的int类型转为无符号的long类型的方法。

```java
private static long normalize(int value) {
    if(value >= 0) return value;
    return (~0L >>> 32) & value;
}
```

太棒了,这个方法返回的结果和我们之前的sizeof函数是相同的。

 but it requires specifyng agent option in your JVM.

事实上，对于合适的，安全的，准确的sizeof函数最好使用`java.lang.instrument`包，但它需要特殊的JVM参数。

#### 浅拷贝

在实现了计算对象浅层大小的基础上，我们可以非常容易的添加对象的拷贝方法。标准的办法需要修改我们的代码和Cloneable。或者你可以实现自定义的对象拷贝函数，但它不会变为通用的函数。

浅拷贝：

```java
static Object shallowCopy(Object obj) {
    long size = sizeOf(obj);
    long start = toAddress(obj);
    long address = getUnsafe().allocateMemory(size);
    getUnsafe().copyMemory(start, address, size);
    return fromAddress(address);
}
```

`toAddress` 和 `fromAddress` 将对象转为它在内存中的地址或者从指定的地址内容转为对象。

```
static long toAddress(Object obj) {
    Object[] array = new Object[] {obj};
    long baseOffset = getUnsafe().arrayBaseOffset(Object[].class);
    return normalize(getUnsafe().getInt(array, baseOffset));
}

static Object fromAddress(long address) {
    Object[] array = new Object[] {null};
    long baseOffset = getUnsafe().arrayBaseOffset(Object[].class);
    getUnsafe().putLong(array, baseOffset, address);
    return array[0];
}
```

该拷贝函数可以用来拷贝任何类型的对象，因为对象的大小是动态计算的。 

**注意** 在完成拷贝动作后你需要将拷贝对象的类型强转为目标类型。


#### 隐藏密码

在Unsafe的直接内存访问方法使用case中有一个非常有趣的用法就是删除内存中不想要的对象。

大多数获取用户密码的API方法的返回值不是byte[]就是char[]，这是为什么呢？

这完全是出于安全原因, 因为我们可以在不需要它们的时候将数组元素置为失效。如果我们获取的密码是字符串类型，则密码字符串是作为一个对象保存在内存中的。要将该密码字符串置为无效，我们只能讲字符串引用职位null，但是该字符串的内容任然存在内存直到GC回收该对象后。

这个技巧在内存创建一个假的大小相同字符串对象来替换原来的:

```java
String password = new String("l00k@myHor$e");
String fake = new String(password.replaceAll(".", "?"));
System.out.println(password); // l00k@myHor$e
System.out.println(fake); // ????????????

getUnsafe().copyMemory(
          fake, 0L, null, toAddress(password), sizeOf(password));

System.out.println(password); // ????????????
System.out.println(fake); // ????????????
```

感觉安全了吗？

其实该方法不是真的安全。想要真的安全我们可以通过反射API将字符串对象中的字符数组`value`字段的值修改为null。

```java
Field stringValue = String.class.getDeclaredField("value");
stringValue.setAccessible(true);
char[] mem = (char[]) stringValue.get(password);
for (int i=0; i < mem.length; i++) {
  mem[i] = '?';
}
```

### 多重继承

在Java中本来是没有多重集成的。除非我们可以将任意的类型转为我们想要的任意类型。

```java
long intClassAddress = normalize(getUnsafe().getInt(new Integer(0), 4L));
long strClassAddress = normalize(getUnsafe().getInt("", 4L));
getUnsafe().putAddress(intClassAddress + 36, strClassAddress);
```

这段代码将String类添加到Integer的超类集合中,所以我们的强转代码是没有运行时异常的。

    (String) (Object) (new Integer(666))
    
有个问题是我们需要先将要转的对象转为Object，然后再转为我们想要的类型。这是为了欺骗编译器。

### 动态类

We can create classes in runtime, for example from compiled .class file. To perform that read class contents to byte array and pass it properly to defineClass method.

我们可以在运行时创建类, 例如通过一个编译好的class文件。将class文件的内容读入到字节数组中然后将该数组传递到合适的`defineClass`方法中。

```java
byte[] classContents = getClassContent();
Class c = getUnsafe().defineClass(
              null, classContents, 0, classContents.length);
    c.getMethod("a").invoke(c.newInstance(), null); // 1
```

读取class文件内如的代码：

```java
private static byte[] getClassContent() throws Exception {
    File f = new File("/home/mishadoff/tmp/A.class");
    FileInputStream input = new FileInputStream(f);
    byte[] content = new byte[(int)f.length()];
    input.read(content);
    input.close();
    return content;
}
```

该方式是非常有用的，如果你确实需要在运行时动态的创建类。比如生产代理类或切面类。


### 抛出一个异常

不喜欢受检异常？这不是问题。

    getUnsafe().throwException(new IOException());
    
该方法抛出一个受检异常，但是你的代码不需要强制捕获该异常就像运行时异常一样。

### 快速序列化

这种使用方式更实用。

每个人都知道java标准的序列化的功能速度很慢而且它还需要类拥有公有的构造函数。

外部序列化是更好的方式，但是需要定义针对待序列化类的schema。

非常流行的高性能序列化库，像[kryo](http://code.google.com/p/kryo/)是有使用限制的，比如在内存缺乏的环境就不合适。

但通过使用Unsafe类我们可以非常简单的实现完整的序列化功能。

**序列化**：

* 通过反射定义类的序列化。 这个可以只做一次。
* 通过Unsafe的`getLong`, `getInt`, `getObject`等方法获取字段真实的值。
* 添加可以恢复该对象的标识符。
* 将这些数据写入到输出

当然也可以使用压缩来节省空间。

**反序列化**:

* 创建一个序列化类的实例，可以通过方法`allocateInstance`。因为该方法不需要任何构造方法。
* 创建schama, 和序列化类似
* 从文件或输入读取或有的字段
* 使用 `Unsafe` 的 `putLong`, `putInt`, `putObject`等方法来填充对象。

Actually, there are much more details in correct inplementation, but intuition is clear.

事实上要正确实现序列化和反序列化需要注意很多细节，但是思路是清晰的。

这种序列化方式是非常快的。

顺便说一句，在 `kryo` 有许多使用`Unsafe`的尝试 [http://code.google.com/p/kryo/issues/detail?id=75](http://code.google.com/p/kryo/issues/detail?id=75)


### 大数组

如你所知Java数组长度的最大值是`Integer.MAX_VALUE`。使用直接内存分配我们可以创建非常大的数组，该数组的大小只受限于堆的大小。

这里有一个`SuperArray`的实现:

```java
class SuperArray {
    private final static int BYTE = 1;

    private long size;
    private long address;

    public SuperArray(long size) {
        this.size = size;
        address = getUnsafe().allocateMemory(size * BYTE);
    }

    public void set(long i, byte value) {
        getUnsafe().putByte(address + i * BYTE, value);
    }

    public int get(long idx) {
        return getUnsafe().getByte(address + idx * BYTE);
    }

    public long size() {
        return size;
    }
}
```

一个简单的用法：

```java
long SUPER_SIZE = (long)Integer.MAX_VALUE * 2;
SuperArray array = new SuperArray(SUPER_SIZE);
System.out.println("Array size:" + array.size()); // 4294967294
for (int i = 0; i < 100; i++) {
    array.set((long)Integer.MAX_VALUE + i, (byte)3);
    sum += array.get((long)Integer.MAX_VALUE + i);
}
System.out.println("Sum of 100 elements:" + sum);  // 300
```

事实上该技术使用了非堆内存`off-heap memory`，在 `java.nio` 包中也有使用。

通过这种方式分配的内存不在堆上，并且不受GC管理。因此需要小心使用`Unsafe.freeMemory()`。该方法不会做任何边界检查，因此任何不合法的访问可能就会导致JVM奔溃。

这种使用方式对于数学计算是非常有用的，因为代码可以操作非常大的数据数组。 同样的编写实时程序的程序员对此也非常感兴趣，因为不受GC限制，就不会因为GC导致非常大的停顿。

### 并发

关于并发编程使用Unsafe的只言片语。`compareAndSwap` 方法是原子的，可以用来实现高性能的无锁化数据结构。

举个例子，多个线程并发的更新共享的对象这种场景：

首先我们定义一个简单的接口 `Counter`:

```java
interface Counter {
    void increment();
    long getCounter();
}
```

我们定义工作线程 `CounterClient`, 它会使用 `Counter`:

```java
class CounterClient implements Runnable {
    private Counter c;
    private int num;

    public CounterClient(Counter c, int num) {
        this.c = c;
        this.num = num;
    }

    @Override
    public void run() {
        for (int i = 0; i < num; i++) {
            c.increment();
        }
    }
}
```

这是测试代码：

```java
int NUM_OF_THREADS = 1000;
int NUM_OF_INCREMENTS = 100000;
ExecutorService service = Executors.newFixedThreadPool(NUM_OF_THREADS);
Counter counter = ... // creating instance of specific counter
long before = System.currentTimeMillis();
for (int i = 0; i < NUM_OF_THREADS; i++) {
    service.submit(new CounterClient(counter, NUM_OF_INCREMENTS));
}
service.shutdown();
service.awaitTermination(1, TimeUnit.MINUTES);
long after = System.currentTimeMillis();
System.out.println("Counter result: " + c.getCounter());
System.out.println("Time passed in ms:" + (after - before));
```

第一个实现-没有同步的计数器

```java
class StupidCounter implements Counter {
    private long counter = 0;

    @Override
    public void increment() {
        counter++;
    }

    @Override
    public long getCounter() {
        return counter;
    }
}
```

Output:

    Counter result: 99542945
    Time passed in ms: 679
    
速度很多，但是没有对所有的线程进行协调所以结果是错误的。第二个版本，使用Java常见的同步方式来实现

```java
class SyncCounter implements Counter {
    private long counter = 0;

    @Override
    public synchronized void increment() {
        counter++;
    }

    @Override
    public long getCounter() {
        return counter;
    }
}
```

Output:

    Counter result: 100000000
    Time passed in ms: 10136

彻底的同步当然会导致正确的结果。但是花费的时间令人沮丧。让我们试试 `ReentrantReadWriteLock`:    

```java
class LockCounter implements Counter {
    private long counter = 0;
    private WriteLock lock = new ReentrantReadWriteLock().writeLock();

    @Override
    public void increment() {
        lock.lock();
        counter++;
        lock.unlock();
    }

    @Override
    public long getCounter() {
        return counter;
    }
}
```

Output:

    Counter result: 100000000
    Time passed in ms: 8065
    
结果依然是正确的，时间也短。那使用原子的类呢？

```java
class AtomicCounter implements Counter {
    AtomicLong counter = new AtomicLong(0);

    @Override
    public void increment() {
        counter.incrementAndGet();
    }

    @Override
    public long getCounter() {
        return counter.get();
    }
}
```

Output:

    Counter result: 100000000
    Time passed in ms: 6552
    
使用`AtomicCounter`的效果更好一点。最后我们试试`Unsafe`的原子方法`compareAndSwapLong`看看是不是更进一步。

```java
class CASCounter implements Counter {
    private volatile long counter = 0;
    private Unsafe unsafe;
    private long offset;

    public CASCounter() throws Exception {
        unsafe = getUnsafe();
        offset = unsafe.objectFieldOffset(CASCounter.class.getDeclaredField("counter"));
    }

    @Override
    public void increment() {
        long before = counter;
        while (!unsafe.compareAndSwapLong(this, offset, before, before + 1)) {
            before = counter;
        }
    }

    @Override
    public long getCounter() {
        return counter;
    }
}    
```

Output:

    Counter result: 100000000
    Time passed in ms: 6454
    
开起来和使用原子类是一样的效果，难道原子类使用了`Unsafe`？答案是YES。

事实上该例子非常简单但表现出了`Unsafe`的强大功能。

就像前面提到的 `CAS`原语可以用来实现高效的无锁数据结构。实现的原理很简单：

* 拥有一个状态
* 创建一个它的副本
* 修改该副本
* 执行 CAS 操作
* 如果失败就重复执行

事实上，在真实的环境它的实现难度超过你的想象，这其中有需要类似ABA，指令重排序这样的问题。

如果你确实对此感兴趣，你可以参考关于无锁HashMap的精彩演示。

### Bonus

Documentation for park method from Unsafe class contains longest English sentence I've ever seen:

> Block current thread, returning when a balancing unpark occurs, or a balancing unpark has already occurred, or the thread is interrupted, or, if not absolute and time is not zero, the given time nanoseconds have elapsed, or if absolute, the given deadline in milliseconds since Epoch has passed, or spuriously (i.e., returning for no "reason"). Note: This operation is in the Unsafe class only because unpark is, so it would be strange to place it elsewhere.

### 结论

尽管Unsafe有这么多有用的应用，但是尽力不要使用。当然了使用JDK中利用了Unsafe实现的类是可以的。或者你对你代码功力非常自信。


### 参考

[https://dzone.com/articles/understanding-sunmiscunsafe](https://dzone.com/articles/understanding-sunmiscunsafe)


















    

