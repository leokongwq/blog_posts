---
layout: post
comments: true
title: hystrix学习总结
date: 2019-08-10 19:36:33
tags:
- hystrix
categories:
---

### hystrix是什么？

官网对hystrix的定义如下：
> Hystrix is a latency and fault tolerance library designed to isolate points of access to remote systems, services and 3rd party libraries, stop cascading failure and enable resilience in complex distributed systems where failure is inevitable.

个人理解如下：

1. 首先 Hystrix 只是一个库，意味着非常容易使用，通过maven或gradle引入即可。
2. Hystrix 的目的是在访问远程系统，服务或使用第三方库的时候由于网络故障，延迟或其他错误可能导致的系统级联故障的点进行隔离。从而使得整个分布式系统具有弹性和容错能力。

<!-- more -->

### hystrix基本用法

#### HystrixCommand

使用Hystrix最简单直接的方式就是继承`HystrixCommand`对象并复写`run`方法。

例子如下：

```java
public class HelloWorldCommand extends HystrixCommand<String> {

    private final String name;

    public HelloWorldCommand(String name) {
        // 通过工厂方法HystrixCommandGroupKey.Factory.asKey
        // 指定CommandGroup的名称
        super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
        this.name = name;
    }

    @Override
    protected String run() {
        return "Hello " + name + "!";
    }
    // 注意只有非 HystrixBadRequestException 才会触发fallback    
    @Override
    public String getFallback() {
        return "fallback: " + name;
    }
}
```

使用方式如下：

```java

String result = new HelloWorldCommand("jiexiu").execute();

Future<String> result = new HelloWorldCommand("jiexiu").queue();

//从名字也能看出来，需要在返回值上注册一个观察者来获取结果
Observable<String> observable = new HelloWorldCommand("jiexiu").observe();
// 或
Observable<String> stringObservable1 = helloWorldCommand.toObservable();
// 注册观察者 获取结果
// 先执行onNext再执行onCompleted
observable.subscribe(new Observer<String>() {
    @Override
    public void onCompleted() {
    	System.out.println("completed");
    }
    @Override
    public void onError(Throwable e) {
    	e.printStackTrace();
    }
    @Override
    public void onNext(String v) {
    	System.out.println("onNext: " + v);
    }
});
		
// 只有成功才会被回调
observable.subscribe(new Action1<String>() {
	// 相当于上面的onNext()
	// @Override
	public void call(String v) {
		System.out.println("call: " + v);
	}
});
```

`execute`方法用来同步获取执行结果（内部还是通过线程池异步执行）内部逻辑如下：

```java
public R execute() {
    try {
        return queue().get();
    } catch (Exception e) {
        throw Exceptions.sneakyThrow(decomposeException(e));
    }
}
```

`queue` 通过返回一个`Future`来异步获取结果。

`observe和toObservable` 通过返回一个`Observable`, 需要用户注册一个回调来接收调用结果。

`observe` 和 `toObservable`的区别是：observe 在注册回调前就已经执行服务调用，toObservable是在注册回调后才开始执行。

#### HystrixObservableCommand

HystrixObservableCommand 和 HystrixCommand 功能是相同的。不过HystrixObservableCommand的定位是用在全异步的环境下。

```java
public class HelloHystrixObservableCommand extends HystrixObservableCommand<String> {

    private String name;

    public HelloHystrixObservableCommand(String name) {
        super(HystrixCommandGroupKey.Factory.asKey("HelloHystrixObservableCommand"));
        this.name = name;
    }

    @Override
    protected Observable<String> construct() {
//        return Observable.just("1", "2", "3");
//        return Observable.just("Hello " + name);
//        return Observable.from(new String[]{"1", "2"});

        // OnSubscribe 是一个Callback， 当 Observable被注册的时候会被执行
        return Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                //remote http call
                try {// 写业务逻辑，注意try-catch
                    if (!subscriber.isUnsubscribed()) {
                        RestTemplate restTemplate = new RestTemplate();
                        String result = restTemplate.getForObject("http://www.jiexiu.com", String.class);
                        //
                        subscriber.onNext(result);
                        subscriber.onCompleted();
                    }
                } catch (Exception e) {
                    subscriber.onError(e);
                }
            }
        }).subscribeOn(Schedulers.io());
    }
    /**
    * fallback 
    */ 
    @Override
    public Observable<String> resumeWithFallback() {
        return Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> observer) {
                try {
                    if (!observer.isUnsubscribed()) {
                        observer.onNext("fallback-jiexiu");
                        observer.onCompleted();
                    }
                } catch (Exception e) {
                    observer.onError(e);
                }
            }
        }).subscribeOn(Schedulers.io());
    }
    public static void main(String[] args) throws IOException {
        HelloHystrixObservableCommand observableCommand =
                new HelloHystrixObservableCommand("jiexiu");
        Observable<String> observable = observableCommand.construct();
        observable.subscribe(System.out::println);

        System.in.read();
    }
}
```

