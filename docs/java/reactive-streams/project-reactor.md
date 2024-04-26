# Project Reactor

ProjectReactor是Spring母公司借鉴RxJava开发的符合响应式编程规范Reactive Streams的响应式编程框架。

本质上还是基于回调的观察者模式实现。

应用：

+ Spring WebFlux
+ Spring cloud Gateway （依赖 Spring WebFlux）
+ RocketMQ

> 没有找到关于Project Reactor 源码分析的资料（本身资料就很少），之前没有响应式编程基础的话仅仅是看官方文档、书籍，写几个简单的Demo，对理解Project Reactor 没有多大帮助，很快会忘的一干二净。终究还是绕不过对源码的学习。
>
> 不过单元测试很多，测试代码比源码代码量都高。
>
> 响应式编程思想是一种趋势，在一些追求高性能的方面应用较为广泛，学习Project Reactor源码封装原理现在看来很有必要。

这里主要研究 reactor-core源码实现。



## Project Reactor 工作原理

### reactive-stream接口规范

首先还是需要清楚`reactive stream`规范定义了什么，定义了哪些固定的组件和流程。

`reactive stream` Java接口规范参考：[reactive-streams-jvm](https://github.com/reactive-streams/reactive-streams-jvm) ，包含`API规范` + `技术兼容性工具TCK`(是对各种实现的一致性测试套件)，规范都在REAME.md中给出了。

如果感觉英文看得累，中文翻译也很多，随便搜了个博客：[JVM平台上的响应式流（Reactive Streams）规范](https://www.cnblogs.com/lixinjie/p/a-reactive-streams-specification-on-jvm.html)

大纲（自己重新理解总结）：

+ 设计目标

  流处理方式（支持无限数量元素处理）、支持按序处理、支持异步处理元素（并行使用计算资源）、传递元素会看接收端处理能力（基于背压实现）

  > 关于背压（backpressure），感觉类似信号处理里的“信号反馈”的概念，下流订阅者向上游发布者发送信号控制上游数据生产速度。

+ API组件

  + Publisher 发布者/生产者

  + Subscriber 订阅者/消费者

  + Subscrition 订阅

    其实是数据源，定义流元素产生和发送。比如实现类 FluxRange$RangeSubscription、DefaultMonoSink。

  + Processor 处理者

    继承Publisher和Subscriber, 既是生产者又是消费者。

+ 接口规范

+ 同步与异步处理过程

+ 订阅者控制的队列边界



### 基本概念

+ 从实现上看依旧体现为观察者模式

+ 两种异步实现的局限性

  + 回调 - 回调地狱

    是可以从代码结构方面优化掉的。

  + Futures - 多任务编排不好用、get() 方法获取结果仍然会导致阻塞

+ 响应流的角色

  + 发布者 Publisher （数据生产）

    + Flux 

      意为“流”，能够发出 0 到 N 个元素的标准的 `Publisher<T>`。

    + Mono

      意为“单声道”，最多发出一个元素的特殊的 `Publisher<T>`。

  + 操作符(算子) Operations （数据包装）

    + publishOn
    + subscribeOn
    + ...

  + 订阅者 Subscriber （数据消费）

    没有订阅，发布者不会生成数据。有订阅，且发送了请求Request信号才会开始生产数据。

  + 调度器

    用于选择执行算子包装操作的线程环境，比如使用当前线程、新建的单线程、弹性线程池、固定大小线程池等。

  + Processors 

    既是一种特别的发布者（Publisher）又是一种订阅者（Subscriber）。不推荐使用。

+ 响应流的信号（事件）

  + 错误信号

  + 完成信号

  + 请求信号

    订阅者向发布者发送的信号。

+ 背压

  下流订阅者向上游发布者发送信号控制上游数据生产速度。

+ 冷发布者 & 热发布者

  冷发布者被订阅的时候才会生成数据，每一个订阅（subscription） 都重新生成数据，如果没有创建任何订阅（subscription），那么就不会生成数据。

  热发布者，即使没有订阅者它们也会生成数据， 同样是订阅后才发出数据。

  > 冷发布者只是强调为每个订阅都重新生成数据，但不确保为两个先后的订阅者生成一样的数据，刚开始容易错误理解为生成一样的数据。

+ 流程中的处理

  + 序列的创建 Generate Create

  + 元素处理 Handle

  + 异常处理

    生产者序列元素发送时的异常处理。

+ Reactor调试

  + 开启调试模式

    ```java
    Hooks.onOperatorDebug();
    ```

    这个方法能够开启调试模式，从而在抛出异常时打印出一些有用的信息。由于这是一种全局性的Hook，会影响到应用中所有的操作符，所以其带来的性能成本也是比较大的。

  + 记录流日志

    ```java
    //比如
    flux.log();
    ```

    这个方法打印序列元素传输过程中的一些信息。

  + checkpoint()

    这个方法用于定位问题出现在那个链上，相对于开启调试模式成本更低。

> 下面章节就研究上面提到的概念是怎么实现的。



### 常用接口方法

#### 核心接口

![](/home/arvin/mywork/java/async/async-program/doc/img/project-reactor-core-interfaces.png)

#### 发布者创建方法

这里主要说明自定义序列发布逻辑的发布者的创建方法。可以借助Sink接口（如MonoSink、FluxSink、SynchronousSink等等）定义序列发布逻辑，使用Sink接口的话Reactor其实已经将接口调用等都连接好了，开发者只需要定义数据生成方式即可。（sink：水槽）

+ generate()

  这是一种同步地， 逐个地产生值的方法，意味着 sink 是一个 SynchronousSink 而且其 next() 方法在每次回调的时候最多只能被调用一次。你也可以调用 error(Throwable) 或者 complete()，不过是可选的。

  generate() 传参的 `Consumer<? super FluxSink<T>> emitter` 会被重复调用，直到调用结束相关方法。

  > 这里的回调指的是onSubscribe()后request()会回调generate()中传入的Consumer#accept() 方法，在这个方法中只能最多调用一次next()方法。参考测试代码。

+ create()
  create 方法的生成方式既可以是同步， 也可以是异步的，并且还可以每次发出多个元素（即可以调用多次next()方法）。

  create() 传参的 `Consumer<? super FluxSink<T>> emitter` 只会被调用一次。需要在其方法内完成序列值发送到结束的流程。

+ push()

+ using()

+ defer()

  延迟创建一个Mono发布者（订阅后执行），每次订阅都会重新执行defer()中的Supplier方法。

  ```java
  @Test
  public void testDefer() {
      Mono<Date> m1 = Mono.just(new Date());
      //每次订阅都会重新执行defer()中的Supplier方法
      //可以通过defer将Mono.just()转成冷发布者
      Mono<Date> m2 = Mono.defer(() -> Mono.just(new Date()));
  
      m1.subscribe(System.out::println);	//Tue Sep 05 00:43:41 CST 2023
      m2.subscribe(System.out::println);	//Tue Sep 05 00:43:41 CST 2023
  
      ThreadUtil.sleep(5000);
      m1.subscribe(System.out::println);	//Tue Sep 05 00:43:41 CST 2023
      m2.subscribe(System.out::println);	//Tue Sep 05 00:43:46 CST 2023
  }
  ```

测试Demo: PublisherSelfDefinedSequenceTest.java

#### 订阅者序列元素传递回调方法 onXxx

+ onSubscribe(Subscription): void

  调用Publisher.subscribe(Subscriber)后会被回调，负责调用Subscription.request(long) 来请求发布者发送数据。

+ onNext(T): void

  发布者发送数据元素的处理回调，传参带过来的就是元素，或者称为事件。

+ onComplete(): void

  正常结束流元素传输，手动主动结束或者元素传输完毕。

+ onError(Throwable): void

  异常结束流元素传输，比如处理过程中抛出异常需要结束流处理。

#### 请求发布后回调监控方法 doOnXxx 

doOnXxx方法都是属于发布者的（看源码可以发现用装饰器模式包装了原发布者），与上面订阅者的onXxx方法是对应的。

```java
.doOnSubscribe(s -> System.out.println("doOnSubscribe ..."))
.doOnRequest(l -> System.out.println("doOnRequest ... , request count: " + l))
//注意是在map等方法之后执行的（因为map修饰的Publisher,doOnEach是Publisher修饰后执行），每个信号（保存有元素对象）发出时调用
.doOnEach(stringSignal -> System.out.println("doOnEach: " + stringSignal.get()))
.doOnNext(str -> System.out.println("doOnNext ... , str: " + str))
//异常发生时调用，然后会异常终止
.doOnError(e -> System.out.println("doOnError ... , error msg: " + e.getMessage()))
//主动取消时调用，然后正常结束
.doOnCancel(() -> System.out.println("doOnComplete ..."))
//正常完成时调用
.doOnComplete(() -> System.out.println("doOnComplete ..."))
//终止时调用，无论是正常还是异常
.doOnTerminate(() -> System.out.println("doOnTerminate ..."))
```

执行顺序（和onXxx回调方法顺序一致）：

```
doOnSubscribe ...
doOnRequest ... , request count: 9223372036854775807
map: 1
doOnEach: 1
doOnNext ... , str: 1
map: 2
doOnEach: null
doOnError ... , error msg: exception for 2
doOnTerminate ...
```

#### 发布者异常处理 onErrorXxx

+ 静态缺省值 onErrorReturn

  异常会终止序列继续发送。

+ 异常恢复 onErrorResume

  切换到一个新的发布者。在新的发布者中可以设置候补序列、也可以捕获异常然后记录日志重新抛出等等。

+ 跳过错误元素继续执行 onErrorContinue

+ 替换错误内容 onErrorMap

  和想象的不一样，从传参可以看到是要替换成新的Throwable错误对象：`Function<? super Throwable, ? extends Throwable> mapper`。

+ try-catch捕获 using、doFinally

+ 重试 retry

测试Demo: PublisherOnErrorTest.java

#### 信号

##### 信号类型 SignalType

#### 操作符（算子）与 全局钩子

#### 异步处理

#### 背压

+ 自行实现

  借助订阅者onNext() request() 方法实现处理完一批再请求处理下一批。

+ limitRate()

#### 冷发布者 & 热发布者

又称冷序列、热序列。

just()生成是一个热序列，它直接在**组装期**就拿到数据，如果之后有谁订阅它，就**重新发送数据**给订阅者。

Reactor 中多数其他的热发布者是扩展自Processor 的。

创建热发布者的方式：

+ UnicastProcessor

  不过从3.5之后已经废弃。

+ 使用share() replay() 将冷发布者转成热发布者

  + Sinks.Many
  + publish() 及其变种方法
  + share() 及其变种方法

TODO: 热序列源码实现原理。

测试Demo: HotAndColdPublisherTest.java



### 从Demo看源码工作流程

当前分析源码版本：3.5.8。

测试Demo代码位于 `async-reactive`。

从几个测试看Reactor源码工作原理：`doc/project-reactor-core-workflow.drawio`。



## 参考资料

> 资料还是要结合源码看，理解不透看和没看是一样的。

+ https://github.com/reactor/reactor-core

+ [Reactor 3 Reference Guide](https://projectreactor.io/docs/core/release/reference/)

  [中文翻译](https://www.cnblogs.com/crazymakercircle/p/14292098.html) (版本可能有点旧，但是差别不大)

+ [由表及里学 ProjectReactor](http://blog.yanick.site/2018/06/26/java/spring/projectreactor/) 

+ 入门博客（看官方文档可能把握不住重点）

  + [Project Reactor学习](https://zhuanlan.zhihu.com/p/35964846)
  + [Spring响应式编程](https://blog.csdn.net/get_set/category_7484996.html)

