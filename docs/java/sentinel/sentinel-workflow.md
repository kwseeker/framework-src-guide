# Sentinel 工作原理

详细源码流程参考： sentinel.drawio，这篇文章只是对流程图的补充。

从 [`spring-cloud-starter-alibaba-sentinel`](https://mvnrepository.com/artifact/com.alibaba.cloud/spring-cloud-starter-alibaba-sentinel)入手分析工作原理。

Sentinel 源码关键模块功能：

```
sentinel-core 核心模块，限流、降级、系统保护等都在这里实现
sentinel-dashboard 控制台模块，可以对连接上的sentinel客户端实现可视化的管理
sentinel-transport 传输模块，提供了基本的监控服务端和客户端的API接口，以及一些基于不同库的实现
sentinel-extension 扩展模块，主要对DataSource进行了部分扩展实现
sentinel-adapter 适配器模块，主要实现了对一些常见框架的适配（比如dubbo、httpclient、grpc、okhttp、reactor、springmvc、gateway、zuul）,为它们整合Sentinel的功能
sentinel-demo 样例模块，可参考怎么使用sentinel进行限流、降级等
sentinel-benchmark 基准测试模块，对核心代码的精确性提供基准测试
```



## 系统组成

<img src="https://user-images.githubusercontent.com/9434884/53381986-a0b73f00-39ad-11e9-90cf-b49158ae4b6f.png" style="zoom: 50%;" />

> 注意上图中标号1，默认并不支持。

线上环境一般包括3部分：

+ **Sentinel 控制台**

  提供机器发现以及健康情况管理、监控（单机和集群），规则管理和推送的功能（默认是用的[原始模式](https://github.com/alibaba/Sentinel/wiki/%E5%9C%A8%E7%94%9F%E4%BA%A7%E7%8E%AF%E5%A2%83%E4%B8%AD%E4%BD%BF%E7%94%A8-Sentinel#%E5%8E%9F%E5%A7%8B%E6%A8%A1%E5%BC%8F)，生产环境一般由配置中心实现规则的管理和推送）。

  > TODO 机器发现实现原理？

+ **规则中心**

  用于规则的管理、推送和持久化，常用配置中心实现，支持很多种数据源（Nacos、Zookeeper、Redis、Apollo、etcd等等），具体参考sentinel-extension。

  **注意** Sentinel 控制台默认并不支持将页面配置的规则写入配置中心，所以测试时可以发现控制台配置好规则后，去配置中心根本看不到配置的规则；**使用配置中心管理规则需要通过配置中心的控制台配置规则**；如果想要支持在 Sentinel 控制台修改规则直接更新到配置中心，可以去修改控制台源码。

+ **Sentinel 客户端**（业务服务）



## 工作原理

![](https://sentinelguard.io/docs/zh-cn/img/sentinel-slot-chain-architecture.png)

> 补充：
>
> 上面的图没有说明逻辑入口，看源码可以发现入口是拦截器SentinelWebInterceptor（暂不管SentinelWebTotalInterceptor）; 
>
> ```java
> public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
>     try {
>         //1 先获取请求路由
>         String resourceName = this.getResourceName(request);
>         //2 为空不校验规则
>         if (StringUtil.isEmpty(resourceName)) {
>             return true;
> 		//3 非空先在请求属性中记录调用次数（+1）
>         //	首次统计也不需要校验规则
>         } else if (this.increaseReferece(request, this.baseWebMvcConfig.getRequestRefName(), 1) != 1) {
>             return true;
>         //4 非首次统计需要校验规则
>         } else {
>             String origin = this.parseOrigin(request);
>             String contextName = this.getContextName(request);
>             ContextUtil.enter(contextName, origin);
>             Entry entry = SphU.entry(resourceName, 1, EntryType.IN);
>             request.setAttribute(this.baseWebMvcConfig.getRequestAttributeName(), entry);
>             return true;
>         }
>     //5 规则校验失败会抛出BlockException
>     } catch (BlockException var12) {
>         BlockException e = var12;
> 
>         try {
>             //6 处理异常
>             this.handleBlockException(request, response, e);
>         } finally {
>             ContextUtil.exit();
>         }
> 
>         return false;
>     }
> }
> ```

### 系统配置

Sentinel 配置项，以 `spring.cloud.sentinel` 开头，对应 [SentinelProperties](https://github.com/alibaba/spring-cloud-alibaba/blob/master/spring-cloud-alibaba-sentinel/src/main/java/com/alibaba/cloud/sentinel/SentinelProperties.java) 配置属性类。

### 规则配置

Sentinel 中包含四种规则配置：流控、熔断、热点、授权。

这节研究规则的定义、存储、加载原理。

线上环境一般使用配置中心存储各种规则，需要在配置中心的控制台配置规则。

#### Sentinel 控制台管理规则

只适用于测试环境，官方称这种方式为原始模式，无法保证各节点规则的一致性，重启即丢失；详细参考流程图。

#### 配置中心管理规则

这种方式和 Sentinel 控制台就没什么关系了，规则的配置是通过配置中心的控制台配置，规则的推送拉取也是借由配置中心的能力。

关于这种模式下规则的拉取推送实现原理在后面 [流控组件初始化](#流控组件初始化) 小节细说。

#### 各种规则定义

+ 流控规则 FlowRuleEntity
+ 熔断规则

### 流控组件初始化



### 请求流量检查



### 流控异常处理

