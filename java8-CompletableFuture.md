---
layout: post
comments: true
title: java8中CompletableFuture详解
date: 2017-01-17 15:23:13
tags:
- java8
categories:
- java
---

本文转自：[使用Java 8的CompletableFuture实现函数式的回调](http://www.infoq.com/cn/articles/Functional-Style-Callbacks-Using-CompletableFuture)

### 背景

最近，在准备一个关于Java并行流相关的演讲时，我意识到“The Free Lunch is Over”（TFLiO）这篇经典的文章已经有超过十年的历史了。对于大多数程序员来说，这篇文章的广泛传播使他们第一次认识到持续四十年的处理器呈指数增长的趋势将要终结——实际上，它已经终结了。取而代之的是另外一种趋势，那就是在每个芯片上增加处理器的数量，按照Herb Sutter的话来讲，程序员必须要“从根本上转向并发”。

<!-- more -->

在TFLiO这篇文章中，Sutter观察到绝大多数的程序员“并没有深刻了解并发”，但是他接着说到“一旦了解之后，其实基于锁的编程并不会比OO难太多”。毫无疑问，第一句话是完全正确的，但是十年来，基于锁的并发体验并没有证明第二句话的正确性。幸好，Java程序员可以在很大程度上避免这样的验证，因为TFLiO发布的时候，Java 5也刚刚可用，它所提供的一些高层级的并发工具开始得到运用。借助它们，能够让Java开发人员避免细粒度地分析同步功能和关键的代码片段。

### 并发与并行

Java 5并发库主要关注于异步任务的处理，它采用了这样一种模式，producer线程创建任务并且利用阻塞队列将其传递给任务的consumer。这种模型在Java 7和8中进一步发展，并且开始支持另外一种风格的任务执行，那就是将任务的数据集分解为子集，每个子集都可以由独立且同质的子任务来负责处理。

这种风格的基础库也就是fork/join框架，它允许程序员规定数据集该如何进行分割，并且支持将子任务提交到默认的标准线程池中，也就是“通用的”ForkJoinPool。（在本文中，非全限定的类和接口名指的都是java.util.concurrent包中的类型。）在Java 8中，fork/join并行功能借助并行流的机制变得更加具有可用性。但是，不是所有的问题都适合这种风格的并行处理：所处理的元素必须是独立的，数据集要足够大，并且在并行加速方面，每个元素的处理成本要足够高，这样才能补偿建立fork/join框架所消耗的成本。

同时，Java 8在并行流方面的革新得到了广泛的关注，这导致大家忽略了并发库中一项新增的重要功能，那就是CompletableFuture<T>类。本文将会探讨CompletableFuture类，有一些系统会依赖于不同类型的异步执行任务，本文将会阐述该类为什么会对这种类型的系统如此重要，并介绍了它是如何补充fork/join风格的并行机制和并行流的。

### 页面渲染器

我们的起始点将是“Java Concurrency in Practice”（JCiP）一书中的样例，这个样例非常经典地阐述了Java 5中的并发工具类。在JCiP的第6.3节中，Brian Goetz探讨了如何开发一个Web页面的渲染器，对于每个页面来说，它的任务就是渲染文本，并下载和渲染图片。图片的下载会耗费较长的时间，在这段时间内CPU无事可做，只能等待。所以，在渲染页面时，一个很明显的策略就是首先初始化所有图片的下载，然后利用它们完成之前的这段时间渲染页面文本，最后渲染下载的图片。

在JCiP中，第一版本的页面渲染器使用了Future的理念，这个接口暴露了一些方法，允许客户端监控任务的执行进度，这里的任务是在一个不同的进程中执行的。在程序清单1中，Callable代表了下载页面中所有图片的任务，它被提交到了Executor中，然后返回一个Future对象，通过它就能询问下载任务的状态。当主线程渲染完页面的文本后，会调用Future.get方法，这个方法会一直阻塞直到所有下载的结果均可用为止，在本例中这个结果是以List<ImageData>的形式来表示的。这种方式一个很明显的缺点在于下载任务的粗粒度性，在所有的图片下载完成之前，我们一张图片也不能渲染。接下来，我们看一下如何缓解这个问题。

```java
public void renderPage(CharSequence source) {
List<ImageInfo> info = scanForImageInfo(source);
//创建Callable，它代表了下载所有的图片
final Callable<List<ImageData>> task = () ->
  info.stream()
    	.map(ImageInfo::downloadImage)
    	.collect(Collectors.toList());
	// 将下载任务提交到executor
	Future<List<ImageData>> images = executor.submit(task);
	// renderText(source);
try {
   // 获得所有下载的图片（在所有图片可用之前会一直阻塞）
   final List<ImageData> imageDatas = images.get();
   // 渲染图片
   imageDatas.forEach(this::renderImage);
} catch (InterruptedException e) {
   // 重新维护线程的中断状态
   Thread.currentThread().interrupt();
   // 我们不需要结果，所以取消任务
   images.cancel(true);
} catch (ExecutionException e) {
  throw launderThrowable(e.getCause()); }
}
```

为了让这个样例及其后面的变种易于管理，这里有一个前提条件：我们假设类型ImageInfo（简单来讲，就是一个URL）和ImageData（图片的二进制数据）以及方法scanForImageInfo、downloadImage、renderText、renderImage、launderThrowable和ImageInfo.downloadImage都已经存在了。实例变量executor是通过ExecutorService类型声明的并进行了恰当的初始化。在本文中，我将JCiP中最初的样例利用Java 8 lambda表达式和流进行了现代化。

在这段代码中，必须要等待所有下载都完成的原因在于，它使用Future接口来代表下载任务，作为异步执行的任务模型，它有很大的局限性。Future允许客户端询问任务执行的结果，如果必要的话，将会产生阻塞，另外还可以询问任务的状态，判断它已经完成还是被取消了。但是，Future本身并不能提供回调方法，假设能够这样做的话，当每个图片下载完成的时候，就能通知页面的渲染线程了。

程序清单2改善了之前样例中粒度较粗的问题，它将页面下载的任务提交到了CompletionService类中，这个类的poll和take方法会产生对应的Future实例，这些实例是按照任务完成的顺序排列的，而不是像程序清单1那样任务是按照提交的顺序处理的。在ExecutorCompletionService接口的平台实现中，为了实现该功能，每项任务都会包装在一个FutureTask中，FutureTask是Future的一个实现，它允许提供完成时的回调。Future的回调行为是在ExecutorCompletionService中创建的，完成的任务会封装到一个队列中，供客户端询问时使用。

```java
public void renderPage(CharSequence source) { 
   List<ImageInfo> info = scanForImageInfo(source); 
   CompletionService<ImageData> completionService = 
     new ExecutorCompletionService<>(executor); 

   // 将每个下载任务提交到completion service中
   info.forEach(imageInfo -> 
     completionService.submit(imageInfo::downloadImage)); 

   renderText(source); 

   // 当每个RunnableFuture可用时（并且我们也准备处理它的时候），
   // 将它们检索出来 
   for (int t = 0; t < info.size(); t++) { 
     Future<ImageData> imageFuture = completionService.take(); 
     renderImage(imageFuture.get()); 
   } 
}
```

**程序清单2：借助CompletionService，当图片可用时立即将其渲染出来（为了保持简洁性，省略掉了中断和错误处理的代码）**

### CompletableFuture简介

程序清单2代表了Java 5所能达到的水准，不过2014年之后，在Java中，编写异步系统的表现性得到了巨大的提升，这是通过引入CompletableFuture (CF)类实现的。这个类是Future的实现，它能够将回调放到与任务不同的线程中执行，也能将回调作为继续执行的同步函数，在与任务相同的线程中执行。它避免了传统回调最大的问题，那就是能够将控制流分离到不同的事件处理器中，而这是通过允许CF实例与回调方法进行组合形成新的CF来实现的。

作为样例，可以参考thenAccept方法，它接受一个 Consumer（用户提供的且没有返回值的函数）并返回一个新的CF。这个新CF所能达到的效果就是在最初CF完成时所得到的结果上，运用Consumer。与很多其他的CF方法类似，thenAccept有两个变种形式，在第一个中，Consumer会由通用fork/join池中的某一个线程来执行；在第二个中，它会由Executor中的某一个线程来负责执行，而Executor是我们在调用时提供的。这形成了三种重载形式：同步运行、在ForkJoinPool中异步运行以及在调用时所提供的线程池中异步运行，CompletableFuture中有近60个方法，上述的这三种重载形式占了绝大多数。

如下是thenAccept的一个样例，借助它重新实现了页面渲染器的功能：

```java
public void renderPage(CharSequence source) { 
        List<ImageInfo> info = scanForImageInfo(source); 
        info.forEach(imageInfo -> 
               CompletableFuture 
       		.supplyAsync(imageInfo::downloadImage) 
       		.thenAccept(this::renderImage)); 
        renderText(source); 
 }
```
*程序清单3：使用CompletableFuture来实现页面渲染功能*

尽管程序清单3比前面的形式更加简洁，但是我们需要练习一下才能更好地阅读它。工厂方法supplyAsync返回一个新的CF，它会在通用的 ForkJoinPool中运行指定的Supplier，完成时，Supplier的结果将会作为CF的结果。方法thenAccept会返回一个新的CF，它将会执行指定的Consumer，在本例中也就是渲染给定的图片，即supplyAsync方法所产生的CF的结果。

需要澄清的是，thenAccept并不是将CF与函数组合起来的唯一方式。将CF与函数组合起来可以接受如下的参数：

- 应用于CF操作结果的函数。此时，可以采用的方法包括：
    - thenCompose：针对返回值为CompletableFuture的函数；
    - thenApply：针对返回值为其他类型的函数；
    - thenAccept：针对返回值为void的函数；
- Runnable。通过thenRun方法，可以接受Runnable参数；
- 函数在处理的过程中，可能正常结束也可能异常退出。CF能够通过方法来分别组合这两种情况：
    - handle，针对接受一个值和一个Throwable，并有返回值的函数；
    - whenComplete，针对接受一个值和一个Throwable，并返回void的函数。

### 扩展页面渲染器

扩展该样例能够阐述CompletableFuture的其他特性。比如，当图片下载超时或失败时，我们想使用一个图标作为可见的指示器。CF暴露了一个名为get(long, TimeUnit)的方法，如果 CF在指定的时间内没有完成的话，将会抛出TimeoutException异常。我们可以使用它来定义一个函数，这个函数会将 ImageInfo转换为ImageData （程序清单4）。

```java
Function<ImageInfo, ImageData> infoToData = imageInfo -> { 
   CompletableFuture<ImageData> imageDataFuture = 
       CompletableFuture.supplyAsync(imageInfo::downloadImage, executor); 
   try { 
       return imageDataFuture.get(5, TimeUnit.SECONDS); 
   } catch (InterruptedException e) { 
       Thread.currentThread().interrupt(); 
       imageDataFuture.cancel(true); 
       return ImageData.createIcon(e); 
   } catch (ExecutionException e) { 
       throw launderThrowable(e.getCause()); 
   } catch (TimeoutException e) { 
       return ImageData.createIcon(e); 
   } 
}
```
*程序清单4：使用CompletableFuture.get来实现超时*

现在，页面可以通过连续调用infoToData来进行渲染。其中每个调用都会同步返回一个下载的图片，所以要并行下载的话，需要为它们各自创建一个新的异步任务。要实现这一功能，合适的工厂方法是CompletableFuture.runAsync()，它与supplyAsync类似，但是接受的参数是Runnable而不是Supplier：

```java
public void renderPage(CharSequence source) throws InterruptedException { 
       List<ImageInfo> info = scanForImageInfo(source); 
       info.forEach(imageInfo -> 
           CompletableFuture.runAsync(() -> 
               renderImage(infoToData.apply(imageInfo)), executor)); 
}
```

现在，我们考虑进一步的需求，当所有的请求完成或超时后，在页面上显示一个指示器，如果对应的所有CompletableFuture都从join方法中返回，就能表示出现了这种场景。静态方法allOf就是为这种需求而提供的，它能够创建一个返回值为空的CompletableFuture，当其所有的组件均完成时，它也会达到完成状态。（join方法通常用来返回某个CF的结果，为了查看allOf方法所组合起来的所有CF的结果，必须要对其进行单独地查询。）

```java
public void renderPage(CharSequence source) { 
       List<ImageInfo> info = scanForImageInfo(source); 
       CompletableFuture[] cfs = info.stream() 
           .map(ii -> CompletableFuture.runAsync( 
               () -> renderImage(mapper.apply(ii)), executor)) 
           .toArray(CompletableFuture[]::new); 
       CompletableFuture.allOf(cfs).join(); 
       renderImage(ImageData.createDoneIcon()); 
}
```

### 联合多个CompletableFuture

另外一组方法允许将多个CF联合在一起。我们已经看见过静态方法allOf，当其所有的组件均完成时，它就会处于完成状态，与之对应的方法也就是anyOf，返回值同样是void，当其任意一个组件完成时，它就会完成。除了这两个方法以外，这个组中其他的方法都是实例方法，它们能够将receiver按照某种方式与另外一个CF联合在一起，然后将结果传递到给定的函数中。

为了展现它们是如何运行的，我们扩展一下JCiP中的另一个例子，这是一个旅行预订的门户，我们将互相关联的订购过程记录在TripPlan对象中，它包含了总价以及所使用服务供应商的列表：

```java
interface TripPlan { 
       List<ServiceSupplier> getSuppliers(); 
       int getPrice(); 
       TripPlan combine(TripPlan); 
}
```   

ServiceSupplier（比如说某个航线或酒店）能够创建一个TripPlan：（当然，在现实中，ServiceSupplier.createPlan 将会接受参数，来反映对应的目的地、旅行等级等信息。）

```java
interface ServiceSupplier { 
    TripPlan createPlan(); 
    String getAlliance();       // 稍后使用 
}
```

为了选择最佳的旅行计划，需要查询每个服务供应商为我们的旅行所给定的规划，然后使用Comparator来对比每个规划结果，这个Comparator反映了我们的选择标准（在本例中，只是简单的选择价格最低者）：

```java
TripPlan selectBestTripPlan(List<ServiceSupplier> serviceList) { 
   List<CompletableFuture<TripPlan>> tripPlanFutures = serviceList.stream() 
     .map(svc -> CompletableFuture.supplyAsync(svc::createPlan, executor)) 
     .collect(toList()); 
    
   return tripPlanFutures.stream() 
     .min(Comparator.comparing(cf -> cf.join().getPrice())) 
     .get().join(); 
}
```

请注意中间的collect操作，在流处理里面，由于中间操作的延迟性（laziness of intermediate operation），它就变得非常必要了。如果没有它的话，流的终止操作（terminal operation）将会是min，它如果要执行的话，首先需要针对tripPlanFutures的每个元素执行join操作。如上述的代码所示，我们并没有这样做，终止操作是collect，它会将map操作所形成的CF值累积起来，这个过程中没有阻塞，因此允许底层的任务并发执行。

如果获取航线和酒店最佳旅行计划的任务是独立的，那么我们会希望它们能够同时初始化，就像前文所述的图片下载一样。要将两个CF按照这种方式联合在一起，我们需要使用CompletableFuture.thenCombine方法，它会并行地执行receiver以及所提供的CF，然后将它们的结果使用给定的函数组合起来（在这里，假设变量 airlines、hotels和（稍后使用的）cars都是以List<TravelService> 类型进行声明的，并且已经进行了恰当的初始化）：

```java
CompletableFuture 
       .supplyAsync(() -> selectBestTripPlan(airlines)) 
       .thenCombine( 
           CompletableFuture.supplyAsync(() -> selectBestTripPlan(hotels)), 
              TripPlan::combine); 
```

对这个样例进行扩展，我们将会学到更多的内容。假设每个服务供应商都属于某一个旅行联盟（travel alliance），通过String类型的属性alliance来表示。在独立订购完航线和酒店后，我们将会确定它们是否属于同一个联盟，如果是的话，那么只有属于同一联盟的租车服务，才在我们的考虑范围之内：

```java
private TripPlan addCarHire(TripPlan p) { 
       List<String> alliances = p.getSuppliers().stream() 
           .map(ServiceSupplier::getAlliance) 
           .distinct() 
           .collect(toList()); 
       if (alliances.size() == 1) { 
           return p.combine(selectBestTripPlan(cars, alliances.get(0))); 
       } else { 
           return p.combine(selectBestTripPlan(cars)); 
       } 
}
```  

selectBestTripPlan方法新的重载形式将会接受一个String类型作为偏爱的联盟，如果这个值存在的话，会使用它来过滤流中的服务：

```java
private TripPlan selectBestTripPlan( 
       List<ServiceSupplier> serviceList, String favoredAlliance) { 

       List<CompletableFuture<TripPlan>> tripPlanFutures = serviceList.stream() 
           .filter(ts -> 
               favoredAlliance == null || ts.getAlliance().equals(favoredAlliance)) 
           .map(svc -> CompletableFuture.supplyAsync(svc::createPlan, executor)) 
           .collect(toList()); 

       ... 
}
```

在本例中，选择租车服务的CF要依赖于航班和酒店预订任务组合所形成的CF。只有航班和酒店都预订之后，它才能完成。实现这种关联关系的方法就是thenCompose：

```java
CompletableFuture.supplyAsync(() -> selectBestTripPlan(airlines)) 
       .thenCombine( 
            CompletableFuture.supplyAsync(() -> selectBestTripPlan(hotels)), 
                TripPlan::combine) 
       .thenCompose(p -> CompletableFuture.supplyAsync(() -> addCarHire(p)));
```

预订航班和酒店联合形成的CF会执行，并且它的结果，也就是联合后的TripPlan，将会作为thenCompose函数参数的输入。结果形成的CF非常简洁地封装了不同异步服务之间的依赖关系。这段代码如此简洁的原因在于，尽管thenCompose联合了两个CF，但是它所返回的并不是我们预期的CompletableFuture<CompletableFuture<TripPlan>>，而是CompletableFuture<TripPlan>。所以，不管在创建CF的时候使用了多少层级的组合，它并不是嵌套的，而是扁平的，要获取它的结果只需要一步操作。这是monad“绑定（bind）”操作（这个名称来源于Haskell）的特性，CF就是这种monad，并且阐明了monad一些非常积极的特征：比如，在本例中，我们能够按照函数式的形式进行编写，如果没有这项功能的话，就需要在各个回调中非常繁琐地显式编写任务定义。

作者 Maurice Naftalin ，译者 张卫滨 发布于 2016年1月19日 | 欲知区块链、VR、TensorFlow等潮流技术和框架，请锁定QCon北京站！ 2  讨论
分享到： 微博 微信 Facebook Twitter 有道云笔记 邮件分享
稍后阅读我的阅读清单
最近，在准备一个关于Java并行流相关的演讲时，我意识到“The Free Lunch is Over”（TFLiO）这篇经典的文章已经有超过十年的历史了。对于大多数程序员来说，这篇文章的广泛传播使他们第一次认识到持续四十年的处理器呈指数增长的趋势将要终结——实际上，它已经终结了。取而代之的是另外一种趋势，那就是在每个芯片上增加处理器的数量，按照Herb Sutter的话来讲，程序员必须要“从根本上转向并发”。

在TFLiO这篇文章中，Sutter观察到绝大多数的程序员“并没有深刻了解并发”，但是他接着说到“一旦了解之后，其实基于锁的编程并不会比OO难太多”。毫无疑问，第一句话是完全正确的，但是十年来，基于锁的并发体验并没有证明第二句话的正确性。幸好，Java程序员可以在很大程度上避免这样的验证，因为TFLiO发布的时候，Java 5也刚刚可用，它所提供的一些高层级的并发工具开始得到运用。借助它们，能够让Java开发人员避免细粒度地分析同步功能和关键的代码片段。

并发与并行

Java 5并发库主要关注于异步任务的处理，它采用了这样一种模式，producer线程创建任务并且利用阻塞队列将其传递给任务的consumer。这种模型在Java 7和8中进一步发展，并且开始支持另外一种风格的任务执行，那就是将任务的数据集分解为子集，每个子集都可以由独立且同质的子任务来负责处理。

这种风格的基础库也就是fork/join框架，它允许程序员规定数据集该如何进行分割，并且支持将子任务提交到默认的标准线程池中，也就是“通用的”ForkJoinPool。（在本文中，非全限定的类和接口名指的都是java.util.concurrent包中的类型。）在Java 8中，fork/join并行功能借助并行流的机制变得更加具有可用性。但是，不是所有的问题都适合这种风格的并行处理：所处理的元素必须是独立的，数据集要足够大，并且在并行加速方面，每个元素的处理成本要足够高，这样才能补偿建立fork/join框架所消耗的成本。

同时，Java 8在并行流方面的革新得到了广泛的关注，这导致大家忽略了并发库中一项新增的重要功能，那就是CompletableFuture<T>类。本文将会探讨CompletableFuture类，有一些系统会依赖于不同类型的异步执行任务，本文将会阐述该类为什么会对这种类型的系统如此重要，并介绍了它是如何补充fork/join风格的并行机制和并行流的。

页面渲染器

我们的起始点将是“Java Concurrency in Practice”（JCiP）一书中的样例，这个样例非常经典地阐述了Java 5中的并发工具类。在JCiP的第6.3节中，Brian Goetz探讨了如何开发一个Web页面的渲染器，对于每个页面来说，它的任务就是渲染文本，并下载和渲染图片。图片的下载会耗费较长的时间，在这段时间内CPU无事可做，只能等待。所以，在渲染页面时，一个很明显的策略就是首先初始化所有图片的下载，然后利用它们完成之前的这段时间渲染页面文本，最后渲染下载的图片。

相关厂商内容

关于红包、SSD云盘等核心技术集锦！ 下一代 DB2更加突出 BLU Acceleration 一项关于提升云主机稳定性的实践 基于KVM架构的新一代云主机，如何让速度更快更稳定？ 看明略任鑫琦如何谈关系挖掘算法
在JCiP中，第一版本的页面渲染器使用了Future的理念，这个接口暴露了一些方法，允许客户端监控任务的执行进度，这里的任务是在一个不同的进程中执行的。在程序清单1中，Callable代表了下载页面中所有图片的任务，它被提交到了Executor中，然后返回一个Future对象，通过它就能询问下载任务的状态。当主线程渲染完页面的文本后，会调用Future.get方法，这个方法会一直阻塞直到所有下载的结果均可用为止，在本例中这个结果是以List<ImageData>的形式来表示的。这种方式一个很明显的缺点在于下载任务的粗粒度性，在所有的图片下载完成之前，我们一张图片也不能渲染。接下来，我们看一下如何缓解这个问题。

public void renderPage(CharSequence source) {
List<ImageInfo> info = scanForImageInfo(source);
//创建Callable，它代表了下载所有的图片
final Callable<List<ImageData>> task = () ->
  info.stream()
    	.map(ImageInfo::downloadImage)
    	.collect(Collectors.toList());
	// 将下载任务提交到executor
	Future<List<ImageData>> images = executor.submit(task);
	// renderText(source);
try {
   // 获得所有下载的图片（在所有图片可用之前会一直阻塞）
   final List<ImageData> imageDatas = images.get();
   // 渲染图片
   imageDatas.forEach(this::renderImage);
} catch (InterruptedException e) {
   // 重新维护线程的中断状态
   Thread.currentThread().interrupt();
   // 我们不需要结果，所以取消任务
   images.cancel(true);
} catch (ExecutionException e) {
  throw launderThrowable(e.getCause()); }
}
程序清单1：使用Future等待所有的图片下载完成

为了让这个样例及其后面的变种易于管理，这里有一个前提条件：我们假设类型ImageInfo（简单来讲，就是一个URL）和ImageData（图片的二进制数据）以及方法scanForImageInfo、downloadImage、renderText、renderImage、launderThrowable和ImageInfo.downloadImage都已经存在了。实例变量executor是通过ExecutorService类型声明的并进行了恰当的初始化。在本文中，我将JCiP中最初的样例利用Java 8 lambda表达式和流进行了现代化。

在这段代码中，必须要等待所有下载都完成的原因在于，它使用Future接口来代表下载任务，作为异步执行的任务模型，它有很大的局限性。Future允许客户端询问任务执行的结果，如果必要的话，将会产生阻塞，另外还可以询问任务的状态，判断它已经完成还是被取消了。但是，Future本身并不能提供回调方法，假设能够这样做的话，当每个图片下载完成的时候，就能通知页面的渲染线程了。

程序清单2改善了之前样例中粒度较粗的问题，它将页面下载的任务提交到了CompletionService类中，这个类的poll和take方法会产生对应的Future实例，这些实例是按照任务完成的顺序排列的，而不是像程序清单1那样任务是按照提交的顺序处理的。在ExecutorCompletionService接口的平台实现中，为了实现该功能，每项任务都会包装在一个FutureTask中，FutureTask是Future的一个实现，它允许提供完成时的回调。Future的回调行为是在ExecutorCompletionService中创建的，完成的任务会封装到一个队列中，供客户端询问时使用。

 public void renderPage(CharSequence source) { 
   List<ImageInfo> info = scanForImageInfo(source); 
   CompletionService<ImageData> completionService = 
     new ExecutorCompletionService<>(executor); 

   // 将每个下载任务提交到completion service中
   info.forEach(imageInfo -> 
     completionService.submit(imageInfo::downloadImage)); 

   renderText(source); 

   // 当每个RunnableFuture可用时（并且我们也准备处理它的时候），
   // 将它们检索出来 
   for (int t = 0; t < info.size(); t++) { 
     Future<ImageData> imageFuture = completionService.take(); 
     renderImage(imageFuture.get()); 
   } 
 }
程序清单2：借助CompletionService，当图片可用时立即将其渲染出来（为了保持简洁性，省略掉了中断和错误处理的代码）

CompletableFuture简介

程序清单2代表了Java 5所能达到的水准，不过2014年之后，在Java中，编写异步系统的表现性得到了巨大的提升，这是通过引入CompletableFuture (CF)类实现的。这个类是Future的实现，它能够将回调放到与任务不同的线程中执行，也能将回调作为继续执行的同步函数，在与任务相同的线程中执行。它避免了传统回调最大的问题，那就是能够将控制流分离到不同的事件处理器中，而这是通过允许CF实例与回调方法进行组合形成新的CF来实现的。

作为样例，可以参考thenAccept方法，它接受一个 Consumer（用户提供的且没有返回值的函数）并返回一个新的CF。这个新CF所能达到的效果就是在最初CF完成时所得到的结果上，运用Consumer。与很多其他的CF方法类似，thenAccept有两个变种形式，在第一个中，Consumer会由通用fork/join池中的某一个线程来执行；在第二个中，它会由Executor中的某一个线程来负责执行，而Executor是我们在调用时提供的。这形成了三种重载形式：同步运行、在ForkJoinPool中异步运行以及在调用时所提供的线程池中异步运行，CompletableFuture中有近60个方法，上述的这三种重载形式占了绝大多数。

如下是thenAccept的一个样例，借助它重新实现了页面渲染器的功能：

public void renderPage(CharSequence source) { 
        List<ImageInfo> info = scanForImageInfo(source); 
        info.forEach(imageInfo -> 
               CompletableFuture 
       		.supplyAsync(imageInfo::downloadImage) 
       		.thenAccept(this::renderImage)); 
        renderText(source); 
 }
程序清单3：使用CompletableFuture来实现页面渲染功能

尽管程序清单3比前面的形式更加简洁，但是我们需要练习一下才能更好地阅读它。工厂方法supplyAsync返回一个新的CF，它会在通用的 ForkJoinPool中运行指定的Supplier，完成时，Supplier的结果将会作为CF的结果。方法thenAccept会返回一个新的CF，它将会执行指定的Consumer，在本例中也就是渲染给定的图片，即supplyAsync方法所产生的CF的结果。

需要澄清的是，thenAccept并不是将CF与函数组合起来的唯一方式。将CF与函数组合起来可以接受如下的参数：

应用于CF操作结果的函数。此时，可以采用的方法包括：
thenCompose：针对返回值为CompletableFuture的函数；
thenApply：针对返回值为其他类型的函数；
thenAccept：针对返回值为void的函数；
Runnable。通过thenRun方法，可以接受Runnable参数；
函数在处理的过程中，可能正常结束也可能异常退出。CF能够通过方法来分别组合这两种情况：
handle，针对接受一个值和一个Throwable，并有返回值的函数；
whenComplete，针对接受一个值和一个Throwable，并返回void的函数。
扩展页面渲染器

扩展该样例能够阐述CompletableFuture的其他特性。比如，当图片下载超时或失败时，我们想使用一个图标作为可见的指示器。CF暴露了一个名为get(long, TimeUnit)的方法，如果 CF在指定的时间内没有完成的话，将会抛出TimeoutException异常。我们可以使用它来定义一个函数，这个函数会将 ImageInfo转换为ImageData （程序清单4）。

Function<ImageInfo, ImageData> infoToData = imageInfo -> { 
   CompletableFuture<ImageData> imageDataFuture = 
       CompletableFuture.supplyAsync(imageInfo::downloadImage, executor); 
   try { 
       return imageDataFuture.get(5, TimeUnit.SECONDS); 
   } catch (InterruptedException e) { 
       Thread.currentThread().interrupt(); 
       imageDataFuture.cancel(true); 
       return ImageData.createIcon(e); 
   } catch (ExecutionException e) { 
       throw launderThrowable(e.getCause()); 
   } catch (TimeoutException e) { 
       return ImageData.createIcon(e); 
   } 
}
程序清单4：使用CompletableFuture.get来实现超时

现在，页面可以通过连续调用infoToData来进行渲染。其中每个调用都会同步返回一个下载的图片，所以要并行下载的话，需要为它们各自创建一个新的异步任务。要实现这一功能，合适的工厂方法是CompletableFuture.runAsync()，它与supplyAsync类似，但是接受的参数是Runnable而不是Supplier：

public void renderPage(CharSequence source) throws InterruptedException { 
       List<ImageInfo> info = scanForImageInfo(source); 
       info.forEach(imageInfo -> 
           CompletableFuture.runAsync(() -> 
               renderImage(infoToData.apply(imageInfo)), executor)); 
}
现在，我们考虑进一步的需求，当所有的请求完成或超时后，在页面上显示一个指示器，如果对应的所有CompletableFuture都从join方法中返回，就能表示出现了这种场景。静态方法allOf就是为这种需求而提供的，它能够创建一个返回值为空的CompletableFuture，当其所有的组件均完成时，它也会达到完成状态。（join方法通常用来返回某个CF的结果，为了查看allOf方法所组合起来的所有CF的结果，必须要对其进行单独地查询。）

public void renderPage(CharSequence source) { 
       List<ImageInfo> info = scanForImageInfo(source); 
       CompletableFuture[] cfs = info.stream() 
           .map(ii -> CompletableFuture.runAsync( 
               () -> renderImage(mapper.apply(ii)), executor)) 
           .toArray(CompletableFuture[]::new); 
       CompletableFuture.allOf(cfs).join(); 
       renderImage(ImageData.createDoneIcon()); 
  }
联合多个CompletableFuture

另外一组方法允许将多个CF联合在一起。我们已经看见过静态方法allOf，当其所有的组件均完成时，它就会处于完成状态，与之对应的方法也就是anyOf，返回值同样是void，当其任意一个组件完成时，它就会完成。除了这两个方法以外，这个组中其他的方法都是实例方法，它们能够将receiver按照某种方式与另外一个CF联合在一起，然后将结果传递到给定的函数中。

为了展现它们是如何运行的，我们扩展一下JCiP中的另一个例子，这是一个旅行预订的门户，我们将互相关联的订购过程记录在TripPlan对象中，它包含了总价以及所使用服务供应商的列表：

 interface TripPlan { 
       List<ServiceSupplier> getSuppliers(); 
       int getPrice(); 
       TripPlan combine(TripPlan); 
   }
ServiceSupplier（比如说某个航线或酒店）能够创建一个TripPlan：（当然，在现实中，ServiceSupplier.createPlan 将会接受参数，来反映对应的目的地、旅行等级等信息。）

interface ServiceSupplier { 
    TripPlan createPlan(); 
    String getAlliance();       // 稍后使用 
}
为了选择最佳的旅行计划，需要查询每个服务供应商为我们的旅行所给定的规划，然后使用Comparator来对比每个规划结果，这个Comparator反映了我们的选择标准（在本例中，只是简单的选择价格最低者）：

TripPlan selectBestTripPlan(List<ServiceSupplier> serviceList) { 
   List<CompletableFuture<TripPlan>> tripPlanFutures = serviceList.stream() 
     .map(svc -> CompletableFuture.supplyAsync(svc::createPlan, executor)) 
     .collect(toList()); 
    
   return tripPlanFutures.stream() 
     .min(Comparator.comparing(cf -> cf.join().getPrice())) 
     .get().join(); 
}
请注意中间的collect操作，在流处理里面，由于中间操作的延迟性（laziness of intermediate operation），它就变得非常必要了。如果没有它的话，流的终止操作（terminal operation）将会是min，它如果要执行的话，首先需要针对tripPlanFutures的每个元素执行join操作。如上述的代码所示，我们并没有这样做，终止操作是collect，它会将map操作所形成的CF值累积起来，这个过程中没有阻塞，因此允许底层的任务并发执行。

如果获取航线和酒店最佳旅行计划的任务是独立的，那么我们会希望它们能够同时初始化，就像前文所述的图片下载一样。要将两个CF按照这种方式联合在一起，我们需要使用CompletableFuture.thenCombine方法，它会并行地执行receiver以及所提供的CF，然后将它们的结果使用给定的函数组合起来（在这里，假设变量 airlines、hotels和（稍后使用的）cars都是以List<TravelService> 类型进行声明的，并且已经进行了恰当的初始化）：

CompletableFuture 
       .supplyAsync(() -> selectBestTripPlan(airlines)) 
       .thenCombine( 
           CompletableFuture.supplyAsync(() -> selectBestTripPlan(hotels)), 
              TripPlan::combine); 
对这个样例进行扩展，我们将会学到更多的内容。假设每个服务供应商都属于某一个旅行联盟（travel alliance），通过String类型的属性alliance来表示。在独立订购完航线和酒店后，我们将会确定它们是否属于同一个联盟，如果是的话，那么只有属于同一联盟的租车服务，才在我们的考虑范围之内：

  private TripPlan addCarHire(TripPlan p) { 
       List<String> alliances = p.getSuppliers().stream() 
           .map(ServiceSupplier::getAlliance) 
           .distinct() 
           .collect(toList()); 
       if (alliances.size() == 1) { 
           return p.combine(selectBestTripPlan(cars, alliances.get(0))); 
       } else { 
           return p.combine(selectBestTripPlan(cars)); 
       } 
   }
selectBestTripPlan方法新的重载形式将会接受一个String类型作为偏爱的联盟，如果这个值存在的话，会使用它来过滤流中的服务：

  private TripPlan selectBestTripPlan( 
       List<ServiceSupplier> serviceList, String favoredAlliance) { 

       List<CompletableFuture<TripPlan>> tripPlanFutures = serviceList.stream() 
           .filter(ts -> 
               favoredAlliance == null || ts.getAlliance().equals(favoredAlliance)) 
           .map(svc -> CompletableFuture.supplyAsync(svc::createPlan, executor)) 
           .collect(toList()); 

       ... 
   }
在本例中，选择租车服务的CF要依赖于航班和酒店预订任务组合所形成的CF。只有航班和酒店都预订之后，它才能完成。实现这种关联关系的方法就是thenCompose：

CompletableFuture.supplyAsync(() -> selectBestTripPlan(airlines)) 
       .thenCombine( 
            CompletableFuture.supplyAsync(() -> selectBestTripPlan(hotels)), 
                TripPlan::combine) 
       .thenCompose(p -> CompletableFuture.supplyAsync(() -> addCarHire(p)));
预订航班和酒店联合形成的CF会执行，并且它的结果，也就是联合后的TripPlan，将会作为thenCompose函数参数的输入。结果形成的CF非常简洁地封装了不同异步服务之间的依赖关系。这段代码如此简洁的原因在于，尽管thenCompose联合了两个CF，但是它所返回的并不是我们预期的CompletableFuture<CompletableFuture<TripPlan>>，而是CompletableFuture<TripPlan>。所以，不管在创建CF的时候使用了多少层级的组合，它并不是嵌套的，而是扁平的，要获取它的结果只需要一步操作。这是monad“绑定（bind）”操作（这个名称来源于Haskell）的特性，CF就是这种monad，并且阐明了monad一些非常积极的特征：比如，在本例中，我们能够按照函数式的形式进行编写，如果没有这项功能的话，就需要在各个回调中非常繁琐地显式编写任务定义。

thenCombine方法只是将两个CF联合起来的方法之一，其他的方法包括：

- thenAcceptBoth：与thenCombine类似，但是它接受一个返回值为void的函数；
- runAfterBoth：接受一个 Runnable，在两个CF都完成后执行；
- applyToEither：接受一个一元函数（unary function），会将首先完成的CF的结果提供给它；
- acceptEither：与applyToEither类似，接受一个一元函数，但是结果为void；
- runAfterEither：接受一个Runnable，在其中一个CF完成后就执行。       

### 结论

我们不可能在一篇短文中，完整地阐述像CompletableFuture这样的API，但是我希望这里的样例能够让你对它所能实现的并发编程形式有一个直观印象。将CompletableFuture与其他CompletableFuture组合，以及与其他函数组合，能够为多项任务构建类似流水线的方案，这样能够控制同步和异步执行以及它们之间的依赖。你想更加详细了解的内容可能会包括异常处理、选择和配置executor的实际经验以及设计异步API所面临的挑战。

我希望已经解释清楚了Java 8所提供的两种异步编程风格之间的联系。在使用fork/join并行机制（包括并行流）的场景中，能够非常高效的将工作内容进行跨核心分发。但是，它的适用条件却非常有限：数据集很大并且能够高效地分割，对某个数据元素的操作与其他元素是（相对）独立的，这些操作的成本应该是比较高昂的，并且应该是CPU密集型的。如果这些条件无法满足的话，尤其是如果你的任务会花费很多时间阻塞在I/O或网络请求上的话，那么 CompletableFuture是更好的替代方案。作为Java程序员，我们非常幸运地有这样一个平台库，它将这些补充的方式集成在了一起。  
                         
查看英文原文：[Functional-Style Callbacks Using Java 8's CompletableFuture](http://www.infoq.com/articles/Functional-Style-Callbacks-Using-CompletableFuture)

### 更多参考

[Java CompletableFuture 详解](http://colobu.com/2016/02/29/Java-CompletableFuture/)