### fallback

Hystrix中实现优雅降级有2中方式：

1. 通过在HystrixCommand中添加`getFallback`方法
2. 通过在HystrixObservableCommand中添加`resumeWithFallback`方法

#### fallback 触发时机

1. run方法或construct方法执行异常（非HystrixBadRequestException异常）。
2. run方法或construct方法超时
3. 线程池满了或信号量为空
4. 断路器处于打开状态

注意: run方法抛出的所有异常中除了`HystrixBadRequestException`异常外都会被记录为失败，触发fallback，更新断路器统计信息。

### Hystrix 隔离机制

Hystrix有2中实现请求故障隔离的机制。一种是利用线程池，一种是使用信号量，默认是使用线程池。

#### 线程池

```java
public HelloWorldCommand(String name) {
//        super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
    super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"))
            .andCommandKey(HystrixCommandKey.Factory.asKey("helloWorld"))
            .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("ExampleGroup-ThreadPool"))
            .andThreadPoolPropertiesDefaults(HystrixThreadPoolProperties.Setter()
                    .withCoreSize(5) //默认是10
                    .withMaximumSize(10) //默认是10
                    .withKeepAliveTimeMinutes(5) //默认是1，单位为分钟
                    .withMaxQueueSize(100) // 默认是-1, 意味着底层使用 SynchronousQueue, 该属性不能动态修改
                    .withQueueSizeRejectionThreshold(80)
            )
            .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
                    .withExecutionTimeoutInMilliseconds(3000) //默认是1s
                    .withCircuitBreakerEnabled(true) //默认是true

            )
    );
    this.name = name;
}
```

线程池机制很容易理解，不同的HystrixCommandGroup底层使用不同的线程池，彼此不干扰。
可以通过`andThreadPoolKey`配置了线程池的名称。如果没有通过`andThreadPoolKey`来设置线程池的名称，默认使用`withGroupKey`设置的名称。

#### 信号量

```java
public HelloWorldCommand(String name) {
    super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"))
                .andCommandKey(HystrixCommandKey.Factory.asKey("helloWorld"))
                .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("ExampleGroup-Semaphore-ThreadPool"))
                .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
                        .withExecutionTimeoutInMilliseconds(3000) //默认是1s
                        .withCircuitBreakerEnabled(true) //默认是true
                        .withExecutionIsolationStrategy(HystrixCommandProperties.ExecutionIsolationStrategy.SEMAPHORE)
                        .withExecutionIsolationSemaphoreMaxConcurrentRequests(3) 
                        //默认值10
                        .withFallbackIsolationSemaphoreMaxConcurrentRequests(1)
                        //默认值10
                )
        );
    this.name = name;
}
```

### 熔断器 CircuitBreaker

通常有2


### 一些细节

HystrixCommandGroupKey 需要指定，用来将不同的HystrixCommand组织起来

HystrixCommandKey 的默认值是 Command的类名称(不带包名)，也可以单独指定。

HystrixThreadPoolKey 的默认值是 HystrixCommandGroupKey。但是建议手动指定HystrixCommand执行的`HystrixThreadPool`的名称。因为同属一个Group下的Command可能需要在不同的`HystrixThreadPool`中隔离执行。

在fallback的处理中，如果需要调用一个远程服务获取值(e.g. 查询缓存) 那么最好使用单独的线程池来执行，否则可能由于主Command的执行线程池已经满了导致fallback不能正常工作。

{% asset_img fallback-via-command-640.png %}





