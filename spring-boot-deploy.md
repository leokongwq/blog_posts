---
layout: post
comments: true
title: SpringBoot应用部署模式
date: 2018-05-15 17:32:00
tags:
- spring
categories:
- web
---

### 背景

越来越多的人开始使用SpringBoot来实现项目的快速开发。每个团队都会面临一个同样的问题：如何部署SpringBoot应用？大部分人会想：这还需要考虑？当然是同步`Fat Jar`的方式部署喽。现实是残酷的！可能的原因有：

- 已有的发布系统不支持
- 团队成员习惯了war包的部署方式
- 外置的Servlet容器更容易配置
- 文件路径相关的代码调整
- 其它原因

<!-- more -->

SpringBoot 应用默认的打包结果是一个jar包。如果需要按照war包的形式进行部署，我们需要做如下的配置就能够实现。

### 第一步 扩展 SpringBootServletInitializer

`SpringBootServletInitializer`是一个实现`WebApplicationInitializer`接口的抽象类，它是servlet 3.0+环境的主要抽象，以便以编程方式配置ServletContext。 它将Servlet，Filter和ServletContextInitializer bean从应用程序上下文绑定到servlet容器。

#### Main 

```java
package com.websystique.springboot;
 
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.builder.SpringApplicationBuilder;
import org.springframework.boot.web.support.SpringBootServletInitializer;
 
@SpringBootApplication(scanBasePackages={"com.websystique.springboot"})// same as @Configuration @EnableAutoConfiguration @ComponentScan
public class SpringBootStandAloneWarApp extends SpringBootServletInitializer{
 
     
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(SpringBootStandAloneWarApp .class);
    }
     
    public static void main(String[] args) {
        SpringApplication.run(SpringBootStandAloneWarApp.class, args);
    }
}
```

### 第二步 修改 Maven打包类型为'war'

```xml
<project xmlns="<a class="vglnk" href="http://maven.apache.org/POM/4.0.0" rel="nofollow"><span>http</span><span>://</span><span>maven</span><span>.</span><span>apache</span><span>.</span><span>org</span><span>/</span><span>POM</span><span>/</span><span>4</span><span>.</span><span>0</span><span>.</span><span>0</span></a>" xmlns:xsi="<a class="vglnk" href="http://www.w3.org/2001/XMLSchema-instance" rel="nofollow"><span>http</span><span>://</span><span>www</span><span>.</span><span>w3</span><span>.</span><span>org</span><span>/</span><span>2001</span><span>/</span><span>XMLSchema</span><span>-</span><span>instance</span></a>"
    xsi:schemaLocation="<a class="vglnk" href="http://maven.apache.org/POM/4.0.0" rel="nofollow"><span>http</span><span>://</span><span>maven</span><span>.</span><span>apache</span><span>.</span><span>org</span><span>/</span><span>POM</span><span>/</span><span>4</span><span>.</span><span>0</span><span>.</span><span>0</span></a> <a class="vglnk" href="http://maven.apache.org/xsd/maven-4.0.0.xsd" rel="nofollow"><span>http</span><span>://</span><span>maven</span><span>.</span><span>apache</span><span>.</span><span>org</span><span>/</span><span>xsd</span><span>/</span><span>maven</span><span>-</span><span>4</span><span>.</span><span>0</span><span>.</span><span>0</span><span>.</span><span>xsd</span></a>">
    <modelVersion>4.0.0</modelVersion>
 
    <groupId>com.websystique.springboot</groupId>
    <artifactId>SpringBootStandAloneWarExample</artifactId>
    <version>1.0.0</version>
    <packaging>war</packaging>
 
    <name>${project.artifactId}</name>
 
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.4.3.RELEASE</version>
    </parent>
......
```

### 第三步 排除内嵌的Servlet容器

