---
layout: post
title: logstash-input-file
comments: true
date: 2015-07-13
categories:
- ELK
---

### logstash-input-file

> Logstash 使用一个名叫 FileWatch 的 Ruby Gem 库来监听文件变化。这个库支持 glob 展开文件路径，而且会记录一个叫 .sincedb 的数据库文件来跟踪被监听的日志文件的当前读取位置。所以，不要担心 logstash 会漏过你的数据。sincedb 文件中记录了每个被监听的文件的 inode, major number, minor number 和 pos。

<!-- more -->

### 配置示例

	input
    	file {
        	path => ["/var/log/*.log", "/var/log/message"]
        	type => "system"
	        start_position => "beginning"
    	}
	}

### 解释

有一些比较有用的配置项，可以用来指定 FileWatch 库的行为：

1. ** path ** 

	path 属性是必填配置，类型是一个数组（该配置项没有默认值）。路径必须是绝对路径不能是相对路径。
	该配置项支持globs, 例如：/var/log/*.log

2. ** discover_interval ** 

	logstash 每隔多久去检查一次被监听的 path 下是否有新文件。默认值是 15 秒。

3. ** exclude **

	不想被监听的文件可以排除出去，这里跟 path 一样支持 glob 展开。

4. ** sincedb_path **

	指定 sincedb 数据库的写入位置 (该数据库保存了监控文件的当前位置).默认为路径是 $HOME/.sincedb*
	(Windows 平台上在 C:\Windows\System32\config\systemprofile\.sincedb)，可以通过这个配置定	义 sincedb 文件到其他位置。

5. ** sincedb_write_interval **

	logstash 写 sincedb 文件的频率，默认是 15 秒。

6. ** stat_interval **

	logstash 每隔多久检查一次被监听文件状态（是否有更新），默认是 1 秒。

7. ** start_position **
	
	logstash 从什么位置开始读取文件数据（只能是beginning 或 end），默认是结束位置(end)
	
8. ** tags **

	数组类型，可以给 event 添加任意你需要的tag, 在后续的处理流程中使用。

9. ** type **

	string 类型， 给所有该input处理的event添加一个type字段； 主要在 filter 阶段使用。type 字段的值作为 event 的一部分，可以在Kibana 中用该字段的值进行搜索。需要注意的是：第一次给event添加了该字段后，在event的生命周期内不可更改。

10. codec 

	类型是 *** codec *** , 默认值 “plain”; codec 用来处理输入的数据， 在数据输入阶段配置codec后，就不需要在Logstash的pipeline 中配置一个filter。

11. ** delimiter **

	数据输入分割符，string 类型， 默认值 '\n'		



	

	