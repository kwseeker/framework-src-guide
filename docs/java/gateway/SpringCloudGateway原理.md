# Gateway 原理篇

源码版本：v3.1.8 ([Reference Doc](https://docs.spring.io/spring-cloud-gateway/docs/3.1.8/reference/html/))，这是支持JDK8的最后一个版本，4.x之后直接在JDK17上开发，基于SpringBoot v2.6.15开发。

前提：需要有WebFlux基础，理解WebFlux工作原理。

首先整理清思路：Gateway终究也只是个**基于WebFlux的Web应用**, 应该和MVC一样有请求Handler Mapping，根据应用篇对Gateway的了解，这里研究Gateway源码就需要先找到这个Handler Mapping 的代码，然后要看路由route组件（包括内部的断言、过滤器）怎么加载执行的，最后可能还需要看下其他限流组件、服务发现组件是怎么起作用的。

即：

+ WebFlux 应用请求处理流程
+ Gateway Handler Mapping 入口
+ Gateway 路由组件加载 & 执行
+ Gateway 其他组件加载 & 执行
  + 处理器
  + 过滤器



## 源码调试准备

### 工程结构

#### v2.0.0.M4 

> 本来找到了这个别人已经加了源码注释的版本，想加快源码阅读进度，但是发现代码太老了，和新项目已经脱节了，遂放弃这个版本，转用v3.1.8。

包含5个模块：

+ **spring-cloud-gateway-core**

  网关逻辑实现。

+ **spring-cloud-gateway-mvc**

  spring-cloud-gateway-mvc是Spring Cloud Gateway的一个插件，它提供了一种在Spring MVC应用程序中使用Gateway的方式。通过使用spring-cloud-gateway-mvc插件，你可以在现有的Spring MVC应用程序中引入Gateway的功能。

+ **spring-cloud-starter-gateway**

  starter 模块只依赖 spring-cloud-gateway-core、spring-boot-starter-webflux。

  spring.provides 定义：

  ```yaml
  # 这里说明自动配置类也是在core中定义的
  provides: spring-cloud-gateway-core
  ```

+ **spring-cloud-gateway-dependencies**

  和其他 dependencies 模块一样用于统一管理各个模块的相同依赖项，简化配置。

+ **spring-cloud-gateway-sample**

#### v3.1.8

包含7个模块（忽略docs）：

+ **spring-cloud-gateway-server**

  对应旧版本的spring-cloud-gateway-core。

+ **spring-cloud-gateway-mvc**

+ **spring-cloud-gateway-webflux**

+ **spring-cloud-gateway-dependencies**

+ **spring-cloud-starter-gateway**

+ **spring-cloud-gateway-integration-tests**

+ **spring-cloud-gateway-sample**

### 源码启动的坑

#### v2.0.0.M4 

启动后访问 http://127.0.0.1:8080/image/webp 会显示一个狼头图片。

启动过程遇到的坑：

+ **依赖下载失败**

  需要选上Maven -> Profiles -> spring。

  这里详细说下这个Profile是做什么的。

  之前看Maven使用方法（参考：[Maven Profile](https://github.com/kwseeker/cloud-native/blob/master/build-tools/Maven.md#maven-profile)），知道这里的Profile是POM文件中配置的，用于做个性化配置的，可以选择是否激活。

  从下面定义上看是注册了Maven依赖和插件仓库，然后你肯定会立马反应过来，因为这里选择的源码版本并不是正式版本，而默认的Maven仓库是不包含非正式版本的依赖的，所以肯定会出现依赖下载失败的问题，所以需要注册下面非正式版本依赖的仓库地址。

  > 关于Spring 项目管理周期：M(Milestone) RC GA 的区别，参考：https://stackoverflow.com/questions/32356053/what-is-a-spring-milestone， 我理解M RC 也归属于SNAPSHOT, 只有GA是RELEASE版本。

  ```xml
  <profile>
      <id>spring</id>
      <repositories>
          <repository>
              <id>spring-snapshots</id>
              <name>Spring Snapshots</name>
              <url>https://repo.spring.io/libs-snapshot-local</url>
              <snapshots>
                  <enabled>true</enabled>
              </snapshots>
              <releases>
                  <enabled>false</enabled>
              </releases>
          </repository>
          <repository>
              <id>spring-milestones</id>
              <name>Spring Milestones</name>
              <url>https://repo.spring.io/libs-milestone-local</url>
              <snapshots>
                  <enabled>false</enabled>
              </snapshots>
          </repository>
          <repository>
              <id>spring-releases</id>
              <name>Spring Releases</name>
              <url>https://repo.spring.io/release</url>
              <snapshots>
                  <enabled>false</enabled>
              </snapshots>
          </repository>
      </repositories>
      <pluginRepositories>
          <pluginRepository>
              <id>spring-snapshots</id>
              <name>Spring Snapshots</name>
              <url>https://repo.spring.io/libs-snapshot-local</url>
              <snapshots>
                  <enabled>true</enabled>
              </snapshots>
              <releases>
                  <enabled>false</enabled>
              </releases>
          </pluginRepository>
          <pluginRepository>
              <id>spring-milestones</id>
              <name>Spring Milestones</name>
              <url>https://repo.spring.io/libs-milestone-local</url>
              <snapshots>
                  <enabled>false</enabled>
              </snapshots>
          </pluginRepository>
          <pluginRepository>
              <id>spring-releases</id>
              <name>Spring Releases</name>
              <url>https://repo.spring.io/libs-release-local</url>
              <snapshots>
                  <enabled>false</enabled>
              </snapshots>
          </pluginRepository>
      </pluginRepositories>
  </profile>
  ```

+ **Kotlin版本太低**

  启动时提示`Kotlin: Language version 1.1 is no longer supported; please, use version 1.3 or greater`，Spring Cloud Gateway 在定义RouteLocator等组件时有使用Kotlin。

  依赖列表中可以看到有下面依赖，可以实现Java调用Kotlin。

  > 据说现在Android开发存在大量 Java Kotlin 混合编程。

  ```properties
  org.jetbrains.kotlin:kotlin-stdlib:1.1.51
  org.jetbrains.kotlin:kotlin-reflect:1.1.51
  ```

  只需要修改根目录和`spring-cloud-gateway-sample`模块下POM文件，指定一个更高的支持版本。

  ```xml
  <kotlin.version>1.6.21</kotlin.version>
  ```


#### v3.1.8



## WebFlux应用请求处理流程

参考 kwseeker/async/async-program/doc/spring-webflux-workflow.drawio 流程图。

这里简述下：

1. Netty EventLoop线程通过本地方法监听到硬件中断事件，然后读取连接（socket文件）数据获取请求数据； 
2. 请求数据交由此连接的 Netty ChannelHandler pipeline 处理，其中包含网关处理逻辑的 ChannelHandler 是 ChannelOperationsHandler;
3. 请求在 ChannelOperationsHandler 内部经过 ReactorHttpHandlerAdapter 适配（转成响应式）最终交给 DispatcherHandler 处理。DispatcherHandler 是一组请求处理器和处理结果处理器的容器，查找与请求匹配的处理器并执行请求处理，然后获取结果，再查找与响应类型匹配的结果处理器，对结果进行处理（比如格式转换、序列化）；最终将结果写入响应式Response，然后再写回原始的Response对象（封装了socket的本地方法write()）。

这里研究Gateway各种组件的原理，就是研究这些组件是怎么注册到 DispatcherHandler、以及被调用的。



## WebFlux DispatcherHandler 入口

`spring-cloud-gateway-sample` 中定义了一个Rest接口：`/localcontroller`，这里加断点，看 Gateway请求处理流程。

调用栈：

```
localController:87, GatewaySampleApplication$TestConfig (org.springframework.cloud.gateway.sample)
...
doInvoke:243, InvocableHandlerMethod (org.springframework.web.reactive.result.method)
lambda$invoke$0:138, InvocableHandlerMethod (org.springframework.web.reactive.result.method)
apply:-1, 30366803 (org.springframework.web.reactive.result.method.InvocableHandlerMethod$$Lambda$870)
...
applyHandler:381, ChannelOperations (reactor.ipc.netty.channel)
onHandlerStart:397, HttpServerOperations (reactor.ipc.netty.http.server)
...
fireChannelRead:965, DefaultChannelPipeline (io.netty.channel)
...
processSelectedKeys:497, NioEventLoop (io.netty.channel.nio)
run:459, NioEventLoop (io.netty.channel.nio)
run:886, SingleThreadEventExecutor$5 (io.netty.util.concurrent)
run:750, Thread (java.lang)
```



## Gateway 路由组件

还是从 spring-cloud-starter-gateway 开始分析。

其 pom.xml 引入了三个主要的依赖

```xml
spring-cloud-gateway-core
spring-boot-starter-webflux
spring-cloud-starter-loadbalancer
```

**先看spring-cloud-gateway-core：**

spring.factories定义

```properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.gateway.config.GatewayClassPathWarningAutoConfiguration,\
org.springframework.cloud.gateway.config.GatewayAutoConfiguration,\
org.springframework.cloud.gateway.config.GatewayHystrixCircuitBreakerAutoConfiguration,\
org.springframework.cloud.gateway.config.GatewayResilience4JCircuitBreakerAutoConfiguration,\
org.springframework.cloud.gateway.config.GatewayLoadBalancerClientAutoConfiguration,\
org.springframework.cloud.gateway.config.GatewayNoLoadBalancerClientAutoConfiguration,\
org.springframework.cloud.gateway.config.GatewayMetricsAutoConfiguration,\
org.springframework.cloud.gateway.config.GatewayRedisAutoConfiguration,\
org.springframework.cloud.gateway.discovery.GatewayDiscoveryClientAutoConfiguration,\
org.springframework.cloud.gateway.config.SimpleUrlHandlerMappingGlobalCorsAutoConfiguration,\
org.springframework.cloud.gateway.config.GatewayReactiveLoadBalancerClientAutoConfiguration

org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.cloud.gateway.config.GatewayEnvironmentPostProcessor
```

GatewayAutoConfiguration 中定义核心功能：



## Gateway 其他组件

### 处理器

### 过滤器

