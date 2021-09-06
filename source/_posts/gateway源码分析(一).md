---
title: Gateway源码分析（一）
date: 2021-07-25 11:16:39
categories: 
- 视频
tags:
- 视频
---

## 背景

当选用了SpringCloud Gateway作为视频项目的网关之后,出于学习的心态，看下视频网关是如何构造起来的。


## Spring WebFlux学习

阅读源码之前首先学习下WebFlux基础概念和相关常用的API

Spring WebFlux简单说是与Spring MVC类似的一门响应式异步非阻塞Web端控制框架。与Spring MVC有着相同功能的注解

1. 什么是响应式编程(Reactive)

官方解释：
>The term, “reactive,”  refers to programming models that are built around reacting to change
响应式编程是一种围绕对变化作出反应而构建的编程模型。

后面给出了例子：
>network components reacting to I/O events, UI controllers reacting to mouse events, and others. In that sense, non-blocking is reactive, because, instead of being blocked, we are now in the mode of reacting to notifications as operations complete or data becomes available.
网络组件对IO事件的响应，UI控制器对鼠标事件的响应。从这个意义上说，非阻塞也是响应式,因为我们现在也是在操作完成或者数据可用的时候作出响应的模式，而不是被阻塞。

2. 什么是背压（back pressure）

官方解析：
>Reactive Streams is a small spec (also adopted in Java 9) that defines the interaction between asynchronous components with back pressure. For example a data repository (acting as Publisher) can produce data that an HTTP server (acting as Subscriber) can then write to the response. The main purpose of Reactive Streams is to let the subscriber control how quickly or how slowly the publisher produces data.
响应式流是一个小小的规范，定义了带有背压异步组件交互。例如，数据仓库（作为发布者）产生数据使HTTP服务（作为订阅者）能响应。主要目的就是响应式能让订阅者控制发布者发布数据的快与慢。

其实上述说白了就是订阅者能够通过生产者需要多小数据，这样能够以免生产者无限量产生数据压垮订阅者。

3. SpringMVC与WebFlux选择
![webFlux&SpringMVC](https://docs.spring.io/spring-framework/docs/current/reference/html/images/spring-mvc-and-webflux-venn.png)

- 如果您有运行正常的Spring MVC应用程序，则无需更改。命令式编程是编写，理解和调试代码的最简单方法。您有最大的库选择空间，因为从历史上看，大多数库都是阻塞的。

- 如果您已经在选择无阻塞的Web堆栈，Spring WebFlux可以提供与该领域其他服务器相同的执行模型优势，还可以选择服务器（Netty，Tomcat，Jetty，Undertow和Servlet 3.1+容器），选择编程模型（带注释的控制器和功能性Web端点），以及选择反应式库（Reactor，RxJava或其他）。

- 如果您对与Java 8 lambda或Kotlin一起使用的轻量级功能性Web框架感兴趣，则可以使用Spring WebFlux功能性Web端点。对于要求较低复杂性的较小应用程序或微服务（可以受益于更高的透明度和控制）而言，这也是一个不错的选择。

- 在微服务架构中，您可以混合使用带有Spring MVC或Spring WebFlux控制器或带有Spring WebFlux功能端点的应用程序。在两个框架中都支持相同的基于注释的编程模型，这使得重用知识变得更加容易，同时还为正确的工作选择了正确的工具。

- 评估应用程序的一种简单方法是检查其依赖关系。如果您要使用阻塞性持久性API（JPA，JDBC）或网络API，则Spring MVC至少是通用体系结构的最佳选择。使用Reactor和RxJava在单独的线程上执行阻塞调用在技术上是可行的，但您不会充分利用非阻塞Web堆栈。

- 如果您的Spring MVC应用程序具有对远程服务的调用，请尝试使用active WebClient。您可以直接从Spring MVC控制器方法返回反应类型（Reactor，RxJava或其他）。每个呼叫的等待时间或呼叫之间的相互依赖性越大，好处就越明显。Spring MVC控制器也可以调用其他反应式组件。



4. WebFlux提供了两种模型
- 注解控制器（Annotated Controllers）：与Spring MVC有一致的注解，都是基于spring-web模型。一个显著的区别是，WebFlux也支持响应式@RequestBody参数。
- 函数终端（Functional Endpoints）：基于Lambda,轻量级和函数编程模型。提供了大量的方法来路由和处理请求。与注解控制器（Annotated Controllers）最大区别就是函数端模型能够负责请求的从头到尾的处理而不是只是通过声明注解然后回调。


### API

两个Publisher接口：Flux与Mono。两者区别在于Flux是代表多个元素的发布者，Mono是单个元素的发布者.

Mono 实现了 Publisher 接口，
```
empty()：创建一个不包含任何元素，只发布结束消息的序列。
just()：可以指定序列中包含的全部元素。创建出来的 Mono序列在发布这些元素之后会自动结束。
justOrEmpty()：从一个 Optional 对象或可能为 null 的对象中创建 Mono。只有 Optional 对象中包含值或对象不为 null 时，Mono 序列才产生对应的元素。
error(Throwable error)：创建一个只包含错误消息的序列。
never()：创建一个不包含任何消息通知的序列。
fromCallable()、fromCompletionStage()、fromFuture()、fromRunnable()和 fromSupplier()：分别从 Callable、CompletionStage、CompletableFuture、Runnable 和 Supplier 中创建 Mono。
delay(Duration duration)和 delayMillis(long duration)：创建一个 Mono 序列，在指定的延迟时间之后，产生数字 0 作为唯一值。
create()：通过 create()方法来使用 MonoSink 来创建 Mono。
```

Flux
```
just：可以指定序列中包含的全部元素。创建出来的 Flux 序列在发布这些元素之后会自动结束。
fromArray、fromIterable、fromStream：可以从一个数组、Iterable 对象或 Stream 对象中创建 Flux 对象。
empty()：创建一个不包含任何元素，只发布结束消息的序列,在响应式编程中，流的传递是基于元素的，empty表示没有任何元素，所以不会进行后续传递，需要用switchIfEmpty等处理
error(Throwable error)：创建一个只包含错误消息的序列。
never()：创建一个不包含任何消息通知的序列。使用示例：
range(int start, int count)：创建包含从 start 起始的 count 个数量的 Integer 对象的序列。
intervalMillis(long period)： interval()方法的作用相同，只不过该方法通过毫秒数来指定时间间隔和延迟时间。
create()：与 generate()方法的不同之处在于所使用的是 FluxSink 对象。FluxSink 支持同步和异步的消息产生，并且可以在一次调用中产生多个元素。
```

## 总结

这一章主要是简单记录下WebFlux的介绍以及简单说明下WebFlux相关的几个概念。WebFlux的官方文档还是相对比较完善的，里面都有挺详细的说明，我就不一一照搬到这里。下一章就开始阅读Gateway的代码，看下Gateway里面是如何使用WebFlux构建网关。



## 参考

- [webFlux官网](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#spring-webflux)


- [外行人都能看得懂的WebFlux](https://zhuanlan.zhihu.com/p/92460075)


- [最近学到的Lambda表达式基础知识](https://mp.weixin.qq.com/s?__biz=MzI4Njg5MDA5NA==&mid=2247485692&idx=1&sn=a6b3f040b13fa2324992b11a927e34dc&chksm=ebd749fddca0c0eb1b05c08ede7ee4a44699584fbc0c3449ec2cac7642fd13819470ec7f44d8&token=1024331018&lang=zh_CN#rd)


