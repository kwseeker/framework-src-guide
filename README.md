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

  几乎所有框架的文档都没法将框架所有功能都讲解清楚。

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

> 下面目录编排参考Github `owesome-java` 和 `owesome-go`。
>
> 后面列举的框架并非都分析过源码（心有余而力不足），有些只是计划，已经分析过源码的都有文档链接。
>
> 有些分析流程图之前记录在其他仓库后续会转移到这里，慢慢补充吧



## Contents

- [Languages](#languages)
  - [Java](#java)
    - [Bean Mapping](#bean-mapping)
    - [Caching](#caching)
    - [CLI](#cli)
    - [Code Generators](#code-generators)
    - [Configuration](#configuration)
    - [Database](#database)
    - [Dependency Injection](#dependency-injection)
    - [Development](#development)
    - [Distributed Applications](#distributed-applications)
    - [Distributed Transactions](#distributed-transactions)
    - [Geospatial](#geospatial)
    - [GUI](#gui)
    - [High Performance](#high-performance)
    - [HTTP Clients](#http-clients)
    - [Imagery](#imagery)
    - [Job Scheduling](#job-scheduling)
    - [JSON](#json)
    - [JVM](#jvm)
    - [Logging](#logging)
    - [Messaging](#messaging)
    - [Microservice](#microservice)
    - [Miscellaneous](#miscellaneous)
    - [Monitoring](#monitoring)
    - [Networking](#networking)
    - [ORM](#orm)
    - [Reactive libraries](#reactive-libraries)
    - [SDK](#sdk)
    - [Search](#search)
    - [Security](#security)
    - [Serialization](#serialization)
    - [Server](#server)
    - [Template Engine](#template-engine)
    - [Utility](#utility)
    - [Version Managers](#version-managers)
    - [Web Crawling](#web-crawling)
    - [Web Frameworks](#web-frameworks)
    - [Workflow Orchestration Engines](#workflow-orchestration-engines)
  - [Go](#go)
    - [Database]()
    - [SDK]()
- [Resources](#resources)
  - [Books](#books)



## Java

### Bean Mapping

+ MapStruct

### Caching

+ Caffeine
+ EhcacheDatabase

### CLI

+ JCommander

### Code Generators
### Configuration
### Database
### Dependency Injection
### Development
### Distributed Applications
### Distributed Transactions
### Geospatial
### GUI
### High Performance
### HTTP Clients
### Imagery
### Job Scheduling
### JSON
### JVM
### Logging
### Messaging
### Microservice
### Miscellaneous
### Monitoring
### Networking
### ORM
### Reactive libraries
### SDK

+ 并发
  + [CompletableFuture](docs/java/jdk/concurrent/CompletableFuture.md)
  + [ForkJoinPool](docs/java/jdk/concurrent/ForkJoinPool.md)


### Search
### Security

处理安全性、身份验证、授权或会话管理的库。

+ [Apache Shiro](docs/java/shiro)

### Serialization
### Server
### Template Engine
### Utility
### Version Managers
### Web Crawling
### Web Frameworks
### Workflow Orchestration Engines



## Go

### Database

+ [Godis](docs/go/godis)

### SDK



## Resources

### Books

