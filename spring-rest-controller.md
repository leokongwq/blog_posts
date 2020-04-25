---
layout: post
comments: true
title: 基于Spring构建RESTFUL风格的controller
date: 2018-05-12 23:23:46
tags:
- spring
categories:
- web
---

### 前言

Spring为开发REST服务提供一流的支持。在本文中，我们将使用Spring 4 `@RestController`注解开发基于Spring 4 MVC的RESTful JSON服务和RESTful XML服务。

Spring在内部使用`HttpMessageConverters`将响应转换为所需的格式[JSON / XML / etc ..]，这些格式基于类路径中可用的某些库，并可以选择使用请求中的`Accept Headers`。

为了服务JSON，我们将使用Jackson库[jackson-databind.jar]。 对于XML，我们将使用Jackson XML扩展[jackson-dataformat-xml.jar]。 只有在类路径中存在这些库才会触发Spring以所需格式转换输出。 此外，我们将进一步通过使用JAXB批注注释域类来支持XML，以防Jackson的XML扩展库由于某种原因而不可用。


**注意**：如果你通过在浏览器中输入网址发送请求，则可以添加后缀[.xml / .json]，以帮助确定要提供的内容的类型。

<!-- more -->

> 文章使用的是SpringBoot 1.5.2版本，并使用MAVEN3管理项目。

### 第一步 添加实体类

```java
public class Message {
 
    String name;
    String text;
 
    public Message(String name, String text) {
        this.name = name;
        this.text = text;
    }
 
    public String getName() {
        return name;
    }
 
    public String getText() {
        return text;
    }
}
```

### 第二步 添加 Controller

```java
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
 
import com.websystique.springmvc.domain.Message;
 
@RestController
public class HelloWorldRestController {
 
    @RequestMapping("/")
    public String welcome() {
        return "Welcome to RestTemplate Example.";
    }
 
    @RequestMapping("/hello/{player}")
    public Message message(@PathVariable String player) {
        Message msg = new Message(player, "Hello " + player);
        return msg;
    }
}
```

如果jackson-dataformat-xml.jar不可用，并且您仍希望获得XML响应，则只需在模型类（Message）上添加JAXB注释，即可启用XML输出支持。 以下是相同的演示

```java
import javax.xml.bind.annotation.XmlElement;
import javax.xml.bind.annotation.XmlRootElement;

@XmlRootElement(name = "player")
public class Message {
 
    String name;
    String text;
 
    public Message(){
         
    }
     
    public Message(String name, String text) {
        this.name = name;
        this.text = text;
    }
 
    @XmlElement
    public String getName() {
        return name;
    }
     
    @XmlElement
    public String getText() {
        return text;
    }

}
```

有了以上的准备，你可以通过下面的请求url来获取指定格式的响应：

#### json

http://127.0.0.1:2223/hello/tom
http://127.0.0.1:2223/hello/tom.json

#### xml

http://127.0.0.1:2223/hello/tom 添加请求头 Accept:application/xml

或 

http://127.0.0.1:2223/hello/tom.xml

### ContentNegotiationStrategy

`ContentNegotiationStrategy`是一个策略接口，作用是将给定的请求解析为媒体类型（`MediaType`）列表。

它有两个重要的实现类，如下所示

#### ServletPathExtensionContentNegotiationStrategy

根据请求路径的扩展名来解析

#### HeaderContentNegotiationStrategy

根据请求头`Accept`来解析


> Spring 在内部会根据请求的MediaType信息和HttpMessageConverter支持的MediaType进行匹配，如果能找到支持该请求的MediaType的HttpMessageConverter，则利用该HttpMessageConverter输出响应。

### REST快速理解

REST代表`Representational State Transfer`。它是一种可用于设计Web服务的架构风格，可从各种客户端使用。 其核心思想是，不使用诸如CORBA，RPC或SOAP之类的复杂机制来连接机器，而是使用简单的HTTP来进行调用。

在基于REST的设计中，对资源的操作是通过一组通用的动词来实现：

- 创建资源：应该使用 HTTP POST
- 检索资源：应使用 HTTP GET
- 更新资源：应该使用 HTTP PUT
- 删除资源：应该使用 HTTP DELETE

这意味着，作为REST服务开发人员或调用方，你应该遵守上述标准。

