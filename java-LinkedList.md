---
layout: post
comments: true
title: java-LinkedList实现解析
date: 2016-10-27 16:05:39
tags:
- java
categories:
- java
---

### 前言
> LinkedList 和 ArrayList 一样，都实现了 List 接口. LinkedList 是基于双向链表实现的. 所以它的插入和删除操作比 ArrayList 更加高效。但也是由于其为基于链表的，所以随机访问的效率要比 ArrayList 差.

### 类定义和类图

### 类定义

```java
public class LinkedList<E>extends AbstractSequentialList<E> implements List<E>, Deque<E>, Cloneable, java.io.Serializable{
    ////
}
```
<!-- more -->

#### 类图

{% asset_img java-java-LinkedList.png %}

LinkedList 继承自 AbstractSequenceList，实现了 List、Deque、Cloneable、java.io.Serializable 接口。AbstractSequenceList 提供了List接口骨干性的实现以减少实现 List 接口的复杂度，Deque 接口定义了双端队列的操作。

在 LinkedList 中除了本身自己的方法外，还提供了一些可以使其作为栈、队列或者双端队列的方法。这些方法可能彼此之间只是名字不同，以使得这些名字在特定的环境中显得更加合适。

### LinkedList 源码解读

#### 数据结构

LinkedList 是基于双向链表结构实现，所以在类中包含了 first 和 last 两个指针(Node)。Node节点中包含了上一个节点和下一个节点的引用。每个 Node 只能知道自己的前一个节点和后一个节点.

```java
/**
 * Pointer to first node.
 * Invariant: (first == null && last == null) ||
 *            (first.prev == null && first.item != null)
 */
transient Node<E> first;

/**
 * Pointer to last node.
 * Invariant: (first == null && last == null) ||
 *            (last.next == null && last.item != null)
 */
transient Node<E> last;

/**
 * Constructs an empty list.
 */
public LinkedList() {
}
//定义链表节点的私有内部类
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

#### 添加元素

**add(E e)** 

LinkedList的`add`方法默认将元素添加到链表的末尾.

```java
public boolean add(E e) {
        linkLast(e);
        return true;
}
/**
 * Links e as last element.
  */
 void linkLast(E e) {
     final Node<E> l = last;
     //新建节点, 并将新节点的pre指向链表的尾节点
     final Node<E> newNode = new Node<>(l, e, null);
     //新节点变为尾节点
     last = newNode;
     if (l == null)
         first = newNode;
     else
        //连接新节点
         l.next = newNode;
     size++;
     modCount++;
 }
```

**add(int index, E element)** 

该方法将新元素添加到链表指定位置, 该方法效率比较低. 因为要从头遍历链表才能找到位置index.

```java
/**
 * Inserts the specified element at the specified position in this list.
 * Shifts the element currently at that position (if any) and any
 * subsequent elements to the right (adds one to their indices).
 *
 * @param index index at which the specified element is to be inserted
 * @param element element to be inserted
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}
/**
 * Returns the (non-null) Node at the specified element index.
 */
Node<E> node(int index) {
    // assert isElementIndex(index);
    //从头节点遍历
    if (index < (size >> 1)) { //这里有位移优化
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        //从尾节点遍历
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
/**
 * Inserts element e before non-null Node succ.
 */
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

**addFirst(E e)**

#### 添加元素到队头

```java
/**
 * Links e as first element.
 */
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}
```    

#### 添加元素到队尾

**addLast(E e)**

```java
/**
 * Appends the specified element to the end of this list.
 *
 * <p>This method is equivalent to {@link #add}.
 *
 * @param e the element to add
 */
public void addLast(E e) {
    linkLast(e);
}
 /**
 * Links e as last element.
 */
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

#### 删除队首元素

**removeFirst**

```java
/**
 * Removes and returns the first element from this list.
 *
 * @return the first element from this list
 * @throws NoSuchElementException if this list is empty
 */
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}
/**
 * Unlinks non-null first node f.
 */
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
```

#### 删除队尾元素 

**removeLast**

```java
/**
 * Removes and returns the last element from this list.
 *
 * @return the last element from this list
 * @throws NoSuchElementException if this list is empty
 */
public E removeLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return unlinkLast(l);
}
/**
 * Unlinks non-null last node l.
 */
private E unlinkLast(Node<E> l) {
    // assert l == last && l != null;
    final E element = l.item;
    final Node<E> prev = l.prev;
    l.item = null;
    l.prev = null; // help GC
    last = prev;
    if (prev == null)
        first = null;
    else
        prev.next = null;
    size--;
    modCount++;
    return element;
}
```

#### 删除指定元素
 
```java
public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
/**
 * Unlinks non-null node x.
 */
E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}
```
 
### 注意事项

1. remove(Object o) 方法不是O(1)的, 而是O(N)的时间复杂度, 同样的remove(int index)也是.
2. add(int index, E element)方法不是O(1)的, 而是O(N)的时间复杂度
3. 查找get(int index) 也是O(N)的时间复杂度
    
### 总结

LinkedList里面的方法很多,但是核心方法就这么多,没有什么难以理解的地方,感觉就是教科书式的实现.在使用的时候注意时间复杂度就可以了.

