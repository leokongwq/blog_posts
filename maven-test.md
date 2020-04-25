---
layout: post
comments: true
title: maven-surefire-plugin简介
date: 2016-10-12 16:04:40
tags:
    - maven
categories:
    - java
    
---

### maven-surefire-plugin简介

Maven本身并不是一个单元测试框架，它只是在构建执行到特定生命周期阶段的时候，通过插件来执行JUnit或者TestNG的测试用例。这个插件就是maven-surefire-plugin，也可以称为测试运行器(Test Runner)，它能兼容JUnit 3、JUnit 4以及TestNG。

<!-- more -->

在默认情况下，maven-surefire-plugin的test目标会自动执行测试源码路径（默认为src/test/java/）下所有符合一组命名模式的测试类。这组模式为：

`**/Test*.java`：任何子目录下所有命名以Test开关的Java类。
`**/*Test.java`：任何子目录下所有命名以Test结尾的Java类。
`**/*TestCase.java`：任何子目录下所有命名以TestCase结尾的Java类。

### 2.跳过测试

要想跳过测试，在命令行加入参数skipTests就可以了。如：

    mvn package -DskipTests  

也可以在pom配置中提供该属性。

    <plugin>  
        <groupId>org.apache.maven.plugins</groupId>  
        <artifactId>maven-surefire-plugin</artifactId>  
        <version>2.5</version>  
        <configuration>  
            <skipTests>true</skipTests>  
        </configuration>  
    </plugin>  

有时候可能不仅仅需要跳过测试运行，还要跳过测试代码的编译：

    mvn package -Dmaven.test.skip=true  

也可以在pom中配置`maven.test.skip:`

    <plugin>  
        <groupId>org.apache.maven.plugin</groupId>  
        <artifactId>maven-compiler-plugin</artifactId>  
        <version>2.1</version>  
        <configuration>  
            <skip>true</skip>  
        </configuration>  
    </plugin>  
    <plugin>  
        <groupId>org.apache.maven.plugins</groupId>  
        <artifactId>maven-surefire-plugin</artifactId>  
        <version>2.5</version>  
        <configuration>  
            <skip>true</skip>  
        </configuration>  
    </plugin>  

### 3.动态指定要运行的测试用例

maven-surefire-plugin提供了一个test参数让Maven用户能够在命令行指定要运行的测试用例。如：

    mvn test -Dtest=RandomGeneratorTest  

也可以使用通配符：

    mvn test -Dtest=Random*Test  

或者也可以使用“，”号指定多个测试类：

    mvn test -Dtest=Random*Test,AccountCaptchaServiceTest  
如果没有指定测试类，那么会报错并导致构建失败。

    mvn test -Dtest  

这时候可以添加-DfailIfNoTests=false参数告诉maven-surefire-plugin即使没有任何测试也不要报错。

    mvn test -Dtest -DfailIfNoTests=false  

由此可见，命令行参数-Dtest -DfailIfNoTests=false是另外一种路过测试的方法

### 4.包含与排除测试用例

如果由于历史原因，测试类不符合默认的三种命名模式，可以通过pom.xml设置maven-surefire-plugin插件添加命名模式或排除一些命名模式。

    <plugin>  
        <groupId>org.apache.maven.plugins</groupId>  
        <artifactId>maven-surefire-plugin</artifactId>  
        <version>2.5</version>  
        <configuration>  
            <includes>  
                <include>**/*Tests.java</include>  
            </includes>  
            <excludes>  
                <exclude>**/*ServiceTest.java</exclude>  
                <exclude>**/TempDaoTest.java</exclude>  
            </excludes>  
        </configuration>  
    </plugin>  

### 5.生成测试报告

#### 5.1 基本测试报告

默认情况下，maven-surefire-plugin会在项目的target/surefire-reports目录下生成两种格式的错误报告。
简单文本格式——内容十分简单，可以看出哪个测试项出错。
与JUnit兼容的XML格式——XML格式已经成为了Java单元测试报告的事实标准，这个文件可以用其他的工具如IDE来查看。

##### 5.2 测试覆盖率报告

