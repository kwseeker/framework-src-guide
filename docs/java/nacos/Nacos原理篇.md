# Nacos 原理篇

Nacos是一个Spring Boot Web项目。

当前分析版本是1.4.1。

[源码下载](https://github.com/alibaba/nacos/releases)

**源码编译与IDEA调试启动**：

```shell
# BUILDING有说明如何编译
mvn -Prelease-nacos -Dmaven.test.skip=true clean install -U
# 启动参数(set -x 执行下启动脚本)
nohup /usr/lib/jvm/java-1.8.0-openjdk-amd64/bin/java 
    -Djava.ext.dirs=/usr/lib/jvm/java-1.8.0-openjdk-amd64/jre/lib/ext:/usr/lib/jvm/java-1.8.0-openjdk-amd64/lib/ext 
    -Xms512m -Xmx512m -Xmn256m 
    -Dnacos.standalone=true 
    -Dnacos.member.list= -Xloggc:/opt/nacos/logs/nacos_gc.log 
    -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps 
    -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M 
    -Dloader.path=/opt/nacos/plugins/health,/opt/nacos/plugins/cmdb 
    -Dnacos.home=/opt/nacos 
    -jar /opt/nacos/target/nacos-server.jar 
    --spring.config.additional-location=file:/opt/nacos/conf/ 
    --logging.config=/opt/nacos/conf/nacos-logback.xml 
    --server.max-http-header-size=524288 nacos.nacos
# 启动入口是console模块的Main方法（SpringBoot工程）
# 前面的参数IDEA中只需要加下面一个就行
-Dnacos.standalone=true
```

源码调试启动后，运行测试DEMO，可正常注册和服务发现后可以开始调试。



## 注册中心原理

![Nacos注册中心原理](imgs/nacos-workflow-naming.png)

### 目标

+ 架构
+ 服务注册与发现
  + 服务列表数据结构
  + 临时节点
  + 持久化节点：`<Nacos安装目录>/data/naming/<命名空间ID>`
+ 服务订阅与推送
+ 集群数据一致性原理
  + Distro 协议
  + Raft协议
+ 健康检查&心跳

### 架构



### 服务注册与发现

#### 客户端启动时自动注册原理

客户端只是配置了`spring.cloud.nacos.discovery.*`等配置参数，以及引入了`spring-cloud-starter-alibaba-nacos-discovery`依赖，那么它是怎么自动注册的？

这个需要对**SpirngBoot启动流程**、**自动配置**（加载外部模块）以及**SpringCloud的机制**比较了解。

大概流程如下：

```txt
@SpringBootApplication 
-> @EnableAutoConfiguration -> @Import -> DeferredImportSelector
-> getAutoConfigurationEntry(获取所有自动配置类) -> loadSpringFactories(读取配置列文件) -> 过滤有效配置类
-> 依次执行所有有效的自动配置类（XxxAutoConfiguration.class）
-> 这里只看Nacos客户端的自动配置类定义文件spring.factories
-> 注册中心主要是 NacosDiscoveryAutoConfiguration、RibbonNacosAutoConfiguration、NacosDiscoveryClientAutoConfiguration

```

Nacos客户端组件spring.factories内容：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.alibaba.cloud.nacos.NacosDiscoveryAutoConfiguration,\
  com.alibaba.cloud.nacos.ribbon.RibbonNacosAutoConfiguration,\
  com.alibaba.cloud.nacos.endpoint.NacosDiscoveryEndpointAutoConfiguration,\
  com.alibaba.cloud.nacos.discovery.NacosDiscoveryClientAutoConfiguration,\
  com.alibaba.cloud.nacos.discovery.configclient.NacosConfigServerAutoConfiguration
org.springframework.cloud.bootstrap.BootstrapConfiguration=\
  com.alibaba.cloud.nacos.discovery.configclient.NacosDiscoveryClientConfigServiceBootstrapConfiguration
```

#### 服务注册表

Nacos服务注册表结构：

```java
//ServiceManager
//命名空间（常用做环境隔离：dev/test/prod）-> group(服务分组,多个服务可以组成一个分组，<GroupName>@@<ServiceName>) -> 服务实例
Map<String, Map<String, Service>> serviceMap
//Service: 服务实例，服务通常有个多个节点，还可能跨地部署
//服务名 -> 服务集群（一个服务[多实例]可能部署到多个集群，比如异地多机房架构）
Map<String, Cluster> clusterMap
//Cluster
//服务集群是包含多个服务实例的集合
Set<Instance> persistentInstances
Set<Instance> ephemeralInstances
```

#### 服务发现



#### 服务订阅与推送

#### 健康检查&心跳

#### 集群数据一致性原理

##### Distro 协议

搜索资料得知Distro是阿里内部制定并实现的临时数据的一致性协议（AP），官网没有说明文档。

先猜下Distro是做什么的？

首先Nacos线上是分布式(多实例)部署，每个实例保存一份服务注册信息，可能每个服务通过不同Nacos实例注册、注销自己（某时间点可能存在数据不一致问题，需要将变更同步到所有Nacos节点），Nacos就是做这个的。

下面通过源码看看协议实现原理。

源码路径：`nacos-core : /src/main/java/com/alibaba/nacos/core/distributed/distro`，20个文件。

##### Raft协议

是一个CP协议。



## 配置中心原理

+ 配置数据存储




## 高性能实现

+ **大量的异步处理**

  比如服务的异步注册，可以降低客户端启动时间。

+ **注册表更新-CopyOnWrite**(写时复制)

  更新注册表时复制了一份注册表数据，对副本进行更新，更新完成后赋值给原来的注册表；
  在更新注册表完成前客户端依然读旧的注册表，不会读取到正在更新的中间态数据。

  对比加锁等待更新完成读取最新的注册表，使用CopyOnWrite大大提升了并发能力，虽然损失了点实时性。但是注册中心场景对实时性要求并不高，注册中心读取到旧的数据表（比如有个服务实例下线了，正在进行注销操作）对业务影响并不大，因为系统一般会做转发处理。

+ **服务注册表变更向客户端推送采用UDP(加了ACK和重试)**

  相较于 ZK 的TCP长连接性能更高、且节省资源。