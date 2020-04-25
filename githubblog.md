---
layout: post
title: Jekyll 快速入门
date: 2015-04-16
comments: true
categories:
- jekyll
tags:
- github
- jekyll
---

## Jekyll 快速入门

### 1. 3分钟-github搭建个人主页

1. 创建一个新的仓库

	访问 [https://github.com](https://github.com "github.com") 并创建一个名为：	**USERNAME.github.com** 的仓库

2. 安装Jekyll-Bootstrap

	选择你想存放博客文件的文件夹，在终端中输入如下命令：
	
		git clone https://github.com/plusjade/jekyll-bootstrap.git USERNAME.github.com
		
		cd USERNAME.github.com git
		
		git remote set-url origin 	git@github.com:USERNAME/USERNAME.github.com.git 
		
		git push origin master

3. 益处
	
	在github花费几分钟对你的blog进行魔法处理后，你就可以在公网通过地址：http://USERNAME.github.com
	访问你的blogle，很神奇对吧！

	***Already have your blog on GitHub?***
	
	我假设你已经在你的机器上安装了Jekyll。在本地启动 Jekyll-Bootstrap ：
	
    	$ git clone https://github.com/plusjade/jekyll-bootstrap.git
    	$ cd jekyll-bootstrap
    	$ jekyll serve

	访问地址：http://localhost:4000.就可以看到结果

### 2. 本机运行Jekyll
	
	
为了在本机预览你的blog,你需要安装 Jekyll ruby gem。 需要注意 gem 的依赖会同时安装。

	$ gem install jekyll

如果在启动的过程中出现问题，可以参考 Jekyll 安装文档. 你也可以Jekyll的github主页提问题。

    $ cd USERNAME.github.com 
    $ jekyll serve

通过地址：http://localhost:4000/ 访问你的blog。

### 3. 创建一个发布

通过rake task创建发布非常容易：

    $ rake post title="Hello World"

rake task 使用合适的格式化文件名称和YAML 文件头格式创建新文件. 确保要使用你自定义的title. 默认文件的名的时间是当前日期。

### 4.创建页面

通过rake task创建页面非常容易:

    $ rake page name="about.md"

Create a nested page:

    $ rake page name="pages/about.md"

Create a page with a "pretty" path:

    $ rake page name="pages/about"
    # this will create the file: ./pages/about/index.html

The rake task automatically creates a page file with properly formatted filename and YAML Front Matter as well as includes the Jekyll Bootstrap "setup" file.

### 5.发布
	
当你完成要发布博文的编写和相关文件的修改，使用下面的命令提交到远程github仓库：

	$ git add .
    $ git commit -m "Add new content"
    $ git push origin master

远程提交完成后，github会自动帮你构建站点文件.

#### 6.定制化

Jekyll-Bootstrap 可以作为一个blog的基础平台。我们有多种方式对其按自己的意图进行深度定制。 下面几种技术可以用对其进行深度定制:

#####  主题

Jekyll-Bootstrap 支持模块化主题。 主题之间可以共存并根据需要启用或停用。 

##### blog 配置

Jekyll and Jekyll-Bootstrap 拥有一个简单且强大的 Jekyll 配置系统。 你可以:

    Specify a custom permalink format for blog posts.
    Specify a commenting engine like disqus, intensedebate, livefyre, or custom.
    Specify an analytics engine like google, getclicky, or custom.

##### 编程接口

API文档页面记录了在Jekyll 和 Jekyll-Bootstrap中可以使用的主要数据结构和变量。通过阅读这些文档你可以知道如何使用，在哪里使用Jekyll提供的这些数据和代码

##### Jekyll 介绍

**高度推荐**

如果你计划定制自己的blog，我高度推荐你阅读Jekyll 介绍文档。 文档介绍了Jekyll的核心工作原理。