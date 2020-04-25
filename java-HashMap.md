---
layout: post
comments: true
title: java-HashMap知识点汇总
date: 2016-10-26 15:44:58
tags:
categories:
- java
---

> java 中的HashMap应该是比较常用的一个类，关于这个类有许许多多的知识点需要我们注意。这篇文章就将我们需要知道的知识点进行汇总。

### HashMap实现

#### HashMap类图

{% asset_img java-HashMap.jpeg %}

#### 实现原理

HashMap内部维护一个`entry`数组(数组中的每个元素称为桶),该数组的初始大小可以通过构造函数控制.当我们将一个键值对放入HashMap中时会
经历如下的步骤:

1.查找该键值对应该存放在entry数组中的下标值.具体算法如下:

```java 
final int hash(Object k) {
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }

    h ^= k.hashCode();

    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}

2.判断该键值对是否已经存在于该下标值对应的桶中, 如果存在,则会覆盖老的value值,如果不存在,则会将该键值对添加到entry链表的表头.

**注意** 如果key为null, 则会将null key对应的键值对放在entry数组中下标为0的桶链表中.

通过指定的key获取HashMap中该key对应的value时会经历如下步骤

1.计算该key对应的键值对所在entry数组中的下标值,
2.遍历该下标对应的桶链表, 取出和参数key equals 的entry对应的value值

**注意** 如果key为null, 则HashMap会遍历entry数组下标为0的桶链表,找出key为null的entry节点并返回对应的value值.


### HashMap的扩展时机

HashMap默认的entry数组大小为16,负载因子为0.75.  如果HashMap中entry的数量和桶数组长度的比值大于负载因子
则HashMap会new一个2倍大小的桶数组,并重新分布所有的entry元素.


### HashMap是否是有序的

根据HashMap的实现原理,我们知道HashMap不是有序的,也就是说我们放入键值对的顺序和我们遍历HashMap取出键值对的顺序是不一致的.这个不难理解.

### 有序的Map实现类有哪些

有序的Map实现类有LinkedHashMap, TreeMap.



### HashMap和WeakHashMap的不同点

1. HashMap实现了Cloneable和Serializable接口，而WeakHashMap没有。HashMap实现Cloneable，意味着它能通过clone()克隆自己。HashMap实现Serializable，意味着它支持序列化，能通过序列化去传输。
2. HashMap的“键”是“强引用(StrongReference)”，而WeakHashMap的键是“弱引用(WeakReference)”。

WeakReference的“弱键”能实现WeakReference对“键值对”的动态回收。当“弱键”不再被使用到时，GC会回收它，WeakReference也会将“弱键”对应的键值对删除。这个“弱键”实现的动态回收“键值对”的原理呢？其实，通过WeakReference(弱引用)和ReferenceQueue(引用队列)实现的。 首先，我们需要了解WeakHashMap中：

* 第一，“键”是WeakReference，即key是弱键。
* 第二，ReferenceQueue是一个引用队列，它是和WeakHashMap联合使用的。当弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。 WeakHashMap中的ReferenceQueue是queue。
* 第三，WeakHashMap是通过数组实现的，我们假设这个数组是table。

接下来，说说“动态回收”的步骤。
(01) 新建WeakHashMap，将“键值对”添加到WeakHashMap中。
将“键值对”添加到WeakHashMap中时，添加的键都是弱键。
实际上，WeakHashMap是通过数组table保存Entry(键值对)；每一个Entry实际上是一个单向链表，即Entry是键值对链表。
(02) 当某“弱键”不再被其它对象引用，并被GC回收时。在GC回收该“弱键”时，这个“弱键”也同时会被添加到queue队列中。
例如，当我们在将“弱键”key添加到WeakHashMap之后；后来将key设为null。这时，便没有外部外部对象再引用该了key。
接着，当Java虚拟机的GC回收内存时，会回收key的相关内存；同时，将key添加到queue队列中。
(03) 当下一次我们需要操作WeakHashMap时，会先同步table和queue。table中保存了全部的键值对，而queue中保存被GC回收的“弱键”；同步它们，就是删除table中被GC回收的“弱键”对应的键值对。
例如，当我们“读取WeakHashMap中的元素或获取WeakReference的大小时”，它会先同步table和queue，目的是“删除table中被GC回收的‘弱键’对应的键值对”。删除的方法就是逐个比较“table中元素的‘键’和queue中的‘键’”，若它们相当，则删除“table中的该键值对”。

### 正确遍历HashMap的姿势

```java
Map<String, String> map = new HashMap<String, String>();
map.put("apple", "苹果");
map.put("watermelon", "西瓜");
Iterator iter = map.entrySet().iterator();
while (iter.hasNext()) {
   ////
}
```

### HashMap中的元素按Key排序

对Map中的元素按key排序是一个经常使用的操作。有一种方法是将Map中的元素放进List集合，然后通过一个比较器对集合进行排序，如下代码示例：

```java
List list = new ArrayList(map.entrySet());
    Collections.sort(list, new Comparator() {
        public int compare(Entry e1, Entry e2) {
            return e1.getKey().compareTo(e2.getKey());
        }
    });
}
```

另一种方法是使用SortedMap，其提供了对Key进行排序的功能；前提是Map中的Key需要实现Comparable接口或者通过构造方法传入比较器；TreeMap是SortedMap的一个实现类，它的构造方法能够接收一个比较器，以下代码展示了如何将普通Map转成SortedMap

```java
SortedMap sortedMap = new TreeMap(new Comparator() {
    @Override
    public int compare(K k1, K k2) {
    return k1.compareTo(k2);
    }
});
sortedMap.putAll(map);
```

### 初始化一个不可修改的Map

有两种方法,更严格的说有一种方法.

方法一:

```java
private static final Map map;
    static {
        map = new HashMap();
        map.put(1, "one");
        map.put(2, "two");
    }
```

很明显上面的方法只能保证不能改变map的指向,但是map中的内容我们依然可以通过修改方法进行改变.

方法二:

```java
private static final Map map;
static {
    Map map = new HashMap();
    map.put(1, "one");
    map.put(2, "two");
    map = Collections.unmodifiableMap(aMap);
}
```

### 总结

关于Java中Map接口的各个实现类,我们都应该仔细了解,阅读源代码查看其具体的实现逻辑.这样才能在开发中做出合理的选择并确保使用姿势的正确性.

    
