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

#### 同步发送&消费

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

     消费者重新负载均衡服务，用于根据当前节点当前主题的消息队列数量和消费者数量，进行负载均衡重新分配。

     默认的负载均衡策略是 AllocateMessageQueueAveragely，会平均分配，假设主题有6个MessageQueue（Q0 Q1 Q2 Q3 Q4 Q5），当前4个消费者(C0 C1 C2 C3)，按此策略各消费者会分得 C0(Q0 Q1) C1(Q2 Q3) C2(Q4) C3(Q5)。
     
     注意：如果消费者个数大于消费队列个数，会有部分消费者闲置，无法消费数据。
     
     > **其他负载均衡算法**：
     >
     > 所有负载均衡算法均实现 AllocateMessageQueueStrategy 接口，主要实现 allocate 方法：
     >
     > ```java
     > List<MessageQueue> allocate(	//返回给当前消费者分配的消息队列列表
     >         final String consumerGroup,	//消费者分组
     >         final String currentCID,	//当前消费者ID
     >         final List<MessageQueue> mqAll,	//当前主题所有的消息队列
     >         final List<String> cidAll	//分组内所有的消费者ID
     >     );
     > ```
     >
     > + 环形分配策略 (AllocateMessageQueueAveragelyByCircle)
     >
     >   源码很简单，就是将消息队列依次分配给各个消费者，
     >
     >   假设主题有6个MessageQueue（Q0 Q1 Q2 Q3 Q4 Q5），当前4个消费者(C0 C1 C2 C3)，按此策略各消费者会分得 C0(Q0 Q4) C1(Q1 Q5) C2(Q2) C3(Q3)。
     >
     > + 手动配置分配策略 (AllocateMessageQueueByConfig)
     >
     >   直接通过配置指定哪个消费者消费哪些消息队列。
     >
     > + 机房分配策略 (AllocateMessageQueueByMachineRoom)
     >
     >   先过滤出mqAll中所有处于 `void setConsumeridcs(Set<String> consumeridcs)` 设置的机房中的Broker的消息队列，然后将过滤出的消息队列平均分配给所有消费者。
     >
     > + 一致性哈希分配策略 (AllocateMessageQueueConsistentHash)
     >
     >   是通过一致性哈希路由器分配的，参考 [ConsistentHashRouter](ConsistentHashRouter.md)。
     >
     > + 靠近机房策略 (AllocateMachineRoomNearby)
     >
     >   这个用于自行定制，需要自行实现 MachineRoomResolver 获取消息队列、消费者被部署的机房信息；allocate 时先将所有消息队列、消费者按机房分组，然后**将消息队列按上面某种策略分配给相同机房的消费者**；如果**消息队列没有同机房的消费者会被所有消费者按上面的某种策略分配**。
     >
     >   这里可以看到并非做到真正的靠近机房分配，没有同机房消费者的消息队列直接分配给了所有消费者，并没有地理位置远近的判断。

### 消息可靠性原理（如何防止丢失）

从源码看可能导致消息丢失的情况有两种：

+ 消息发送失败且编码不当没有检查发送结果
+ 消息还未持久化到磁盘服务器宕机
+ 消息消费失败且编码不当返回了消费成功（不太可能出现）

针对第一种情况需要确认消息发送返回结果即是否发送成功，不能使用 OneWay 这种发送方式；

针对第二种情况需要使用同步刷盘策略。

```properties
## 默认情况为 ASYNC_FLUSH 
flushDiskType = SYNC_FLUSH 
```

### 消息堆积能力

从源码看只有存储时间（5.x版本最多保存72小时）限制，并没有空间限制，所以取决于服务器可用的内存和磁盘空间。

从源码看消息所在 CommitLog 文件一旦超时，**不会管里面到底还有没有未消费的消息**，都会将文件删除。

### 避免消息重复消费的方案

RocketMQ中可能引起消息被重复消费的原因：

以集群消费模式（CLUSTERING）为例，比如消息拉取到 Consumer 后执行消费回调方法时，可能业务逻辑中会请求其他微服务接口，接口执行正常但是返回结果时出现网络抖动 Consumer 没有收到结果，这时后续处理就是当作消费失败处理，一般是在消费回调方法中返回 **RECONSUME_LATER**，后面 Consumer 会将此消息重新发回给 Broker 重试消费。

