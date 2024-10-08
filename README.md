# 后端技术栈框架源码分析

这里主要分析Java和Go后端技术栈常用框架的源码，从整体到局部探究内部工作原理，并输出可视化流程图。

源码分析输出主要为`drawio`流程图（`.drawio`文件）和 `Markdown`文本，以流程图为主（主要展示框架主流程），Markdown作为补充，详细内容参考[docs](./docs)。

这里的流程图不是常规流程图，实际是借鉴的时序图的编排方式，另外还添加了简单的`UML`图。

**读框架源码的初衷**：

+ **解决生产环境BUG**

  不熟悉源码原理，可能即使定位到问题所在代码行也不知道是什么原因应该怎么改。

+ **从0到1构建项目时，定位导致产生不符合预期结果的原因**

  这个碰到太多了，通过Google、官方文档可以解决大部分，但还是有一些无法查到的问题。

+ **拓展框架功能**（很多框架都有预留一些扩展点）

  拓展方式比如接口、SPI、插件、JVMTI Agent什么的。

+ **更好更合理地使用框架**

  几乎所有框架的文档都没法将框架所有功能的细节都讲解清楚，熟悉原理可以帮助开发时避坑。

+ **满足对框架内部工作原理的好奇心，也便于后期出现BUG排查BUG**

+ **学习代码架构设计、代码风格规范、对依赖框架的封装和使用、提取轮子等**

**对框架源码的认识**：

+ **读源码需要画图辅助记忆和理解**

  这一点是最关键的，解决看源码看了后面的逻辑忘了前面的逻辑的问题（面向生产的框架代码量一般都比较大，应该没有人看了成千上万行代码后对前面的流程细节还记得清清楚楚的吧），这个问题是制约我刚开始学后端迟迟无法深入源码的主要问题。

  曾经为了解决这个问题，试着写过源码分析文档（Markdown，写的多了回顾发现和重新看源码一样无法立刻回想起这段代码的作用和其他模块的关系）、画过思维导图、流程图（流程复杂了看着很乱）、时序图（绘制复杂且空间利用率低，即不方便看）但是效果都不好，最后突然想到为何不按时序图的编排方式画流程图，即使流程复杂也不会乱且格式较时序图更紧凑且方便绘制，后来因为流程图只能体现”算法“逻辑无法体现”数据结构“，又添加进去了简化的UML图，最终形成了现在的流程图。

  通过现在的流程图，可以快速回顾框架的主流程以及架构。

+ **大部分框架源码是复杂但不是难，只是梳理过程比较费时间**

+ **大部分框架都有一个主流程逻辑，主流程逻辑占全体代码量一般并不高，也是主要要看的**；除了一些工具类框架，比如`Hutool`、`Redisson`

  面向生产的框架，里面很多功能点，一般都会为功能点提供较全的实现方案，它们的接口和功能类似，通常只需要梳理一种方案实现即可。

  比如框架中常见的配置解析，可能支持Properties 、Json、Yaml、Toml等配置方式；又比如通信框架中可能支持TCP UDP HTTP WebSocket等各种协议实现，梳理主流程时只需要关注其中一种实现。

+ **需要清楚代码阅读边界**

  框架里面可能依赖其他框架，不熟悉被依赖的框架，不要立即跳转进去看源码，先通过被依赖框架官方文档了解接口功能即可，以当前框架为主。

+ **很多框架在代码架构上有用一些相同的架构、模式**

+ **想了解某一部分源码的逻辑可以结合相应的单元测试代码调试**
+ **检验对源码的熟悉度可以通过尝试从中提取Mini版的框架**

> 后面列举的框架并非都分析过源码（心有余而力不足），有些只是计划，已经分析过源码的都有文档链接。
>
> 有些分析流程图之前记录在其他仓库后续会转移到这里，慢慢补充吧



## 分类

