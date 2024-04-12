# [Sentinel](https://github.com/alibaba/Sentinel/wiki/%E4%BB%8B%E7%BB%8D) 应用

sentinel：哨兵。

[github](https://github.com/alibaba/Sentinel)

Sentinel 用于协调系统的处理能力与外部接入请求速度。

Sentinel 具有以下特征:

- **丰富的应用场景**：Sentinel 承接了阿里巴巴近 10 年的双十一大促流量的核心场景，例如秒杀（即突发流量控制在系统容量可以承受的范围）、消息削峰填谷、集群流量控制、实时熔断下游不可用应用等。
- **完备的实时监控**：Sentinel 同时提供实时的监控功能。您可以在控制台中看到接入应用的单台机器秒级数据，甚至 500 台以下规模的集群的汇总运行情况。
- **广泛的开源生态**：Sentinel 提供开箱即用的与其它开源框架/库的整合模块，例如与 Spring Cloud、Apache Dubbo、gRPC、Quarkus 的整合。您只需要引入相应的依赖并进行简单的配置即可快速地接入 Sentinel。同时 Sentinel 提供 Java/Go/C++ 等多语言的原生实现。
- **完善的 SPI 扩展机制**：Sentinel 提供简单易用、完善的 SPI 扩展接口。您可以通过实现扩展接口来快速地定制逻辑。例如定制规则管理、适配动态数据源等。

这里只是经过源码深入理解后对官方文档的补充。



## 工作原理

原理篇讲。

## 使用流程

1. 引入依赖；
2. 配置Sentinel的配置参数，主要是规则的数据源、DashBoard（主要用于开发环境）、拦截请求地址（SentinelWebInterceptor默认将请求路由作为资源）等；
3. 在数据源中配置规则定义（像Nacos这些数据源可以自动将规则推送到各个节点的规则管理器，节点启动时也会主动去拉取一次）；
4. SentinelWebInterceptor 执行过程中会按 ProcessorSlot 责任链执行各个 ProcessorSlot，进而筛选所有与资源匹配的规则，一一执行规则校验逻辑；
5. 规则校验不通过会抛出对应的异常，这些异常都继承 BlockException; 
6. 在MVC统一异常处理中对 BlockException类异常进行处理（当然也可以通过配置BaseWebMvcConfig blockExceptionHandler进行异常处理）。

## 流量控制

流量控制（flow control），其原理是监控应用流量的 QPS 或并发线程数等指标，当达到指定的阈值时对流量进行控制，以避免被瞬时的流量高峰冲垮，从而保障应用的高可用性。

## 集群流控

为什么要使用集群流控呢？假设我们希望给某个用户限制调用某个 API 的总 QPS 为 50，但机器数可能很多（比如有 100 台）。这时候我们很自然地就想到，找一个 server 来专门来统计总的调用量，其它的实例都与这台 server 通信来判断是否可以调用。这就是最基础的集群流控的方式。

## 网关流控

即将Sentinel集成到网关里面。

## 熔断降级

对不稳定的**弱依赖服务调用**进行熔断降级。

## 热点参数限流

热点参数限流会统计传入参数中的热点参数，并根据配置的限流阈值与模式，对包含热点参数的资源调用进行限流。

## 系统自适应限流

从整体维度对应用入口流量进行控制，结合应用的 Load、CPU 使用率、总体平均 RT、入口 QPS 和并发线程数等几个维度的监控指标，通过自适应的流控策略，让系统的入口流量和系统的负载达到一个平衡，让系统尽可能跑在最大吞吐量的同时保证系统整体的稳定性。

？

## 黑白名单控制

根据调用来源来判断该次请求是否允许放行。

## 实时监控数据

Sentinel 提供对所有资源的实时监控。

## 动态规则

当资源定义成功后可以动态增加各种流控降级规则。

Sentinel 提供两种方式修改规则：

- 通过 API 直接修改 (`loadRules`)
- 通过 `DataSource` 适配不同数据源修改

## 控制台

控制台需要额外部署对接。默认地址：http://127.0.0.1:8080/ 。

```shell
wget https://github.com/alibaba/Sentinel/releases/download/1.8.6/sentinel-dashboard-1.8.6.jar
nohup java -jar sentinel-dashboard-1.7.1.jar &
```

Docker部署：

看源码中没有Docker相关的文件，估计官方没有做Docker镜像，而且看第三方做的镜像都比较老了，需要自己制作。

https://github.com/kwseeker/cloud-native/blob/master/docker/dockerfiles/sentinel-dashboard/1.8.6/Dockerfile

## 生产环境使用 Sentinel

## 阿里云企业版 Sentinel

## OpenSergo 控制面

官方文档说的有些抽象，其实就是通过 OpenSergo 规范与动态规则源（就是数据源）交互，用于获取规则源中定义的规则。

通过模块 [`sentinel-datasource-opensergo`](https://github.com/alibaba/Sentinel/tree/master/sentinel-extension/sentinel-datasource-opensergo)实现，对应下图中间的部分，上边部分对应各个数据源根据OpenSergo规范提供的数据输出接口。下面部分是Sentinel Core 通过 sentinel-datasource-opensergo 获取并加载动态规则到本地。

![](https://user-images.githubusercontent.com/9434884/186125289-efb5e75a-0d6d-486c-a577-f986024ad911.png)

## 启动配置项

就是一些个性化配置项。

## 注解埋点支持

这里说的埋点就是定义资源的意思。

## 主流框架适配

## Envoy 集群流量控制

为 Envoy 服务网格提供集群流量控制的能力。Envoy是面向Service Mesh 的高性能网络代理服务。

## 多语言生态