# Dubbo 使用

## 常用功能

> 下面这些功能也将作为源码分析的研究重点。

+ @EnableDubbo
+ Dubbo 配置 `dubbo.*`
+ 服务注册、发现、调用
  + @DubboService 
  + @ServiceReference
+ 服务多版本
  + @DubboService(version = "1.0.0")  
  + @DubboReference(version = "1.0.0") 
  + dubbo.provider.version=1.0.0 
  + dubbo.consumer.version=1.0.0
+ 服务分组
  + @DubboService(group = "xxx")  
  + @DubboReference(group = "xxx") 
  + dubbo.provider.group=xxx
  + dubbo.consumer.group=xxx

+ 负载均衡

  服务提供者往往是多实例，通过负载均衡选择一个实例请求。

  ```java
  @DubboService(loadbalance = "random")
  @DubboReference(loadbalance = "random")
  ```

  + Random
  + RoundRobin
  + LeastActive 最少使用
  + ConsistentHash 一致哈希
  + Shorttest Response 最快响应

+ 服务超时时间

+ 集群容错

+ 服务降级

+ 本地伪装

+ 本地存根

+ 参数回调

+ 异步调用

+ 泛化调用

+ 泛化服务

+ 拓展机制

