eg.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>
</dependency>
```

### 完整的pom文件

```xml
<project xmlns="<a class="vglnk" href="http://maven.apache.org/POM/4.0.0" rel="nofollow"><span>http</span><span>://</span><span>maven</span><span>.</span><span>apache</span><span>.</span><span>org</span><span>/</span><span>POM</span><span>/</span><span>4</span><span>.</span><span>0</span><span>.</span><span>0</span></a>" xmlns:xsi="<a class="vglnk" href="http://www.w3.org/2001/XMLSchema-instance" rel="nofollow"><span>http</span><span>://</span><span>www</span><span>.</span><span>w3</span><span>.</span><span>org</span><span>/</span><span>2001</span><span>/</span><span>XMLSchema</span><span>-</span><span>instance</span></a>"
    xsi:schemaLocation="<a class="vglnk" href="http://maven.apache.org/POM/4.0.0" rel="nofollow"><span>http</span><span>://</span><span>maven</span><span>.</span><span>apache</span><span>.</span><span>org</span><span>/</span><span>POM</span><span>/</span><span>4</span><span>.</span><span>0</span><span>.</span><span>0</span></a> <a class="vglnk" href="http://maven.apache.org/xsd/maven-4.0.0.xsd" rel="nofollow"><span>http</span><span>://</span><span>maven</span><span>.</span><span>apache</span><span>.</span><span>org</span><span>/</span><span>xsd</span><span>/</span><span>maven</span><span>-</span><span>4</span><span>.</span><span>0</span><span>.</span><span>0</span><span>.</span><span>xsd</span></a>">
    <modelVersion>4.0.0</modelVersion>
 
    <groupId>com.websystique.springboot</groupId>
    <artifactId>SpringBootStandAloneWarExample</artifactId>
    <version>1.0.0</version>
    <packaging>war</packaging>
 
    <name>${project.artifactId}</name>
 
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.4.3.RELEASE</version>
    </parent>
 
    <properties>
        <java.version>1.8</java.version>
        <startClass>SpringBootStandAloneWarApp</startClass>
    </properties>
 
    <dependencies>
        <!-- Add typical dependencies for a web application -->
        <!-- Adds Tomcat and Spring MVC, along others -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-freemarker</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
            <scope>provided</scope>
        </dependency>
    </dependencies>
 
 
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                        <!-- this will get rid of version info from war file name -->
                    <finalName>${project.artifactId}</finalName>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### 打包

```shell
mvn clean package 
```

然后将war包部署到已有的Servlet容器中。

### 同时支持jar和war类型

上面的pom.xml配置能很好的支持war包类型的部署，但是在开发阶段我们还是希望通过Main方法的方式进行运行。此时我们可以通过Maven的profile功能实现。

```xml
<project xmlns="<a class="vglnk" href="http://maven.apache.org/POM/4.0.0" rel="nofollow"><span>http</span><span>://</span><span>maven</span><span>.</span><span>apache</span><span>.</span><span>org</span><span>/</span><span>POM</span><span>/</span><span>4</span><span>.</span><span>0</span><span>.</span><span>0</span></a>" xmlns:xsi="<a class="vglnk" href="http://www.w3.org/2001/XMLSchema-instance" rel="nofollow"><span>http</span><span>://</span><span>www</span><span>.</span><span>w3</span><span>.</span><span>org</span><span>/</span><span>2001</span><span>/</span><span>XMLSchema</span><span>-</span><span>instance</span></a>"
    xsi:schemaLocation="<a class="vglnk" href="http://maven.apache.org/POM/4.0.0" rel="nofollow"><span>http</span><span>://</span><span>maven</span><span>.</span><span>apache</span><span>.</span><span>org</span><span>/</span><span>POM</span><span>/</span><span>4</span><span>.</span><span>0</span><span>.</span><span>0</span></a> <a class="vglnk" href="http://maven.apache.org/xsd/maven-4.0.0.xsd" rel="nofollow"><span>http</span><span>://</span><span>maven</span><span>.</span><span>apache</span><span>.</span><span>org</span><span>/</span><span>xsd</span><span>/</span><span>maven</span><span>-</span><span>4</span><span>.</span><span>0</span><span>.</span><span>0</span><span>.</span><span>xsd</span></a>">
    <modelVersion>4.0.0</modelVersion>
 
    <groupId>com.websystique.springboot</groupId>
    <artifactId>SpringBootStandAloneWarExample</artifactId>
    <version>1.0.0</version>
    <packaging>${artifact-packaging}</packaging>
 
    <name>${project.artifactId}</name>
 
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.4.3.RELEASE</version>
    </parent>
 
    <properties>
        <java.version>1.8</java.version>
        <startClass>SpringBootStandAloneWarApp</startClass>
        <!-- Additionally, Please make sure that your JAVA_HOME is pointing to 
            1.8 when building on commandline -->
    </properties>
 
    <dependencies>
        <!-- Add typical dependencies for a web application -->
        <!-- Adds Tomcat and Spring MVC, along others -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-freemarker</artifactId>
        </dependency>
    </dependencies>
 
 
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
 
    <profiles>
        <profile>
            <id>local</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
                <artifact-packaging>jar</artifact-packaging>
            </properties>
        </profile>
        <profile>
            <id>remote</id>
            <properties>
                <artifact-packaging>war</artifact-packaging>
            </properties>
            <dependencies>
                <dependency>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-tomcat</artifactId>
                    <scope>provided</scope>
                </dependency>
            </dependencies>
            <build>
                <plugins>
                    <plugin>
                        <groupId>org.springframework.boot</groupId>
                        <artifactId>spring-boot-maven-plugin</artifactId>
                        <version>1.4.3.RELEASE</version>
                        <configuration>
                                   <!-- this will get rid of version info from war file name -->
                            <finalName>${project.artifactId}</finalName>
                        </configuration>
                    </plugin>
                </plugins>
            </build>
        </profile>
    </profiles>
</project>
```

现在我们就可以在本地以FAT jar的方式运行。当发布到生产环境时以war的方式进行部署了。




