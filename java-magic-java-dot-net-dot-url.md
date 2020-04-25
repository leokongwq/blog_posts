---
layout: post
comments: true
title: java魔法之java.net.Url
date: 2016-12-31 23:05:18
tags:
categories:
- java
---

本文翻译自[http://mishadoff.com/blog/java-magic-part-1-java-dot-net-dot-url/](http://mishadoff.com/blog/java-magic-part-1-java-dot-net-dot-url/)


最近, 我在reddit上发现了一个非常有趣的Java代码片段

```java
HashSet set = new HashSet();
set.add(new URL("http://google.com"));
set.contains(new URL("http://google.com"));
Thread.sleep(60000);
set.contains(new URL("http://google.com"));
```
<!-- more -->

你猜一下第三代码和第五行代码的结果会是什么？

很明显不是`true, true`。如果被问到该问题，最后思考两分钟。

好了，在大多数时候结果是`true, false` 这是因为你连接了互联网，否则你怎么能看到这篇文章呢？关闭你的网络连接你将会得到`true, true`。

问题的原因在于该类方法`hashCode()` 和 `equals()`的实现逻辑。

让我们看下它是如何计算hashCode的：

```java
public synchronized int hashCode() {
  if (hashCode != -1)
    return hashCode;
  hashCode = handler.hashCode(this);
  return hashCode;
}
```

我们可以看到`hashCode`是一个实例变量并且只计算一次。这是由意义的，因为URL是不可变的。`handler`是什么？它是`URLStreamHandler`子类的实例，具体依赖于不同的协议类型（file, http, ftp）。看下java.net.URL的Java文档说明：

> 针对URL的比较操作来说，URL的hashCode的计算依赖所有的URL组件。因此,这个操作是一个阻塞操作。

停! 阻塞式操作？！

- Sorry, I couldn't check email yesterday due to hashCode calculation.

or even better

- No, mom, I can't watch porn, It's hashCode, you know.

Ok, let it be blocking. Another exciting part, that handler resolves host IP address for hashCode calculation. Tries to resolve, to be honest. If it can not do this, it calculates hashCode based on host, which is google.com for our example. Shit happens when IP is dynamic, or host have request balancer that also changes host IP. In that case we got different hashCodes for one host name, and will have two (or even more) instances in HashSet. Not good at all. By the way, hashCode and equals performance is terrible because of URLStreamHandler opens URLConnection. But it's another topic.

### 如何避免

- Use java.net.URI instead of java.net.URL. It's not the best choice though, but have deterministic hashCode implementation.
- Do not use java.net.URL in collections. Good option to maintain collection of String objects (that represent host name) and get URL when needed.
- Disable your network adapter during hashCode calculation. (It's a joke, but it helps)
- Use your own subclass of URLStreamHandler with proper implementation of hashCode.

Finally, I'm pretty sure java.net.URL class has lot of useful applications. But not that way.

### 总结

在使用该类时小心点，不要使用到该类的hashCode方法和equals方法，也就是说不要在集合类和Map类中使用。这种场景也是比较的少啊！



