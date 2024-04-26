# Gateway 应用篇

官方文档：[Spring Cloud Gateway](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/)

Spring Cloud Gateway 构建于 Spring Boot、Spring WebFlux 和 Project Reactor 之上。

引入 spring-cloud-starter-gateway 但是不想开启网关可以设置 spring.cloud.gateway.enabled=false。

常见网关框架对比：

| 中间件       | Nginx               | Kong                                                 | Netflix Zuul                           | Spring Cloud Gateway              | shenyu                       |
| :----------- | :------------------ | :--------------------------------------------------- | :------------------------------------- | :-------------------------------- | :--------------------------- |
| 主要开发语言 | C                   | Lua                                                  | Java                                   | Java                              | Java                         |
| 依赖关系     | 无                  | 基于 Nginx_Lua模块                                   | 无                                     | 无                                | 无                           |
| 支持协议     | HTTP                | HTTP, GRPC                                           | HTTP,                                  | HTTP                              | HTTP, WebSocket, Dubbo, GRPC |
| 扩展         | 基于 Lua 脚本       | 基于 Lua 脚本                                        | Java 写过滤器                          | Java 写过滤器、断言               | Java 写插件                  |
| 编程模型     | 多进程 + io多路复用 | 基于 Nginx                                           | Zuul 1 采用 Servlet, Zuul 2 采用 Netty | Spring WebFlux（Netty Reactor）   | Netty Reactor                |
| 配置页面     | 无                  | 丰富                                                 | 无                                     | 无                                | 丰富                         |
| 负载均衡     | 写死的              | 支持 Consul(间接可以支持使用 Consul 的 Spring Cloud) | Spring Cloud 相关                      | Spring Cloud 相关                 | 通过各种插件实现             |
| GitHub       | nginx/nginx         | Kong/kong                                            | Netflix/zuul                           | spring-cloud/spring-cloud-gateway | apache/incubator-shenyu      |

> 后面将 Spring Cloud Gateway 简称 SCG。
>
> 测试版本：v3.1.8。



## 工作原理

### 网关负责的功能

+ 路由转发
+ 认证
+ 熔断
+ 限流
+ 日志监控

### SCG实现原理

SCG 基于 WebFlux 响应式Web框架，WebFlux内默认使用Netty异步Web容器。

**特征**：

+ 基于Spring Framework 5, Project Reactor 和 Spring Boot 2.0 进行构建；

+ 动态路由：能够匹配任何请求属性；

+ 集成 Spring Cloud 服务发现功能；

+ 可以对路由指定 Predicate（断言）和 Filter（过滤器）；

+ 易于编写的 Predicate（断言）和 Filter（过滤器）；

+ 集成Hystrix的断路器功能；

+ 请求限流功能；

+ 支持路径重写。

#### 工作流程

官方配图，有点粗略。

<img src="https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/images/spring_cloud_gateway_diagram.png" style="zoom: 80%;" />

+ **路由（route)**
  路由是网关中最基础的部分，路由信息包括一个ID、一个目的URI（被代理微服务URI）、一组断言工厂、一组Filter组成。如果断言为真，则说明请求的URL和配置的路由匹配。
  
  > 即断言和过滤器是路由的组成部分，路由中定义的过滤器是局部过滤器。
  
+ **断言(predicates)**
  Java8中的断言函数，SpringCloud Gateway中的断言函数类型是Spring5.0框架中的ServerWebExchange。断言函数允
  许开发者去定义匹配Http request中的任何信息，比如请求头和参数等。
  
+ **过滤器（Filter)**
  SpringCloud Gateway中的filter分为Gateway FilIer和Global Filter。Filter可以对请求和响应进行处理。

