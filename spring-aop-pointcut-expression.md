---
layout: post
comments: true
title: spring aop 切点表达式介绍
date: 2016-10-12 16:04:23
tags:
    - aop
    - spring
categories:
    - java
    - spring
    
---
                                                
### Pointcut定义

> Pointcut 是指那些需要被执行"AOP"的方法,是由"Pointcut Expression"来描述的.Pointcut可以有下列方式来定义或者通过 && || 和!的方式进行组合. 

<!-- more -->

 - execution()
 - args()
 - @args()
 - this()
 - target()
 - @target()
 - within()
 - @within()
 - @annotation
 
其中execution 是用的最多的,其格式为:
    
**execution(modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern(param-pattern)throws-pattern?)**

`returning type pattern` , `name pattern`, `parameters pattern` 是必须的。
`ret-type-pattern`:可以为`*`表示任何返回值,全路径的类名等.
`name-pattern`:指定方法名,`*`代表所有,set*,代表以set开头的所有方法.
`parameters pattern`:指定方法参数(声明的类型),(..)代表所有参数,(*)代表一个参数,(*,String)代表第一个参数为任何值,第二个为String类型.

**举例说明:**

任意公共方法的执行：

    execution(public * *(..))

任何一个以“set”开始的方法的执行：

    execution(* set*(..))
    
AccountService 接口的任意方法的执行：

    execution(* com.xyz.service.AccountService.*(..))

定义在service包里的任意方法的执行：

    execution(* com.xyz.service.*.*(..))

定义在service包和所有子包里的任意类的任意方法的执行：

    execution(* com.xyz.service..*.*(..))
    
定义在pointcutexp包和所有子包里的JoinPointObjP2类的任意方法的执行：

    execution(* com.test.spring.aop.pointcutexp..JoinPointObjP2.*(..))")
    
***> 最靠近(..)的为方法名,靠近.*(..))的为类名或者接口名,如上例的JoinPointObjP2.*(..))

**pointcutexp**包里的任意类.

    within(com.test.spring.aop.pointcutexp.*)

pointcutexp包和所有子包里的任意类.

    within(com.test.spring.aop.pointcutexp..*)

实现了Intf接口的所有类,如果Intf不是接口,限定Intf单个类.

    this(com.test.spring.aop.pointcutexp.Intf)
    
**当一个实现了接口的类被AOP的时候,用getBean方法必须cast为接口类型,不能为该类的类型.**

带有@Transactional标注的所有类的任意方法.

    @within(org.springframework.transaction.annotation.Transactional)
    @target(org.springframework.transaction.annotation.Transactional)

带有@Transactional标注的任意方法.

    @annotation(org.springframework.transaction.annotation.Transactional)
    
**@within和@target针对类的注解,@annotation是针对方法的注解**

参数带有@Transactional标注的方法.

    @args(org.springframework.transaction.annotation.Transactional)
    
参数为String类型(运行时决定)的方法.
    
    args(String)
    
Pointcut 可以通过Java注解和XML两种方式配置,如下所示:

{% codeblock lang:xml %} 
    <aop:config>  
        <aop:aspectrefaop:aspectref="aspectDef">  
            <aop:pointcutidaop:pointcutid="pointcut1"expression="execution(* com.test.spring.aop.pointcutexp..JoinPointObjP2.*(..))"/>  
            <aop:before pointcut-ref="pointcut1" method="beforeAdvice" />  
        </aop:aspect>  
    </aop:config>  
{% endcodeblock %}

{% codeblock lang:java %}
 
    @Component  
    @Aspect  
    public class AspectDef {  
        //@Pointcut("execution(* com.test.spring.aop.pointcutexp..JoinPointObjP2.*(..))")  
        //@Pointcut("within(com.test.spring.aop.pointcutexp..*)")  
        //@Pointcut("this(com.test.spring.aop.pointcutexp.Intf)")  
        //@Pointcut("target(com.test.spring.aop.pointcutexp.Intf)")  
        //@Pointcut("@within(org.springframework.transaction.annotation.Transactional)")  
        //@Pointcut("@annotation(org.springframework.transaction.annotation.Transactional)")  
        @Pointcut("args(String)")  
        public void pointcut1() {  
        }  
        @Before(value = "pointcut1()")  
        public void beforeAdvice() {  
            System.out.println("pointcut1 @Before...");  
        }
    }    

{% endcodeblock %}                    
                    