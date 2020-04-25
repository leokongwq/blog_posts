---
layout: post
comments: true
title: maven-resource-plugin
date: 2017-06-22 10:03:00
tags:
- maven
categories:
- 开发工具
---

### 背景

有个项目在测试环境和线上环境需要包含不同的配置文件， 这就需要结合maven的profile和resource插件配置来解决了。

<!-- more -->

### 解决办法

```xml

<filters>
  <filter>src/main/filters/${profileActive}.properties</filter>
</filters>
<resources>
  <resource>
      <directory>${project.basedir}/src/main/resources</directory>
      <filtering>true</filtering>
      <excludes>
          <exclude>test/</exclude>
          <exclude>production/</exclude>
      </excludes>
  </resource>
</resources>

<plugin>
    <artifactId>maven-resources-plugin</artifactId>
    <version>3.0.2</version>
    <executions>
        <execution>
            <id>copy-resources</id>
            <phase>generate-resources</phase>
            <goals>
                <goal>copy-resources</goal>
            </goals>
            <configuration>
                <outputDirectory>${project.build.outputDirectory}/</outputDirectory>
                <resources>
                    <resource>
                        <directory>src/main/resources/${profileActive}</directory>
                        <filtering>true</filtering>
                    </resource>
                </resources>
            </configuration>
        </execution>
    </executions>
</plugin>
```