RocketMQ 本身并不会避免重复消费，需要用户自行实现，一般都是通过保证**接口幂等**（即检测消息是否已经处理过[消息带有唯一ID]，已处理过直接返回上次处理结果）。实现幂等的方式比如：数据库唯一键约束、使用缓存记录已成功处理的消息ID。

源码入口：

```java
// 这里展示消息消费返回 RECONSUME_LATER 的后续重试逻辑
ConsumeMessageConcurrentlyService.this.processConsumeResult(status, context, this);
```

### 顺序消息原理

> 发现网上搜索到的方案说法大都不严谨，基本都是只强调写入同一个消息队列。

即如何保证消息顺序发送和顺序消费，从RocketMQ工作流程看，严谨地实现顺序消息需要保证三点：

+ **顺序生产**

  一组有序的消息需要使用同一个生产者生产，如果使用多个生产者，可能导致乱序写入消息队列；

+ **写入同一消息队列**

  默认情况下一个主题下面会创建多个消息队列，一个消息队列中的消息消费是天然可以保证顺序写入就被顺序输出的，但是多个消息队列无法保证；

+ **顺序消费**

  即使消息被顺序地从Broker拉取到消费者，也不能保证被顺序消费，比如消费者开启多个线程并发消费，却没有同步处理同样可能导致消息被乱序消费，需要使用 `MessageListenerOrderly`。

  `MessageListenerOrderly` 通过 `ConsumeMessageOrderlyService`处理，同样使用了线程池进行消费，因为只需要保证同一个队列内的消息顺序消费即可，`ConsumeMessageOrderlyService`是使用消息队列锁实现的同队列消息顺序消费的，这样仍然可以保证不同队列的消息的并发消费，性能比单线程好的多。

  另外`ConsumeMessageOrderlyService`对**消费失败的处理**也和 `ConsumeMessageConcurrentlyService`不一样，没有将消费失败的消息重新发回给 Broker, 而是**将它们直接重新加入处理队列重试消费，重试消费时不会消费后续的消息**。

  将消息发回给Broker可能导致破坏消息顺序性，举个例子：一组顺序消息包含20个消息，假设消费者每次拉取10个，由于拉取和消费是异步的，可能第一次拉取的10个消息消费失败，发回给Broker只可能是排在第二批消息的后面，这就会破坏消息的顺序性。

  > RocketMQ 一个消息队列只会被一个消费者消费，不用再强调单消费者消费了。

#### 如何将顺序消息写入同一消息队列

自然可以想到两种方式实现顺序消息：

+ **能否设置这个主题下只有一个消息队列？**（只有一个队列自然可以保证消息顺序消费）

  总结：可以通过 `DefaultMQProducer#setDefaultTopicQueueNums()` 设置消息队列个数为1，不过这种方式不推荐，无法发挥多节点优势，可能会有性能问题。

  ```java
  DefaultMQProducer producer = new DefaultMQProducer("ordered_messages");
  producer.setDefaultTopicQueueNums(1);
  ```

+ **这组有顺序的消息如何实现都发给同一个消息队列？**（只用其中一个队列）

  总结：通过在 `MQProducer#send()` 设置自定义 `MessageQueueSelector`。

**先看：能否设置这个主题下只有一个消息队列？**

先分析清楚主题下消息队列个数是哪里配置的？分析源码找到主题中消息队列的个数是在 `DefaultMQProducerImpl#sendKernelImpl(...)` 中通过 `requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());`设置的，而 defaultTopicQueueNums 可以通过 `DefaultMQProducer#setDefaultTopicQueueNums(int defaultTopicQueueNums)`设置。

详细流程参考流程图。

经测试这种方式是可行的。

源码入口：

```java
//1 Broker SendMessageProcessor#sendMessage(...)
// 消息存储目标消息队列ID
int queueIdInt = requestHeader.getQueueId();
// 这里用到的主题配置就是在第一次创建主题时创建的
TopicConfig topicConfig = 
	this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());
if (queueIdInt < 0) {    //如果队列ID小于0,则随机选择一个队列存储消息
    queueIdInt = randomQueueId(topicConfig.getWriteQueueNums());
}

//2 查找 requestHeader 来源，找到 DefaultMQProducerImpl#sendKernelImpl(...)
SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
requestHeader.setTopic(msg.getTopic()); //实际主题，默认主题TBW102用于创建这个新主题
requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey()); //默认主题，TBW102
// >>>>>>> 突破点
requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums()); // 主题消息队列个数，实际用于设置新主题读写队列个数，默认是4
requestHeader.setQueueId(mq.getQueueId()); //消息存储目标消息队列ID
requestHeader.setSysFlag(sysFlag);
requestHeader.setBornTimestamp(System.currentTimeMillis());
requestHeader.setFlag(msg.getFlag());
requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
requestHeader.setReconsumeTimes(0);
requestHeader.setUnitMode(this.isUnitMode());
requestHeader.setBatch(msg instanceof MessageBatch);
requestHeader.setBrokerName(brokerName);
```

