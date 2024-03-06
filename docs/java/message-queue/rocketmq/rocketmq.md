# RocketMQ

RocketMQ源码量也极其庞大（30W），最新已经5.0了，这里依旧先只分析主流程。

主流程的流程图还是参考drawio文件。

参考资料：

+ [官方文档](https://rocketmq.apache.org/zh/docs/)
+ 《RocketMQ实战与原理解析》
+ 《RocketMQ技术内幕》

开源项目：

> 如果公司项目里面没有用过RocketMQ或者使用质量不好，可以找下工程性的开源实践项目。

+ 

主要研究问题：

+ 架构
+ 基本的消息发布&订阅
+ 消息可靠性原理（如何防止丢失）
+ 消息堆积能力
+ 消息去重&避免重复消费原理

+ 顺序消息原理
+ 分布式事务实现原理



## 本地源码编译与启动

本地先启动一个 NameServer、一个Broker。需要测试Broker高可用和节点负载功能时再多部署几个Broker。

先切换到最新正式发行版 5.2.0, 再 `mvn clean package -Dmaven.test.skip=true`；

**修改 NameServer 配置并启动**：

安装包中先后通过 mqnamesrv、runserver.sh 两个shell 文件启动命名服务，主要做了下面事情：

1. 设置环境变量 `ROCKETMQ_HOME` ，从 mqnamesrv 路径检测安装包的路径设置为 `ROCKETMQ_HOME`;

2. 执行 runserver.sh 脚本，NamesrvStartup 是命名服务的主类；

   ```shell
   # -D 设置了logback配置文件
   sh ${ROCKETMQ_HOME}/bin/runserver.sh -Drmq.logback.configurationFile=$ROCKETMQ_HOME/conf/rmq.namesrv.logback.xml org.apache.rocketmq.namesrv.NamesrvStartup $@
   ```

3. runserver.sh 读取 JAVA_HOME，获取 java 命令、添加 CLASSPATH、配置 GC 日志输出目录、配置一些 JVM 启动参数，最后启动 NamesrvStartup 主类。

   ```shell
   # 配置文件通过这行加载到CLASSPATH的（配置文件包括）
   export CLASSPATH=.:${BASE_DIR}/conf:${BASE_DIR}/lib/*:${CLASSPATH}
   ```

源码启动直接从 NamesrvStartup.java 启动即可，只需要设置下`ROCKETMQ_HOME`环境变量，比如设置为`-DROCKETMQ_HOME=./` 虽然源码中检查了这个环境变量但是NameServer貌似并没有用到，所以随便设置。

源码启动成功后打印：

```
The Name Server boot success. serializeType=JSON, address 0.0.0.0:9876
```

**修改 Broker 配置并启动**：

安装包中先后通过 mqbroker、runbroker.sh 两个shell 文件启动Broker服务。类似NameServer的脚本。

不过源码启动除了设置 `ROCKETMQ_HOME`环境变量，还需要设置两个参数，即配置文件路径和连接的命名服务IP端口。

```shell
# 安装包启动
nohup sh bin/mqbroker -c conf/broker.conf -n 127.0.0.1:9876 &
# 源码启动配置添加参数
-c conf/broker.conf -n 127.0.0.1:9876
# 环境变量，在源码根目录创建conf,并将安装包中broker.conf拷贝到conf
ROCKETMQ_HOME=/home/arvin/mywork/github/rocketmq
```

broker.conf

```properties
brokerClusterName = DefaultCluster
# 当前Broker名
brokerName = broker-a
# 0表示Master, >0表示Slave
brokerId = 0
# 消息删除时间，04即凌晨4点
deleteWhen = 04
# 磁盘上保存时间，48小时
fileReservedTime = 48
# 主Broker, 与Slave异步同步消息
brokerRole = ASYNC_MASTER
# 异步刷盘
flushDiskType = ASYNC_FLUSH
```

源码启动成功后打印：

```
The broker[broker-a, 192.168.1.5:10911] boot success. serializeType=JSON and name server is 127.0.0.1:9876
```

**测试消息的生产&消费**：

使用安装包的shell脚本测试，源码启动正常的话下面shell脚本会打印很多消息发送成功和接收成功的日志。

```shell
export NAMESRV_ADDR=127.0.0.1:9876
# 就是调用源码包 example 模块下的测试类
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
```



## 源码包

```txt
.
├── acl
├── bazel
├── broker			Broker实现，接受生产者发来的消息并存储（通过调用rocketmq-store），消费者从这里取得消息
├── BUILD.bazel
├── BUILDING
├── client			客户端接口，提供发送、接受消息的客户端API
├── common
├── container
├── CONTRIBUTING.md
├── controller
├── dev
├── distribution
├── docs
├── example	
├── filter			消息过滤器
├── LICENSE
├── namesrv			NameServer实现，类似于Zookeeper，保存消息的TopicName，队列等运行时的元信息
├── NOTICE
├── openmessaging
├── pom.xml
├── proxy
├── README.md
├── remoting		基于Netty4的client/server + fastjson序列化 + 自定义二进制协议
├── srvutil
├── store
├── style
├── target
├── test
├── tieredstore
├── tools
└── WORKSPACE
```



## 基本架构

### 组件

+ 命名服务器（NameServer）
+ 生产者（Producer）
+ 消息服务器（Broker）
+ 消费者（Consumer）

### Broker集群模式

+ 单主
+ 双主
+ 两主两从（异步复制）
+ 两主两从（同步复制）

+ [Dledger集群](https://github.com/apache/rocketmq/blob/master/docs/cn/dledger/deploy_guide.md)

> 一般 RocketMQ 的集群部署方案推荐如下：
>
> - 如果对高性能有比较强的诉求，使用两主两从，异步复制，异步刷盘。
> - 如果对可靠性有比较强的诉求，建议使用 [Dledger 集群](https://github.com/apache/rocketmq/blob/master/docs/cn/dledger/deploy_guide.md)，至少三节点。



##  基本的消息发布&订阅





## 消息可靠性原理（如何防止丢失）



## 消息堆积能力



## 消息去重&避免重复消费原理



## 顺序消息原理



## 分布式事务实现原理
