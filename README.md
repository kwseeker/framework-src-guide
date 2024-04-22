# 后端技术栈框架源码分析

这里主要分析Java和Go后端技术栈常用框架的源码，探究内部工作原理。

源码分析输出主要为`drawio`流程图（`.drawio`文件）和 `Markdown`文本，以流程图为主（主要展示框架主流程），Markdown作为补充，详细内容参考[docs](./docs)。

这里的流程图不是常规流程图，实际是借鉴的时序图的编排方式，另外还添加了简单的`UML`图。

**读框架源码的初衷**：

+ **解决生产环境BUG**

  不熟悉源码原理，可能即使定位到问题所在代码行也不知道应该怎么改。

+ **从0到1构建项目时，定位导致产生不符合预期结果的原因**

  这个碰到太多了，通过Google、官方文档可以解决大部分，但还是有一些无法查到的问题。

+ **拓展框架功能**（很多框架都有预留一些扩展点）

  拓展方式比如接口、SPI、插件、JVMTI Agent什么的。

+ **更好更合理地使用框架**

  几乎所有框架的文档都没法将框架所有功能的细节都讲解清楚。

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
    - 消息队列
    - 作业调度
    - 服务器
    - Web框架
    - 微服务框架
  
    + 注册中心/配置中心
    + 服务保障
  
    + 链路追踪
  
    + 安全框架
    + ORM框架
    + 数据库相关]
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

    代码量较小，很容易理解。不过也仅仅只包含一些很核心的功能实现（.class文件解析、类加载、实例化、Native方法、方法调用、异常处理），GC、多线程、双亲委托、JIT优化等等都没有。

    其实感觉对深入理解JVM帮助不是很大，但是比较适合初学者。

    Github Repo: 

    + [guxingke/mini-jvm](https://github.com/guxingke/mini-jvm)
    
    源码流程图：
    
    + [mini-jvm.drawio](docs/java/jvm/mini-jvm.drawio)
    + [mini-jvm.drawio.png](docs/java/jvm/mini-jvm.drawio.png)

+ **并发**

  + **AbstractQueuedSynchronizer**

    源码流程图：

    + [AbstractQueuedSynchronizer.drawio](docs/java/jdk/concurrent/AbstractQueuedSynchronizer.drawio)
    + [AbstractQueuedSynchronizer.drawio.png](docs/java/jdk/concurrent/AbstractQueuedSynchronizer.drawio.png)

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

    + [ForkjoinPool.drawio](docs/java/jdk/concurrent/ForkjoinPool.drawio)
    + [ForkjoinPool.drawio.png](docs/java/jdk/concurrent/ForkjoinPool.drawio.png)

  + [ThreadLocal](docs/java/jdk/concurrent/ThreadLocal.md)

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

+ **Spring Cloud Gateway**
+ **Zuul**

### 服务调用

+ **Dubbo**
+ **Feign**
+ **Grpc**
+ **Thift**

### 消息队列

+ **Kafka**
+ **RocketMQ**
+ **RabbitMQ**

### 作业调度

+ **Elastic-Job**
+ **XXL-Job**

### 服务器

+ **Jetty**

+ **Netty**

  源码流程图：

  + [netty.drawio](docs/java/netty/netty.drawio)
  + [netty.drawio.png](docs/java/netty/netty.drawio.png)

+ **Tomcat**

### Web框架

+ **Spring**
+ **Spring MVC**
+ **Spring Boot**
+ **Spring WebFlux**

### 响应式

+ **Reactor**
+ **RxJava**

### 微服务框架

+ **Spring Cloud**

### 注册中心/配置中心

+ **Apollo**
+ **Nacos**
+ **Zookeeper**

### 服务保障

+ [**Sentinel**](docs/java/sentinel/sentinel-workflow.md)

  源码流程图：

  + [sentinel.drawio](docs/java/sentinel/sentinel.drawio)
  + [sentinel.drawio.png](docs/java/sentinel/sentinel.drawio.png)

### 链路追踪

+ **Skywalking**

### 安全框架

+ [**Apache Shiro**](docs/java/shiro)
+ **Spring Security**
+ **JCasbin**

### ORM框架

+ **Mybatis**
+ **MybatisPlus**

### 数据库相关

+ **Canal**

### 分布式事务

### 搜索引擎

+ **ElasticSearch**

### 规则引擎

+ **Drools**

### 工作流引擎

+ **Activiti**
+ **Spring Workflow**

### 缓存

+ **Caffeine**
+ **Ehcache**

### 分布式

+ **一致性协议**

  + **BRaft**

  + **Raft4J**

  + **JRaft**

### 命令行

+ **JCommander**

### 日志

+ **Slf4j**
+ **Log4j2**
+ **Logback**

### 工具类

+ **Arthas**
+ **EasyExcel**
+ **Fastjson**
+ **Guava**
+ **Hutool**
+ **jvm-sandbox**
+ **MapStruct**

### 语法解析器

+ **Antlr**



## Go

### SDK

### 注册中心/配置中心

+ **Consul**
+ **Etcd**

### 数据库相关

+ [**Godis**](docs/go/godis)



## Resources

### Books

