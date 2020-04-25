---
layout: post
comments: true
title: java中xml解析器介绍
date: 2017-02-18 22:09:09
tags:
- xml
categories:
- java
---


### What is XML Parser?

xml 解析器提供了访问或修改用来表示数据的xml文件的能力。Java中提供了多种方式来解析xml文件。下面列出了Java中常用的xml解析器

<!-- more -->

- Dom Parser - 将整个xml文件读入内存并构造一个含有继承关系的树结构
- SAX Parser - 基于事件方式的解析器，不需要将整个文档读入内存。
- JDOM Parser - 以类似于DOM解析器的方式解析文档，但更容使用。
- StAX Parser - 以类似于SAX解析器的方式解析文档，但是以更有效的方式。
- XPath Parser - 基于表达式解析XML，并与XSLT结合广泛使用。
- DOM4J Parser - 一个使用 XPath 和 XSLT 和Java 集合框架解析XML文件的库，并提供了对DOM, SAX 和 JAXP 的支持。
- Digester - xml和Java对象之间进行转换。
- JAXB - xml和Java对象之间进行转换。
- XStream  - 一个在xml和Java对象之间进行转换的库

### Digester Example

#### maven dependency

```
<dependency>
    <groupId>commons-digester</groupId>
    <artifactId>commons-digester</artifactId>
    <version>1.8</version>
</dependency>
```

#### xml file 

```
<?xml version="1.0" encoding="UTF-8"?>  
<root>  
    <student id="1" group="1">  
        <name>张三</name>  
        <sex>男</sex>  
        <age>18</age>  
        <birthday>1990-09-09</birthday>   
    </student>  
    <student id="2" group="2">  
        <name>李四</name>  
        <sex>女</sex>  
        <age>17</age>  
        <birthday>1990-10-10</birthday>  
    </student>  
</root>
```

#### java object 

```java
public class Root {
    public ListstudentList = new ArrayList();
     
    public void addStudent(Student student) {
        this.studentList.add(student);
    }
}
public class Student {
	private Long id;
	private String group;
	private String name;
	private String sex;
	private Integer age;
	private String birthday;
   
   .... setter , getter    	
}
```

#### parse

```java
public class XmlDigesterTester {
    public static void main(String[] args) throws IOException, SAXException {
        // XML文件的路径
        String path = XmlDigesterTester.class.getClassLoader().getResource("com/xml/student.xml").getPath();
        File input = new File(path);
        Digester digester = new Digester();
        // 初始化根对象Root（本案例的XML根目录命名为ROOT）
        digester.addObjectCreate("root", "com.xml.Root");
        // 将root节点上的属性全部注入到Root对象中。
        digester.addSetProperties("root");
        // 将所有匹配到的目录root/student转成javaBean对象Student（参见Student类）
        digester.addObjectCreate("root/student", "com.xml.Student");
        // 将节点root/student上的属性注入到Student对象中
        digester.addSetProperties("root/student");
        // 以下是root/student包含的节点根据名称映射，将节点值setter到对应的Student对象中。
        digester.addBeanPropertySetter("root/student/name");
        digester.addBeanPropertySetter("root/student/sex");
        digester.addBeanPropertySetter("root/student/age");
        digester.addBeanPropertySetter("root/student/birthday");
        // 将所有匹配root/student节点，转化成了Student对象后，传入Root对象的addStudent方法中（参见Root类）
        digester.addSetNext("root/student", "addStudent", "com.xml.Student");
        // 定义好了上面的解析规则后，就可以开始进行解析工作了
        Root root = (Root) digester.parse(input);
        // 打印输入生成的对象结构
        System.out.println(JSON.toJSONString(root));
    }
}
```

### XStream Example 

#### 对象类定义

```
public class Person {
  private String firstname;
  private String lastname;
  private PhoneNumber phone;
  private PhoneNumber fax;
  // ... constructors and methods
}

public class PhoneNumber {
  private int code;
  private String number;
  // ... constructors and methods
}
```

#### 初始化XStream 

```java
XStream xstream = new XStream();
//or 
XStream xstream = new XStream(new DomDriver()); // does not require XPP3 library
//or
XStream xstream = new XStream(new StaxDriver()); // does not require XPP3 library starting with Java 6

xstream.alias("person", Person.class);
xstream.alias("phonenumber", PhoneNumber.class);
```

#### Serializing an object to XML

```java
Person joe = new Person("Joe", "Walnes");
joe.setPhone(new PhoneNumber(123, "1234-456"));
joe.setFax(new PhoneNumber(123, "9999-999"));
String xml = xstream.toXML(joe);
```

#### Deserializing an object back from XML

```java
Person newJoe = (Person)xstream.fromXML(xml);
```

### JAXB Example 

#### 对象类定义

```java
import javax.xml.bind.annotation.XmlAttribute;
import javax.xml.bind.annotation.XmlElement;
import javax.xml.bind.annotation.XmlRootElement;

@XmlRootElement
public class Customer {

	String name;
	int age;
	int id;

	public String getName() {
		return name;
	}

	@XmlElement
	public void setName(String name) {
		this.name = name;
	}

	public int getAge() {
		return age;
	}

	@XmlElement
	public void setAge(int age) {
		this.age = age;
	}

	public int getId() {
		return id;
	}

	@XmlAttribute
	public void setId(int id) {
		this.id = id;
	}
}
```

#### 对象到xml文件

```java
import java.io.File;
import javax.xml.bind.JAXBContext;
import javax.xml.bind.JAXBException;
import javax.xml.bind.Marshaller;

public class JAXBExample {
	public static void main(String[] args) {

	  Customer customer = new Customer();
	  customer.setId(100);
	  customer.setName("mkyong");
	  customer.setAge(29);

	  try {

		File file = new File("C:\\file.xml");
		JAXBContext jaxbContext = JAXBContext.newInstance(Customer.class);
		Marshaller jaxbMarshaller = jaxbContext.createMarshaller();

		// output pretty printed
		jaxbMarshaller.setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, true);

		jaxbMarshaller.marshal(customer, file);
		jaxbMarshaller.marshal(customer, System.out);

	      } catch (JAXBException e) {
		e.printStackTrace();
	      }

	}
}
```

#### xml到对象

```java
package com.mkyong.core;

import java.io.File;
import javax.xml.bind.JAXBContext;
import javax.xml.bind.JAXBException;
import javax.xml.bind.Unmarshaller;

public class JAXBExample {
	public static void main(String[] args) {

	 try {

		File file = new File("C:\\file.xml");
		JAXBContext jaxbContext = JAXBContext.newInstance(Customer.class);

		Unmarshaller jaxbUnmarshaller = jaxbContext.createUnmarshaller();
		Customer customer = (Customer) jaxbUnmarshaller.unmarshal(file);
		System.out.println(customer);

	  } catch (JAXBException e) {
		e.printStackTrace();
	  }

	}
}
```

### 参考

[https://www.tutorialspoint.com/java_xml/java_xml_parsers.htm](https://www.tutorialspoint.com/java_xml/java_xml_parsers.htm)

[http://www.everycoding.com/coding/78.html](http://www.everycoding.com/coding/78.html)

[https://www.mkyong.com/java/jaxb-hello-world-example/](https://www.mkyong.com/java/jaxb-hello-world-example/)

[http://www.mkyong.com/tutorials/java-xml-tutorials/](http://www.mkyong.com/tutorials/java-xml-tutorials/)

[http://x-stream.github.io/](http://x-stream.github.io/)


