---
layout: post
comments: true
title: java魔法之0xCAFEBABE
date: 2016-12-31 23:47:55
tags:
categories:
- java
---

本文翻译自[http://mishadoff.com/blog/java-magic-part-2-0xcafebabe/](http://mishadoff.com/blog/java-magic-part-2-0xcafebabe/)

你知道所有的Java class文件都是以相同的4字节内容开头吗？十六进制的格式为：`CAFEBABE`.

为了确定这个问题，创建一个名为`Hello.java`的文件：

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hell, O'World!");
    }
}
```
<!-- more -->

使用`javac`编译`Hello.java`然后用十六进制编辑器打开编译后的class文件

{% asset_img VztJknQ.png %}

James Gosling [解释](http://radio-weblogs.com/0100490/2003/01/28.html)了该问题：

> We used to go to lunch at a place called St Michael's Alley. According to local legend, in the deep dark past, the Grateful Dead used to perform there before they made it big. It was a pretty funky place that was definitely a Grateful Dead Kinda Place. When Jerry died, they even put up a little Buddhist-esque shrine. When we used to go there, we referred to the place as Cafe Dead. Somewhere along the line it was noticed that this was a HEX number. I was re-vamping some file format code and needed a couple of magic numbers: one for the persistent object file, and one for classes. I used CAFEDEAD for the object file format, and in grepping for 4 character hex words that fit after "CAFE" (it seemed to be a good theme) I hit on BABE and decided to use it. At that time, it didn't seem terribly important or destined to go anywhere but the trash-can of history. So CAFEBABE became the class file format, and CAFEDEAD was the persistent object format. But the persistent object facility went away, and along with it went the use of CAFEDEAD - it was eventually replaced by RMI.

`0xCAFEBABE` 的十进制值是 `3405691582`。 如果我们将每一位的数字相加得到结果是43。一个超过42（ 对生命,宇宙和一切的终极答案）。顺便说一下,43岁是一个质数。你看,魔术无处不在。即使在最后一句话。

### 后记

关于为什么42是对生命，宇宙，一切事情的终极答案，可以参考这个回答[https://www.quora.com/Why-and-how-is-42-the-answer-to-life-the-universe-and-everything](https://www.quora.com/Why-and-how-is-42-the-answer-to-life-the-universe-and-everything)

更多关于class文件格式的内容可以google。我是从一本名叫[深入理解JAVA虚拟机](https://item.jd.com/1069428318.html)的书获知的。


