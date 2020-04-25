---
layout: post
comments: true
title: java 中的内存映射
date: 2019-09-12 22:57:50
tags:
- mmap
- RMQ
categories:
---

### 背景

最近在阅读公司内部扩展的RocketMQ中延迟消息的实现代码，其中使用到了一个组件`MappedFile`。`MappedFile`就代表了一个mmap的文件。其实在很早前就了解到RMQ实现用使用了mmap技术，但是一直没有深入了解，借此机会就将Java中mmap的内容进行一次总结。

### mmap

 `mmap`是一个系统调用。它的作用是将一个文件或者其它对象的一部分内容映射到进程的地址空间。这样进程就可以直接读写这一段内存，而系统会自动回写脏页面到对应的文件磁盘上，即完成了对文件的操作而不必再调用read,write等系统调用函数。相反，内核空间对这段区域的修改也直接反映用户空间。通过`mmap`也可以实现不同进程间的共享内存。
 
 <!-- more -->
 
### Java中如何使用mmap

示例代码：

```java
private static final int TEN_MB = 1024 * 1024 * 10;

public static void main(String[] args) throws Exception {
    RandomAccessFile  raf = new RandomAccessFile("/Users/jiexiu/a.txt", "rw");
    MappedByteBuffer mappedByteBuffer = raf.getChannel().map(FileChannel.MapMode.READ_WRITE, 0, TEN_MB);
    raf.close();
}
```

### `FileChannel.map`方法详解

```java
public abstract MappedByteBuffer map(MapMode mode,
                                         long position, long size)
        throws IOException;
```

1. 将FileChannle对于的文件的一部分直接映射到内存。（这里的内存是堆外内存）
2. 映射模式可以是`MapMode.READ_ONLY`(只读)，`MapMode.READ_WRITE`(读写)，`MapMode.PRIVATE`（私有，写时复制）
3. 如果是只读模式映射，那么文件通道必须是以只读模式打开。
4. 方法返回的`MappedByteBuffer`的`position`是0，`limit` 和 `capacity`的值是参数`size`的值。
5. 一旦映射完成，那么该`MappedByteBuffer`就和创建它的`FileChannel`无关。关闭`FileChannel`不会影响该`MappedByteBuffer`。
6. 该方法的底层实现依赖于具体操作系统中mmap系统调用的实现逻辑。不同的操作系统可能表现不同。例如：程序修改了`MappedByteBuffer`的内容，操作系统何时将变更写入到磁盘，这个是不确定的。
7. 对大多数操作系统来说该方法都是一个昂贵的操作，如果仅仅是映射很小范围，那么不建议使用。针对大文件推荐使用该操作。


### MappedByteBuffer 

`MappedByteBuffer`本质上是一块对外内存，也就是`DirectByteBuffer`。通过`FileChannel.map`来创建，直到被GC才结束它的生命。

它有两个非常重要的方法，分别是`load`和`force`

```java
/**
 * 加载该缓存的内容到物理内存中。这是因为mapp完成后，OS并没有直接读取文件的内容，当真正要访问的时候，通过缺页异常来进行读磁盘操作。
 */
public final MappedByteBuffer load() {
}
```

force 方法

```java
/**
 * 强制将修改后的的内容写入到存储设备上。
 * 需要注意的是：如果是本地设备，那么该方法返回时，确保自从该缓存区创建后或该方法最后一次调用后，变更的内容一定写入了设备，如果是网络文件则没有该保证。
 * 如果不是通过`MapMode.READ_WRITE`模式映射的，调用该方法没有任何影响。
 */
public final MappedByteBuffer force() {
}
```

关于force方法多说一点：即使我们不手动调动该方法写缓存区的更改写入底层设备，操作系统底层也会定时将变更的脏页刷到设备上，不过时间不确定。

MappedByteBuffer 在我们关闭FileChannel和文件后如果还没有被GC，那么对于的文件也是无法删除的，因为底层的文件句柄还没有释放。在RocketMQ中有专门针对该问题编写的代码。具体如下：

```java MappedFile.clean
public static void clean(final ByteBuffer buffer) {
    if (buffer == null || !buffer.isDirect() || buffer.capacity() == 0)
        return;
    invoke(invoke(viewed(buffer), "cleaner"), "clean");
}
```

作用是获取`DirectByteBuffer`中的`Cleaner`，然后调用它的`clean`方法来回收该`DirectByteBuffer`，也就是`MappedByteBuffer`。

`Cleaner`底层是通过`unsafe.freeMemory(address);`来释放内存的。


### 题外话 - 如何创建指定大小的文件

#### 方法一

```java
RandomAccessFile  raf = new RandomAccessFile("/Users/jiexiu/a.txt", "rw");
raf.setLength(1024 * 1024 * 10); // 10M
raf.close()
```

#### 方法二

```java
for (int i = 0; i < 10; i++) {
    byte[] buffer = new byte[1024 * 1024];
    fos.write(buffer);
}
```

#### 方法三

```shell
dd if=/dev/zero of=hello.txt bs=100M count=1
```

### 参考

[文件IO操作的一些最佳实践](https://www.cnkirito.moe/file-io-best-practise/)
[Java网络编程与NIO详解8：浅析mmap和Direct Buffer](https://how2playlife.com/2018/05/27/Javanet8/)
[认真分析mmap：是什么 为什么 怎么用](https://www.cnblogs.com/huxiao-tee/p/4660352.html)
[使用sun.misc.Cleaner或者PhantomReference实现堆外内存的自动释放](https://blog.csdn.net/aitangyong/article/details/39455229)
[Java网络编程与NIO详解8：浅析mmap和Direct Buffer](https://how2playlife.com/2018/05/27/Javanet8/)
 

