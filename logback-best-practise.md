---
layout: post
comments: true
title: logback最佳实践
date: 2019-12-14 18:51:31
tags:
- logback
categories:
- java
---

### 背景

在最近的一次项目性能优化过程中，通过火焰图工具发现logback占用CPU很多，因此有了这篇总结文章。


### logback 同步 vs 异步 

同步写日志一般配置如下：

```xml
 <appender name="ORDER_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>/data/logs/AAA/order.log</file>
    <encoder>
        <pattern>[%-5p] [%d{yyyy-MM-dd HH:mm:ss.SSS}] [%X{tracing_id}] [%C{1}:%M:%L] %m%n</pattern>
        <immediateFlush>false</immediateFlush>  
    </encoder>
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>INFO</level>
    </filter>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <fileNamePattern>/data/logs/AAA/order.log.%d{yyyy-MM-dd_HH}</fileNamePattern>
    </rollingPolicy>
</appender>
```

<!-- more -->

从配置可以看出来，写日志到日志文件的操作由`RollingFileAppender`完成，该类的继承结构如下：

{% asset_img RollingFileAppender.jpeg %}


从继承结构可以知道`RollingFileAppender`继承了`UnsynchronizedAppenderBase`，根据doc文档说明，`RollingFileAppender`需要自己处理多线程同步的问题。 在内部它确实也自己做了同步。

```java RollingFileAppender
@Override
protected void subAppend(E event) {
    // The roll-over check must precede actual writing. This is the
    // only correct behavior for time driven triggers.

    // We need to synchronize on triggeringPolicy so that only one rollover
    // occurs at a time
    synchronized (triggeringPolicy) {
      if (triggeringPolicy.isTriggeringEvent(currentlyActiveFile, event)) {
        rollover();
      }
    }
    super.subAppend(event);
}
```

OutputStreamAppender.java

```java OutputStreamAppender.java
/**
* All synchronization in this class is done via the lock object.
*/
protected LogbackLock lock = new LogbackLock();
  
protected void subAppend(E event) {
    if (!isStarted()) {
      return;
    }
    try {
      // this step avoids LBCLASSIC-139
      if (event instanceof DeferredProcessingAware) {
        ((DeferredProcessingAware) event).prepareForDeferredProcessing();
      }
      // the synchronization prevents the OutputStream from being closed while we
      // are writing. It also prevents multiple threads from entering the same
      // converter. Converters assume that they are in a synchronized block.
      synchronized (lock) {
        writeOut(event);
      }
    } catch (IOException ioe) {
      // as soon as an exception occurs, move to non-started state
      // and add a single ErrorStatus to the SM.
      this.started = false;
      addStatus(new ErrorStatus("IO failure in appender", this, ioe));
    }
}
// 通过 encode进行日志序列化，格式化写入
protected void writeOut(E event) throws IOException {
    this.encoder.doEncode(event);
}
```

LayoutWrappingEncoder.java

```java
LayoutWrappingEncoder.java
private boolean immediateFlush = true;
public void doEncode(E event) throws IOException {
    String txt = layout.doLayout(event);
    outputStream.write(convertToBytes(txt));
    //是否立即写入
    if (immediateFlush)
      outputStream.flush();
    }
}    
```

通过上面的代码逻辑可以得出如下结论：

1. RollingFileAppender 写日志自己实现了多线程同步。 
2. RollingFileAppender 写日志是直接接入到日志文件中的。
3. RollingFileAppender 默认日志到文件后会立刻flush，保证日志不丢失。
4. 因为是顺序写文件，速度还是很高的，但是应为每次都flush，这会影响性能。

为了防止写日志影响应用性能， 我们需要使用异步写日志的方式。

异步写日志一般配置如下：

```xml
<appender name ="ASYNC_ORDER_LOG" class= "ch.qos.logback.classic.AsyncAppender">
    // 不丢弃日志
    <discardingThreshold>0</discardingThreshold>
    <queueSize>512</queueSize>
    <includeCallerData>true</includeCallerData>
    // 如果设置为true，队列满了会直接丢弃信息，而不是阻塞（其实就是使用的offer而不是put方法）
    <neverBlock>false</neverBlock>
    // 指定底层真实使用的Appender
    <appender-ref ref ="ORDER_LOG"/>
</appender>
```

{% asset_img AsyncAppender.jpeg %}

`AsyncAppender` 继承自 `AsyncAppenderBase`。 `AsyncAppenderBase`的doc文档有如下的描述：

> AsyncAppenderBase的子类写日志是异步的方式，内部使用了 BlockingQueue（异步写日志就是一个生产者-消费者模式）。 
> BlockingQueue 的使用者负责在应用关闭时关闭BlockingQueue，来确保不丢失日志。

注意：不丢失日志，可以通过如下的方式实现:

```xml
<configuration debug="true">
   <shutdownHook class="ch.qos.logback.core.hook.DelayingShutdownHook" />
</configuration>
```

或

```java
Runtime.addShutdownHook(new Thread (() -> {
   LoggerContext loggerContext = (LoggerContext) LoggerFactory.getILoggerFactory();
    loggerContext.stop();
}));
```

`AsyncAppenderBase`代码分析:

```java
public static final int DEFAULT_QUEUE_SIZE = 256;
// 默认阻塞队列大小
int queueSize = DEFAULT_QUEUE_SIZE;
static final int UNDEFINED = -1;
// 初始值为 -1， 在启动时默认值是通过queueSize的大小计算出来的。
int discardingThreshold = UNDEFINED;

// 异步写日志的线程
Worker worker = new Worker();

@Override
public void start() {
    if (appenderCount == 0) {
      addError("No attached appenders found.");
      return;
    }
    if (queueSize < 1) {
      addError("Invalid queue size [" + queueSize + "]");
      return;
    }
    blockingQueue = new ArrayBlockingQueue<E>(queueSize);

    if (discardingThreshold == UNDEFINED)
      discardingThreshold = queueSize / 5;
    addInfo("Setting discardingThreshold to " + discardingThreshold);
    worker.setDaemon(true);
    worker.setName("AsyncAppender-Worker-" + worker.getName());
    // make sure this instance is marked as "started" before staring the worker Thread
    super.start();
    worker.start();
}
  
@Override
protected void append(E eventObject) {
    // 可以的队列容量小于配置的值 并且 日志事件的级别为INFO及一下，那么就丢弃日志。
    if (isQueueBelowDiscardingThreshold() && isDiscardable(eventObject)) {
      return;
    }
    preprocess(eventObject);
    put(eventObject);
}

private boolean isQueueBelowDiscardingThreshold() {
    return (blockingQueue.remainingCapacity() < discardingThreshold);
}

protected boolean isDiscardable(ILoggingEvent event) {
    Level level = event.getLevel();
    return level.toInt() <= Level.INFO_INT;
}

protected void preprocess(ILoggingEvent eventObject) {
    eventObject.prepareForDeferredProcessing();
    // 是否需要获取调用者信息，这也是一个耗时操作，默认是false（内部通过创建异常对象，获取堆栈信息，从而计算日志发送的类，方法，行号信息）。
    if(includeCallerData)
      eventObject.getCallerData();
}

private void put(E eventObject) {
    try {
        // 阻塞操作
        blockingQueue.put(eventObject);
    } catch (InterruptedException e) {
    }
}

class Worker extends Thread {

    public void run() {
      AsyncAppenderBase<E> parent = AsyncAppenderBase.this;
      AppenderAttachableImpl<E> aai = parent.aai;

      // loop while the parent is started
      while (parent.isStarted()) {
        try {
          // 从队列中获取数据，其实就是消费者
          E e = parent.blockingQueue.take();
          aai.appendLoopOnAppenders(e);
        } catch (InterruptedException ie) {
          break;
        }
      }

      addInfo("Worker thread will flush remaining events before exiting. ");
      for (E e : parent.blockingQueue) {
        aai.appendLoopOnAppenders(e);
      }

      aai.detachAndStopAllAppenders();
    }
}
```

### 小结

1. 同步写日志而且immediateFlush=true的配置下，性能最差。原因是写磁盘导致其他线程等待时间过长，虽然是顺序写，但是毕竟是持久化数据到磁盘。当然了，SSD能好一点。
2. 异步写在JVM突然crash的时候有丢失数据的风险，但是性能很高，原因在于避免了直接写磁盘带来的性能消耗。但是需要注意的是多线程操作同一个阻塞队列也会因为锁争用的问题影响性能。
    a. 不同的模块配置不同的日志文件和Appender，能减少锁争用的问题。
    b. 减少不必要的日志输出。
    c. 增加阻塞队列的大小，在`neverBlock=false`的情况下避免线程等待问题。
    d. 多个Appender（SiftingAppender），底层还是写同一个文件。好处是减少了多线程在阻塞队列上的锁竞争问题。

### SiftingAppender

SiftingAppender是logback根据mdc中的变量动态创建appender的代理，只要我们将一个线程号作为日志名分发器discriminator注入到SiftingAppender中，它就可以动态的为我们创建不同的appender，达到分线程的目的，配置方式举例如下：

```xml
<!-- 分线程输出源 -->
<appender name="frameworkthread" class="ch.qos.logback.classic.sift.SiftingAppender">
        <discriminator class="ThreadDiscriminator">
            <key>threadName</key>
        </discriminator>
        <sift>
            <appender name="FILE-${threadName}" class="ch.qos.logback.core.rolling.RollingFileAppender">
                <encoder>
                    <Encoding>UTF-8</Encoding>
                    <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS}[%c][%thread][%X{tradeNo}][%p]-%m%n</pattern>
                </encoder>
                <rollingPolicy
                    class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">        
                    <fileNamePattern>D:/test/threadlogs/${threadName}-%d{yyyy-MM-dd}.%i.log
                    </fileNamePattern>
                    <maxFileSize>100MB</maxFileSize>
                    <maxHistory>60</maxHistory>
                    <totalSizeCap>20GB</totalSizeCap>
                </rollingPolicy>
            </appender>
        </sift>
</appender> 
```    

### 参考资料

[http://logback.qos.ch/manual/appenders.html](http://logback.qos.ch/manual/appenders.html)
[一次logback多线程调优的经历](https://segmentfault.com/a/1190000016204970)