相关配置示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: echo_route                  # 路由ID，全局唯一，建议配合服务名
          uri: http://localhost:8080      # 被代理的微服务的请求地址和端口
          #uri: lb://srv-echo              # lb 整合负载均衡器ribbon,loadbalancer
          predicates:					  # 断言工厂，用于路由匹配
            # Path路径匹配
            - Path=/echo/**
            # 匹配在指定的日期时间之后发生的请求，入参是ZonedDateTime类型
            #- After=2021-05-16T20:50:57.511+08:00[Asia/Shanghai]
            # Cookie匹配
            #- Cookie=username, fox
            # Header匹配  请求中带有请求头名为 x-request-id，其值与 \d+ 正则表达式匹配
            #- Header=X-Request-Id, \d+
            # 自定义CheckAuth断言工厂
            #- name: CheckAuth
            #  args:
            #  name: fox
            #- CheckAuth=monkey
          filters:							 # 过滤器工厂,对匹配此路由的请求做额外处理（比如修改请求信息、认证等）
            - AddRequestHeader=X-Request-color, red # 添加请求头
            - AddRequestParameter=color, blue       # 添加请求参数
            #- PrefixPath=/echo                     # 添加前缀，对应微服务需要配置context-path
            #- RedirectTo=302, http://baidu.com     # 重定向到百度
            #- CheckAuth=fox,男                      # 配置自定义的过滤器工厂
```



官方原理图太粗略重新画一张。



常与服务注册中心配合使用。



## SCG 各功能配置

Spring Cloud Gateway 中可以实现诸如：限流、熔断、安全认证、负载均衡、监控、跨域、请求日志等等功能。

### [Actuator API](https://docs.spring.io/spring-cloud-gateway/docs/3.1.8/reference/html/#actuator-api)

为了方便测试，先开启gateway端点监控，详细参考 [Actuator API](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/#actuator-api)：

```java
management:
  endpoint:
    gateway:
      enabled: true
  endpoints:
    web:
      exposure:
        include: gateway
```

测试DEMO地址是 http://localhost:9999，对应各端点查看url是：

+ 路由列表 http://localhost:9999/actuator/gateway/routes

  查询指定路由信息：http://localhost:9999/actuator/gateway/routes/{id}

+ 全局过滤器列表 http://localhost:9999/actuator/gateway/globalfilters

+ 路由过滤器列表 http://localhost:9999/actuator/gateway/routefilters

+ 刷新路由缓存  http://localhost:9999/actuator/gateway/refresh

### 路由定义

有下面两种定义路由的方式，如果同一路由规则配置两种方式，虽然都会被加载到**RoutePredicateHandlerMapping**的成员**routeLocator**中，只会使用优先级高的（按order排序）。

#### application.yml + spring.cloud.gateway.routes

#### @Bean + RouteLocatorBuilder 

对于网关的某些用途，通过属性配置（application.yml等）就足够了，但某些生产用例受益于从外部源（例如数据库）加载配置。未来的里程碑版本将具有基于 Spring Data Repositories 的 RouteDefinitionLocator 实现，例如 Redis、MongoDB 和 Cassandra。

从配置中心加载配置应该还是比较常用的。TODO

#### 断言配置定义

+ 快捷方式配置

  ```yaml
  # 属于spring.cloud.gateway.routes的属性
  predicates:
  # Cookie是过滤器的名称，匹配名称为mycookie，值为mycookievalue的Cookie
  - Cookie=mycookie,mycookievalue
  ```

+ 完全拓展的参数

  ```yaml
  #与上面快捷方式配置等同
  predicates:
  - name: Cookie
    args:
    	name: mycookie
    	regexp: mycookievalue
  ```

#### 断言工厂

##### 预定义的断言工厂

位于模块spring-cloud.gateway-server，包org.springframework.cloud.gateway.handler.predicate。

Route中需要指定至少一个断言。

像兜底的路由（其他路由都不匹配就都导向到这个路由）尽量写到后面或者指定一个低优先级的order值。断言按优先级顺序与请求匹配，如果优先级相同按解析顺序匹配。

+ **AfterRoutePredicateFactory**

  指定路由生效时间，指定的时间之前发起的请求都会被拦截。

  日期格式为`ZonedDateTime`，即`2023-10-01T20:00:00.000000+08:00[Asia/Shanghai]`这种格式。

+ **BeforeRoutePredicateFactory**

  指定路由失效时间，指定的时间之后发起的请求都会被拦截。

  日期格式为`ZonedDateTime`。

+ **BetweenRoutePredicateFactory**

  相当于AfterRoutePredicateFactory和BeforeRoutePredicateFactory的组合，两个时间点用逗号分隔。

+ **CookieRoutePredicateFactory**

  路由生效需要传递指定的Cookie名（cookie），Cookie值需要匹配设置的正则表达式（regexp）。

  ```yaml
  predicates:
  - Cookie=chocolate, ch.p
  ```

+ **HeaderRoutePredicateFactory**

  路由生效需要传递指定的Header名（header），Header值需要匹配设置的正则表达式（regexp）。

  ```yaml
  - Header=X-Request-Id, \d+ 	#用于匹配一串数字
  ```

+ **HostRoutePredicateFactory**

  路由生效需要请求的Host匹配一组主机名。

+ **MethodRoutePredicateFactory**

  路由生效需要请求的Method在指定的范围。

  ```yaml
  - Method=GET,POST 	#即此路由只接受GET,POST请求
  ```

+ **PathRoutePredicateFactory**

  路由生效需要请求的路径在指定的范围。包含两个参数：

  + Spring PathMatcher patterns列表

  + matchTrailingSlash （默认true）

    是否匹配尾随的斜杠，比如下面规则，默认可以匹配“/red/1/”, 如果设置false, 则不可以匹配。

    > matchTrailingSlash 配置是不是只能通过完全拓展参数配置？TODO yaml工具解析配置原理。

  ```
  - Path=/red/{segment},/blue/{segment}
  ```

  关于提取路径参数：

  ```java
  Map<String, String> uriVariables = ServerWebExchangeUtils.getUriTemplateVariables(exchange);
  String segment = uriVariables.get("segment");
  ```

+ **QueryRoutePredicateFactory**

  路由生效需要请求带有查询参数，包含两个参数：

  + param 查询参数名
  + regexp 参数值的正则表达式（可选）

  ```yaml
  - Query=color 	#查询参数中需要带有color这个请求参数，比如 ?color=green
  ```

+ **RemoteAddrRoutePredicateFactory**

  路由生效需要请求来源地址匹配设置的值（CIDR 表示法），包含一个参数：

  + sources 来源地址列表 (RemoteAddrRoutePredicateFactory.Config)。

  ```yaml
  - RemoteAddr=192.168.1.1/24,192.168.1.1/24
  ```

  > [CIDR](https://zh.wikipedia.org/zh-cn/%E6%97%A0%E7%B1%BB%E5%88%AB%E5%9F%9F%E9%97%B4%E8%B7%AF%E7%94%B1)表示法：**无类别域间路由**（英语：Classless Inter-Domain Routing，简称**CIDR**[/ˈsaɪdər, ˈsɪ-/](https://zh.wikipedia.org/wiki/Help:英語國際音標)）是一个用于给用户分配[IP地址](https://zh.wikipedia.org/wiki/IP地址)以及在[互联网](https://zh.wikipedia.org/wiki/互联网)上有效地路由IP[数据包](https://zh.wikipedia.org/wiki/数据包)的对IP地址进行归类的方法。

  自定义 RemoteAddressResolver

  默认情况下，RemoteAddr 使用传入请求中的远程地址。如果 Spring Cloud Gateway 位于代理层后面，这可能与实际客户端 IP 地址不匹配。这时可以自定义 RemoteAddressResolver 实现其他方式传入来源地址。官方已经提供了一种基于 X-Forwarded-For 的解决方式，还提供了对应的断言工厂XForwardedRemoteAddrRoutePredicateFactory。

+ **XForwardedRemoteAddrRoutePredicateFactory**

  路由生效需要请求 X-Forwarded-For HTTP标头的值与设置的值匹配。

  ```yaml
  - XForwardedRemoteAddr=192.168.1.1/24
  ```

+ **WeightRoutePredicateFactory**

  用于实现按权重的负载均衡。这个断言一般不单独使用。如果单独使用（路由的断言列表只有这种断言）那么这个路由可以适配所有请求。

  这个断言的一个重要应用是做**灰度发布**，比如服务升级后开始上线需要慢慢灰度，先灰度1%用户到v2, 99%用户仍然请求v1。

+ **CloudFoundryRouteServiceRoutePredicateFactory**

  用于根据Cloud Foundry平台提供的路由服务（Route Service）来匹配请求的路由。

+ **ReadBodyRoutePredicateFactory**

  用于根据请求体的内容来进行路由匹配。

##### 自定义断言工厂

TODO

#### 过滤器

看Spring Cloud Gateway源码可以看到出现了4种过滤器（关于过滤器自动配置和被用到的时机参考源码流程图）：

+ **Web过滤器**（请求处理前的Web过滤器）

  实现 WebFilter 接口；比如 WeightCalcuatorWebFilter。

  Spring Cloud Gateway 本身是基于 WebFlux 实现的一个响应式的Web应用。有着与SpringMVC类似的请求处理逻辑（都基于spring-web模块）也包括Web过滤器，WebFilter 定位对应 SpringMVC的过滤器。

+ **全局过滤器**（属于请求处理流程，针对所有路由）

  实现 GlobalFilter 接口。

  GlobalFilter过滤器链组成了处理到网关的请求（R表示）的主要处理逻辑。比如 URL转换、负载均衡、发起到后端服务的请求（各种schema有特定的过滤器处理），读取后端服务响应写回R的Response，资源释放 等。

+ **Gateway过滤器**（属于请求处理流程）

  实现 GatewayFilter 接口。

  配置文件中通过`spring.cloud.gateway.routes.filters`配置，则作用于当前路由。

  配置文件中通过`spring.cloud.gateway.default-filters`配置，则作用于所有路由。

+ **HttpHeadersFilter**（属于请求处理流程，NettyRoutingFilter中解析请求的Header）

> 注意除了Web过滤器，其他过滤器，名为过滤，都是干的处理器的活。

##### Web过滤器

+ **WeightCalculatorWebFilter**

  Spring Cloud Gateway （v3.1.8）本身只新增了这个WebFilter，调试代码时可能还会看到一些其他模块定义的WebFilter, 比如spring-boot-actuator中定义的MetricsWebFilter。

##### 全局过滤器

全局过滤器（GlobalFilter）和 Gateway过滤器 一起构成了网关请求处理的完整逻辑。

+ ForwardRoutingFilter

+ ReactiveLoadBalancerClientFilter

  用于负载均衡配置。

  ```yaml
  uri: lb://service-echo
  ```

  注意新版本移除了**spring-cloud-start-loadbalancer**依赖，需要手动添加配置。官方文档并没有提醒。

  调试代码会发现没有手动添加此依赖的话，自动配置类并不会自动加载 ReactiveLoadBalancerClientFilter 这个Bean，最终处理请求走的仍然是NoLoadBalancerClientFilter。然后就会抛异常（会被统一异常处理捕获）。

  ```java
  //NoLoadBalancerClientFilter
  if (url != null && ("lb".equals(url.getScheme()) || "lb".equals(schemePrefix))) {
    throw NotFoundException.create(this.use404, "Unable to find instance for " + url.getHost());
  } else {
    return chain.filter(exchange);
  }
  ```

+ NettyRoutingFilter

+ NettyWriteResponseFilter

+ RouteToRequestUrlFilter

+ WebSocketRoutingFilter

+ GatewayMetricsFilter

+ ...

##### Route Gateway过滤器工厂

位于包：org.springframework.cloud.gateway.filter.factory；实现很多，这里只举常用的局部过滤器工厂。

+ AddRequestHeaderGatewayFilterFactory

+ RemoveRequestHeaderGatewayFilterFactory

+ AddRequestParameterGatewayFilterFactory

+ RemoveRequestParameterGatewayFilterFactory

+ AddResponseHeaderGatewayFilterFactory

+ RemoveResponseHeaderGatewayFilterFactory

+ StripPrefixGatewayFilterFactory

  提取请求路径的`parts`，比如Gateway手动的请求路径为`/name/blue/red`, 过滤器指定`- StripPrefix=2`, 则发往后端服务的请求路径为`/red`。

+ DedupeResponseHeaderGatewayFilterFactory

  用于删除重复的响应头。

  包含两个参数：

  + name （Header名，多个响应头用空格分隔）
  + strategy（保留策略，可选参数，包括RETAIN_FIRST（默认）、RETAIN_LAST 和 RETAIN_UNIQUE）

  ```yaml
  - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
  ```

  如上配置在网关 CORS 逻辑和下游逻辑都添加 Access-Control-Allow-Credentials 和 Access-Control-Allow-Origin 响应标头的情况下，这会删除它们的重复值。

+ **SpringCloudCircuitBreakerFilterFactory**

  使用 Spring Cloud CircuitBreaker API 将网关路由包装在断路器中。 Spring Cloud CircuitBreaker 支持多个可与 Spring Cloud Gateway 一起使用的库。 Spring Cloud 开箱即用地支持 Resilience4J。

  包含两个参数：

  + name
  + fallbackUri（可选，当前仅支持schema为forward）

  ```yaml
  # 全拓展参数定义
  - name: CircuitBreaker
  	args:
  		name: myCircuitBreaker
  	fallbackUri: forward:/inCaseOfFailureUseThis
  ```

+ **FallbackHeadersGatewayFilterFactory**

  配合SpringCloudCircuitBreakerFilterFactory使用。后面详细研究。

+ MapRequestHeaderGatewayFilterFactory

  用于传递请求头的值到下游服务。

  包含两个参数：

  + fromHeader
  + toHeader

  ```yaml
  # 这会将 X-Request-Red:<values> 标头添加到下游请求，其中包含来自传入 HTTP 请求的 Blue 标头的更新值。
  - MapRequestHeader=Blue, X-Request-Red
  ```

+ PrefixPathGatewayFilterFactory

  为下游请求路径添加前缀。

  ```yaml
  - PrefixPath=/v2
  ```

+ PreserveHostHeaderGatewayFilterFactory

  为请求添加一个preserveHostHeader=true的属性，路由过滤器会检查该属性以决定是否要发送原始的Host。

+ RequestRateLimiterGatewayFilterFactory

  用于对请求限流，限流算法为令牌桶。后面详细研究。

+ RedirectToGatewayFilterFactory

  根据响应状态码将原始请求重定向到指定的URL。

  ```yaml
  - RedirectTo=302, https://acme.org
  ```

+ RequestHeaderSizeGatewayFilterFactory

  设置请求标头允许的最大数据大小（包括键和值）。

+ RequestSizeGatewayFilterFactory

+ RewritePathGatewayFilterFactory

  用于使用 Java 正则表达式来灵活地重写请求路径。

  ```
  - RewritePath=/red/?(?<segment>.*), /$\{segment}
  ```

+ RewriteLocationResponseHeaderGatewayFilterFactory
+ RewriteResponseHeaderGatewayFilterFactory
+ SaveSessionGatewayFilterFactory
+ SecureHeadersGatewayFilterFactory
+ SetPathGatewayFilterFactory
+ SetRequestHeaderGatewayFilterFactory
+ SetResponseHeaderGatewayFilterFactory
+ SetStatusGatewayFilterFactory
+ RetryGatewayFilterFactory
+ SetRequestHostHeaderGatewayFilterFactory
+ ModifyRequestBody
+ ModifyResponseBody 
+ TokenRelayGatewayFilterFactory
+ CacheRequestBodyGatewayFilterFactory
+ JsonToGrpcGatewayFilterFactory
+ ...

##### 自定义过滤器

+ **自定义全局过滤器**

  首先需要了解全局过滤器接口定义、自动配置、调用时期。

  参考原理流程图。

+ **自定义Gateway过滤器工厂**



## SCG常用场景与实现



### 动态路由

#### 基于注册中心的动态路由

通过服务注册与发现，动态创建路由，会自动为所有注册到注册中心的服务创建路由。

自动创建的路由ID: `ReactiveCompositeDiscoveryClient_${spring.application.name}`，路由规则和下面手动配置的路由规则相同。

此动态路由的问题：

+ 会暴露注册中心所有服务，但实际上有些底层服务并不需要暴露出去
+ 路由规则太简单不符合实际业务

[The DiscoveryClient Route Definition Locator](https://docs.spring.io/spring-cloud-gateway/docs/3.1.8/reference/html/#the-discoveryclient-route-definition-locator)

```yaml
# 基于配置中心的动态路由
spring:
  cloud:
    gateway:
      routes:
        # 动态路由
        # - id: route-service-echo-dynamic
        #   uri: lb://service-echo
        #   predicates:
        #     - Path=/service-echo/**
        #   filters:
        #     - RewritePath=/service-echo/(?<remaining>.*), /${remaining}   # 将 /service-echo 前缀剔除
        # - id: route-service-time-dynamic
        #   uri: lb://service-time
        #   predicates:
        #     - Path=/service-time/**
        #   filters:
        #     - RewritePath=/service-time/(?<remaining>.*), /${remaining}   # 将 /service-time 前缀剔除
      # 通过服务注册与发现，动态创建路由，即配置了discovery.locator后就不需要配置上面的路由了
      # 自动创建的路由和上面手动定义的路由相同，只不过路由ID是 ReactiveCompositeDiscoveryClient_${spring.application.name}
      # 请求时会自动将服务名替换为服务节点地址
      discovery:
        locator:
          enabled: true                           # 是否开启，默认为 false
          lower-case-service-id: true
#          url-expression: "'lb://' + serviceId"   # 路由的目标地址的表达式，默认为 "'lb://' + serviceId"
```

从调试信息可以看到 discovery.locator 自动创建的路由信息：

```text
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition ReactiveCompositeDiscoveryClient_service-time applying {pattern=/service-time/**} to Path
o.s.c.g.filter.GatewayMetricsFilter      : New routes count: 3
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition ReactiveCompositeDiscoveryClient_service-time applying filter {regexp=/service-time/?(?<remaining>.*), replacement=/${remaining}} to RewritePath
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition matched: ReactiveCompositeDiscoveryClient_service-time
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition ReactiveCompositeDiscoveryClient_service-echo applying {pattern=/service-echo/**} to Path
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition ReactiveCompositeDiscoveryClient_service-echo applying filter {regexp=/service-echo/?(?<remaining>.*), replacement=/${remaining}} to RewritePath
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition matched: ReactiveCompositeDiscoveryClient_service-echo
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition ReactiveCompositeDiscoveryClient_gateway-sample applying {pattern=/gateway-sample/**} to Path
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition ReactiveCompositeDiscoveryClient_gateway-sample applying filter {regexp=/gateway-sample/?(?<remaining>.*), replacement=/${remaining}} to RewritePath
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition matched: ReactiveCompositeDiscoveryClient_gateway-sample
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition ReactiveCompositeDiscoveryClient_service-time applying {pattern=/service-time/**} to Path
o.s.c.g.filter.GatewayMetricsFilter      : New routes count: 3
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition ReactiveCompositeDiscoveryClient_service-time applying filter {regexp=/service-time/?(?<remaining>.*), replacement=/${remaining}} to RewritePath
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition matched: ReactiveCompositeDiscoveryClient_service-time
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition ReactiveCompositeDiscoveryClient_service-echo applying {pattern=/service-echo/**} to Path
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition ReactiveCompositeDiscoveryClient_service-echo applying filter {regexp=/service-echo/?(?<remaining>.*), replacement=/${remaining}} to RewritePath
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition matched: ReactiveCompositeDiscoveryClient_service-echo
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition ReactiveCompositeDiscoveryClient_gateway-sample applying {pattern=/gateway-sample/**} to Path
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition ReactiveCompositeDiscoveryClient_gateway-sample applying filter {regexp=/gateway-sample/?(?<remaining>.*), replacement=/${remaining}} to RewritePath
o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition matched: ReactiveCompositeDiscoveryClient_gateway-sample
```

#### 基于配置中心的动态路由

将 `spring.cloud.gateway` 配置项统一存储在 Nacos 中，从配置中心动态刷新路由规则配置数据。

> 注意：测试Demo中用的2021.0.5.0版本，需要引入依赖`spring-cloud-starter-bootstrap`，并将配置nacos配置放到bootstrap.yml。
>
> ```yaml
> <dependency>
> 	<groupId>org.springframework.cloud</groupId>
> 	<artifactId>spring-cloud-starter-bootstrap</artifactId>
> </dependency>
> ```
>
> TODO: spring cloud bootstrap 属性加载原理。

### 实现限流

#### 自定义限流

#### 配合 sentinel  实现限流



### 实现熔断

#### 配合Hystrix实现熔断



### 实现安全认证

比如Token校验。



### 实现负载均衡

#### 基于WeightRoutePredicateFactory

```
routes:
	- id: route-service-echo-v1
		uri: http://localhost:8081/v1
		predicates:
		- Path=/echo/**
		- Weight=service-echo, 99
	- id: route-service-echo-v2
		uri: http://localhost:8081/v2
		predicates:
		- Path=/echo/**
		- Weight=service1, 1
```

#### 基于spring-cloud-starter-loadbalancer

参考前面动态路由。

### 实现灰度发布

可以基于权重路由断言工厂 WeightRoutePredicateFactory实现；

但是默认提供的WeightRoutePredicateFactory功能太简单粗暴不符合企业级应用要求。

**灰度发布的硬性要求**：

+ **方便快速回滚**

  即：

  + 业务逻辑应该向下兼容
  + 应该提供配置页面方便快速回滚（不需要重启服务）

+ **同一用户应该始终访问同一版本**

  用户不会在两个版本反复横跳。因为反复横跳而不会出异常要求业务逻辑向上和向下都要兼容，基本不太可能。复杂一点的义务光是向下兼容都不太容易。

  即：

  + 应该以用户特征值为基础制定分流策略

    比如使用用户userId尾号分流，00-09 访问新版本服务，10-99访问旧版本服务。

**基于SCG的一个通用的方案**：

同一服务多版本发布，多版本服务节点注册到注册中心的服务名相同，但元数据信息不同（比如新版本部署2个节点，旧版本部署8个节点，服务名都是service-echo, 新版本服务节点元数据信息为 version=v1.0.1，旧版本服务节点元数据信息为 version=v1.0.0）；

参考ReactiveLoadBalancerClientFilter自定义全局过滤器，实现基于用户ID尾号的分流策略（比如00-09访问新版本服务，10-99访问旧版本服务）先经过用户ID尾号对应应访问的版本号过滤出所有此版本的服务节点，然后再从节点中选择一个节点；

> 另外，路由，分流策略可以配置在配置中心；分流策略也可以从多个维度实现。

```yaml
# TODO
routes:
	- id: route-service-echo-v1.0.1
		uri: lb://service-echo-v1.0.1
		predicates:
		- Path=/echo/**
		- GrayWeight=0, 9				# 尾号00-09
	- id: route-service-echo-v1.0.0
		uri: lb://service-echo-v1.0.0
		predicates:
		- Path=/echo/**
		- GrayWeight=10, 99			# 尾号10-99
```

对应详细实现流程：

1. 用户请求到达网关，先经过认证过滤器，获取用户ID，存储到属性Map;
2. 经过自定义负载均衡过滤器，计算用户ID灰度到的服务版本；
3. 按版本筛选服务节点，最后从筛选出的节点中选择一个。

### 实现请求监控

#### 请求日志

### [跨域CORS配置](https://docs.spring.io/spring-cloud-gateway/docs/3.1.8/reference/html/#cors-configuration)

### 其他自定义功能

#### 活动限时访问



