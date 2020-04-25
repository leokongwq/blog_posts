---
layout: post
title: Guava-Spilitor
date: 2015-04-22
comments: true
categories:
- guava
- java
tags:
- guava
---

### Guava常用工具类-Spilitor

Spilitor和Joiner的功能恰好相反，它主要用来以特定的分隔符来拆分字符串

1. 基本用法

```java
Iterator<String> iterator = Splitter.on(",").split("1,3,5").iterator();
while (iterator.hasNext()){
    String str = iterator.next();
    System.out.println(str);
}
```
<!-- more -->
    
2. trimResults

有时候我们会遇到这种格式的字符串："1 , 3, 5";针对这种格式的字符串如果我们用上面的方法来解析就会出现某些元素包含空格情况；针对这样个格式字符串，Spilitort提供了trimResults功能，它会将结果中的每个元素两边的空格去掉。
	
		Iterator<String> iterator = Splitter.on(",").trimResults().split("1,3,5").iterator();
    	while (iterator.hasNext()){
    		String str = iterator.next();
    	    System.out.println(str);
    	}
3.omitEmptyStrings

有时候我们会遇到这种格式的字符串："1,,3,,5";针对这种格式的字符串如果我们用上面的方法来解析就会出现某些元素为空的情况；针对这样个格式字符串，Spilitort提供了omitEmptyStrings功能，它会抛弃结果中空元素。
	
		Iterator<String> iterator = Splitter.on(",").omitEmptyStrings().split("1,,3,,5").iterator();
    	while (iterator.hasNext()){
    		String str = iterator.next();
    	    System.out.println(str);
    	}
    	
4. MapSplitter
	如果我们有这样一种需求：将http请求的查询字符串解析为一个Map对象，通过MapSplitter会非常的方便。
	
		String str = "name=tom&age=22";
		MapSplitter splitter = Splitter.on("&").withKeyValueSeparator("=");
		Map<String, String> map = splitter.split(str);
		System.out.println(map);
				
	