**再看：这组有顺序的消息如何实现都发给同一个消息队列？**

~~还是分析源码流程，从消息发送流程可以看到在 `DefaultMQProducerImpl$sendDefaultImpl(...)` 中实现了消息存储目标消息队列的选择（**队列ID递增轮询 + MQFaultStrategy 过滤**）；~~

> 队列ID递增轮询，其实也体现了生产者端的负载均衡实现。

~~不过这个不支持自定义拓展 QueueFilter，另外控制顺序消息发到同一消息队列也和 MQFaultStrategy 本意不符。~~

```java
//从 TopicPublishInfo 中选择一个 MessageQueue 实例存储当前消息，内部借助 MQFaultStrategy 实现对对消息队列的选择
//其中 BrokerFilter 的作用是如果lastBrokerName不为null将一个主题的消息优先发送到不同的Broker节点, 也即重试发送时将消息尝试发送到其他Broker节点，只有重试发送时 BrokerFilter 才有效
MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName, resetIndex);

public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName, final boolean resetIndex) {
    // 每个线程中都会创建一个 BrokerFilter，不过只是用于发送失败后重试
    BrokerFilter brokerFilter = threadBrokerFilter.get();
    brokerFilter.setLastBrokerName(lastBrokerName);
    if (this.sendLatencyFaultEnable) { //如果开启延时容错
        if (resetIndex) {
            tpInfo.resetIndex();
        }
        // availableFilter 优先级高于 reachableFilter
        MessageQueue mq = tpInfo.selectOneMessageQueue(availableFilter, brokerFilter);
        if (mq != null) {
            return mq;
        }

        mq = tpInfo.selectOneMessageQueue(reachableFilter, brokerFilter);
        if (mq != null) {
            return mq;
        }
        // 兜底，不使用过滤器过滤
        return tpInfo.selectOneMessageQueue();
    }
    // 如果没有开启延迟容错
    MessageQueue mq = tpInfo.selectOneMessageQueue(brokerFilter);
    if (mq != null) {
        return mq;
    }
    // 兜底，不使用过滤器过滤
    return tpInfo.selectOneMessageQueue();
}
```

另外发现生产者发送消息接口（MQProducer）中已经提供了一组方法可以指定 MessageQueueSelector（即选择存储消息的MessageQueue），可以自定义实现，将有顺序的消息发送到同一消息队列。

```java
// MQProducer
public SendResult send(Message msg, MessageQueueSelector selector, Object arg)
public SendResult send(Message msg, MessageQueueSelector selector, Object arg, long timeout)
public void send(Message msg, MessageQueueSelector selector, Object arg, SendCallback sendCallback)
public void send(Message msg, MessageQueueSelector selector, Object arg, SendCallback sendCallback, long timeout)

// 官方已经提供了Demo: rocketmq-example/.../ordermessage/Producer.java
SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        Integer id = (Integer) arg;
        int index = id % mqs.size();
        return mqs.get(index);
    }
}, orderId);

// 内部原理也很简单，就是先选择消息队列，然后将消息直接发送到这个消息队列
// 先选择消息队列
MessageQueue mq = this.defaultMQProducerImpl
    .invokeMessageQueueSelector(msg, selector, arg, this.getSendMsgTimeout());
mq = queueWithNamespace(mq);
// 将消息直接发送到这个消息队列
if (this.getAutoBatch() && !(msg instanceof MessageBatch)) {
    return sendByAccumulator(msg, mq, null);
} else {
    return sendDirect(msg, mq, null);
}
```

#### 消息顺序拉取到消费者端后如何保证顺序消费？

消息消费回调接口 `MessageListener`，有两个子接口 `MessageListenerOrderly`、`MessageListenerConcurrently`，分别由 `ConsumeMessageOrderlyService` 和 `ConsumeMessageConcurrentlyService`处理。

保证顺序消费需要使用 `MessageListenerOrderly`。

