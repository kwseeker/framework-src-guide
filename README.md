# 后端技术栈框架源码分析

这里主要分析Java后端技术栈常用框架的源码（后面也会考虑加入Go、Rust技术栈），从整体到局部探究内部工作原理，并输出可视化流程图。

源码分析输出主要为`Drawio`流程图（`.drawio`文件）和 `Markdown`文档，以流程图为主（主要展示框架数据结构和主流程），`Markdown`文档作为补充，详细内容参考[docs](./docs)。

这里的流程图不是常规流程图，实际是借鉴的时序图的编排方式，另外还添加了重要类的`UML`图（UML：体现数据结构 ，流程：体现算法， **UML比流程更重要**，早期画的一些流程图没配UML图，后面有空的话会加上）。

个人认为读源码（不一定是框架源码）是每个程序员都应该养成的习惯，尤其是在做架构设计或方案设计时，如果对某个业务不是很熟悉，第一步应该做的是检索一些开源实现，快速过一遍源码，梳理业务开发中需要考虑哪些设计要点、有哪些实现方式、不同实现方式的优缺点等等，见到很多优秀的程序员都是这么做的，兼顾效率与质量，一些框架也会互相借鉴优点。

读完源码后练手：有些东西自己不写一遍总会感觉手生或会忽略一些设计细节，抄别人的源码感觉又比较无聊，个人感觉有3种方式可以练手：1. 提取公共组件（比如配置文件解析、线程池管理、Netty通信组件、生产者消费者模式等）；2. 开发异构框架（可能Java有的框架，Go、Rust中没有，可以使用Go、Rust技术栈参考既有流程重新开发）；3. 参与开源项目（有什么好的拓展想法都可以提issue并实现、或者取已有的issue修复）。

**目录**：

