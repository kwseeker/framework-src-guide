# Ribbon & LoadBalancer 原理篇

详细流程参考 netflix-ribbon-workflow.drawio, 此文档只是对流程图的补充。



## Ribbon工作原理

这里就研究官方的例子ribbon-examples开始。代码位于：SpringBoot-Labs/netflix-ribbon/ribbon-examples。

依赖：

```groovy
compile project(':ribbon-core')
compile project(':ribbon-httpclient')
compile project(':ribbon-transport')
compile project(':ribbon-loadbalancer')
compile project(':ribbon')
```

### 负载均衡器 ILoadBalancer

使用构建者模式LoadBalancerBuilder创建负载均衡器。

负载均衡器的几种实现：

+ BaseLoadBalancer

+ DynamicServerListLoadBalancer

  继承 BaseLoadBalancer。

+ ZoneAwareLoadBalancer

  继承 DynamicServerListLoadBalancer。

#### 配置项 IClientConfig

#### 负载均衡规则 IRule

#### 服务可用性检查 IPing

### 重试处理器 RetryHandler

里面并没有重试逻辑，只是重试配置信息，改成 RetryConfiguration估计更好。

重试逻辑是借助RxJava Observable retry() 方法实现的。



## spring-cloud-starter-netflix-ribbon 工作原理

### 自动配置阶段

### 请求处理阶段



## spring-cloud-starter-loadbalancer 工作原理

版本：v3.1.5。

这里研究基于Spring Cloud LoadBalancer 和 Nacos 实现的负载均衡工作流程。

 ### 自动配置阶段

负载均衡处理依靠自动配置阶段加载的很多Bean。

`spring-cloud-starter-alibaba-nacos-discovery` `spring-cloud-starter-loadbalancer` 这两个starter加载了很多Bean。

核心的Bean:

+ **LoadBalancerClientsProperties** 

  `spring.cloud.loadbalancer` 前缀的配置属性。

  约定的默认配置（及不修改配置的情况下默认值）：

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

+ **BlockingLoadBalancerClient**

  由 LoadBalancerClientFactory 这个Bean创建。

  

### 请求处理阶段

参考 `loadbalancer-ribbon-workflow.drawio`。