- 语言
  
  - [Java](#Java)
  
    - SDK
    - 网关
    - 服务调用
    - 负载均衡
    - 消息队列
    - 作业调度
    - 服务器
    - Web框架
    - 微服务框架
    
    + 注册中心/配置中心
    + 分布式协调
    + 服务保障
    + 链路追踪
    + 监控系统
    + 安全框架
    + ORM框架
    + 数据库相关
    + 分布式事务
    + 搜索引擎
    + 规则引擎
    + 工作流引擎
    + 缓存
    + 分布式
    + 命令行
    + 日志
    + 工具类
    + 语法解析器
    
  - [Go](#go)
    
    - SDK
    - 注册中心/配置中心
    - 数据库相关
  
- [Resources](#resources)
  
  - [Books](#books)



## Java

### SDK

+ **JVM**

  + **HotSpot**

    已经搭建好了源码调试环境（参考：jvm-debug），但是看JVM源码需要很多操作系统系统编程、内核方面的知识，不是短时间就能看明白的，所以暂停了。

    源码流程图：

    + [jvm8-hotspot.drawio](docs/java/jvm/jvm8-hotspot.drawio) (未完成)
    + [java-exec-process.drawio](docs/java/jvm/java-exec-process.drawio) (流程概要)
    + [java-exec-process.drawio.png](docs/java/jvm/java-exec-process.drawio.png)

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

  + **ThreadPoolExecutor**

  + [**ForkJoinPool**](docs/java/jdk/concurrent/ForkJoinPool.md)

    源码流程图：

    + [ForkjoinPool.drawio](docs/java/jdk/concurrent/ForkJoinPool.drawio)
    + [ForkJoinPool.drawio.png](docs/java/jdk/concurrent/ForkJoinPool.drawio.png)

  + [**ThreadLocal**](docs/java/jdk/concurrent/ThreadLocal.md)

    源码流程图：

    + [ThreadLocal.drawio](docs/java/jdk/concurrent/ThreadLocal.drawio)
    + [ThreadLocal.drawio.png](docs/java/jdk/concurrent/ThreadLocal.drawio.png)

+ **容器类**

+ **IO**

  + **NIO**

    源码流程图：

    + [java-nio.drawio](docs/java/jdk/io/java-nio.drawio)
    + [java-nio.drawio.png](docs/java/jdk/io/java-nio.drawio.png)
    + [java-nio-overview.png](docs/java/jdk/io/java-nio-overview.png)

    > NIO 使用的多路复用IO模型的逻辑实现其实是在 epoll 这个系统调用里。

### 网关

+ **[Spring Cloud Gateway](docs/java/gateway/SpringCloudGateway原理.md)**

  源码流程图：

  + [spring-cloud-gateway.drawio](docs/java/gateway/spring-cloud-gateway.drawio)
  + [spring-cloud-gateway.drawio.png](docs/java/gateway/spring-cloud-gateway.drawio.png)

  原理简述：

  本质是一个 WebFlux 应用，内部处理逻辑和 Spring MVC 也有点类似，主要也是定义一组请求处理映射（用于定义路由处理，路由包括一组断言和过滤器），通过断言进行请求与路由的匹配，通过路由的过滤器对请求进行处理（比如：修改路径进行转发）。

+ **Zuul**

### 服务调用

+ **Dubbo**

  源码代码量很高，部分组件拆开。

  + **Dubbo主流程**

    源码流程图：

    + [dubbo3.drawio](docs/java/dubbo/dubbo3.drawio) (尚未完成)

  + [Dubbo SPI](docs/java/dubbo/dubbo-spi.md)

    源码流程图：

    + [dubbo3-spi.drawio](docs/java/dubbo/dubbo3-spi.drawio)
    + [dubbo3-spi.png](docs/java/dubbo/imgs/dubbo3-spi.png)

+ **Feign**

+ **Grpc**

+ **Thift**

### 负载均衡

+ **Ribbon**

  源码流程图：

  + [ribbon.drawio](docs/java/ribbon/ribbon.drawio)

+ **Spring Cloud LoadBalancer**

  源码流程图：

  + [spring-cloud-loadbalancer.drawio](docs/java/spring-cloud-loadbalancer/spring-cloud-loadbalancer.drawio)
  + [spring-cloud-loadbalancer.drawio.png](docs/java/spring-cloud-loadbalancer/spring-cloud-loadbalancer.drawio.png)

### 消息队列

+ **Disruptor**

  线程间高性能低延迟消息传递框架。

+ **Kafka**

+ **[RocketMQ](docs/java/message-queue/rocketmq/rocketmq.md)**

  源码流程图：

  + [rocketmq.drawio](docs/java/message-queue/rocketmq/rocketmq.drawio)

    + NameServer: [rocketmq-NameServer.drawio.png](docs/java/message-queue/imgs/rocketmq-NameServer.drawio.png)

    + Broker:  [rocketmq-Broker.drawio.png](docs/java/message-queue/imgs/rocketmq-Broker.drawio.png)
    + Producer: [rocketmq-Producer.drawio.png](docs/java/message-queue/imgs/rocketmq-Producer.drawio.png)
    + Consumer: [rocketmq-Consumer.drawio.png](docs/java/message-queue/imgs/rocketmq-Consumer.drawio.png)

  + [rocketmq-messagestore.drawio](docs/java/message-queue/rocketmq/rocketmq-messagestore.drawio) (消息存储服务原理)

    + MessageStore: [rocketmq-messagestore.drawio.png](docs/java/message-queue/imgs/rocketmq-messagestore.drawio.png)

  + [transaction-message.drawio](docs/java/message-queue/rocketmq/transaction-message.drawio) (事务消息原理)

    + [rocketmq-message.drawio.png](docs/java/message-queue/imgs/rocketmq-message.drawio.png)

+ **RabbitMQ**

### 作业调度

+ **Elastic-Job**

+ **XXL-Job**

  源码流程图：

  + [xxl-job.drawio](docs/java/job-schedule/xxl-job.drawio)

  + [xxl-job-executor.drawio.png](docs/java/job-schedule/xxl-job-executor.drawio.png)

    任务执行器工作流程。

  + [xxl-job-admin.drawio.png](docs/java/job-schedule/xxl-job-admin.drawio.png)

    任务管理器工作流程。

### 服务器

+ **Jetty**

+ **Netty**

  源码流程图：

  + [netty.drawio](docs/java/netty/netty.drawio)
  + [netty.drawio.png](docs/java/netty/netty.drawio.png)

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

### Web框架

+ **Spring**

  + **基于注解的应用上下文**

    这里主要分析 IOC 和 Spring Bean 生命周期。

    源码流程图：

    + [spring-context.drawio](docs/java/spring/spring-context.drawio)
    + [spring-context.drawio.png](docs/java/spring/spring-context.drawio.png)

  + **切面**

  + **事务**

    源码流程图：

    + [spring-transaction.drawio](docs/java/spring/spring-transaction.drawio)
    + [spring-transaction.drawio.png](docs/java/spring/spring-transaction.drawio.png)

  + **测试**

  + **其他**

    分析IOC时有涉及但是不够详细，所以这里重新绘制单独的流程图。

    + **FactoryBean**

      包括FactoryBean和Bean生命周期的详细分析。

      源码流程图：

      + [spring-beans-factorybean.drawio](docs/java/spring/spring-beans-factorybean.drawio)
      + [spring-beans-factorybean.drawio.png](docs/java/spring/spring-beans-factorybean.drawio.png)

    + **循环依赖处理**

      源码流程图：

      + [spring-beans-circular-dependency.drawio](docs/java/spring/spring-beans-circular-dependency.drawio)
      + [spring-beans-circular-dependency.drawio.png](docs/java/spring/spring-beans-circular-dependency.drawio.png)

    + **Bean创建顺序控制**

    + **四种依赖注入实现**

      分析构造器注入、setter方法注入、工厂方法注入（分为静态工厂方法、实例工厂方法 [结合factory-bean使用]）。

    + **五种不同方式的自动装配**

      no、byName、byType、constructor、autodetect。

+ **Spring MVC**

  这里只是分析请求处理流程，Servlet容器初始化流程看Spring Boot的部分（XML配置方式已经不流行了），初始化流程应该也可以参考WebFlux流程图估计差别不大。 

  源码流程图：

  + [spring-mvc.drawio](docs/java/spring/spring-mvc.drawio)
  + [spring-mvc.drawio.png](docs/java/spring/spring-mvc.drawio.png)

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

  之前看源码输出到了Markdown文件，但是内容多了Markdown可读性很差，待补充原理图。

+ **Spring WebFlux**

  源码流程图：

  + [spring-webflux.drawio](docs/java/web/spring-webflux.drawio)

  + [spring-webflux.drawio.png](docs/java/web/spring-webflux.drawio.png)

  原理简述：

  和 SpringMVC 处理流程基本一致，只不过 WebFlux 默认使用 Reactor-Netty 作为Web容器，接口都基于 ProjectReactor 封装成了响应式接口，支持 HTTP 协议 和 自定义协议通信。

  与 SpringMVC 的 `DispatcherServlet` 对应有一个 `DispatcherHandler` 类，这个类定义了请求处理的主要流程。

  > 前面说 Tomcat NIO 模式下请求处理的部分操作依然是阻塞的，只要有操作是阻塞的就会白占线程，降低系统吞吐量，而通过多创建线程提升系统吞吐量则又会引入线程上下文切换的开销。

### 响应式

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

### 微服务框架

+ **Helidon**

+ **Micronaut**

+ **Quarkus**
+ **Spring Cloud**

### 注册中心/配置中心

+ **Apollo**

+ **Nacos**

  源码流程图：

  + [nacos.drawio](docs/java/nacos/nacos.drawio)

### 分布式协调

+ **Zookeeper**
  + 源码流程图：

    + [zookeeper-server.drawio](docs/java/zookeeper/zookeeper-server.drawio)
    
      > 很久之前画的没有UML，而且图太过罗嗦，TODO 重画。


### 服务保障

+ [**jd-hotkey**](docs/java/jd-hotkey/jd-hotkey.md)

  源码流程图：

  + [jd-hotkey.drawio](docs/java/jd-hotkey/jd-hotkey.drawio)

  + [jd-hotkey.drawio.png](docs/java/jd-hotkey/imgs/jd-hotkey.drawio.png)

  业务使用：

  + [jd-hotkey-usage.png](docs/java/jd-hotkey/imgs/jd-hotkey-usage.png)

+ [**Sentinel**](docs/java/sentinel/sentinel-workflow.md)

  源码流程图：

  + [sentinel.drawio](docs/java/sentinel/sentinel.drawio)
  + [sentinel.drawio.png](docs/java/sentinel/sentinel.drawio.png)

+ Hystrix

### 链路追踪

+ **Skywalking**

  + **Agent**

    源码流程图：

    + [skywalking-agent.drawio](docs/java/skywalking/skywalking-agent.drawio)
    + [skywalking-agent.drawio.png](docs/java/skywalking/skywalking-agent.drawio.png)

+ **Sleuth**

### 监控系统

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

### 安全框架

+ **Apache Shiro**

+ **[Spring Security](docs/java/spring-security/spring-security.md)**

  + 主流程

    源码流程图：

    + [spring-security.drawio](docs/java/spring-security/spring-security.drawio)
    + [spring-security.drawio.png](docs/java/spring-security/spring-security.drawio.png)

  + OAuth2

    源码流程图：

    + [spring-security-oauth2.drawio](docs/java/spring-security/spring-security-oauth2.drawio)
    + [spring-security-oauth2.drawio.png](docs/java/spring-security/spring-security-oauth2.drawio.png)

+ **JCasbin**

### ORM框架

+ **Mybatis**

  + 主流程

    源码流程图：

    + [mybatis.drawio](docs/java/mybatis/mybatis.drawio)
    + [mybatis.drawio.png](docs/java/mybatis/mybatis.drawio.png)

    > 也是好久之前画的图，UML不详细，SQL前后置处理以及连接池部分还有细节逻辑没有梳理，TODO 重画。

  + Mybatis-Spring

+ **Mybatis Plus**

### 数据库相关

+ **Canal**

  > 之前梳理了个半成品，TODO 重画。

+ **Sharding-JDBC**

  > TODO 流程图迁移。

+ **连接池**
  + **Druid**
  + **HikariCP**

### 分布式事务

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

  前身是Fescar。

  源码流程图：

  + [seata.drawio](docs/java/seata/seata.drawio)  (尚未完成)

+ **TCC-Transaction**

### 搜索引擎

+ **ElasticSearch**

### 规则引擎

+ **Drools**

### 工作流引擎

+ **Activiti**
+ **Spring Flowable**

### 缓存

+ **Caffeine**
+ **Ehcache**

### 分布式

+ **一致性协议**

  + **[Raft](docs/java/distributed/consistency/raft.md)**

    说实话想自己按 Raft 论文写一个实现也不是个简单的事，关键是需要将论文中的所有细节理解清楚，里面有一些证明能否理解透彻，即便写出来，怎么测试实现的没有漏洞也是个问题。

    + **BRaft**
  
      Raft协议的C++实现。
  
    + **JRaft**
  
      看 Nacos 当前最新版本（2.3.2）中依赖的是 SOFAJRaft，简称JRaft，是参考 BRaft 的 Java 实现。
  
    + **Raft4J**
  
    + **Raft-Java**
  
      这个只是DEMO性质的实现，主要是代码实现简单（5K多行，其他生产级别的实现都是几W行），适合快速学习Raft协议细节。
  
      源码流程图：
    
      + [raft-java.drawio](docs/java/distributed/consistency/raft-java.drawio)
      + [raft-java.drawio.png](docs/java/distributed/consistency/raft-java.drawio.png)
  

### 命令行

+ **JCommander**
+ **CLI**

### 日志

+ **Slf4j**

+ **Log4j2**

+ **Logback**

  虽然 logback-core、logback-classic 代码量加起来有 6W 行，但是核心流程比较简单，且数据结构很清晰，直接看源码可能比看官方文档和书能更快地理解 Logback 的工作原理和使用细节。

  源码流程图：

  + [logback.drawio](docs/java/logging/logback.drawio)
  + [logback.drawio.png](docs/java/logging/imgs/logback.drawio.png)

### 工具类

+ **Arthas**

+ **EasyExcel**

+ **Fastjson**

+ **Guava**

+ **Hutool**

+ **jvm-sandbox**

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

      多数锁继承 RedissonLock，梳理清 RedissonLock 再看其他锁的实现会简单很多。

    + [redisson-lock-RedissonReadWriteLock.drawio.png](docs/java/redisson/redisson-lock-RedissonReadWriteLock.drawio.png)

    + [redisson-lock-RedissonFencedLock.drawio.png](docs/java/redisson/redisson-lock-RedissonFencedLock.drawio.png)

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

+ **JETCD**

  + **分布式锁**

    对比Redisson实现（TODO）。


### 语法解析器

+ **Antlr**

  Sharding-JDBC 中 借助 Antlr 实现对 SQL 语句的重构（原SQL -> AST -> 新SQL），将 SQL 转换成针对某个分表的 SQL。



## Go

### SDK

### 注册中心/配置中心

+ **Consul**
+ **Etcd**

### 数据库相关

+ [**Godis**](docs/go/godis)

### 微服务

+ **Istio**

+ **K8S**

### 分布式事务

+ **DTM**



## Rust



## Resources

### Books

