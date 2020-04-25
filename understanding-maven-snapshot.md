---
layout: post
comments: true
title: 理解maven中SNAPSHOT版本的作用
date: 2017-08-24 15:43:19
tags:
- maven
categories:
- 项目管理
---

### 背景

一次针对现有的http服务开发了一个SNAPSHOT版本的调用SDK jar包。QA同学部署到测试环境后，我又更新了一下jar包的内容，此时QA同学再次部署时并没有拉去到最新的jar包，这个就比较奇怪了。记忆中maven不是每次都从私服去检查
SNAPSHOT类型的jar包是否有更新吗？怎么对我就不起作用了呢？原来也是一直这么使用的的，换个公司就不行了？最后通过阅读官方文档才发现自己的理解不到位。

### 为什么使用SNAPSHOT类型的依赖？

答案当然是不想每次有点代码改动都升级一下版本。

<!-- more -->

### maven如何处理SNAPSHOT类型的依赖？

第一次构建的时候会把该库从远程仓库中下载到本地仓库缓存中，然后根据pom文件的配置不定期检查该快照版本是否有变更。如果有变更则会重新拉去最新的jar。

#### 检查频率配置

```xml
<repository>
    <id>myRepository</id>
    <url>...</url>
    <snapshots>
        <enabled>true</enabled>
        <updatePolicy>更新策略</updatePolicy>
    </snapshots>
</repository>
```

更新策略有一下几种：

- always 每次构建都检查远程仓库中该依赖jar包是否有更新
- daily 每天检查一次 (默认策略)
- interval:XXX 指定检查时间间隔，单位是分钟。
- never 从不检查。该策略就和正式版本的处理规则一样了。

### 参考

[http://maven.apache.org/ref/3.5.0/maven-settings/settings.html](http://maven.apache.org/ref/3.5.0/maven-settings/settings.html)