测试覆盖率是衡量项目代码质量的一个重要的参考指标。Cobertura是一个优秀的开源测试覆盖率统计工具（详见http://cobertura.sourceforge.net/)，Maven通过cobertura-maven-plugin与之集成，用户可以使用简单的命令为Maven项目生成测试覆盖率报告。运行下面命令生成报告：

    mvn cobertura:cobertura  

### 6.运行TestNG测试

TestNG是Java社区中除了JUnit之外另一个流行的单元测试框架。TestNG在JUnit的基础上增加了很多特性，其站点是http://testng.org/ .添加TestNG依赖：

    <dependency>  
        <groupId>org.testng</groupId>  
        <artifactId>testng</artifactId>  
        <version>5.9</version>  
        <scope>test</scope>  
        <classifier>jdk15</classifier>  
    </dependency>  

下面是JUnit和TestNG的常用类库对应关系

|       JUnit类      |       TestNG类     |  作用 |
| ------------- | ------------- | ----- |
| org.junit.Test | org.testng.annotations.Test | 标注方法为测试方法 | 
| org.junit.Assert | org.testng.Assert | 检查测试结果 |
| org.junit.Before | org.testng.annotations.BeforeMethod |  标注方法在每个测试方法之前运行 |
| org.junit.After | org.testng.annotations.AfterMethod | 标注方法在每个测试方法之后运行 |
| org.junit.BeforeClass | org.testng.annotations.BeforeClass | 标注方法在所有测试方法之前运行 |  
| org.junit.AfterClass | org.testng.annotations.AfterClass | 标注方法在所有测试方法之后运行 |  
		
TestNG允许用户使用一个名为testng.xml的文件来配置想要运行的测试集合。如在类路径上添加testng.xml文件，配置只运行RandomGeneratorTest

    <?xml version="1.0" encoding="UTF-8"?>  
    <suite name="Suite1" verbose="1">  
        <test name="Regression1">  
            <classes>  
                <class name="com.juvenxu.mvnbook.account.captcha.RandomGeneratorTest" />  
            </classes>  
        </test>  
    </suite>  

同时再配置maven-surefire-plugin使用该testng.xml，如：

    <plugin>  
        <groupId>org.apache.maven.plugins</groupId>  
        <artifactId>maven-surefire-plugin</artifactId>  
        <version>2.5</version>  
        <configuration>  
            <suiteXmlFiles>  
                <suiteXmlFile>testng.xml</suiteXmlFile>  
            </suiteXmlFiles>  
        </configuration>  
    </plugin>  

TestNG较JUnit的一大优势在于它支持测试组的概念。如可以在方法级别声明测试组：

    @Test(groups={"util","medium"})  

然后可以在pom中配置运行一个或多个测试组：

    <plugin>  
        <groupId>org.apache.maven.plugins</groupId>  
        <artifactId>maven-surefire-plugin</artifactId>  
        <version>2.5</version>  
        <configuration>  
            <groups>util,medium</groups>  
        </configuration>  
    </plugin>  

### 7.重用测试代码

当命令行运行`mvn package`的时候，Maven只会打包主代码及资源文件，并不会对测试代码打包。如果测试代码中有需要重用的代码，这时候就需要对测试代码打包了。
这时候需要配置maven-jar-plugin将测试类打包，如：

    <plugin>  
        <groupId>org.apache.maven.plugins</groupId>  
        <artifactId>maven-jar-plugin</artifactId>  
        <version>2.2</version>  
        <executions>  
            <execution>  
                <goals>  
                    <goal>test-jar</goal>  
                </goals>  
            </execution>  
        </executions>  
    </plugin>  
    
maven-jar-plugin有两个目标，分别为jar和test-jar。这两个目标都默认绑定到default生命周期的package阶段运行，只是test-jar并没有在超级POM中配置，因此需要我们另外在pom中配置。

现在如要引用test-jar生成的测试代码包，可以如下配置：

    <dependency>  
        <groupId>com.juvenxu.mvnbook.account</groupId>  
        <artifactId>account-captcha</artifactId>  
        <version>1.0.0-SNAPSHOT</version>  
        <type>test-jar</type>  
        <scope>test</scope>  
    </dependency>  
                        
                    
                    