* [Java](#Java)
  + [SDK](#SDK)
  + [网关](#网关)
  + [服务调用](#服务调用)
  + [负载均衡](#负载均衡)
  + [消息队列](#消息队列)
  + [作业调度](#作业调度)
  + [服务器](#服务器)
  + [Web相关](#Web相关)
  + [响应式](#响应式)
  + [微服务](#微服务-1)
  + [服务网格](#服务网格)
  + [注册中心/配置中心](#注册中心/配置中心)
  + [分布式协调](#分布式协调)
  + [服务保障](#服务保障)
  + [链路追踪](#链路追踪)
  + [监控系统](#监控系统)
  + [安全框架](#安全框架)
  + [ORM框架](#ORM框架)
  + [数据库相关](#数据库相关)
  + [分布式事务](#分布式事务)
  + [搜索引擎](#搜索引擎)
  + [规则引擎](#规则引擎)
  + [工作流引擎](#工作流引擎)
  + [缓存](#缓存)
  + [分布式](#分布式)
  + [命令行](#命令行)
  + [日志](#日志)
  + [序列化](#序列化)
  + [工具类](#工具类)
  + [语法解析器](#语法解析器)
  + [表达式引擎](#表达式引擎)
  + [自动化测试](#自动化测试)
* [AI](#AI)
  * [接入工具](#接入工具)
  * [产品](#产品)
* [Go](#Go)
  + [SDK](#SDK-1)
  + [服务调用](#服务调用-1)
  + [微服务](#微服务-1)
  + [注册中心/配置中心](#注册中心/配置中心-1)
  + [数据库相关](#数据库相关-1)
  + [分布式事务](#分布式事务-1)
  + [音视频](#音视频)
* [Rust](#Rust)
  + [异步框架](#异步框架)
  + [其他](#其他)
* [为何读源码](#为何读源码)
* [资源](#资源)
  + [书籍](#书籍)

> 后面列举的框架并非都分析过源码（心有余而力不足），有些只是计划，已经分析过源码的都有流程图链接。
>
> 有些分析流程图之前记录在其他仓库后续会转移到这里，慢慢补充吧。
>
> 学过的东西不使用过一段事件都只能留个大概的印象，这个无法避免，为此打算后面再罗列一些要点辅助快速回忆流程。



## 1 Java

### 1.1 SDK

> JDK 一些类的源码少于2000行的一般没必要画流程图，数据结构和逻辑不画图也能梳理清楚，时间久了忘记了重新看也花不了多长时间。

+ **JVM**

  + **HotSpot**

    已经搭建好了源码调试环境（参考仓库：jvm-debug），但是看JVM源码需要很多操作系统系统编程、内核方面的知识，不是短时间就能把整个流程看明白的，所以暂停了。还是碰到问题先针对性地看对应的代码块吧，更简单些。

    源码流程图：

    + [jvm8-hotspot.drawio](docs/java/jvm/jvm8-hotspot.drawio) (未完成)

    + [java-exec-process.drawio](docs/java/jvm/java-exec-process.drawio) (流程概要)

    + [java-exec-process.drawio.png](docs/java/jvm/java-exec-process.drawio.png)

      里面有些错误，TODO 修改。

    > 不过Github上有一些简单JVM开源实现，可以参考下以加深对JVM的理解，这里只分析星数最高的一个实现[mini-jvm](#mini-jvm)。

  + [**mini-jvm**](docs/java/jvm/mini-jvm.md)

    代码量较小，很容易理解。不过也仅仅只包含一些很核心的功能实现（.class文件解析、类加载、实例化、Native方法、方法调用、异常处理），像GC、多线程、双亲委托、JIT优化等等都没有。

    其实感觉对深入理解JVM帮助不大，但是比较适合初学者，比单纯啃几本干巴巴的JVM书籍好太多了。

    Github Repo: 

    + [guxingke/mini-jvm](https://github.com/guxingke/mini-jvm)
    
    源码流程图：
    
    + [mini-jvm.drawio](docs/java/jvm/mini-jvm.drawio)
    + [mini-jvm.drawio.png](docs/java/jvm/mini-jvm.drawio.png)

+ **并发**

  + **AbstractQueuedSynchronizer**

    源码流程图：

    + [AbstractQueuedSynchronizer.drawio](docs/java/jdk/concurrent/AbstractQueuedSynchronizer.drawio)

  + **ReentrantLock**

    源码流程图：

    + [ReentrantLock.drawio](docs/java/jdk/concurrent/ReentrantLock.drawio)
    + [ReentrantLock.drawio.png](docs/java/jdk/concurrent/ReentrantLock.drawio.png)

  + [**CompletableFuture**](docs/java/jdk/concurrent/CompletableFuture.md)

    源码流程图：

    + [CompletableFuture.drawio](docs/java/jdk/concurrent/CompletableFuture.drawio)
    + [CompletableFuture.drawio.png](docs/java/jdk/concurrent/CompletableFuture.drawio.png)

  + **ThreadPoolExecutor**

  + [**ForkJoinPool**](docs/java/jdk/concurrent/ForkJoinPool.md)

    源码流程图：

    + [ForkjoinPool.drawio](docs/java/jdk/concurrent/ForkJoinPool.drawio)
    + [ForkJoinPool.drawio.png](docs/java/jdk/concurrent/ForkJoinPool.drawio.png)

  + [**ThreadLocal**](docs/java/jdk/concurrent/ThreadLocal.md)

    源码流程图：

    + [ThreadLocal.drawio](docs/java/jdk/concurrent/ThreadLocal.drawio)
    + [ThreadLocal.drawio.png](docs/java/jdk/concurrent/ThreadLocal.drawio.png)

    关键问题：

    + ThreadLocal 为何会内存泄漏？

      要点：不使用remove() 主动清理、两条引用链（使用线程池核心线程的话从这个线程对象开始到ThreadLocalMap Entry 到   value 对象的强引用始终存在，如果后续没有读写操作也不会通过检查清除过期的值，那么只要线程存在这个值就不会回收）。

    + 弱引用是怎么在一定程度上解决内存泄漏问题的？

      要点：弱引用 + 读写时检查清除过期Entry。

      > key 是 ThreadLocal 对象的弱引用，ThreadLocal 被回收后，即使之前没有调用 remove() , key 在下次GC 也会被回收，当后续读写时发现 entry 不为null 但是 key 为 null，会执行 expungeStateEntry 或 replaceStateEntry 清理过期 entry。

+ **容器类**

  之前都是写的 Markdown文档，TODO 补充流程图。

+ **IO**

  + **NIO**

    源码流程图：

    + [java-nio.drawio](docs/java/jdk/io/java-nio.drawio)
    + [java-nio.drawio.png](docs/java/jdk/io/java-nio.drawio.png)
    + [java-nio-overview.png](docs/java/jdk/io/java-nio-overview.png)（详细缩略图）

    > 注意即使是使用NIO这种非阻塞模型，也不意味着你的服务一定是非阻塞的。因为NIO模型仅仅是为了支持使用较少的线程处理大量并发请求，它说的非阻塞是说借助select线程避免当通道没有数据时阻塞**Channel读写线程**执行（这个Channel没有数据可以去处理其他Channel就绪的数据）；业务服务是否阻塞还取决于**业务线程**读取Channel读线程返回的值的方式，比如Channel线程读取到请求数据后还需要交给业务线程处理，如果业务线程还是使用 FutureTask 这种方式读取返回值那就是阻塞的，如果使用 Callback Promise 这些方式读取返回值就是非阻塞的。
    >
    > 总之 **NIO 非阻塞是说的是不会阻塞 Channel 读写线程，响应式Web框架中服务非阻塞则是说服务中所有线程读写都不会阻塞，包括 Channel读写线程**。看响应式框架（比如Reactor ）
    >
    > Tomcat 8.x 默认使用 NIO 处理请求但是不妨碍它 “读请求体”、“写响应头”、“写响应体” 依然是阻塞的。

+ **Reference**

  源码流程图：

  + [java-reference.drawio](docs/java/jdk/reference/java-reference.drawio)
  + [java-reference.drawio.png](docs/java/jdk/reference/java-reference.drawio.png)

### 1.2 网关

+ **[Spring Cloud Gateway](docs/java/gateway/SpringCloudGateway原理.md)**

  源码流程图：

  + [spring-cloud-gateway.drawio](docs/java/gateway/spring-cloud-gateway.drawio)
  + [spring-cloud-gateway.drawio.png](docs/java/gateway/spring-cloud-gateway.drawio.png)

  原理简述：

  本质是一个 WebFlux 应用，内部处理逻辑和 Spring MVC 也有点类似，主要也是定义一组请求处理映射（用于定义路由处理，路由包括一组断言和过滤器），通过断言进行请求与路由的匹配，通过路由的过滤器对请求进行处理（比如：修改路径、进行转发等）。

+ **Zuul**

### 1.3 服务调用

+ **Dubbo**

  源码代码量很大，部分组件拆开。

  + **Dubbo主流程**

    源码流程图：

    + [dubbo3.drawio](docs/java/dubbo/dubbo3.drawio) (尚未完成)

  + [Dubbo SPI](docs/java/dubbo/dubbo-spi.md)

    源码流程图：

    + [dubbo3-spi.drawio](docs/java/dubbo/dubbo3-spi.drawio)
    + [dubbo3-spi.png](docs/java/dubbo/imgs/dubbo3-spi.png)

+ **Feign**

+ **Grpc**

+ **Thrift**

### 1.4 负载均衡

+ **Ribbon**

  源码流程图：

  + [ribbon.drawio](docs/java/ribbon/ribbon.drawio)

+ **Spring Cloud LoadBalancer**

  源码流程图：

  + [spring-cloud-loadbalancer.drawio](docs/java/spring-cloud-loadbalancer/spring-cloud-loadbalancer.drawio)
  + [spring-cloud-loadbalancer.drawio.png](docs/java/spring-cloud-loadbalancer/spring-cloud-loadbalancer.drawio.png)

### 1.5 消息队列

+ **[Disruptor](docs/java/message-queue/disruptor/disruptor.md)**

  线程间高性能低延迟消息传递框架， 4.x 和 3.x 源码变化还是挺大的。

  源码流程图(4.x)：

  + [disruptor.drawio](docs/java/message-queue/disruptor/disruptor.drawio)
  + [disruptor.drawio.png](docs/java/message-queue/imgs/disruptor.drawio.png)

+ **Kafka**

+ **[RocketMQ](docs/java/message-queue/rocketmq/rocketmq.md)**

  源码流程图：

  + [rocketmq.drawio](docs/java/message-queue/rocketmq/rocketmq.drawio)
    + NameServer: [rocketmq-NameServer.drawio.png](docs/java/message-queue/imgs/rocketmq-NameServer.drawio.png)
    + Broker:  [rocketmq-Broker.drawio.png](docs/java/message-queue/imgs/rocketmq-Broker.drawio.png)
    + Producer: [rocketmq-Producer.drawio.png](docs/java/message-queue/imgs/rocketmq-Producer.drawio.png)
    + Consumer: [rocketmq-Consumer.drawio.png](docs/java/message-queue/imgs/rocketmq-Consumer.drawio.png)
  + [rocketmq-messagestore.drawio](docs/java/message-queue/rocketmq/rocketmq-messagestore.drawio) (消息存储服务原理)
    + MessageStore: [rocketmq-messagestore.drawio.png](docs/java/message-queue/imgs/rocketmq-messgestore.drawio.png)
  + [transaction-message.drawio](docs/java/message-queue/rocketmq/transaction-message.drawio) (事务消息原理)
  + [transaction-message.drawio.png](docs/java/message-queue/imgs/transaction-message.drawio.png)

  关键问题（下面问题详细参考文档 [rocketmq.md](docs/java/message-queue/rocketmq/rocketmq.md)）：

  + 同步发送&消费原理

  + 顺序消息原理

    这个问题实现原理其实挺复杂的（整个流程涉及的源码很多），网上基本没有找到能解释清楚且全面的，个人调试源码很久得出下面结论：

    需要满足四点：**顺序生产**（一组顺序消息需要使用一个生产者实例生产，以避免乱序写入消息队列）、**写入同一消息队列**、**同一消息队列的消息总是发给同一个消费者实例**（由消费者负载均衡策略保证）、**顺序消费**（顺序消息支持单消费者多线程并发消费，通过锁保证消费顺序性），简单几句话说不清，参考文档。

  + 消息消费失败重试&死信队列机制

    直接参考流程图，这两个问题不算复杂。

  + RocketMQ 消息消费到底是推还是拉

    其实是**推拉结合**的方式，流程简述就是：消费者向Broker发送拉取请求（Netty请求），Broker收到消息后会判断队列中是否有消息，有则批量推送给消费者，没有则将请求缓存起来，待有消息后再推送给被缓存的请求来源消费者，并清除缓存的请求； Netty客户端监听到 Broker 响应后消费消息，消费完成后再发出新的拉取请求。

    这样既可以避免拉模式的**无间隔轮询空转**或**有间隔轮询的处理延迟**问题，也能避免推模式推送速度大于消费者消费速度导致消息在客户端**堆积**的问题。

+ **RabbitMQ**

### 1.6 作业调度

+ **Elastic-Job**

+ **XXL-Job**

  源码流程图：

  + [xxl-job.drawio](docs/java/job-schedule/xxl-job.drawio)

  + [xxl-job-executor.drawio.png](docs/java/job-schedule/xxl-job-executor.drawio.png)

    任务执行器工作流程。

  + [xxl-job-admin.drawio.png](docs/java/job-schedule/xxl-job-admin.drawio.png)

    任务管理器工作流程。

### 1.7 服务器

+ **Jetty**

+ **Netty**

  源码流程图：

  + [netty.drawio](docs/java/netty/netty.drawio)

  + [netty.drawio.png](docs/java/netty/netty.drawio.png)

    关于 Netty Pipeline 中各种 ChannelHandler 的调用顺序、触发方式解析参考后面 RocketMQ 流程图。
    
    > 好久之前画的图，UML部分数据结构依赖关系不清晰，流程部分一次完整的请求流程处理过程也不是很清晰，需要重画。

  细节问题分析：

  + [Netty如何通过参数配置支持三种Reactor模型](docs/java/netty/netty-reactor.md)

    实际多用Reactor主从多线程模型，RocketMQ 中 NettyRemotingServer 就是Reactor主从多线程模型。

  + Netty的串行无锁化设计体现在哪里

  + [Netty解码器中处理TCP粘包、拆包的实现原理以及反序列化处理流程](docs/java/netty/netty-codec.md)

    + RocketMQ 自定义解码器粘包、拆包处理及反序列化处理流程

      RocketMQ 自定义解码器 NettyDecoder 基于 LengthFieldBaseFrameDecoder 实现，根据报文长度提取消息主体。

    + 基于 WebSocket 协议的粘包、拆包处理及反序列化处理流程

      使用Netty不一定需要自己定义通信协议和实现编解码器，Netty默认支持通过 HTTP协议、WebSocket协议通信，本人接触到的公司项目就是用的 WebSocket 协议。

  **开源框架是怎么用Netty的**：

  + RocketMQ

    已将 RocketMQ 中通过 Netty 通信的代码抽离到了仓库 SpringBoot-Labs/netty/**netty-rocketmq** 模块，去除了业务处理，共2K行代码，可以作为模板用在自己的项目中；

    实现的功能包括：Reactor主从多线程模型、同步或异步传输、连接空闲超时断线、针对连接的请求信号量限流、断线重连、消息重发，另外还有Socks代理、TLS支持、监控告警（写缓存水位线监控告警）不过这些暂时没有提取出来，还有可以借鉴《Netty权威指南》支持 Protobuf 编解码、HTTP协议、WebSocket协议。

+ **Reactor-Netty**

  基于 Project Reactor 对 Netty 的响应式封装。

  源码流程图：

  + [reactor-netty.drawio](docs/java/reactive-streams/reactor-netty.drawio)
  + [reactor-netty.drawio.png](docs/java/reactive-streams/reactor-netty.drawio.png)

+ **Tomcat**

  > 之前看过源码但是没有画图时间久了就细节全忘记了，TODO 有空重看下并画下图。
  
  原理简述：
  1、创建一个 Acceptor 线程来接收用户连接，接收到之后扔到 events queue 队列里面，默认情况下只有一个线程来接收；
  2、创建 Poller 线程，数量 <= 2；Poller 对象是 NIO 的核心，在Poller中，维护了一个 Selector 对象；当 Poller 从队列中取出 Socket 后，注册到该 Selector 中；然后通过遍历 Selector，找出其中可读的 Socket，然后扔到线程池中处理相应请求，这就是典型的NIO多路复用模型。
  3、扔到线程池中的 SocketProcessorBase 处理请求。
  
  > 不过 Tomcat 虽然使用了 NIO 模型，只是优化了请求处理流程中部分操作由阻塞转成了非阻塞，比如 “读请求头”、“等待下一个请求”、“SSL握手”；“读请求体”、“写响应头”、“写响应体”依然是阻塞的（有种说法是需要遵循传统的接口规范以致于无法对所有操作进行非阻塞改写）。 
  >
  > 参考：[Connector Comparsion](https://tomcat.apache.org/tomcat-8.0-doc/config/http.html#/Connector_Comparison)

+ **Vertx**

  基于 Netty 实现的事件驱动服务器。

### 1.8 Web相关

+ **Spring**

  + **基于注解的应用上下文**

    这里主要分析 IOC 和 Spring Bean 生命周期（包括优雅关闭）。

    源码流程图：

    + [spring-context.drawio](docs/java/spring/spring-context.drawio)

      很久之前画的，图有点丑，关于Spring Bean 实例化和初始化流程，参考后面的 [spring-beans-factorybean.drawio](docs/java/spring/spring-beans-factorybean.drawio)，更详细。

    + [spring-context.drawio.png](docs/java/spring/imgs/spring-context.drawio.png)

  + **切面**

    + **Spring CgLib and ASM**

      Spring 代理实现基础，Spring 没有直接引入CgLib和ASM的依赖，而是将它们的部分源码直接迁移到了Spring核心，细节还是挺多的，比较枯燥，流程图省略了很多细节。

      + [spring-cglib-and-asm.drawio](docs/java/spring/spring-cglib-and-asm.drawio)
      + [spring-cglib-and-asm.drawio.png](docs/java/spring/imgs/spring-cglib-and-asm.drawio.png)

    + **AOP**

      关键是生成代理对象，像Spring声明式事务以及Seata数据源代理都注册了一个继承`AbstractAutoProxyCreator`的Bean，即代理构造器，其本质是一个BeanPostProcessor，作用是在其他Bean初始化后判断是否需要为Bean生成代理对象，是的话查找所有对应的 Advice、Advisor创建代理对象。  
      
      参考声明式事务源码分析或者Seata数据源代理源码分析（Seata这部分工作流程图单独提出来了）。

  + **事务**

    源码流程图：

    + [spring-transaction-5.3.27.drawio](docs/java/spring/spring-transaction-5.3.27.drawio) （编程式事务源码分析）

    + [spring-transaction-5.3.27.drawio.png](docs/java/spring/imgs/spring-transaction-5.3.27.drawio.png)（编程式事务流程图）

    + [spring-transaction.drawio](docs/java/spring/spring-transaction.drawio) (声明式事务源码分析，sheet2)

      不完整（事务处理部分逻辑没画），声明式事务处理实现和编程式有点小区别但差别也不是很大，知道事务操作入口是 `TransactionInterceptor` 就行了，懒得再画一遍了。

    + [spring-transaction-declarative.drawio.png](docs/java/spring/imgs/spring-transaction-declarative.drawio.png) （声明式事务流程图）

    注意：

    如果有两个操作 a()，b()，a() 调用 b()，a() 在事务中执行，**b() 方法无论是新建事务执行还是普通执行，如果产生异常没有被捕获，都会被 a() 所在事务捕获到，进而导致 a() 所在事务也回滚**。

    **正确的处理方式**：b() 中应该捕获异常，然后通过是否设置 rollbackOnly 标志决定是否让 a() 所在事务也回滚。

    **事务传播个人认为只是强调多个操作是纳入一个事务管理还是多个事务分开管理，而不是说互不干扰**。

    >  初学者容易认为 `REQUIRED_NEW ` 新建事务后，即使抛出异常也不会影响之前的事务，这是不对的（对可能存在外部事务的传播类型都做了测试无一例外）；原因无论新建事务执行还是普通执行，抛出的异常只要不处理都会最终抛到外部事务中（Spring事务源码发现异常即使捕获也会再次抛出）。

  + **测试**

  + **其他**

    分析IOC时有涉及但是不够详细，所以这里重新绘制单独的流程图。

    + **FactoryBean**

      包括FactoryBean和Bean生命周期的详细分析。

      源码流程图：

      + [spring-beans-factorybean.drawio](docs/java/spring/spring-beans-factorybean.drawio)
      + [spring-beans-factorybean.drawio.png](docs/java/spring/imgs/spring-beans-factorybean.png)

    + **循环依赖处理**

      源码流程图：

      + [spring-beans-circular-dependency.drawio](docs/java/spring/spring-beans-circular-dependency.drawio)
      + [spring-beans-circular-dependency.drawio.png](docs/java/spring/imgs/spring-beans-circular-dependency.png)

    + **Bean创建顺序控制**

    + **四种依赖注入实现**

      分析构造器注入、setter方法注入、工厂方法注入（分为静态工厂方法、实例工厂方法 [结合factory-bean使用]）。

    + **五种不同方式的自动装配**

      依赖no、byName、byType、constructor、autodetect。

+ **Spring MVC**

  这里只是分析请求处理流程，Servlet容器初始化流程看Spring Boot的部分（XML配置方式已经不流行了），初始化流程应该也可以参考WebFlux流程图估计差别不大。 

  源码流程图：

  + [spring-mvc.drawio](docs/java/spring/spring-mvc.drawio)
  + [spring-mvc.drawio.png](docs/java/spring/imgs/spring-mvc.drawio.png)

  原理简述：

  本质是向Servlet容器（比如Tomcat）注册了一个名为 `DispatcherServlet` 的Servlet，通过这个类处理HTTP请求。

  1. Tomcat 经过一系列处理  Connector -> Processor -> Valve -> Filter链 -> DispatcherServlet，即最终将请求交给`DispatcherServlet` 处理; 

     > 在 Spring MVC中注册的过滤器在会添加到Filter链。

  2. `DispatcherServlet`先从请求处理器映射中根据请求方法类型、参数、请求头、请求与响应类型等信息匹配请求处理器；

  3. 然后装配上与请求匹配的拦截器链，生成 HandlerExecutionChain，然后根据请求处理器定义方式获取合适的请求处理适配器，常用的是 RequestMappingHandlerAdapter;

     > 不同的Controller定义方式处理方式不同，现在常用的定义方式是通过 @RequestMapping， 对应的请求处理器适配器是 RequestMappingHandlerAdapter。

  4. 处理请求前先遍历执行所有匹配的拦截器的 preHandle()；

  5. 通过请求处理器适配器调用请求处理方法，处理流程主要包括：方法参数解析（包括请求消息转换成方法参数类型）、反射调用

     Controller业务方法、方法返回数据类型转成响应消息类型，最终通过 HttpResponse 写响应状态、响应主体；

  6. 遍历执行所有匹配的拦截器的 postHandle()；

  > 像 Model View 在前后端分离的项目基本不会使用，暂略。

+ **Spring Boot**

  很久之前看源码输出到了Markdown文件，但是内容多了Markdown可读性很差，待补充原理图。

+ **Spring WebFlux**

  源码流程图：

  + [spring-webflux.drawio](docs/java/web/spring-webflux.drawio)

  + [spring-webflux.drawio.png](docs/java/web/spring-webflux.drawio.png)

  原理简述：

  和 SpringMVC 处理流程基本一致，只不过 WebFlux 默认使用 Reactor-Netty 作为Web容器，接口都基于 ProjectReactor 封装成了响应式接口，支持 HTTP 协议 和 自定义协议通信。

  与 SpringMVC 的 `DispatcherServlet` 对应有一个 `DispatcherHandler` 类，这个类定义了请求处理的主要流程。

  > 前面说 Tomcat NIO 模式下请求处理的部分操作依然是阻塞的，只要有操作是阻塞的就会白占线程，降低系统吞吐量，而通过多创建线程提升系统吞吐量则又会引入线程上下文切换的开销。

+ **Vertx Web**

  基于 Vertx Core 实现的 Web 服务器。

+ Helidon

+ Micronaut

+ **[Quarkus](docs/quarkus)**

  Quarkus是一个为GraalVM和HotSpot定制的Kubernetes原生Java框架，Quarkus HTTP服务器基于 Vertx-Web 实现。Quarkus 为了**支持本地化编译**将启动流程分为了**引导阶段**和**应用启动阶段**，引导阶段主要是**执行 BuildStep 生成一套应用代码**（应用代码里面很多类是在**增强**阶段用自己封装的字节码工具Gizmo（基于ASM）**动态生成**的），这也导致源码**不容易理解**（相当于从元编程的角度编写应用代码），另外还有很多动态类加载、函数式代码、事件驱动代码提升了读码难度。

  > Quarkus 引导阶段才是核心，理解Quarkus的工作原理本质就是理解 BuildStep 的原理。
  >
  > 看源码必备：
  >
  > 官方提供的 Maven 插件 quarkus-maven-plugin 有提供输出增强类字节码的工具（goal） generate-code、generate-code-tests,  执行 generate-code 可以在 target/quarkus-app/quarkus/ 下生成 **generated-bytecode.jar**，解压可以看到生成的所有字节码文件，看源码是一定需要这些文件辅助的，可以在IDEA中关联此 jar 包调试生成的代码。
  >
  > 另外可以通过设置下面配置打印源码中的 TRACE 日志，辅助理解内部流程。
  >
  > ```xml
  > quarkus.log.category."io.quarkus".level=TRACE
  > # 默认只允许打印 DEBUG 及以上级别日志，如果要打印低于 DEBUG 级别的日志，需要额外设置 min-level
  > quarkus.log.category."io.quarkus".min-level=TRACE
  > ```
  
  看 Quarkus 源码最核心的需要梳理清对 **CDI**（相当于 Spring Boot 的 IoC）的支持原理、**拓展机制**（相当于 Spring Boot 的自动配置，集成其他框架时需要）的实现，Quarkus **Web**部分需要先熟悉下 Vertx Web 的接口和使用方法。
  
  > 有 Spring Boot 开发经验使用 **Quarkus 开发应用还是比较容易的**很多概念都是相通的，有资料说几天完全可以上手；个人感觉主要**难在拓展组件的开发**，需要清晰地理解 Quarkus 内部机制；另外实战时将一个项目改造成 Quarkus 项目后，打包 Jar 包很容易，但是**打包本地可执行文件时暴露出一堆坑**（很多代码不经意就引入了动态特性就导致构建失败）。
  
  + **CDI**
  
    并没有找到适合用于理解Quarkus CDI工作原理的单元测试供调试，建了一个Quarkus最小应用，清除了一些不必要的依赖，用来调试CDI流程，参考 quarkus-lab/quarkus-010-smallest-system。
  
    源码流程图：
  
    + [quarkus-core.drawio](docs/java/quarkus/quarkus-core.drawio)
  
    + [quarkus-core.drawio.png](docs/java/quarkus/imgs/quarkus-core.drawio.png)
  
      仅仅梳理了增强阶段启动流程、ApplicationImpl 类生成、Bean 创建和存储、依赖注入等大概原理，另外它们的原理还涉及到增强阶段 BuildStep 的执行，但是由于流程过于繁琐，短时间没法梳理清楚先暂停。
  
  + **拓展机制**
  
    相当于SpringBoot自动配置（Starter开发），Quarkus 做应用开发还是比较好上手的，但是做组件拓展开发就没有那么容易了，仅框架内置就有上百个 BuildStep 和 几百个 BuildItem 它们共同配合才生成应用代码，需要理解其工作原理才能清楚在集成第三方工具时应该怎么引入已有的 BuildItem 实现自己的拓展组件，拓展组件原理解析和开发资料基本没有，只能参考开源的拓展组件慢慢摸索。
  
    常用拓展组件:
  
    + [quarkus-mybatis](docs/java/quarkus/quarkus-mybatis.md)
    + [redisson-quarkus](docs/java/quarkus/quarkus-mybatis.md)
  
  + **Web**

### 1.9 响应式

响应式编程框架有点类似工具类框架没有主线或者说只有各个接口类实现的独立的主线。

+ **[Reactor](docs/java/reactive-streams/project-reactor.md)**

  源码流程图：

  + [project-reactor-core.drawio](docs/java/reactive-streams/project-reactor-core.drawio)

  + [project-reactor-core.drawio.png](docs/java/reactive-streams/project-reactor-core.drawio.png)

  + [mono-delay.drawio](docs/java/reactive-streams/mono-delay.drawio)

  + [mono-delay.drawio.png](docs/java/reactive-streams/mono-delay.drawio.png)

    这个流程图分析 Mono.delay() 实现原理（可以看作是 Thread.sleep() 的异步非阻塞实现），研究如果实现对**阻塞操作**的**异步非阻塞**改造 。

  要求：

  **所有操作都不阻塞**，项目中要用响应式，需要将项目中所有组件的阻塞操作都进行**异步化改造**，这样才能实现更少的线程更大的吞吐量（同时更少的线程也能减少线程上下文切换）。

  只要项目中混入了任何阻塞式操作的组件，都会让对应调用链的性能大打折扣，因此使用响应式不是引入Reactor 、WebFlux 这些响应式框架就行了，还需要整个项目生态的组件全部做异步化改造，这也是主要导致 WebFlux 久久无法火起来的原因（其他原因响应式规范不符合人类思维习惯、调试困难[函数式接口、被拆散的流程]）。

  不过现在JDK21也开始支持虚拟线程了，类似Go协程的概念，从语言层面作出了优化，感觉这东西在后端更难流行起来了。

  > 说所有操作都不阻塞感觉有点不是很严谨，像 Mono.delay() 其实相当于将阻塞操作从本线程中移到了ScheduledThreadPoolExecutor的工作者线程，即工作者线程中为了实现延迟还是有阻塞的，不过多个 Mono.delay() 调用可以复用工作者线程统一处理阻塞操作；有点像 Reactor 模式线程复用的思想。

+ **RxJava**

### 1.10 微服务框架

+ **Spring Cloud**

  其实就是将一堆微服务组件通过自动配置集成到 SpringBoot，从这点看其他框架如 Quarkus 其社区也提供了一堆微服务组件集成拓展，说 Quarkus 也是微服务框架貌似也可以。

+ **K8S**

  K8S 通过自身容器管理功能以及集成微服务组件容器实现微服务支持，参考 Go 部分。

### 1.11 服务网格

+ **Istio**

  服务都会启动SideCar代理，可以拦截和处理来自其他服务实例的网络流量并提供各种功能，包括负载均衡、故障转移、熔断、限流、安全、监控等等。

### 1.12 注册中心/配置中心

+ **Apollo**

+ **Nacos**

  源码流程图：

  + [nacos.drawio](docs/java/nacos/nacos.drawio)

### 1.13 分布式协调

+ **Zookeeper**
  
  + 源码流程图：
  
    + [zookeeper-server.drawio](docs/java/zookeeper/zookeeper-server.drawio)
    
      > 很久之前画的没有UML，而且图太过罗嗦，TODO 重画。
  
  + Curator
  
    + 分布式锁 InterProcessMutex
  
      TODO。


### 1.14 服务保障

+ [**Jd-Hotkey**](docs/java/jd-hotkey/jd-hotkey.md)

  热Key指的是频繁访问的数据；Jd-Hotkey 依赖的关键组件：Netty、Etcd、Caffeine、Guava。

  源码流程图：

  + [jd-hotkey-v0.0.4.drawio](docs/java/jd-hotkey/jd-hotkey-v0.0.4.drawio)

  + [jd-hotkey-v0.0.4.drawio.png](docs/java/jd-hotkey/imgs/jd-hotkey-v0.0.4.drawio.png)

    关键问题：

    + 为何引入本地缓存 Caffeine
    + 热key探测实现流程（上报到同一Worker进行滑动窗口统计）
    + 规则命中率统计流程？CounterConsumer 中规定 map.size() >= 300 才会向 Etcd 推送规则命中率数据，这里300的含义到底是什么
    + 热key探测规则发布原理
    + jd-hotkey 使用本地缓存存储热key的值，如何保证一致性
    + 官方列举的一些应用场景应该怎么实现

  + [jd-hotkey.drawio](docs/java/jd-hotkey/jd-hotkey.drawio)（旧流程图）

  + [jd-hotkey.drawio.png](docs/java/jd-hotkey/imgs/jd-hotkey.drawio.png)（旧流程图）

  业务使用：

  + [jd-hotkey-usage.png](docs/java/jd-hotkey/imgs/jd-hotkey-usage.png)

  > 得物出了一个[基于Redis内核的热key统计实现方案](https://tech.dewu.com/article?id=149)，实现原理是对 Redis Server 进行二开，拦截客户端请求**通过 LRU 进行热 key 统计**，然后外部可以通过**订阅通道**读取统计数据。

+ [**Sentinel**](docs/java/sentinel/sentinel-workflow.md)

  源码流程图：

  + [sentinel.drawio](docs/java/sentinel/sentinel.drawio)
  + [sentinel.drawio.png](docs/java/sentinel/sentinel.drawio.png)

  Sentinel 中系统自适应保护规则的 QPS \ RT 检测基于**滑动窗口**，热点参数限流规则和流量控制规则内部有多种规则，以流控规则中的校验规则为例 DefaultController基于**滑动窗口**、RateLimiterController、WarmUpRateLimiterController 基于**漏桶**算法，有资料说 WarmUpController 基于**令牌桶**算法（不过还没确认）。

  > 上面的漏桶算法说是漏桶算法只是看到官方文档称是漏桶算法，从定义上看个人感觉其实是**令牌桶**算法，其实不必纠结于这种概念，能符合业务需要随便怎么实现。

  注意令牌桶算法使用不当也可能无法应对**突发流量**（只是说可能，需要看令牌发放方式），比如1s限制1000次访问，令牌桶算法如果是1s内匀速发放1000个令牌（比如每10ms发放10个令牌），面对前100ms内有1000次请求后面900ms没有请求的情况会被限流（不一定直接拒绝请求「**非阻塞式**」，也可能是阻塞等待直到有可用令牌「**阻塞式**，强行整流会拉长响应时间RT」，看具体使用的方法），这种聚集但少量的请求不被限流更适合。

  > 上面阻塞式的令牌桶和漏桶算法的效果已经很像了，感觉**令牌桶和漏桶算法的边界并没有那么清晰**。
  >
  > 令牌桶令牌发放看实现基本都是隔一段时间发一批，并不一定是每隔一段时间发一个令牌，这种发放方式可以解决突发流量问题。

+ Hystrix

  已经不维护了，Spring 官方推荐使用  Resilience4j 替代 Hystrix。

+ Resilience4j

  提供限流、熔断等功能，限流基于令牌桶算法。

+ Guava RateLimiter

  单纯的限流组件，基于令牌桶算法。

+ **Redisson RateLimiter**

  单纯的限流组件，基于令牌桶算法。参考 Redisson 章节。

+ Bucket4j

  单纯的限流组件，基于令牌桶算法。

### 1.15 链路追踪

+ **Skywalking**

  + **Agent**

    源码流程图：

    + [skywalking-agent.drawio](docs/java/skywalking/skywalking-agent.drawio)
    + [skywalking-agent.drawio.png](docs/java/skywalking/skywalking-agent.drawio.png)

+ **Sleuth**

+ OpenTelemetry

### 1.16 监控系统

+ **Prometheus**

  暂只关注数据采集和上报的原理，业务定制也主要是这部分。

  + **client_java**

    感觉和 Skywalking Agent 库的代码组织有点像也有一堆根据服务实例中各种组件定制的**数据采集组件**，可以按需引入。

    源码流程图：
  
    + [prometheus-client-java.drawio](docs/java/prometheus/prometheus-client-java.drawio)
    + [prometheus-client-java.drawio.png](docs/java/prometheus/imgs/prometheus-client-java.drawio.png)
  
    拓展：
  
    + [目标分位数问题与CKMS算法](docs/java/prometheus/target-quantile-problem-and-CKMS.md)
  
  + **micrometer-registry-prometheus**
  
    Micrometer 适配 Prometheus上报接口的数据采集器组件，代码感觉有点乱，还有一堆函数式编程，代码可读性较差。
  
    源码流程图：
    
    + [micrometer-registry-prometheus.drawio](docs/java/prometheus/micrometer-registry-prometheus.drawio)
    + [micrometer-registry-prometheus.drawio.png](docs/java/prometheus/imgs/micrometer-registry-prometheus.drawio.png)

### 1.17 安全框架

+ **Apache Shiro**

+ **pac4j**

+ **Sa-Token**

  参考后面分析的 Sa-Token OAuth2 工作流程。

+ **[Spring Security](docs/java/spring-security/spring-security.md)**

  + 主流程

    源码流程图：

    + [spring-security.drawio](docs/java/spring-security/spring-security.drawio)
    + [spring-security.drawio.png](docs/java/spring-security/spring-security.drawio.png)

+ **JCasbin**

  JCasbin 提供了简单易用、灵活、强大的权限校验模型，可以替换 Spring Security、Shiro 中的权限校验模块。

  源码流程图：

  + [jcasbin.drawio](docs/java/jcasbin/jcasbin.drawio)
  + [jcasbin.drawio.png](docs/java/jcasbin/imgs/jcasbin.drawio.png)

+ **认证协议**

  系统多个服务（包括外部服务）整合统一的认证授权服务时会用到，使用最多的是 OIDC 和 OAuth2。

  + **OAuth2**

    包括**四种角色**（资源所有者、客户端[APP、Web服务等等]、认证服务器、资源服务器，认证服务器和资源服务器可以是一个服务）、**四种模式**，认证服务器包含**一组固定的HTTP接口**处理客户端的认证请求。

    不只是可以用在第三方授权，企业内部系统也是可以的，还可以用于实现**单点登录**，之前公司内部系统就是用 OAuth2 对接 GitLab、YAPI、Kibana、自己的管理后台、监控后台等等服务。

    > 学习 OAuth2 工作原理和细节看再多的资料不如看一遍源码理解的清晰，推荐看 Sa-Token 的源码比较简单，一天即可看完主流程实现包括内部细节。

    + Spring Security OAuth2
  
      源码流程图：
  
      + [spring-security-oauth2.drawio](docs/java/spring-security/spring-security-oauth2.drawio)
  
    + Sa-Token OAuth2
  
      源码流程图：
      
      + [satoken-oauth2.drawio](docs/java/authentication-protocols/oauth2/satoken-oauth2.drawio)
      + [satoken-oauth2.drawio.png](docs/java/authentication-protocols/imgs/satoken-oauth2.drawio.png)
  
  + **OIDC** (OpenId Connect)
  
    从 Sa-Token 的源码实现看就是**基于 OAuth2 认证流程、使用 JWT 形式包装用户信息**；看文档定义（[OIDC 是什么](https://auth.wiki/zh/openid-connect)）的话可能感觉OIDC不好理解，但是从源码上理解其实超级简单，以SaToken为例，OAuth2 和 OIDC 就是 UserIdScopeHandler 和 OidcScopeHandler 这两个类的区别。
  
  + SAML
  + CAS
  
  + JWT
  
    比较简单不详述了。


### 1.18 ORM框架

+ **Mybatis**

  + 主流程

    源码流程图：

    + [mybatis.drawio](docs/java/mybatis/mybatis.drawio)
    + [mybatis.drawio.png](docs/java/mybatis/imgs/mybatis.drawio.png)

    > 也是好久之前画的图，UML不详细，导致回看不好理解，SQL前后置处理以及连接池部分还有细节逻辑没有梳理，TODO 重画。
  
  + 重要组件
  
     + [mybatis-cache.drawio](docs/java/mybatis/mybatis-cache.drawio) (Mybatis 两级缓存工作原理)
  
      + [mybatis-cache.drawio.png](docs/java/mybatis/imgs/mybatis-cache.drawio.png) 
  
        + 两个事务中同时执行同一条查询语句使用二级缓存是怎么保证不会出现脏读的
        + 二级缓存联表查询数据不一致问题产生的原因
        + SpringBoot 集成 Mybatis 非事务方式连续两次执行同一条查询，为何第二次不会命中一级缓存
  
        > 综上：SpringBoot Mybatis 项目默认配置下其实根本不会用到 Mybatis的缓存。
  
      + [mybatis-plugin.drawio](docs/java/mybatis/mybatis-plugin.drawio) (Mybatis 插件工作原理)
  
      + [mybatis-plugin.drawio.png](docs/java/mybatis/imgs/mybatis-plugin.drawio.png)
  
        插件原理：通过 JDK 动态代理将插件通过 Interceptor 接口定义的拓展逻辑封装到 Mybatis SQL执行组件中，可以
        拦截 Exuecutor StatementHandler ParameterHandler ResultSetHandler 的方法，对SQL语句、参数、返回值进行额外处理；
        
        这里以 PageHelper 为例分析插件的XML配置解析、实例化、初始化、注册原理，插件都可以增强数据库访问中哪些过程和组件（Executor、StatementHandler、ParamterHandler、ResultSetHandler），以及插件增强原理（JDK动态代理层层封装）。
        
        > 由于 PageHelper 分页基于 LIMIT ?, ? 实现，这种分页方式有深度分页问题，只适合小数据量的表，所以 PageHelper 平时使用的并不多。

+ Mybatis-Spring
+ **Mybatis Plus**

### 1.19 数据库相关

+ **Canal**

  > 之前梳理了个半成品，TODO 重画。

+ DataX

  数据库迁移工具，阿里云DataWorks的开源版本。


+ **Redis**

  Redis的源码看起来并不难理解，平时使用基本用不着看源码；只在需要理解某些重要问题（比如某个命令是否有性能问题）底层原理时看对应部分源码就行了（也可以看《Redis源码剖析与实战》了解大概流程，碰到具体问题能快速找到对应处理代码即可）。

  源码流程图（6.2.6）：

  + [redis-client-server.drawio](docs/java/redis/redis-client-server.drawio) (Redis Client/Server 流程，sheet1: 客户端、sheet2: 服务端)

    + [redis-server.drawio.png](docs/java/redis/imgs/redis-server.drawio.png) （服务端对请求处理流程）

      这里主要关注 Redis Server 基于 epoll IO多路复用模型对请求的处理流程，以及 Redis 命令解析分发和执行流程（需要知道客户端输入一个命令，命令最终是由 Server 中哪个方法执行的）。
      
      最好先参考《Linux系统编程》等书籍回顾下 epoll 是怎么使用的，不然可能看不懂上面的流程图；或者参考这个 epoll 示例：[epoll_server.c](https://github.com/kwseeker/netty/blob/master/epoll/epoll_server.c) 。

  + [redis-cmd-set.drawio](docs/java/redis/redis-cmd-set.drawio) (Redis Server set命令处理流程)

  + [redis-cmd-rpush.drawio](docs/java/redis/redis-cmd-rpush.drawio) (Redis Server rpush命令处理流程)

  + [redis-cmd-zadd.drawio](docs/java/redis/redis-cmd-zadd.drawio) (Redis Server zadd命令处理流程)

  + [redis-inner-ds.drawio](docs/java/redis/redis-inner-ds.drawio) (Redis Server 内部基础数据结构)

    + [obj_encoding_skiplist2.png](docs/java/redis/imgs/obj_encoding_skiplist2.png) (跳表的数据结构)

      发现技术圈很喜欢讨论跳表，这里单独列出来之前画的一张ZSet跳表的结构图以及参考Redis源码使用Java重新实现的跳表[ZSkipList.java](https://github.com/kwseeker/redis-model/blob/master/redis-theory/src/main/java/top/kwseeker/redis/theory/datastructure/ZSkipList.java)，此跳表实现原理说明参考 [redis-data-structure.md](docs/java/redis/redis-data-structure.md) 。

  + redis-pubsub (Redis发布订阅)

    后面 Redisson 发布订阅源码分析中仅仅是展示Redisson对发布订阅的封装，不包含Redis服务端内部对发布订阅的实现。

    >  TODO： Redis发布订阅命令源码流程图补充。

  + redis-stream (Redis Stream)

    > TODO： 源码流程图补充。

  + redis-transaction (Redis事务)

    Redis事务其实说的只是并发的原子性，保证事务中的多个命令执行时中间不会插入其他客户端的命令。没有事务控制时为何可能插入其他客户端的命令？看上面**IO多路复用模型**就明白了：**多个客户端同时发送命令，先获取到哪个客户端的命令就绪事件无法确定**，所以高并发场景下一个客户端先后发送两个命令，中间很可能插入其他客户端的命令；
    要实现Redis事务其实就是要么将多个命令合并成一个命令一起发送；要么就是先缓存到服务端的队列，当提交EXEC后，再将这批命令一起取出来一起执行；根据一些资料看实现原理是第二种方案。
    
    > TODO： 源码流程图补充。

  + [Redis 过期key定期删除（主动删除）原理](docs/java/redis/redis-expired-keys-clear.md)

    定期删除任务由 `bio_lazy_free` 线程执行，但是此线程只是负责扫描过期key并加入异步队列，**过期的key最终是还是由时间事件循环交由主线程执行删除命令进行删除**，所以定期删除扫描任务中的思想是**少量多次有间隔地删除**，而不是扫描全部的过期key一起删除以防止阻塞客户端命令的执行。

    如果有一批key同时过期，定期删除策略也只能缓解对其他key读写效率的影响。

  重要问题分析：

  + Redis 6.0 开始到底哪里支持了多线程

    看上面 [redis-server.drawio.png](docs/java/redis/imgs/redis-server.drawio.png) 会发现没有用到多线程啊？这是因为**IO多线程默认是关闭的**需要修改服务端配置（redis.conf），然后 `redis-server redis.conf` 启动，启动后 `ps -T -p <pid>` 可以看到多线程模式相对于单线程模式多出来几个线程 `io_thd_<n>`，暂时没时间看这部分源码，可以先参考[Redis 6.0的多线程](https://cloud.tencent.com/developer/article/1940123) 这篇文章，后面会添加流程图（TODO）。
    
    ```shell
    # 开启网络IO多线程
    io-threads-do-reads yes
    # 设置IO线程数量为8
    io-threads 8
    # 查看 redis-server 进程下的所有线程，上面io-threads 8 包括main线程
    ~ ps -T -p 96007
        PID    SPID TTY          TIME CMD
      96007   96007 pts/4    00:00:00 redis-server		# 主线程，即处理客户端命令的线程
      96007   96028 pts/4    00:00:00 bio_close_file
      96007   96029 pts/4    00:00:00 bio_aof_fsync
      96007   96030 pts/4    00:00:00 bio_lazy_free		# 这个线程是过期key定期删除的扫描线程
      96007   96031 pts/4    00:00:00 io_thd_1
      96007   96032 pts/4    00:00:00 io_thd_2
      96007   96033 pts/4    00:00:00 io_thd_3
      96007   96034 pts/4    00:00:00 io_thd_4
      96007   96035 pts/4    00:00:00 io_thd_5
      96007   96036 pts/4    00:00:00 io_thd_6
      96007   96037 pts/4    00:00:00 io_thd_7
    ```

  + Redis ZSet 中最多只存储20W热点数据，每次新增热点数据时使用 zcard 查看数量，超过 20W 使用 zremrangeByRank 删除最早添加的数据，这个数据量 zcard zremrangeByRank 有没有性能问题

    这两个命令肯定没有性能问题，插入操作和检索操作需要评估下，这数据量一定是跳表，使用跳表读写时间复杂度是 O(Log(N))，20W 数据属于大key，一般规定 Set 类型不要超过1W数据，20W数据量相对于 1W 数据估计大概多5次循环，内存中5次循环还是可以接受的，不过建议还是做拆分，比如按地域、按用户尾号拆分到不同的key中，主要是避免大key导致在集群中出现数据倾斜。

+ **Sharding-JDBC**

  源码流程图（5.4.1）：

  + [ShardingSphere-JDBC.drawio](docs/java/sharding-jdbc/Sharding-JDBC.drawio) (未完待续)
  + [ShardingSphere-JDBC.drawio.png](docs/java/sharding-jdbc/ShardingSphere-JDBC.drawio.png)

+ **连接池**

  这些连接池可以对比着看源码，比较下各自优缺点。
  
  
    + Mybatis PooledDataSource
  
  
    + **Druid**
  
  
    + **HikariCP**
  


### 1.20 分布式事务

+ **Atomikos**

+ **ByteTCC**

+ **EasyTransaction**

+ **Hmily**

  源码流程图：

  + [hmily.drawio](docs/java/distributed/transaction/hmily.drawio)

  + [hmily.drawio.png](docs/java/distributed/transaction/hmily.drawio.png)

  + [hmily-producer-consumer.drawio.png](docs/java/distributed/transaction/hmily-producer-consumer.drawio.png)

    Hmily 基于 Disruptor 实现的事件发布订阅（生产者消费者模式）。

+ **LCN**

+ **RocketMQ事务消息**

  参考 RocketMQ 部分。

+ **Seata**

  包含一套分布式事务方案适用于不同要求的一致性场景，总代码量很大，但是核心代码量没有那么吓人，只需要关注**seata-server**、**seata-core**、一种配置中心的支持(如seata-config-nacos)、一种服务发现的支持（如seata-discovery-nacos）、一种通信方式支持（如 RestTemplate、Feign、GRPC、Dubbo）、**三大组件（RM TM TC）**、各种**事务实现模式（AT、TCC、Saga、XA）**。

  源码流程图：

  + [seata-server.drawio](docs/java/seata/seata-server.drawio) (Seata服务器，作为TC)

  + [seata-server.drawio.png](docs/java/seata/imgs/seata-server.drawio.png)

    这里主要分析 AT 模式。TC 、TM 、RM 组件之间通过 Netty 通信，Netty 通信端口默认是 HTTP 通信端口 + 1000，Netty 通信组件封装和 RocketMQ 中 Netty 组件的封装大致相同。

  + [seata-client.drawio](docs/java/seata/seata-client.drawio) (Seata客户端，作为TM 和 RM)

  + [seata-client.drawio.png](docs/java/seata/imgs/seata-client.drawio.png)

    这里主要分析 AT 模式。每个客户端启动时都会启动 TM RM 的 Netty 客户端，但是只有全局事务发起者才会使用 TM 客户端，如果一个服务仅仅作为事务参与者只会用到 RM 客户端。

  + [seata-at-overview.drawio](docs/java/seata/seata-at-overview.drawio) （Seata AT模式流程概图）

  + [seata-at-overview.drawio.png](docs/java/seata/imgs/seata-at-overview.drawio.png)

    感觉官方文档中[原理图](https://seata.apache.org/zh-cn/assets/images/solution-1bdadb80e54074aa3088372c17f0244b.png)太过简略了，无法体现很多重要信息，重新绘制了AT模式工作原理概图。

    AT模式也是基于**业务补偿**的**两阶段提交**；业务补偿方面类似TCC，但是AT模式可以自行生成用于补偿的 UndoLog，而不是像 TCC 模式需要手动编码实现补偿逻辑，也即**没有业务侵入**；AT模式的两阶段提交，**第一阶段**：实现各分支事务的注册、执行、提交（**注意第一阶段分支事务就会提交**），**第二阶段**：如果是提交的话其实主要是执行事务数据的清理，比如删除 UndoLog（可以异步执行），如果是回滚的话，主要是查询各分支事务 UndoLog 并执行进行补偿。

  + [seata-datasourceproxy.drawio](docs/java/seata/seata-datasourceproxy.drawio) (数据源代理 DataSourceProxy)

  + [seata-datasourceproxy.drawio.png](docs/java/seata/imgs/seata-datasourceproxy.drawio.png)

    Seata @GlobalTransactional 增强的逻辑中并不包含事务操作，事务操作是通过 **DataSourceProxy** 实现，分支事务执行可能会同时经过 @Transactional 和 DataSourceProxy 事务操作，但是两者并不冲突，DataSourceProxy 中会先判断当前是否已经开启事务，如果已经开启不会重复开启。

    DataSourceProxy 本身**不是基于 Spring AOP 实现**，调用数据源接口时切换到 DataSourceProxy 的流程是基于 Spring AOP 实现的，详细参考 `SeataAutoDataSourceProxyCreator` 流程，它继承 `AbstractAutoProxyCreator`，本质是一个`BeanPostProcessor`。

    DataSourceProxy 通过**装饰器模式**对数据源操作扩展事务处理逻辑，比如通过 ConnectionProxy 为连接操作拓展事务处理包括向TC注册分支事务信息、记录UndoLog等等。

  + [seata-at-globallock.drawio](docs/java/seata/seata-at-globallock.drawio)

  + [seata-at-globallock.drawio.png](docs/java/seata/imgs/seata-at-globallock.drawio.png)

    Seata 全局锁用于保证不同分布式事务的**写隔离**和**读隔离**，本质就是通过**一组分布式行锁**保证同步执行（排队），详细参考：[seata-at-globallock.md](docs/java/seata/seata-at-globallock.md)。

    全局锁在 TC 中实现，实现原理其实很简单就是**基于主键唯一索引进行插入，插入成功获取锁成功，插入失败获取锁失败**，和Redis set nx 原理一样。

    `@GlobalLock` 用在不需要全局事务而又需要检查全局锁避免脏读脏写的场景（比如某个纯粹的本地事务查询全局事务可能会修改的某行数据，这种场景应该还是挺常见的），这种场景使用`@GlobalLock`注解更加轻量。

    > 这里纯粹的本地事务指不属于任何全局事务的本地事务。

  + [seata-configurationfactory.drawio.png](docs/java/seata/imgs/seata-configurationfactory.drawio.png)

    Seata 支持多种注册中心、配置中心的**适配器**实现；
    
    这种适配功能很常用，初学者不清楚怎么实现可以参考下，其实很简单就是定义一组通用接口方法，使用各个不同的注册中心、配置中心重新实现这些接口。

  关键问题分析（部分节选自官方FAQ）：

  + [@GlobalTransactional 注释的方法明明没有像 @Transactional 拓展事务操作为何这个方法中执行的SQL还是可以在出现异常后被回滚](docs/java/seata/seata-global-transactional-rollback.md)

  + [事务发起者和参与者服务方法上到底需不需要加 @Transactional 注解](docs/java/seata/seata-biz-method-transactional.md)

  + [全局事务ID是怎么传递给事务参与者服务的](docs/java/seata/seata-xid-propagation.md)

    > 注意 TransactionPropagationInterceptor 中的事务传播只是说 XID 的传递，和 Spring 事务传播不是一个概念。

  + [TCC AT XA 性能为何依次降低](docs/java/seata/seata-transaction-mode.md)

  + 怎么使用Seata框架来保证事务的隔离性

    即上面的全局锁。

  + 脏数据回滚失败如何处理

    结合告警、 UndoLog 手动处理。

  + 抛出异常后事务未回滚

    官方文档直接列举了一些可能原因，但是感觉这种回答方式不太好，应该结合抛出异常到事务回滚的流程一步步排查，清楚内部流程的话应该怎么排查很清楚。

+ **TCC-Transaction**

### 1.21 搜索引擎

+ **ElasticSearch**

  + 客户端

    + Spring Data ElasticSearch

    + elasticsearch-rest-high-level-client

    + x-pack-sql-jdbc

      ElasticSearch 官方提供的一个组件，支持使用 JDBC 接口规范访问 ES。


### 1.22 规则引擎

+ **Drools**

### 1.23 工作流引擎

+ **Activiti**
+ **Spring Flowable**

### 1.24 缓存

+ **Caffeine**

+ **Ehcache**

  TODO。

### 1.25 分布式

+ **一致性协议**

  + **[Raft](docs/java/distributed/consistency/raft.md)**

    说实话想自己按 Raft 论文写一个实现也不是个简单的事，关键是需要将论文中的所有细节理解清楚，里面有一些证明能否理解透彻，即便写出来，怎么测试实现的没有漏洞也是个问题。

    + **BRaft**
  
      Raft协议的C++实现。
  
    + **JRaft**
  
      看 Nacos 当前最新版本（2.3.2）中依赖的是 SOFAJRaft，简称JRaft，是参考 BRaft 的 Java 实现。
  
      可以参考 Seata RaftServer 了解如何使用 JRaft 实现一个分布式一致性文件系统。
  
    + **Raft4J**
  
    + **Raft-Java**
  
      这个只是DEMO性质的实现，主要是代码实现简单（5K多行，其他生产级别的实现都是几W行），适合快速学习Raft协议细节。
    
      源码流程图：
    
      + [raft-java.drawio](docs/java/distributed/consistency/raft-java.drawio)
      + [raft-java.drawio.png](docs/java/distributed/consistency/raft-java.drawio.png)
  

### 1.26 命令行

+ **JCommander**
+ **CLI**

### 1.27 日志

+ **Slf4j**

+ **Log4j2**

+ **Logback**

  虽然 logback-core、logback-classic 代码量加起来有 6W 行，但是核心流程比较简单，且数据结构很清晰，直接看源码可能比看官方文档和书能更快地理解 Logback 的工作原理和使用细节。

  源码流程图：

  + [logback.drawio](docs/java/logging/logback.drawio)
  + [logback.drawio.png](docs/java/logging/imgs/logback.drawio.png)

### 1.28 序列化

+ **Fastjson**

  基于FastJson2源码分析，从源码实现上看主体逻辑其实比较简单，之所以源码代码量很高更多地是因为编程语言语法的复杂性（需要适配各种各样的类型和语法，而且要用ASM定义ObectWriter动态类，调试时可以发现有很多if判断，还有一些对特定类型的定制实现，但是实际的类型可能只会用到其中少部分处理逻辑）。

  序列化主体逻辑可以分为3部分（以  ObjectWriterCreatorASM 为例）：
  1）通过Class解析一个类型需要序列化的部分（比如公共基本类型字段、有实现 Getter 方法的私有基本类型字段、对象类型字段、继承的字段等等），针对每个字段会创建一个 FieldWriter 实现对这个字段的序列化；

  2）为此类型通过 ASM 字节码动态生成一个专属的 ObjectWriter 类进而加载并实例化，用于专门实现对这个类型对象的序列化写操作，通过这种方式避免了反射的低效性，而且 Fastjson 会缓存这些动态生成的 ObjectWriter 类，后续再序列化会很快；

  3）使用 ObjectWriter 对每个字段进行序列化并追加到最终的序列化结果。

  源码流程图：

  + [fastjson.drawio](docs/java/fastjson/fastjson.drawio)
  + [fastjson.drawio.png](docs/java/fastjson/imgs/fastjson.drawio.png)

+ **Hessian**

+ **Jackson**

  TODO。

+ **Kryo**

+ **Protostuff**

### 1.29 工具类

+ **Arthas**

+ **EasyExcel**

+ **Guava**

+ **Hutool**

+ **JETCD**

  + **分布式锁**

    对比Redisson实现（TODO）。

+ **jvm-sandbox**

  > TODO 流程图迁移

+ **MapStruct**

+ **Redisson**

  + **发布订阅**

    这里分析Redisson Netty连接管理、通信管道和发布订阅流程。

    源码流程图：

    + [redisson-pubsub.drawio](docs/java/redisson/redisson-pubsub.drawio)
    + [redisson-pubsub.drawio.png](docs/java/redisson/redisson-pubsub.drawio.png)

  + **[分布式锁](docs/java/redisson/distributed-locks-redisson.md)**

    源码流程图：

    + [redisson-lock.drawio](docs/java/redisson/redisson-lock.drawio)

      + [redisson-lock-RedissonLock.drawio.png](docs/java/redisson/redisson-lock-RedissonLock.drawio.png)

        多数锁继承 RedissonLock，梳理清 RedissonLock 再看其他锁的实现会简单很多，可以看到Lua脚本才是核心；
        包括**加锁解锁流程**以及**看门狗**实现原理。

      + [redisson-lock-RedissonReadWriteLock.drawio.png](docs/java/redisson/redisson-lock-RedissonReadWriteLock.drawio.png)

      + [redisson-lock-RedissonFencedLock.drawio.png](docs/java/redisson/redisson-lock-RedissonFencedLock.drawio.png)
    
    注意：
    
    红锁（RedLock）在新版本已经废弃。

  + **分布式延迟队列**

    + RDelayedQueue (TODO)

      基于ZSet数据结构实现，延迟时间作为分值，定时任务扫描，取出到期的延迟消息给业务线程处理。
      扫描延迟消息的定时任务可以用计划任务线程池或Timer实现（看其他框架实现时见到好多次了），Redisson 用的哪种后面有空再看。


  + **[RateLimiter](docs/java/redisson/RateLimiter.md)**

    Redisson 限流器实现原理也挺简单的，是使用 **Lua 脚本**实现的，流程分为两步：

    1.  如果有过期的令牌，先删除过期的令牌（使用 ZSet 类型的键 {RateLimiterName}:permits 存储，按时间戳排序）并恢复可用令牌数量（使用 String 类型的 {RateLimiterName}:value 存储）；
    2.  判断可用令牌是否足够，不够就返回回收令牌需要等待的时间ms；足够的话就生成令牌ID存储到{myLimiter}:permits并将可用令牌计数减1。

    令牌的“发放”是通过回收过期的令牌实现的。

  + **Future、Promise模式**

    源码流程图：

    + [redisson-future-promise.drawio](docs/java/redisson/redisson-future-promise.drawio)
    + [redisson-future-promise.drawio.png](docs/java/redisson/redisson-future-promise.drawio.png)

  + **[misc](docs/java/redisson/redisson-misc.md)**

    + **AsyncSemaphore**

      借助 CompletableFuture 实现的异步非阻塞的 Semaphore，设计也挺巧妙的仅仅用了80行代码就实现了一个异步非阻塞的信号量。不过注意这个不是分布式的信号量，分布式信号量还是需要使用 RedissonSemaphore。

      源码流程图：

      + [redisson-AsyncSemaphore.drawio](docs/java/redisson/redisson-AsyncSemaphore.drawio)
      + [redisson-AsyncSemaphore.drawio.png](docs/java/redisson/redisson-AsyncSemaphore.drawio.png)

+ [**sensitive-word**](https://github.com/houbb/sensitive-word)

  基于 DFA （Deterministic Finite Automaton，确定有穷自动机）算法的敏感词工具（基于**前缀树**数据结构，此工具借助 HashMap 存储下一级字符节点），支持很多细节控制（比如格式转换、全角半角、大小写、邮箱检测、网址检测、白名单等等）。

  有些关于DFA算法的介绍（参考维基百科[确定有限状态自动机](https://zh.wikipedia.org/wiki/%E7%A1%AE%E5%AE%9A%E6%9C%89%E9%99%90%E7%8A%B6%E6%80%81%E8%87%AA%E5%8A%A8%E6%9C%BA)）还有什么事件、状态转换，但是从一些实现看并没有涉及这些东西，感觉是概念将问题描述的复杂化了，实现其实很简单。

  源码流程图：

  + [sensitive-word.drawio](docs/java/sensitive-word/sensitive-word.drawio)

  + [sensitive-word.drawio.png](docs/java/sensitive-word/imgs/sensitive-word.drawio.png)

    源码量不高，作者加了很多注释，代码很容易理解，这里不详细画流程图了，只看数据结构的 UML就能推导出逻辑 。


### 1.30 语法解析器

+ **Antlr**

  Sharding-JDBC 中 借助 Antlr 实现对 SQL 语句的重构（原SQL -> AST -> 新SQL），将 SQL 转换成针对某个分表的 SQL。

### 1.31 表达式引擎

+ **Aviator**

### 1.32 自动化测试

+ [goreply](https://github.com/buger/goreplay)
+ [JSONassert](https://github.com/skyscreamer/JSONassert)

### 1.33 AI

+ [**langchain4j**](https://github.com/langchain4j/langchain4j)

+ [**Spring-AI**](https://docs.spring.io/spring-ai/reference/)

## 2 Python

### 2.1 AI

+ [**CoorAgent**](https://github.com/LeapLabTHU/cooragent)

+ [**MetaGPT**](https://github.com/FoundationAgents/MetaGPT)

    源码流程图：

    + [metagpt.drawio](docs/python/ai/agent/metagpt.drawio.png)
    + [metagpt.drawio.png](docs/python/ai/agent/metagpt.drawio.png)

## 3 Go

### 3.1 SDK

### 3.2 服务调用

+ **gorilla/websocket**

### 3.3 服务器

+ [**pocketbase**](https://github.com/pocketbase/pocketbase)

  一个开源的 Go 后端应用。包括：具有实时订阅的嵌入式数据库 （SQLite）、内置文件和用户管理、方便的 Admin dashboard UI

  和简单的 REST 式 API，另外**还可以使用 Go / JS 进行业务拓展**（比如自定义路由、接入数据库、事件钩子、任务调度、模板引擎、实时消息、文件系统、日志等等）。可以作为标准的后端应用、开发框架、工具包。

  > 感觉定位上更接近是一个支持灵活拓展的低代码平台。

### 3.4 微服务

+ **gopub**

  基于 Beego + Vue 开发的一个 K8S 运维管理发布系统，支持 GitLab、Jenkins 发布，支持 Docker & K8S 管理服务镜像。

  学习 K8S 运维，个人推荐使用 [`gopub`](https://github.com/linclin/gopub) 结合 [`sock-shop`](https://github.com/microservices-demo/microservices-demo)（可以替换为自己的Web项目）实战部署学习。

+ **Istio**

+ **K8S**

### 3.5 注册中心/配置中心

+ **Consul**
+ **Etcd**

### 3.6 数据库相关

+ [**Godis**](docs/go/godis)

### 3.7 分布式事务

+ **DTM**

### 3.8 通信框架

+ **WebRTC**

  [WebRTC](https://github.com/RTC-Developer/WebRTC-Documentation-in-Chinese/blob/master/resource/Chapter1/1%20%E4%BB%8B%E7%BB%8D.md) 是一种Web实时通信规范，提供点对点通信、流媒体传输等功能。

  + **[pion/webrtc](docs/go/pion/pion-webrtc.md)**
  
    连接建立流程：
  
    + [webrtc.drawio.png](docs/go/pion/imgs/webrtc.drawio.png)
  
    使用 Docker 虚拟网络模拟打洞流程参考：[kwseeker/p2p](https://github.com/kwseeker/p2p)。

### 3.9 音视频

+ **lal**

  从源码实现上看 lal 其实就是基于 **TCP** + **RTMP 等传输协议**实现的一个音视频流**转发**服务器。音视频发布者上传音视频数据块后，lal 会先读取并缓存在 Stream 对象（Stream 本质是个带着**读写双指针的缓冲**，最大存储 1MB 数据），直到读取到一个完整的消息（一个完整的消息可能比较大需要分成多个块传输）后会将这个消息**广播**发送给所有音视频订阅者（广播的实现方式是遍历订阅者连接会话逐个写）。

  源码流程图：

  + [lal.drawio](docs/go/lal/lal.drawio)
  + [lal.drawio.png](docs/go/lal/lal.drawio.png)



## 4 Rust

### 4.1 异步框架

+ **Tokio**

### 4.2 其他

+ **音乐**

  + [netease-cloud-music-gtk](https://github.com/gmg137/netease-cloud-music-gtk)

    网易云音乐 Linux 版已经不支持下载了，而且网上版本很老，然后发现了这个项目，基于 GTK + 网易云音乐API 实现。

    TODO：可以研究下源码（当前2.5.0版本，代码量大概8K行），打包自用，也可以了解下 Rust + GTK Linux 桌面程序开发和打包。



## 5 为何读源码

**读框架源码的初衷**：

+ **解决生产环境BUG**

  不熟悉源码原理，可能即使定位到问题所在代码行也不知道是什么原因应该怎么改。

+ **从0到1构建项目时，定位导致产生不符合预期结果的原因**

  这个碰到太多了，通过Google、官方文档可以解决大部分，但还是有一些无法查到的问题。

+ **拓展框架功能**（很多框架都有预留一些扩展点）

  拓展方式比如扩展钩子、SPI、插件、JVMTI Agent什么的。

+ **更好更合理地使用技术和框架**

  通过源码可以更透彻地理解某个技术的实现原理和细节，因为文档很难描述某些代码逻辑，也很难透漏所有细节，这也导致有些文档难以理解，甚至根据文档理解的是片面的或错误的。

  几乎所有框架的文档都没法将框架所有功能的细节都讲解清楚，熟悉原理可以帮助开发时避坑。

+ **满足对框架内部工作原理的好奇心，也便于后期出现BUG排查BUG**

+ **借鉴方案设计**

  对一陌生的场景进行方案设计时，一定要先多参考已有的设计，梳理设计要点，比较不同方案的优缺点，不要闭门造车，否则很容易设计出存在很多漏洞的方案；

+ **学习代码架构设计、代码风格规范、对依赖框架的封装和使用、提取轮子等**

+ **以不变应万变**

  对框架源码有清晰认识之后，很多问题没必要死记硬背（不是天天用肯定会忘），用到再看下对应模块源码即可，有什么坑怎么解决源码会告诉我们一切。

> 做企业级的方案设计的时候，如果不是对业务很熟悉（熟知需要关注哪些点、会存在什么坑）；首先需要多参考开源的方案设计，对比优缺点，总结都需要注意哪些坑，在此基础上再设计符合自己业务的方案。

**对读框架源码的心得**：

+ **读源码需要画图辅助记忆和理解**

  这一点是最关键的，解决看源码看了后面的逻辑忘了前面的逻辑的问题（面向生产的框架代码量一般都比较大，应该没有人看了成千上万行代码后对前面的流程细节还记得清清楚楚的吧），这个问题是制约我刚开始学后端迟迟无法深入源码的主要问题。

  曾经为了解决这个问题，试着写过源码分析文档（Markdown，写的多了回顾发现和重新看源码一样无法立刻回想起这段代码的作用和其他模块的关系）、画过思维导图、流程图（流程复杂了看着很乱）、时序图（绘制复杂且空间利用率低，即不方便看）但是效果都不好，最后突然想到为何不按时序图的编排方式画流程图，即使流程复杂也不会乱且格式较时序图更紧凑且方便绘制，后来因为流程图只能体现”算法“逻辑无法体现”数据结构“，又添加进去了简化的UML图，最终形成了现在的流程图。

  通过现在的流程图，可以快速回顾框架的主流程以及架构。

  > 也可能是我属于视觉空间学习者，所以需要借助图表理解感觉更清晰更容易接受。

+ **大部分框架源码是复杂但不是难，只是梳理过程比较费时间**

  研究清楚某个逻辑的完整实现流程，可能会涉及几十个源码文件，很多层调用。

+ **大部分框架都有一个主流程逻辑，主流程逻辑占全体代码量一般并不高，也是主要要看的**；除了一些工具类框架，比如`Hutool`、`Redisson`没有主流程

  面向生产的框架，里面很多功能点，一般都会为功能点提供较全的实现方案，它们的接口和功能类似，通常只需要梳理一种方案实现即可。

  比如框架中常见的配置解析，可能支持Properties 、Json、Yaml、Toml等配置方式；又比如通信框架中可能支持TCP UDP HTTP WebSocket等各种协议实现，梳理主流程时只需要关注其中一种实现。

+ **需要清楚代码阅读边界**

  框架里面可能依赖其他框架，不熟悉被依赖的框架，不要立即跳转进去看源码，先通过被依赖框架官方文档了解接口功能即可，以当前框架为主。

+ **很多框架在代码架构上有用一些相同的架构思想、设计模式、技术实现方法**

  多总结这些内容，可以加速对新框架源码的理解。

+ **想了解某一部分源码的逻辑可以结合相应的单元测试代码调试**

+ **检验对源码的熟悉度可以通过尝试从中提取Mini版的框架**

+ **读源码时组件初始化代码可以先不看细节，在分析流程逻辑时用到再回来找**

**文档规范**：

后面新 Markdown 文档统一使用 4W2H 规则编写。

4W2H：What (是什么)、When/Where(什么时候使用或者用在哪里)、Why(为什么选择这个)、How(怎么实现的)、How(怎么使用)，
即 **介绍**（包括功能）、**场景**、**优劣**、**原理**、**应用**。



## 6 资源

### 6.1 书籍

有电子版的书基本上在 [ZLibrary](https://zh.z-lib.gs/) 都能找到。