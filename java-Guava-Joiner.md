---
layout: post
title: Guava-Joiner
date: 2015-04-22
comments: true
categories:
- guava
- java
tags:
- guava
---

### Guava常用工具类-Joiner

在日常工作中我们经常会有找个的需要，将一个数字或列表变为一个以固定分隔符分割的字符串。通常的做法是遍历该数组或List,通过使用StringBuilder将每个元素和分隔符连起来，如：

	List<String> names = new ArrayList<String>(Arrays.asList("tom", "lily"));
	String delimiter = ",";
    StringBuilder sb = new StringBuilder();
    for (String name : names){
    	sb.append(name).append(delimiter);
    }
    return sb.substring(0, sb.length() - 1).toString();
 	
由上面的代码我们可以看出，要实现这个简单的功能我们要写好多行代码，而且通常情况下我们还要对数组或List中的元素进行判空操作以防止NullPointException。而如果通过使用Guava给我们提供的工具类Joiner,我们可以非常容易的实现该功能。

<!-- more -->

1. 连接List或数组

		//names 可以是List,也可以是Object[]
		String delimiter = ",";
		// Joiner类一旦创建则不可变，满足不可变性，因此线程安全
		Joiner joiner = Joiner.on(delimiter);
    	// 忽略null
		String excludeNullString = joiner.skipNulls().join(names);
		// 将null替代为指定的字符串
		String replaceNullString = joiner.useForNull("empty").join(names);
		System.out.println("excludeNullString: " + excludeNullString);
		System.out.println("replaceNullString: " + replaceNullString); 
		// 不对null处理，默认会抛NullPointerException
		String defaultNullString = joiner.join(names);
		System.out.println("defaultNullString: " + defaultNullString);

2. 连接多个参数元素

		String delimiter = ",";
		// Joiner类一旦创建则不可变，满足不可变性，因此线程安全
		Joiner joiner = Joiner.on(delimiter).skipNulls();
		//利用jdk5提供的可变参数特性
		String res = joiner.join(null, "foo","bar");
		System.out.println(res); //foo,bar
		
3. 追加到实现了Appendable接口的类中：

		//append到StringBuilder
		StringBuilder stringBuilder = new StringBuilder();
		Joiner joiner = Joiner.on(",").skipNulls();
		joiner.appendTo(stringBuilder, "appendTo", "StringBuilder");
		System.out.println(stringBuilder); //appendTo,StringBuilder
         
		//append到输出流
		FileWriter writer = new FileWriter("append_text.txt");
		joiner.appendTo(writer, "appendTo", "FileWriter");
		writer.close();
		
4.链接Map对象
	在有时我们会有这样的需要，将一个Map对象变成http请求查询字符形式的字符串，通过Joiner我们非常容易实现这样的功能，如下代码：
	
		Map<String, String> map = new HashMap<>();
		map.put("name", "tom");
		map.put("age", "22");
		MapJoiner mapJoiner = Joiner.on("&").withKeyValueSeparator("=");
		String str = mapJoiner.join(map);
		System.out.println(str);//结果如:name=tom&age=22
		