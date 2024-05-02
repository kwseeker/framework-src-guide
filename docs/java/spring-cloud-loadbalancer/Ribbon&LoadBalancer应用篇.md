# Ribbon & LoadBalancer 应用篇

Ribbon

+ [github](https://github.com/Netflix/ribbon)

Spring Cloud LoadBalancer

+ [Getting Started Guide](https://spring.io/guides/gs/spring-cloud-loadbalancer/) 

+ [Reference Doc v3.1.5](https://docs.spring.io/spring-cloud-commons/docs/3.1.5/reference/html/#spring-cloud-loadbalancer)

涉及的测试DEMO参考： SpringBoot-Labs/loadbalancer-ribbon。

使用方法这里不详述了，看官方文档就行了。



## 简介

### Ribbon

Ribbon 是一个进程间通信的客户端库，该库包括 Netflix 客户端负载均衡器和中间层服务客户端。

提供以下功能：

+ 负载均衡

  Ribbon实现的是客户端负载均衡，对应还有服务端负载均衡，比如：Nginx负载均衡。

  相比来说，客户端级别的负载均衡可以有**更好的性能**，因为不需要多经过一层代理服务器。并且，服务端级别的负载均衡需要额外考虑代理服务的高可用，以及请求量较大时的负载压力。因此，在微服务场景下，一般采用客户端级别的负载均衡为主。

+ 容错机制

+ 支持 HTTP、TCP、UDP 等多种协议，并支持异步和响应式的调用方式

+ 缓存与批处理

主要模块：

```txt
├── ribbon				//API定义模块，在其他Ribbon模块和Hystrix之上集成负载平衡、容错、缓存/批处理等功能的API
├── ribbon-archaius		//配置库
├── ribbon-core			//客户端配置API和其他共享API
├── ribbon-eureka		//使用Eureka客户端为云提供动态服务器列表的API 
├── ribbon-loadbalancer	//负载平衡器API，可以独立使用，也可以与其他模块配合使用
├── ribbon-evcache		
├── ribbon-guice
├── ribbon-httpclient	//基于Apache HttpClient构建的REST客户端，与负载均衡器集成（已弃用）
├── ribbon-transport	//使用具有负载均衡功能的RxNetty传输，支持 HTTP、TCP 和 UDP 协议的客户端
├── ribbon-test
├── ribbon-examples
```

以 v2.7.18（最后的版本，Ribbon已经停止了迭代更新）为例，分析使用和原理。

貌似没有官方文档，只有Github上的README和Wiki且太简略。这里就自己从源码中发掘使用方法。

### Spring Cloud LoadBalancer

由于Netflix Ribbon 已经停止了迭代更新，Spring Cloud 自己实现了 LoadBalancer 用于替代 Netflix Ribbon。

位于`spring-cloud-commons`组件中, package: `org.springframework.cloud.client.loadbalancer`。

后面切换版本测试的时候会发现较新的版本（这里测试的Spring Cloud 2021.0.5）已经没有`spring-cloud-starter-netflix-ribbon`这个依赖了，取而代之的是`spring-cloud-starter-loadbalancer`。



## Ribbon单独使用

这里可以参考官方的例子ribbon-examples。代码位于：SpringBoot-Labs/netflix-ribbon/ribbon-examples。

### LoadBalancerCommand

### 功能定制



## Spring Cloud Ribbon

### 版本差异

Spring Cloud 旧版本使用的`spring-cloud-starter-netflix-ribbon`， 较新的版本使用 `spring-cloud-starter-loadbalancer`。

```xml
<!-- 父POM -->
<!-- 原测试版本 -->
<properties>
    <spring-boot.version>2.2.4.RELEASE</spring-boot.version>
    <spring-cloud.version>Hoxton.SR1</spring-cloud.version>
    <spring-cloud-alibaba.version>2.2.0.RELEASE</spring-cloud-alibaba.version>
</properties>
<!-- 最后的支持JDK8的Spring Cloud Alibaba版本 -->
<properties>
    <spring-boot.version>2.6.13</spring-boot.version>
    <spring-cloud.version>2021.0.5</spring-cloud.version>
    <spring-cloud-alibaba.version>2021.0.5.0</spring-cloud-alibaba.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-parent</artifactId>
            <version>${spring-boot.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>${spring-cloud-alibaba.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- 使用旧版本的客户端POM -->
<!-- 引入 Spring Cloud Alibaba Nacos Discovery 相关依赖，将 Nacos 作为注册中心，并实现对其的自动配置，旧版本会自动引入 spring-cloud-starter-netflix-ribbon -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

### 客户端负载均衡方式

+ **@LoadBalanced RestTemplate** 

+ **@LoadBalanced WebClient**

  提供非阻塞IO、响应式接口等，spring boot 3 之后建议用 WebClient 替代 RestTemplate。

+ **WebClient.builder().filter(lbFunction)** 

  通过手动给客户端注册一个负载均衡过滤器（LoadBalancedExchangeFilterFunction）实现客户端负载均衡。LoadBalancedExchangeFilterFunction 的两种实现：

  + ReactorLoadBalancerExchangeFilterFunction
  + RetryableLoadBalancerExchangeFilterFunction

+ **LoadBalancerClient RestTemplate**

  `LoadBalancerClient extends ServiceInstanceChooser `，即服务实例选择器，负载均衡实现原理是先选择服务实例，再执行到此服务实例的请求。

参考：

- [Spring RestTemplate as a Load Balancer Client](https://docs.spring.io/spring-cloud-commons/docs/3.1.5/reference/html/#rest-template-loadbalancer-client)
- [Spring WebClient as a Load Balancer Client](https://docs.spring.io/spring-cloud-commons/docs/3.1.5/reference/html/#webclinet-loadbalancer-client)
- [Spring WebFlux WebClient with `ReactorLoadBalancerExchangeFilterFunction`](https://docs.spring.io/spring-cloud-commons/docs/3.1.5/reference/html/#webflux-with-reactive-loadbalancer)

### 自定义Ribbon配置

配置分为**客户端级别配置**和**全局配置**。

#### 配置文件

配置文件中定义的配置都是客户端级别的配置，通过在配置文件中添加 `{clientName}.ribbon.{key}={value}` 配置项，对应源码配置类 [CommonClientConfigKey](https://github.com/Netflix/ribbon/blob/master/ribbon-core/src/main/java/com/netflix/client/config/CommonClientConfigKey.java)。

常用配置：

```yaml
service-application-name:	# 这里是服务提供者的应用名称
  ribbon: 
    # 制定使用的负载均衡规则
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule # 负载均衡规则全类名
    
```

#### Spring JavaConfig 配置类

#### 功能定制

### 第三方支持

服务注册与发现框架基本都有对负载均衡接口的支持。

#### Nacos



## Spring Cloud LoadBalancer

[Getting Started Guide](https://spring.io/guides/gs/spring-cloud-loadbalancer/) : 讲了需要哪些依赖，如何自定义用于负载均衡的固定的服务实例列表（@LoadBalancerClient + ServiceInstanceListSupplier），客户端服务怎么负载均衡访问固定的基础服务（@LoadBalanced 或者 WebClient.builder().filter(lbFunction)）。

[Reference Doc v3.1.5](https://docs.spring.io/spring-cloud-commons/docs/3.1.5/reference/html/#spring-cloud-loadbalancer)

### 依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
<!-- 较新的版本使用 loadbalancer 取代 ribbon, 较新的版本spring-cloud-starter-alibaba-nacos-discovery不会自动引入spring-cloud-starter-loadbalancer -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

### 客户端负载均衡方式

同旧版本。

### 自定义LoadBalancer配置

#### 配置文件

对应类`class LoadBalancerClientsProperties extends LoadBalancerProperties`的属性。

约定的默认配置：

```properties
properties = {LoadBalancerClientsProperties@6009} 
 clients = {HashMap@6030}  size = 0
 healthCheck = {LoadBalancerProperties$HealthCheck@6031} 
  initialDelay = {Duration@6037} "PT0S"
  interval = {Duration@6038} "PT25S"
  refetchInstancesInterval = {Duration@6039} "PT25S"
  path = {LinkedCaseInsensitiveMap@6040}  size = 0
  port = null
  refetchInstances = false
  repeatHealthCheck = true
 hint = {LinkedCaseInsensitiveMap@6032}  size = 0
 hintHeaderName = "X-SC-LB-Hint"
 retry = {LoadBalancerProperties$Retry@6034} 
  enabled = true
  retryOnAllOperations = false
  maxRetriesOnSameServiceInstance = 0
  maxRetriesOnNextServiceInstance = 1
  retryableStatusCodes = {HashSet@7051}  size = 0
  backoff = {LoadBalancerProperties$Retry$Backoff@7052} 
 stickySession = {LoadBalancerProperties$StickySession@6035} 
  instanceIdCookieName = "sc-lb-instance-id"
  addServiceInstanceCookie = false
 useRawStatusCodeInResponseData = false
 xForwarded = {LoadBalancerProperties$XForwarded@6036} 
  enabled = false
```

配置项说明：

```yaml
spring:
  cloud:
    loadbalancer:
      # 是否开启负载均衡功能，默认为true
      enabled: true
      # LoadBalancer缓存，不配置默认使用默认的LoadBalancer缓存实现DefaultLoadBalancerCache，这时日志中还会看到一条Warn
      cache:
        # 是否开启缓存
        enabled: true
        # 如果类路径中有com.github.ben-manes.caffeine:caffeine，也会使用caffeine的实现
        caffeine:
          # 指定自定义的caffeine规范，TODO
          spec:
        ttl: 35s      # 默认35s
        capacity: 256 # 默认256个
      # 基于区域的负载均衡，依赖服务发现组件
      # 借助 ZonePreferenceServiceInstanceListSupplier 过滤检索到的实例，仅返回同一区域内的实例。如果区域为 null 或同一区域内没有实例，则返回所有检索到的实例
      #zone:
      # 负载均衡服务实例的健康检查
      health-check:
        initial-delay: 0
        interval: 25s
        refetch-instances: false	# 是否自动刷新实例列表，默认为false
        refetch-instances-interval: 25s
        #path:        # 默认使用 /actuator/health 做健康检查，依赖 spring-boot-starter-actuator
          #default:
        #port:        # 设置运行状况检查请求的自定义端口
        repeat-health-check: true
      # 使用 SameInstancePreferenceServiceInstanceListSupplier 总是将请求发给之前选择的实例
      #configurations: same-instance-preference
      # 总是优先选择请求 cookie 中提供的 instanceId 的实例
      configurations: request-based-sticky-session
      sticky-session:
        add-service-instance-cookie: true
        instance-id-cookie-name: instance-id-cookie-name # 如果需要修改 RequestBasedStickySessionServiceInstanceListSupplier Cookie名
      # 用于使用请求头中传递的提示值来过滤服务实例
      hint-header-name: X-SC-LB-Hint
      # 用于请求失败重试
      retry:
        enabled: true # 是否开启重试，默认false
        max-retries-on-same-service-instance: 3
        max-retries-on-next-service-instance: 3
        retryable-status-codes: 403,404,500  # 仅对某些状态码执行重试
        backoff:      # 回退策略，默认只支持响应式实现，非响应式请求需要自行实现LoadBalancedRetryFactory createBackOffPolicy()方法
          enabled: false
      #use-raw-status-code-in-response-data: true
```

配置项目对应着 Spring Cloud LoadBalancer 的一个个功能点：

+ 负载均衡功能开关

+ 负载均衡缓存

+ 基于区域的负载均衡 ZonePreferenceServiceInstanceListSupplier

  ```java
  @Bean
  public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
      ConfigurableApplicationContext context) {
      return ServiceInstanceListSupplier.builder()
          .withDiscoveryClient()
          .withZonePreference()
          .withCaching()
          .build(context);
  }
  ```

+ 负载均衡实例健康检查 HealthCheckServiceInstanceListSupplier

+ 总是使用之前选择的服务实例 SameInstancePreferenceServiceInstanceListSupplier

  ```java
  ServiceInstanceListSupplier.builder().withSameInstancePreference()
  ```

+ 基于请求的粘性会话 RequestBasedStickySessionServiceInstanceListSupplier

  总是优先选择请求 cookie 中提供的 instanceId 的实例。

  ```
  ServiceInstanceListSupplier.builder().withRequestBasedStickySession()
  ```

+ 基于提示的负载均衡 HintBasedServiceInstanceListSupplier

  HintBasedServiceInstanceListSupplier 检查提示请求标头（默认 header-name 为 X-SC-LB-Hint，但您可以通过更改 spring.cloud.loadbalancer.hint-header-name 属性的值来修改它），如果找到提示请求标头，使用标头中传递的提示值来过滤服务实例。TODO 具体的过滤规则？
  
  ```
  ServiceInstanceListSupplier.builder().withHints()
  ```

+ 使用`spring.cloud.loadbalancer.clients.<clientId> `对客户端进行单独配置

  上面配置文件中的配置都是全局的，但是如果想对某个客户端进行独立的配置就可以使用`spring.cloud.loadbalancer.clients.<clientId> `对名为"clientId"的 loadbalancer 进行单独的配置。

+ 请求失败重试

  [retrying failed requests](https://docs.spring.io/spring-cloud-commons/docs/3.1.5/reference/html/#retrying-failed-requests)

#### ServiceInstanceListSupplier 自定义服务实例列表

```java
@Bean
@Primary
ServiceInstanceListSupplier serviceInstanceListSupplier() {
    return new DemoServiceInstanceListSuppler("demo-provider");
}
```

内置的实现：

+ CachingServiceInstanceListSupplier
+ DelegatingServiceInstanceListSupplier
+ DiscoveryClientServiceInstanceListSupplier
+ HealthCheckServiceInstanceListSupplier
+ HintBasedServiceInstanceListSupplier
+ NoopServiceInstanceListSupplier
+ RequestBasedStickySessionServiceInstanceListSupplier
+ RetryAwareServiceInstanceListSupplier
+ SameInstancePreferenceServiceInstanceListSupplier
+ ZonePreferenceServiceInstanceListSupplier

#### 功能定制(拓展点)

##### @LoadBalancerClient & @LoadBalancerClients

@LoadBalancerClient 注解可以传入自定义的负载均衡器客户端配置，为某客户端设置客户端级别的定制的配置。

@LoadBalancerClients 注解则可以传递多个配置。

另外配置文件中也可以通过 `spring.cloud.loadbalancer.configurations`配置 ServiceInstanceListSupplier 的实现。

##### 自定义 LoadBalancer 实现

只需要实现 `ReactorLoadBalancer<ServiceInstance> ` Bean的定义，然后通过 @LoadBalancerClient 加载。

> 官方提示：
>
> 作为 @LoadBalancerClient 或 @LoadBalancerClients 配置参数传递的类不应使用 @Configuration 进行注释，也不应位于组件扫描范围之外。
>
> 否则会成为全局的。

##### LoadBalancerRequestTransformer 请求转换器

https://docs.spring.io/spring-cloud-commons/docs/3.1.5/reference/html/#transform-the-load-balanced-http-request

比如给请求添加个头、参数等等。

##### 为自定义 LoadBalancer 设置生命周期回调

### 第三方支持

#### Nacos

+ **NacosLoadBalancer**

   Nacos实现的负载均衡器，使用**随机权重**算法，支持**总是连接同集群同IP类型服务实例**。

  启用方法：

  ```
  @LoadBalancerClients(defaultConfiguration = NacosLoadBalancerClientConfiguration.class)
  ```

  