通常基于Rest的Web服务返回JSON或XML作为响应，尽管它不仅限于这些类型。 客户端可以指定（使用HTTP Accept头）他们感兴趣的资源类型，服务器可以返回资源，指定它正在服务的资源的Content-Type。 想要详细了解REST，这个[StackOverflow](https://stackoverflow.com/questions/671118/what-exactly-is-restful-programming)是必须要阅读的。

### RestController

以下是一个基于Rest的`Contrller`，实现了REST API。 

该`Contrller`是提供了如下的API：

* GET request to /api/user/ returns a list of users
* GET request to /api/user/1 returns the user with ID 1
* POST request to /api/user/ with a user object as JSON creates a new user
* PUT request to /api/user/3 with a user object as JSON updates the user with ID 3
* DELETE request to /api/user/4 deletes the user with ID 4
* DELETE request to /api/user/ deletes all the users

```java
import java.util.List;
 
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.util.UriComponentsBuilder;
 
import com.websystique.springmvc.model.User;
import com.websystique.springmvc.service.UserService;
 
@RestController
public class HelloWorldRestController {
 
    @Autowired
    UserService userService;  //Service which will do all data retrieval/manipulation work
 
     
    //-------------------Retrieve All Users--------------------------------------------------------
     
    @RequestMapping(value = "/user/", method = RequestMethod.GET)
    public ResponseEntity<List<User>> listAllUsers() {
        List<User> users = userService.findAllUsers();
        if(users.isEmpty()){
            return new ResponseEntity<List<User>>(HttpStatus.NO_CONTENT);//You many decide to return HttpStatus.NOT_FOUND
        }
        return new ResponseEntity<List<User>>(users, HttpStatus.OK);
    }
 
 
    //-------------------Retrieve Single User--------------------------------------------------------
     
    @RequestMapping(value = "/user/{id}", method = RequestMethod.GET, produces = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<User> getUser(@PathVariable("id") long id) {
        System.out.println("Fetching User with id " + id);
        User user = userService.findById(id);
        if (user == null) {
            System.out.println("User with id " + id + " not found");
            return new ResponseEntity<User>(HttpStatus.NOT_FOUND);
        }
        return new ResponseEntity<User>(user, HttpStatus.OK);
    }
 
     
     
    //-------------------Create a User--------------------------------------------------------
     
    @RequestMapping(value = "/user/", method = RequestMethod.POST)
    public ResponseEntity<Void> createUser(@RequestBody User user,    UriComponentsBuilder ucBuilder) {
        System.out.println("Creating User " + user.getName());
 
        if (userService.isUserExist(user)) {
            System.out.println("A User with name " + user.getName() + " already exist");
            return new ResponseEntity<Void>(HttpStatus.CONFLICT);
        }
 
        userService.saveUser(user);
 
        HttpHeaders headers = new HttpHeaders();
        headers.setLocation(ucBuilder.path("/user/{id}").buildAndExpand(user.getId()).toUri());
        return new ResponseEntity<Void>(headers, HttpStatus.CREATED);
    }
 
     
    //------------------- Update a User --------------------------------------------------------
     
    @RequestMapping(value = "/user/{id}", method = RequestMethod.PUT)
    public ResponseEntity<User> updateUser(@PathVariable("id") long id, @RequestBody User user) {
        System.out.println("Updating User " + id);
         
        User currentUser = userService.findById(id);
         
        if (currentUser==null) {
            System.out.println("User with id " + id + " not found");
            return new ResponseEntity<User>(HttpStatus.NOT_FOUND);
        }
 
        currentUser.setName(user.getName());
        currentUser.setAge(user.getAge());
        currentUser.setSalary(user.getSalary());
         
        userService.updateUser(currentUser);
        return new ResponseEntity<User>(currentUser, HttpStatus.OK);
    }
 
    //------------------- Delete a User --------------------------------------------------------
     
    @RequestMapping(value = "/user/{id}", method = RequestMethod.DELETE)
    public ResponseEntity<User> deleteUser(@PathVariable("id") long id) {
        System.out.println("Fetching & Deleting User with id " + id);
 
        User user = userService.findById(id);
        if (user == null) {
            System.out.println("Unable to delete. User with id " + id + " not found");
            return new ResponseEntity<User>(HttpStatus.NOT_FOUND);
        }
 
        userService.deleteUserById(id);
        return new ResponseEntity<User>(HttpStatus.NO_CONTENT);
    }
 
     
    //------------------- Delete All Users --------------------------------------------------------
     
    @RequestMapping(value = "/user/", method = RequestMethod.DELETE)
    public ResponseEntity<User> deleteAllUsers() {
        System.out.println("Deleting All Users");
 
        userService.deleteAllUsers();
        return new ResponseEntity<User>(HttpStatus.NO_CONTENT);
    }
 
}
```

#### 详解

- @RestController：首先，我们使用Spring 4的新的`@RestController`注解。此注释避免我们在每个方法上添加`@ResponseBody``注解。在Spring-MVC内部下，`@RestController`本身是用`@ResponseBody`注解的，可以被认为是`@Controller`和`@ResponseBody`的组合。
- @RequestBody：如果一个方法参数使用`@RequestBody`进行注解，那么Spring会将传入的HTTP请求主体（针对该方法的@RequestMapping中提到的URL）绑定到该参数。原理是Spring内部使用`HttpMessageConverter`将HTTP请求体转换为域对象[将请求主体反序列化为域对象]，这是基于请求中存在的`ACCEPT`或`Content-Type`头。
- @ResponseBody：如果一个方法用@ResponseBody注解，Spring会将返回值绑定到传出的HTTP响应正文。在这样做的过程中，Spring将在内部使用`HttpMessageConverter`将返回值转换为HTTP响应主体[将对象序列化到响应主体]，并基于请求HTTP头中的Content-Type。如前所述，在Spring 4中，你可能会停止使用此注释。
- ResponseEntity 它代表整个HTTP响应。好的一点是你可以控制任何进入它的东西。你可以指定状态码，标题和正文。它带有几个构造函数来携带您想要在HTTP响应中发送的信息。
- @PathVariable：这个注解表示一个方法参数应该绑定到一个URI模板变量['}']。基本上，`@RestController`，`@RequestBody`，`ResponseEntity`和`@PathVariable`是你在Spring 4中实现一个REST API所需要知道的。另外，spring提供了几个支持类来帮助你实现一些自定义的东西。
- MediaType：使用`@RequestMapping`注释，你可以另外指定要通过特定控制器方法生成或使用的`MediaType`（使用生成或消费属性），以进一步缩小映射范围。


### 使用RestTemplate编写REST客户端

PostMan是一个很棒用来测试Rest API的客户端。 但是，如果你想要在应用程序中调用基于REST的Web服务，则需要为你的应用程序提供REST客户端。 最受欢迎的HTTP客户端之一是Apache HttpComponents HttpClient。 但是，该客户端提供的功能过于基础，需要自己编写大量符合REST风格的代码。

Spring提供的RestTemplate提供了更高级别的方法，这些方法对应于六种主要的HTTP方法中的每一种，这些方法使得调用许多RESTful服务只需一行代码，并成为实施REST的最佳实践。

下面显示了HTTP方法和相应的RestTemplate方法来处理这种类型的HTTP请求。

HTTP 方法和 RestTemplate 方法对应关系:

* HTTP GET : getForObject, getForEntity
* HTTP PUT : put(String url, Object request, String…​urlVariables)
* HTTP DELETE : delete
* HTTP POST : postForLocation(String url, Object request, String…​ urlVariables), postForObject(String url, Object request, Class responseType, String…​ uriVariables)
* HTTP HEAD : headForHeaders(String url, String…​ urlVariables)
* HTTP OPTIONS : optionsForAllow(String url, String…​ urlVariables)
* HTTP PATCH and others : exchange execute


#### 自定义REST客户端，使用先前创建的REST服务

```java
import java.net.URI;
import java.util.LinkedHashMap;
import java.util.List;
 
import org.springframework.web.client.RestTemplate;
 
import com.websystique.springmvc.model.User;
 
public class SpringRestTestClient {
 
    public static final String REST_SERVICE_URI = "<a class="vglnk" href="http://localhost:8080/Spring4MVCCRUDRestService" rel="nofollow"><span>http</span><span>://</span><span>localhost</span><span>:</span><span>8080</span><span>/</span><span>Spring4MVCCRUDRestService</span></a>";
     
    /* GET */
    @SuppressWarnings("unchecked")
    private static void listAllUsers(){
        System.out.println("Testing listAllUsers API-----------");
         
        RestTemplate restTemplate = new RestTemplate();
        List<LinkedHashMap<String, Object>> usersMap = restTemplate.getForObject(REST_SERVICE_URI+"/user/", List.class);
         
        if(usersMap!=null){
            for(LinkedHashMap<String, Object> map : usersMap){
                System.out.println("User : id="+map.get("id")+", Name="+map.get("name")+", Age="+map.get("age")+", Salary="+map.get("salary"));;
            }
        }else{
            System.out.println("No user exist----------");
        }
    }
     
    /* GET */
    private static void getUser(){
        System.out.println("Testing getUser API----------");
        RestTemplate restTemplate = new RestTemplate();
        User user = restTemplate.getForObject(REST_SERVICE_URI+"/user/1", User.class);
        System.out.println(user);
    }
     
    /* POST */
    private static void createUser() {
        System.out.println("Testing create User API----------");
        RestTemplate restTemplate = new RestTemplate();
        User user = new User(0,"Sarah",51,134);
        URI uri = restTemplate.postForLocation(REST_SERVICE_URI+"/user/", user, User.class);
        System.out.println("Location : "+uri.toASCIIString());
    }
 
    /* PUT */
    private static void updateUser() {
        System.out.println("Testing update User API----------");
        RestTemplate restTemplate = new RestTemplate();
        User user  = new User(1,"Tomy",33, 70000);
        restTemplate.put(REST_SERVICE_URI+"/user/1", user);
        System.out.println(user);
    }
 
    /* DELETE */
    private static void deleteUser() {
        System.out.println("Testing delete User API----------");
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.delete(REST_SERVICE_URI+"/user/3");
    }
 
 
    /* DELETE */
    private static void deleteAllUsers() {
        System.out.println("Testing all delete Users API----------");
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.delete(REST_SERVICE_URI+"/user/");
    }
 
    public static void main(String args[]){
        listAllUsers();
        getUser();
        createUser();
        listAllUsers();
        updateUser();
        listAllUsers();
        deleteUser();
        listAllUsers();
        deleteAllUsers();
        listAllUsers();
    }
}
```

输出：

```
Testing listAllUsers API-----------
User : id=1, Name=Sam, Age=30, Salary=70000.0
User : id=2, Name=Tom, Age=40, Salary=50000.0
User : id=3, Name=Jerome, Age=45, Salary=30000.0
User : id=4, Name=Silvia, Age=50, Salary=40000.0
Testing getUser API----------
User [id=1, name=Sam, age=30, salary=70000.0]
Testing create User API----------
Location : <a class="vglnk" href="http://localhost:8080/Spring4MVCCRUDRestService/user/5" rel="nofollow"><span>http</span><span>://</span><span>localhost</span><span>:</span><span>8080</span><span>/</span><span>Spring4MVCCRUDRestService</span><span>/</span><span>user</span><span>/</span><span>5</span></a>
Testing listAllUsers API-----------
User : id=1, Name=Sam, Age=30, Salary=70000.0
User : id=2, Name=Tom, Age=40, Salary=50000.0
User : id=3, Name=Jerome, Age=45, Salary=30000.0
User : id=4, Name=Silvia, Age=50, Salary=40000.0
User : id=5, Name=Sarah, Age=51, Salary=134.0
Testing update User API----------
User [id=1, name=Tomy, age=33, salary=70000.0]
Testing listAllUsers API-----------
User : id=1, Name=Tomy, Age=33, Salary=70000.0
User : id=2, Name=Tom, Age=40, Salary=50000.0
User : id=3, Name=Jerome, Age=45, Salary=30000.0
User : id=4, Name=Silvia, Age=50, Salary=40000.0
User : id=5, Name=Sarah, Age=51, Salary=134.0
Testing delete User API----------
Testing listAllUsers API-----------
User : id=1, Name=Tomy, Age=33, Salary=70000.0
User : id=2, Name=Tom, Age=40, Salary=50000.0
User : id=4, Name=Silvia, Age=50, Salary=40000.0
User : id=5, Name=Sarah, Age=51, Salary=134.0
Testing all delete Users API----------
Testing listAllUsers API-----------
No user exist----------
```

### REST API 添加 CORS 支持

在访问REST API时，您可能会面临有关同源策略的问题。

可能的错误如下：

- “no” Access-Control-Allow-Origin“标题出现在请求的资源上。 原因'http://127.0.0.1:8080'因此不被允许访问。“或
- “XMLHttpRequest无法加载`http://abc.com/bla`。 原始`http：// localhost：12345`不被Access-Control-Allow-Origin允许。“在这种情况下很常见。

解决方案是Cross-Origin Resource Sharing(跨源资源共享)。 基本上，在服务器端，我们可以返回额外的CORS访问控制头和响应，这最终将允许进一步的域间通信。

在Spring中，我们可以编写一个简单的过滤器，在每个响应中添加这些CORS特定的响应头信息。

```java
import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletResponse;
 

@WebFilter 
public class CORSFilter implements Filter {
 
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
        System.out.println("Filtering on...........................................................");
        HttpServletResponse response = (HttpServletResponse) res;
        response.setHeader("Access-Control-Allow-Origin", "*");
        response.setHeader("Access-Control-Allow-Methods", "POST, GET, PUT, OPTIONS, DELETE");
        response.setHeader("Access-Control-Max-Age", "3600");
        response.setHeader("Access-Control-Allow-Headers", "x-requested-with");
        chain.doFilter(req, res);
    }
 
    public void init(FilterConfig filterConfig) {}
 
    public void destroy() {}
 
}
```

### 参考资料

[spring-mvc-4-restful-web-services-crud-example-resttemplate](http://websystique.com/springmvc/spring-mvc-4-restful-web-services-crud-example-resttemplate/)















