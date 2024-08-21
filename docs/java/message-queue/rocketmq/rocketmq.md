# RocketMQ

RocketMQ源码量也极其庞大（30W），最新已经5.0了，这里依旧先只分析主流程。

主流程的流程图还是参考drawio文件。

参考资料：

+ [官方文档](https://rocketmq.apache.org/zh/docs/)
+ 《RocketMQ实战与原理解析》
+ 《RocketMQ技术内幕》

开源项目：

> 如果公司项目里面没有用过RocketMQ或者使用质量不好，可以找下工程性的开源实践项目。

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
# 默认是用户根目录下store文件夹
storePathRootDir = /home/arvin/mywork/github/rocketmq/data/store
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

**源码选择性修改**：

```java
#NettyClientConfig.java
//private boolean disableNettyWorkerGroup = false;	
//源码默认为false，执行每个ChannelHandler的事件处理方法时都会提交到线程池中执行，导致调试ChannelPipeline很不方便
private boolean disableNettyWorkerGroup = true;
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



## 关键业务流程

下面流程请结合源码和流程图看（细节还是得去看流程图和源码）。

NameServer、Broker 逻辑的核心入口：请求的各种Processor、线程池注册的计划任务，从这些源码可以快速了解当前模块实现了什么功能。

### 启动阶段

NameServer、Broker启动流程：

1. 首先启动  NameServer，启动阶段主要是启动 Netty服务端和客户端（负责与其他模块交互）、创建几个计划任务线程池、注册关机钩子；

   **NameServer模块功能**：

   + **Broker节点的注册和发现**

     Broker 节点的路由信息由 RouteInfoManager 存储。

   + **Topic路由信息注册和查询**

     Topic 路由信息（即发送和消费消息究竟应该访问哪些Broker节点）由 TopicRouteData 存储。Broker 节点启动后会向所有NameServer 报备自己的地址信息（`registerBrokerAll()`），最终是向所有 NameServer 发送 REGISTER_BROKER 请求上报自己的信息，里面还会向所有默认主题添加当前Broker节点的路由信息（包括 TBW102）。

   + ... (随着后面的分析慢慢补充)...

2. 启动 Broker 代理服务器，启动阶段主要是启动Netty服务端和客户端（负责与其他模块交互）, 启动一堆计划任务线程池或单独的线程，创建了一堆组件、容器，注册关闭钩子；

   **Broker 模块功能**：

   + **Broker 用户自定义主题的创建**
   
     用户自定义主题是用户向某一个 TBW102 Broker 节点发送消息后，Broker检查自定义主题不存在后，才临时创建的；本地创建TopicConfig 先注册到本地然后再向所有 NameServer 报备。
   
   + ... (随着后面的分析慢慢补充)...

### 消息发布&订阅

#### 同步发送&同步消费

1. 生产者（DefaultMQProducer）实例创建并启动；由于每个生产者都是放到MQClientInstance中保存并维护的，如果MQClientInstance实例不存在会先创建并启动 MQClientInstance 实例，最核心的就是创建Netty客户端；然后将生产者实例注册到 MQClientInstance 的 produceTable 成员变量；

   > + 每个MQ客户端服务实例有一个MQClientInstance实例，一个MQClientInstance本质是Netty客户端 NettyRemotingClient（里面至少两个Netty Bootstrap，一个连接NameServer集群的某个实例，一个连接Broker集群的某个实例），另外可能存储多个生产者和消费者，另外还存储主题路由表，主题、队列、Broker代理服务器映射。
   >
   > + 每个 MQClientInstance 有自己唯一的 ClientId，其生成规则：
   >
   >   由 ClientIP@InstanceName@UnitName@StreamRequestType 组成，
   >   其中 InstanceName由PID#System.nanoTime() 组成，后面两部分可能没有；比如：192.168.1.5@74854#15296806586016。

2. 启动一堆计划任务线程池或单独的线程（详细参考流程图）；

   > 其中比较重要的任务：
   >
   > + timerTaskScanAvailableNameSrv
   >
   >   即定期请求 NameServer 集群的某个实例，获取所有可用的 NameServer 节点地址信息。
   >
   > + updateTopicRouteInfoFromNameServer

3. 创建Message实例，额外装配上命名空间；发送前需要根据主题信息查询这个消息应该发给哪个 Broker, 最终是通过向 NameServer 发送 `GET_ROUTEINFO_BY_TOPIC` 请求查询的；如果这个这个主题是首次使用，第一请求会返回没有此主题的路由信息，然后会去查询 TBW102 这个主题的路由信息并返回其中一个Broker节点路由；

   > + 如果没有设置命名空间，命名空间默认为空
   > + TBW102 这个主题就是用于创建新主题，它的Broker路由信息来源于Broker节点注册，每注册一个 Broker 节点就会为TBW102这些默认主题添加一个路由信息。 

4. 从Broker节点路由信息中选择一个消息队列，装配上其他信息，最终向被选中的 Broker 节点及队列发送消息（请求码 `SEND_MESSAGE_V2`）；

   > 一个主题里面可能配置了多个队列，消息需要发送到其中一个队列存储，默认的选择规则：
   >
   > 先使用 BrokerFilter 过滤，内部轮询查找和本线程上次发送消息拥有相同的Broker节点名的队列，如果是线程首次发送lastBrokerName一定为 null 就直接轮询（RoundRobin）选择下一个消息队列。
   >
   > 即同一个线程中多次发送某主题消息总是发送到相同的Broker节点。
   >
   > 详细参考：TopicPublishInfo$selectOneMessageQueue(...) 方法。

5. Broker 节点接收到`SEND_MESSAGE_V2`请求会转交给 SendMessageProcessor 处理：如果 Topic 不存在会创建 TopicConfig 先注册到本地 TopicConfigManager 然后向所有 NameServer 发送请求进行报备；之后会通过 **MessageStore** 组件存储消息；MessageStore 组件将消息写入 CommitLog 文件对应的缓冲（ByteBuffer），然后由 FlushCommitLogService 线程异步将 CommitLog 中的新消息写入文件；由 **ReputMessageService** 异步向 ConsumeQueue、IndexFile 中追加新写入的消息信息，然后分别由 FlushConsumeQueueService、IndexService 线程异步将消息信息写入各自的文件。

   > CommitLog 包含一个 MappedFile 队列，一个 MappedFile 实例对应 CommitLog 中一个文件（因为 RocketMQ 要求CommitLog 文件大小不能超过 1G，存满后需要新建一个文件 ），MappedFile 包含两个重要成员对象：写缓冲（ByteBuffer）和 文件内存映射缓冲（MappedByteBuffer），消息先写到 ByteBuffer，后面再由 FlushCommitLogService 线程异步通过 MappedByteBuffer 将 ByteBuffer 中的消息写入到文件（磁盘）。
   >
   > 消息内容是在 CommitLog 中存储的；IndexFile （也是一组 MappedFile） 中存储所有消息的**索引**信息（用于通过消息唯一KEY进行检索）；ConsumeQueue 中存储某个主题某个队列的消息的**索引**信息，而 ConsumeQueueStore 包含所有主题、所有队列的 ConsumeQueue。
   >
   > **通过消息唯一KEY检索流程**：
   >
   > 1. 先对消息唯一Key哈希取绝对值，然后取模哈希槽数量（5M），去20MB空间中检索，获取消息加入索引文件后的IndexCount；
   > 2. 通过IndexCount计算消息索引信息位置，去400MB空间中检索，获取消息的索引信息；
   > 3. 通过消息的索引信息去CommitLog中检索消息内容。
   
6. 消费者（DefaultMQConsumer）实例创建并启动，一个服务实例中同样可能包含多个分组的多个消费者实例；配置（消费者分组、消费主题、TAG）好消费者后启动，同样是创建了 Netty 客户端 和 一批线程池或单独的线程；

   这里列举比较重要的线程任务：

   + PullMessageService

     线程会一直轮询以阻塞方式从messageRequestQueue中读取 MessageRequest 命令，命令读取成功后会向 Broker 发送PULL_MESSAGE等请求拉取消息，如果有消息被 Found 则拉取成功并回调处理。处理完成后会再调用 executePullRequestLater() 或 executePullRequestImmediately() 用于触发消息的下次消费，实现消费者可以对消息进行持续消费。

     Broker 端和消费者端**消费进度**的管理：

     以默认的负载均衡策略来看，集群中一个分组的所有消费者会平均分担一个主题下所有的消息队列，**一个消息队列最多由一个消费者进行消费，不会出现多线程并发消费同一个消息队列的情况，这样的话就可以实现在消费者端维护消费进度**。

     以 Cluster 消费模式的 RemoteBrokerOffsetStore 为例，每个消费者本地会通过 RemoteBrokerOffsetStore 存储消费位点，消费消息后（无论成功失败）会更新消费位点，并通过线程池定期或发现消费位点有错误时，和Broker OffsetManager 同步消费位点。

     > MessageRequest 定义对哪个主题的哪个队列进行消费，以及从哪个偏移位置开始消费。
     >
     > 消息拉取是批量拉取，默认是每隔 500 ms 拉取一次。

   + RebalanceService

     重新负载均衡服务，用于根据当前节点当前主题的消息队列数量和消费者数量，进行负载均衡重新分配。

     默认的负载均衡策略是 AllocateMessageQueueAveragely，会平均分配，假设主题有6个MessageQueue（Q0 Q1 Q2 Q3 Q4 Q5），当前4个消费者(C0 C1 C2 C3)，按此策略各消费者会分得 C0(Q0 Q1) C1(Q2 Q3) C2(Q4) C3(Q5)。

### 消息可靠性原理（如何防止丢失）

### 消息堆积能力

### 消息去重&避免重复消费原理

### 顺序消息原理

### 分布式事务实现原理



## 从源码深入解析常见面试题

### RocketMQ Broker 中的消息消费后会立即删除么

不会立即删除，每条消息都会持久化到CommitLog中，每个Consumer连接到Broker后会维持消费进度信息，当有消息消费后只是当前Consumer的消费进度（CommitLog的offset）更新了。

由于内存和磁盘空间限制的，为防止消息堆积，RocketMQ 会周期性地删除过期的 CommitLog 文件（默认保存72小时），以及对应的 ConsumeQueue IndexFile 中的过期数据；默认是在凌晨4点进行删除操作，但是也可以在磁盘空间不足时执行删除操作、或者手动执行删除操作。

**源码入口**：

```java
// DefaultMessageStore$CleanCommitLogService
public void run() {
    ...
    this.deleteExpiredFiles();
    this.reDeleteHangedFile();
    ...
}
// DefaultMessageStore$CleanConsumeQueueService
public void run() {
    ...
    this.deleteExpiredFiles();
    ...
}
// MessageStoreConfig
private int fileReservedTime = 72;	//文件没有修改操作后72小时过期
private String deleteWhen = "04";	//凌晨4点执行删除
private int diskMaxUsedSpaceRatio = 75;	//磁盘使用超过75%
// 手动删除过期 CommitLog, AdminBrokerProcessor, netty 请求命令码 RequestCode.DELETE_EXPIRED_COMMITLOG
case RequestCode.DELETE_EXPIRED_COMMITLOG:
	return this.deleteExpiredCommitLog();
```

### RocketMQ消费模式有几种

+ 集群消费

  1.一条消息只会被同Group中的一个Consumer消费

  2.多个Group同时消费一个Topic时，每个Group都会有一个Consumer消费到数据

+ 广播消费

  消息将对一 个Consumer Group 下的各个 Consumer 实例都消费一遍。即使这些 Consumer 属于同一个Consumer Group ，消息也会被 Consumer Group 中的每个 Consumer 都消费一次。

### 消费消息是 push 还是 pull？为什么要主动拉取消息而不使用事件监听方式？

从 PushConcumer 实现类 DefaultMQPushConsumer 源码看，push 也是基于 pull 实现的；

从启动流程可知会启动 PullMessageService 这个线程，这个线程会一直轮询以阻塞方式从 messageRequestQueue 中读取 MessageRequest 再向 Broker 发送 **PULL_MESSAGE** 等请求拉取消息，如果有消息则可以拉取成功并直接返回给消费者端，这就实现了拉模式；

拉取成功后还会通过提交新的 MessageRequest 实现持续拉取；

但是可能发请求时并没有可以消费的消息，这时Broker会将PullRequest（即消费者中的MessageRequest）存储到 PullRequestHoldService pullRequestTable，待有新消息到来会重放 pullRequestTable 中对应的 PullRequest 请求再次读取消息并通过netty长连接通道推送给消费者，这就实现了推模式。

> MessageRequest 请求则是在消费者重新负载均衡时，每添加一个新的MessageQueue，就会提交一个PullRequest （MessageRequest 子类），启动对这个消息队列的消费流程。

主动拉取方式是拉取一批消息缓存到消费者本地，如果缓存的未处理的消息超过可缓存的最大阈值就不会再继续拉取，直到消费者消费掉一部分消息，使缓存的未处理消息数量低于阈值，才会继续拉取。这样做的好处是可以根据消费者的消费能力动态负载均衡，避免消息在消费者端持续累积。

**拉取阈值的源码**：

```java
// DefaultMQPushConsumerImpl
public void pullMessage(final PullRequest pullRequest) {
	...
    // 从数量和大小两个维度设置阈值，超过阈值直接return, 不会执行后面拉取消息的方法
	long cachedMessageCount = processQueue.getMsgCount().get();
    long cachedMessageSizeInMiB = processQueue.getMsgSize().get() / (1024 * 1024);
    if (cachedMessageCount > this.defaultMQPushConsumer.getPullThresholdForQueue()) {
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_CACHE_FLOW_CONTROL);
        if ((queueFlowControlTimes++ % 1000) == 0) {
            log.warn(...);
        }
        return;
    }
    if (cachedMessageSizeInMiB > this.defaultMQPushConsumer.getPullThresholdSizeForQueue()) {
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_CACHE_FLOW_CONTROL);
        if ((queueFlowControlTimes++ % 1000) == 0) {
            log.warn(...);
        }
        return;
    }
    
    ...
	this.pullAPIWrapper.pullKernelImpl(...);
}
```

