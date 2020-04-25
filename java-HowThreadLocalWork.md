---
layout: post
title: ThreadLocal正解
categories:
- java
- jdk
tags:
- java
---

### 写在前面的话  
   		
1. 首先，ThreadLocal 不是用来解决共享对象的多线程访问问题的！一般情况下，通过ThreadLocal.set() 到线程中的对象是该线程自己使用的对象，其他线程是不需要访问的，也访问不到的。各个线程中访问的是不同的对象。 

2. 另外，说ThreadLocal使得各线程能够保持各自独立的一个对象，并不是通过ThreadLocal.set()来实现的，而是通过每个线程中的new 对象 的操作来创建的对象，每个线程创建一个，不是什么对象的拷贝或副本。通过ThreadLocal.set()将这个新创建的对象的引用保存到各线程的自己的一个map中，每个线程都有这样一个map，执行ThreadLocal.get()时，各线程从自己的map中取出放进去的对象，因此取出来的是各自自己线程中的对象，ThreadLocal实例是作为map的key来使用的。 

3. 如果ThreadLocal.set()进去的东西本来就是多个线程共享的同一个对象，那么多个线程的ThreadLocal.get()取得的还是这个共享对象本身，还是有并发访问问题。 

4. ThreadLocal的本意是用来解决当前线程的执行栈中参数传递问题的, 如果一个参数在当前线程调用栈的每个调用中都需要, 不需要将该参数层层传递到每个方法中.

<!-- more -->

### ThreadLocal 关键源码分析
<pre>
 //该方法首先获取当前线程对象，然后获取当前线程对象的threadLocals属性，threadLocals保存的是以该TheadLocal对象为key,当前线程独自持有的对象value的一个条目
 public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
 }
 private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
}
//看下面的代码我们发现通过ThreadLocal.set的值，其实最终是保存在当前线程的一个Map属性中，TheadLocal实例本身并不保存线程持有的值
public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
}
public void remove() {
	ThreadLocalMap m = getMap(Thread.currentThread());
    	if (m != null)
        	m.remove(this);
}

/**
* 该方法返回的是当前的线程对象的属性
*/
ThreadLocalMap getMap(Thread t) {
	return t.threadLocals;
}
//给当前线程的treadLocals属性赋值
void createMap(Thread t, T firstValue) {
	t.threadLocals = new ThreadLocalMap(this, firstValue);
}
</pre>

### Thead类关键代码

```java
Thread 类关键代码：
 /*ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. 
 */
 ThreadLocal.ThreadLocalMap threadLocals = null;
```

### hibernate中典型的ThreadLocal的应用

```java
private static final ThreadLocal threadSession = new ThreadLocal();  
  
public static Session getSession() throws InfrastructureException {  
    Session s = (Session) threadSession.get();  
    try {  
        if (s == null) {  
            s = getSessionFactory().openSession();  
            threadSession.set(s);  
        }  
    } catch (HibernateException ex) {  
        throw new InfrastructureException(ex);  
    }  
    return s;  
}  
```

可以看到在getSession()方法中，首先判断当前线程中有没有放进去session，如果还没有，那么通过sessionFactory().openSession()来创建一个session，再将session set到线程中，实际是放到当前线程的ThreadLocalMap这个map中，这时，对于这个session的唯一引用就是当前线程中的那个ThreadLocalMap（下面会讲到），而threadSession作为这个值的key;要取得这个session可以通过threadSession.get()来得到，里面执行的操作实际是先取得当前线程中的ThreadLocalMap，然后将threadSession作为key将对应的值取出。这个session相当于线程的私有变量，而不是public的。 
显然，其他线程中是取不到这个session的，他们也只能取到自己的ThreadLocalMap中的东西。要是session是多个线程共享使用的，那还不乱套了。 
试想如果不用ThreadLocal怎么来实现呢？可能就要在action中创建session，然后把session一个个传到service和dao中，这可够麻烦的。或者可以自己定义一个静态的map，将当前thread作为key，创建的session作为值，put到map中，应该也行，这也是一般人的想法，但事实上，ThreadLocal的实现刚好相反，它是在每个线程中有一个map，而将ThreadLocal实例作为key，这样每个map中的项数很少，而且当线程销毁时相应的东西也一起销毁了，不知道除了这些还有什么其他的好处。 

#### 总结

ThreadLocal不是用来解决对象共享访问问题的，而主要是提供了保持对象的方法和避免参数传递的方便的对象访问方式。归纳了两点： 

1. 每个线程中都有一个自己的ThreadLocalMap类对象，可以将线程自己的对象保持到其中，各管各的，线程可以正确的访问到自己的对象。 

2. 将一个共用的ThreadLocal静态实例作为key，将不同对象的引用保存到不同线程的ThreadLocalMap中，然后在线程执行的各处通过这个静态ThreadLocal实例的get()方法取得自己线程保存的那个对象，避免了将这个对象作为参数传递的麻烦。 

当然如果要把本来线程共享的对象通过ThreadLocal.set()放到线程中也可以，可以实现避免参数传递的访问方式，但是要注意get()到的是那同一个共享对象，并发访问问题要靠其他手段来解决。但一般来说线程共享的对象通过设置为某类的静态变量就可以实现方便的访问了，似乎没必要放到线程中。 

ThreadLocal的应用场合，我觉得最适合的是按多线程多实例（每个线程对应一个实例）的对象的访问，并且这个对象很多地方都要用到。 

### 一件小事
有次有人问我这样一个问题：当多个线程往同一个ThreadLocal对象中set东西，且从不调用ThreadLocal的remove方法，这样会不会造成内存泄露，我当时回答肯定不会！我当时想JDK的编写者不会设计这样的类，直到我查看源代码和看相关的文章才知道原理，通过ThreadLocal set进去的值，其实是保存在每个Thread中的，这样当每个Thread销毁后，内存自动就释放了。