看 `ConsumeMessageOrderlyService` 和 `ConsumeMessageConcurrentlyService`执行原理，可以看到两者实现类似，都使用了线程池消费拉取到的消息数据，区别是  `ConsumeMessageOrderlyService` 线程任务消费数据前会**先加消息队列同步锁**，从而实现同步消费。

```java
final Object objLock = messageQueueLock.fetchLockObject(this.messageQueue);
// 通过消息队列同步锁实现对这个消息队列的消费总是顺序消费
synchronized (objLock) {
	...消费消息...
    //消费可能失败，ConsumeMessageConcurrentlyService 会直接将消息重新发给Broker,
    //但是 ConsumeMessageOrderlyService 不能这么做，因为可能会破坏消息的顺序性，ConsumeMessageOrderlyService 是直接将这批消息重新放到处理队列 ProcessQueue 重试消费
    continueConsume = ConsumeMessageOrderlyService.this.processConsumeResult(msgs, status, context, this);
}

//ConsumeMessageOrderlyService#processConsumeResult(...)
public boolean processConsumeResult(
    final List<MessageExt> msgs,
    final ConsumeOrderlyStatus status,
    final ConsumeOrderlyContext context,
    final ConsumeRequest consumeRequest
) {
    boolean continueConsume = true;
    long commitOffset = -1L;
    if (context.isAutoCommit()) {
        switch (status) {
            case COMMIT:
            case ROLLBACK:
                log.warn(...);
            case SUCCESS:
                commitOffset = consumeRequest.getProcessQueue().commit();
                this.getConsumerStatsManager().incConsumeOKTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
                break;
            case SUSPEND_CURRENT_QUEUE_A_MOMENT:
                this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
                if (checkReconsumeTimes(msgs)) {
                    consumeRequest.getProcessQueue().makeMessageToConsumeAgain(msgs);
                    this.submitConsumeRequestLater(
                        consumeRequest.getProcessQueue(),
                        consumeRequest.getMessageQueue(),
                        context.getSuspendCurrentQueueTimeMillis());
                    continueConsume = false;
                } else {
                    commitOffset = consumeRequest.getProcessQueue().commit();
                }
                break;
            default:
                break;
        }
    } else {
        switch (status) {
            case SUCCESS:
                this.getConsumerStatsManager().incConsumeOKTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
                break;
            // 这两个用于事务消息
            case COMMIT:
                commitOffset = consumeRequest.getProcessQueue().commit();
                break;
            case ROLLBACK:
                consumeRequest.getProcessQueue().rollback();
                this.submitConsumeRequestLater(
                    consumeRequest.getProcessQueue(),
                    consumeRequest.getMessageQueue(),
                    context.getSuspendCurrentQueueTimeMillis());
                continueConsume = false;
                break;
            // 消费失败
            case SUSPEND_CURRENT_QUEUE_A_MOMENT:
                this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
                if (checkReconsumeTimes(msgs)) {
                    //将 msgs 重新加入处理队列，直接进行重试，由于外部的消息队列同步锁还没有释放，也不会消费后续的消息
                    //从而保证消息重试消费还是正确的熟顺序
                    consumeRequest.getProcessQueue().makeMessageToConsumeAgain(msgs);
                    this.submitConsumeRequestLater(
                        consumeRequest.getProcessQueue(),
                        consumeRequest.getMessageQueue(),
                        context.getSuspendCurrentQueueTimeMillis());
                    continueConsume = false;
                }
                break;
            default:
                break;
        }
    }

    if (commitOffset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {
        this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), commitOffset, false);
    }

    return continueConsume;
}
```

### 消息消费失败重试队列与死信队列机制

以 PushConsumer ConsumeMessageConcurrentlyService 为例，消息消费失败（比如返回 RECOSUME_LATER）在处理消费结果的方法 `ConsumeMessageConcurrentlyService#processConsumeResult(...)` 中会将消息发送回 Broker，通过请求码 `RequestCode
.CONSUMER_SEND_MSG_BACK`发送；

Broker 端在 SendMessageProcessor 中处理此请求码，如果消息重试次数没有超过最大重试次数（5.2版本默认是3次），就将消息以**延迟消息**的形式发送到当前消费者分组的**重试主题** `%RETRY%consumerGroup` ，如果重试次数超过最大重试次数，就将消息发送到**死信队列**`%DLQ%consumerGroup`；

消费者启动时会额外订阅当前消费者分组的重试主题，参考 `copySubscription()` 方法。

另外注意死信队列的消息同样是默认保存72消息，如果没有消费也会被删除，需要注意及时处理。

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

